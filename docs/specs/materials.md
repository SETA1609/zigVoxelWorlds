# Materials + Shader Management Spec

> PBR materials, runtime shader registry, Vulkan pipeline-state cache. Gap: [`gaps.md` § 2.1.G](../gaps.md). Reference patterns: [`gap-references.md` § 2.1.G](../gap-references.md).

## Scope

Mesh entities and shipped scenes need surface descriptors (material) and the renderer needs a runtime pipeline cache (shader management). Voxels have a simpler implicit material per voxel-ID; meshes need the full PBR set.

## Material data model

Author-time (TOML):

```toml
[material.iron_sword]
albedo_tex = "textures/iron_albedo.png"        # GUID resolved
normal_tex = "textures/iron_normal.png"
metallic = 0.95
roughness = 0.40
emissive = [0.0, 0.0, 0.0]
emissive_strength = 0.0
ao_tex = "textures/iron_ao.png"
shader = "pbr_opaque"                          # references a shader in shader registry
```

Runtime (Zig):

- `MaterialHandle` (opaque, u64)
- Material loaded into a GPU buffer at import time (texture handles + scalar params)
- Material instances inherit from a base material with parameter overrides (Unreal MaterialInstance pattern)

## Shader registry

- All shaders compiled GLSL → SPIR-V at import time (Phase 4)
- Runtime registry maps `shader_id` → `VkShaderModule` + reflection data (uniform layout, push-constant ranges)
- `PipelineKey { shader_id, render_pass, vertex_layout, blend_mode }` → cached `VkPipeline`
- Cache populated on first draw with that key; LRU evict if memory pressure
- Hot-reload: shader file change → recompile → invalidate matching pipelines

## Pipeline-state object

```zig
const PipelineKey = struct {
    shader_id: ShaderId,
    render_pass: RenderPassId,
    vertex_layout: VertexLayoutId,
    blend_mode: BlendMode,
    cull_mode: CullMode,
    depth_test: bool,
};
```

`RenderServer` requests a pipeline by key; backend looks up or creates.

## Borrowed pattern

Godot's `material_storage.cpp` ([`gap-references.md` § 2.1.G](../gap-references.md)) — material as typed resource, pipeline-state cache keyed by shader + format. Unreal's MaterialInstance hierarchy for parameterized variants.

## Draw-call sorting

- Opaque: front-to-back by depth (early-Z culling)
- Transparent: back-to-front (correct blending)
- Sort key includes `(material_id, mesh_id)` for instancing batches

## Open decisions

- Shading model: pure forward, forward+, or deferred? (Forward+ recommended for voxel workloads on iGPU)
- Material variants — texture array vs bindless?
- Push constants vs uniform buffers for per-draw params

## Milestone

Phase 7.5 (Presentation Layer). Load a glTF mesh with PBR materials; render at target FPS on iGPU. Hot-reload a shader file; pipeline cache invalidates the affected pipeline; next frame uses the updated shader.
