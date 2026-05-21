# Localization (i18n / l10n) Spec

> Translations via gettext `.po` as the canonical source format. Compact binary at runtime. Hot-reload in the editor. Gap: [`gaps.md` § 2.2.B + § 3 #15](../gaps.md). Reference patterns: [`gap-references.md` § 2.2.B](../gap-references.md).

## Scope

A localization system that works from Phase 2 onward (so retrofitting is unnecessary), even though v1.0 ships English-only. Translators get the industry-standard `.po` format directly; players get language switching in settings; code uses string keys, never literals.

## Canonical format — gettext `.po`

`.po` is the source-of-truth. Checked into git. Edited by translators directly in their tool of choice. No TOML middle layer (rejected — see § Decision rationale).

Source layout:

```text
<project>/assets/locales/
├── messages.pot       # template (English keys + empty strings)
├── en.po              # English (canonical for the dev)
├── de.po              # German
├── es.po              # Spanish
└── ja.po              # Japanese
```

`.po` file shape:

```pofile
# Player-facing main menu
#: ui/menu.zig:34
msgctxt "menu"
msgid "ui.menu.start"
msgstr "Start Game"

# Skill increase notification — {level} substituted at runtime
#: gameplay/skill.zig:120
msgid "gameplay.skill.mining_increased"
msgstr "Your mining skill increased to {level}."

# Plural form for item collection
msgid "ui.inventory.items_collected"
msgid_plural "ui.inventory.items_collected_plural"
msgstr[0] "{count} item collected"
msgstr[1] "{count} items collected"
```

## Code idiom

```zig
const t = @import("core").localize.t;

// at use site
const msg = t("ui.menu.start");

// with substitution
const msg = t.fmt("gameplay.skill.mining_increased", .{ .level = 42 });

// pluralized
const msg = t.plural("ui.inventory.items_collected", count);

// with context (disambiguation)
const msg = t.ctx("menu", "ui.menu.close");      // "close menu"
const msg = t.ctx("door", "ui.menu.close");      // "close (door verb)"
```

## Build-time pipeline

1. **Extract** — `xgettext`-equivalent extractor walks Zig source for `t()` / `t.fmt()` / `t.plural()` / `t.ctx()` calls, emits `messages.pot`
2. **Merge** — `msgmerge`-equivalent updates each locale's `.po` against the new template (preserves existing translations, marks new/changed strings as fuzzy)
3. **Compile** — `.po` files → compact binary table (key hash → translated string + plural rules), one per locale, written to `<project>/.import/locales/<locale>.bin`
4. **Runtime** — engine loads the active locale's binary; lookups are O(1) hash table

## Format substitution

Placeholders: `{name}` (named substitution), with format specifiers:

- `{count:int}`
- `{value:float:.2}`
- `{name:string}`

## Plural rules

[CLDR plural rules](https://cldr.unicode.org/index/cldr-spec/plural-rules) embedded at build time. Each locale's `.po` header declares its plural form:

```pofile
"Plural-Forms: nplurals=2; plural=(n != 1);\n"            # English / German
"Plural-Forms: nplurals=3; plural=(n%10==1 && n%100!=11 ? 0 : n%10>=2 && n%10<=4 && (n%100<10 || n%100>=20) ? 1 : 2);\n"  # Russian
```

Code uses the abstract `t.plural(key, count)` API; runtime picks the correct form per locale.

## Font fallback

Per-locale font chain in a separate config file (not part of `.po`):

```toml
# <project>/assets/locales/font_chains.toml
[en]
chain = ["Inter", "NotoSans", "NotoColorEmoji"]

[ja]
chain = ["NotoSansCJK-JP", "Inter", "NotoSans"]

[zh]
chain = ["NotoSansCJK-SC", "Inter", "NotoSans"]
```

UI engine ([`specs/ui.md`](ui.md)) renders text by walking the chain — for each glyph, pick the first font that has it.

## Hot-reload

Editor watches `<project>/assets/locales/*.po`; on change:

1. Recompile that locale's binary cache
2. Re-render all text widgets in active scenes
3. Show changes immediately in playtest

## Translator workflow

`.po` is **the** standard for FOSS / indie game localization. Mature toolchain:

- **Editors**: [Poedit](https://poedit.net/) (free), [Lokalize](https://apps.kde.org/lokalize/) (KDE), Emacs po-mode
- **Platforms**: [Weblate](https://weblate.org/) (self-hostable, FOSS), [Crowdin](https://crowdin.com/), [Transifex](https://www.transifex.com/) — all support `.po` natively
- **Glossary + translation memory** + **fuzzy match** — standard features across all `.po` tools
- **CLI tooling**: `msgfmt`, `msgmerge`, `xgettext` — universal, stable

Dev workflow:

1. Write code with `t("key")` calls
2. Run `zig build extract-strings` → updates `messages.pot`
3. Run `zig build merge-locales` → propagates new/changed strings into every `<locale>.po` (existing translations preserved; new strings marked fuzzy)
4. Hand `.po` files to translators
5. Translators submit updated `.po` files
6. `zig build` compiles `.po` → binary tables; game uses them

## Right-to-left (RTL) — deferred

Arabic + Hebrew need RTL text rendering + UI mirroring. Not in v1.0. Architecture should leave room — UI engine renders text via the font subsystem, which can flip in RTL mode later.

## Decision rationale — why `.po`, not TOML or JSON

This decision was litigated at length. Conclusion:

| Format | Why not chosen |
| --- | --- |
| **TOML (project default)** | Translators don't know it. No mature TOML translation tools. Plural forms + `msgctxt` would need hand-rolled conventions. Loses the ecosystem we'd get free with `.po` |
| **JSON (i18next)** | JS-web pattern, weak FOSS-game translator support. Translators end up exporting to `.po` anyway — backwards workflow |
| **XLIFF** | Enterprise / commercial localization standard. Tools exist but more for vendor-driven workflows. Heavier than needed |
| **`.po`** ✅ chosen | Industry standard for FOSS games + GNOME / KDE / Mozilla / Wikipedia. First-class plurals, `msgctxt`, comments, fuzzy markers. Diff-friendly. Universally tool-supported |

The decision deliberately deviates from the "TOML everywhere for hand-authored content" principle in [`tech-stack.md`](../tech-stack.md). Exceptions are allowed when a better-fit standard exists — `.po` for translations is in the same category as glTF for models or KTX2 for textures.

## Borrowed patterns

[`gap-references.md` § 2.2.B](../gap-references.md):

- Godot `core/string/translation_po.cpp` — gettext `.po` reader (study the parser as the canonical implementation reference)
- Godot `core/string/translation.cpp` — TranslationServer + locale lookup
- Unreal `Internationalization/` (`Culture.h`, `FastDecimalFormat.h`) — culture + number/date formatting + CLDR data

## Open decisions

- Source-extractor implementation — port `xgettext` patterns or write Zig-native? (Zig-native; tighter integration with build system)
- Allowing pseudo-locale (Pig-Latin / accented English) for catching unlocalized strings in dev
- Whether to ship the full CLDR plural-rules database or per-locale subsets

## Milestone

Phase 2 — `t()` API exists and is used in every UI string from day one. Phase 12 — locale switch in settings menu reflects immediately in editor playtest, edit `.po` in Poedit, see change live. Phase 13 — exported game ships with at least one alternate locale (German or Spanish) to validate the pipeline end-to-end.
