# Security Policy

## Supported versions

zVoxRealms is in **Phase 0 (Foundation)** — pre-1.0, no production releases yet. There is no stability commitment and no backport policy. The only "supported" version is `main`.

Once a 1.0 stable channel ships on Steam, this policy will be revised with a versioned-support table.

## Reporting a vulnerability

If you find a security issue in zVoxRealms — the engine, an adapter sub-repo, the export pipeline, the modding ABI, the runtime networking layer, or anything that could compromise a player's machine or data — please report it **privately**, not in a public issue.

**Use [GitHub Security Advisories](https://github.com/SETA1609/zigVoxelWorlds/security/advisories)** — GitHub's private vulnerability-disclosure flow. Once the repository is public, the "Report a vulnerability" button is the canonical channel.

Include in the report:

- A clear description of the vulnerability
- Steps to reproduce (minimal repro preferred)
- The affected component (engine binary / adapter / export pipeline / mod ABI / etc.)
- The impact (RCE / data exfiltration / save corruption / DoS / etc.)
- A suggested fix or mitigation, if you have one

Do **not** open public issues for security problems. Do not post in Discussions. Use the Security Advisory channel only.

## What I'll do

- Acknowledge receipt within 7 days (faster if it's exploitable in the wild)
- Investigate and confirm or dispute the report
- Develop a fix, working with you if you wish to be credited
- Coordinate a disclosure timeline (typical: 90 days, less if the bug is being actively exploited)
- Credit you in the release notes / `CHANGELOG.md` unless you prefer anonymity

## What I won't do

- Pay bug bounties — zVoxRealms has no commercial revenue yet
- Disclose your contact details without consent
- Penalize good-faith research

## Scope

In scope:

- The zVoxRealms engine binary
- Any adapter sub-repo published under the same maintainer
- The export pipeline (the part that produces shipped games)
- The mod / script ABI surface (anything an `.so`/`.dll` mod can do to harm a player)
- Save-file deserialization (malicious save files)
- Network protocol parsing (malicious server / client messages, once multiplayer ships)
- The asset import pipeline (malicious source files)

Out of scope:

- Third-party libraries — report to upstream (Jolt, ImGui, Tracy, etc.) directly
- Mods made by third parties — report to the mod author
- Steam infrastructure — report to Valve
- Issues in games _made with_ zVoxRealms by third-party developers — report to the game's author

## Threat model

For context, the threats I'm prioritizing:

1. **Malicious mods executing code on player machines.** The mod system uses native plugins (`.so`/`.dll`) which means full code execution. This is the largest attack surface. The architecture treats mods as semi-trusted; the security model is "you trust mods you install" — but engine bugs that escalate trusted-mod-author intent into ACE on shipped games are in scope.
2. **Malicious save files.** Binary deserialization is a classic vector. The save format is versioned + bounds-checked; bugs there are in scope.
3. **Malicious network messages (post Phase 10).** Authoritative server, but a compromised peer can still feed malformed packets. Parser hardening is in scope.
4. **Malicious asset import.** A player who installs an art pack containing a malicious PNG/glTF/audio file shouldn't be able to compromise the editor or runtime. The asset importers must bounds-check vendor library outputs.
5. **License compliance bypass.** Not a "security" issue per se, but reports of GPL/LGPL code being smuggled into the engine (which would violate the Apache 2.0 strategy) are welcome via the same channel.

## Related documents

- [`LICENSE`](LICENSE) — Apache License 2.0
- [`docs/licensing.md`](docs/licensing.md) — licensing strategy + adapter sub-repos
- [`docs/engine-vs-game.md`](docs/engine-vs-game.md) — what ships where; defines the trust boundary
- [`docs/cpp-style.md`](docs/cpp-style.md) — C ABI hardening rules (`noexcept`, bounds-checking at the boundary)
