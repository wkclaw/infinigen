# Infinigen Usage Guide (wkclaw fork)

This fork includes macOS compatibility fixes for `customgt` and custom configs for generating scenes with animals on memory-constrained systems.

## Quick Start

```bash
conda activate infinigen

# Desert scene with animals (16GB RAM safe)
python -m infinigen.datagen.manage_jobs \
    --output_folder outputs/my_scene --num_scenes 1 --specific_seed 0 \
    --configs desert.gin animals_plants.gin \
    --pipeline_configs local_16GB.gin monocular.gin opengl_gt.gin \
    --pipeline_overrides LocalScheduleHandler.use_gpu=False
```

## Ground Truth Pipelines

### blender_gt (CPU, no GPU required)
Generates: RGB, Depth, SurfaceNormal, ObjectSegmentation, InstanceSegmentation, Flow.

Does NOT generate 3D bounding boxes.

```bash
--pipeline_configs local_16GB.gin monocular.gin blender_gt.gin
```

### opengl_gt (requires GPU / Linux EGL)
Generates everything from blender_gt PLUS:
- 3D bounding boxes (`min`, `max`, `model_matrices` in Objects JSON)
- Occlusion boundaries
- Tag segmentation
- 3D flow

```bash
--pipeline_configs local_16GB.gin monocular.gin opengl_gt.gin
```

**Note:** `opengl_gt` requires `customgt` binary. On Linux with EGL, it works out of the box. On macOS, see [macOS Fixes](#macos-customgt-fixes) below.

### Extracting 3D Bounding Boxes (opengl_gt only)

```bash
# List available object names
python -m infinigen.tools.ground_truth.bounding_boxes_3d outputs/my_scene/0 48

# Draw 3D boxes for specific objects
python -m infinigen.tools.ground_truth.bounding_boxes_3d outputs/my_scene/0 48 --query rock
python -m infinigen.tools.ground_truth.bounding_boxes_3d outputs/my_scene/0 48 --query herbivore
```

The Objects JSON (`frames/Objects/camera_0/Objects_*.json`) contains per-object:
- `min` / `max`: object-space bounding box corners
- `model_matrices`: 4x4 transformation matrices for each instance
- `instance_ids`: unique IDs per instance

## Scene Types

Available in `infinigen_examples/configs_nature/scene_types/`:
- `desert.gin` - sand, cacti, snakes
- `plain.gin` - grassland, herbivores, carnivores, birds
- `forest.gin` - dense trees, varied fauna
- `mountain.gin`, `snowy_mountain.gin`, `arctic.gin`
- `coast.gin`, `cliff.gin`, `cave.gin`, `river.gin`

## Creature Factories

| Factory | Type | Notes |
|---------|------|-------|
| `HerbivoreFactory` | Ground | Deer-like animals. Set `hair=False` to avoid crash. |
| `CarnivoreFactory` | Ground | Predators. Set `hair=False` to avoid crash. |
| `BirdFactory` | Ground | Ground birds |
| `FlyingBirdFactory` | Flying | Birds in flight |
| `SnakeFactory` | Ground | Snakes |
| `BeetleFactory` | Ground | Insects |
| `CrabFactory` | Ground | Crustaceans |

## Custom Configs (this fork)

### animals_plants.gin
Scene config with creatures enabled, optimized for 16GB RAM:
- Creatures: HerbivoreFactory, CarnivoreFactory, BirdFactory, FlyingBirdFactory, SnakeFactory
- Hair disabled (avoids `NotImplementedError: Dynamic hair`)
- Monocots, ferns, ground leaves disabled (RAM savings)
- Caves disabled (hangs on macOS)
- Resolution: 1280x720, 1024 samples

### custom_desert.gin
Desert scene with reduced scatter and moderate quality.

### blender_gt_savemesh.gin
Pipeline config combining blender_gt + savemesh step.

## Performance Tuning

### Resolution and Samples
```gin
execute_tasks.generate_resolution = (1280, 720)   # default: (1280, 720)
configure_render_cycles.num_samples = 1024         # default: 8192
```

### Memory (16GB systems)
```gin
# Reduce view distance
compose_nature.inview_distance = 25
placement.populate_all.dist_cull = 25

# Disable heavy vegetation
compose_nature.monocots_chance = 0.0
compose_nature.ferns_chance = 0.0
compose_nature.ground_leaves_chance = 0.0

# Disable caves (can hang)
scene.caves_chance = 0.0

# Lower mesh detail
OpaqueSphericalMesher.pixels_per_cube = 3
target_face_size.global_multiplier = 1.5
```

### Typical Timings (16GB Mac, CPU only, 1280x720)
- Coarse: ~2 min
- Populate: ~5 min
- Render (1 frame, 1024 samples): ~5-10 min
- blender_gt: ~2 min
- **Total per frame: ~15-20 min**

## macOS customgt Fixes

This fork includes fixes for running `customgt` on macOS (Metal/OpenGL translation layer):

1. **SYSTEM_NUM branch fix** (`main.cpp` line 264): Changed `#elif SYSTEM_NUM == 0` to `#elif SYSTEM_NUM == 1` to enable GLFW code path on macOS.

2. **GLSL version downgrade**: All 9 shader files (`glsl/*.vert/*.frag/*.geom`) changed from `#version 440 core` to `#version 410 core` (macOS max OpenGL 4.1).

3. **Bool in shader interface blocks**: Changed `bool has_flow` to `float has_flow` in shader varying blocks (GLSL 4.10 limitation).

4. **Non-fatal GL errors** (`src/utils.cpp`): `glCheckError` changed from throw to warning (macOS GL errors are often non-fatal).

### Known Issue
`GL_LINES_ADJACENCY` draw calls return `GL_INVALID_VALUE` on macOS due to Metal translation layer limitations. The customgt rendering output may be blank/incorrect. **For production use, run opengl_gt on Linux with EGL support.**

## Camera Data Format

- Intrinsics `K`: 3x3 pinhole matrix
- Extrinsics `T`: 4x4 camera-to-world matrix
- Coordinate system: +X right, +Y down, +Z forward (CV convention)
- Stored in `frames/camview/camera_0/camview_*.npz`
