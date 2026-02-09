# Infinigen Usage Guide

Custom configs and usage notes for generating 3D scenes with animals. Use `opengl_gt` pipeline on Linux for 3D bounding box ground truth.

## Quick Start

Generate a scene with animals using our pre-configured setup:

```bash
# Basic animal scene generation
python -m infinigen_examples.generate_nature \
  --seed 0 \
  --task coarse \
  --output_folder outputs/my_animal_scene \
  infinigen_examples/configs_nature/animals_plants.gin

# Generate multiple camera views
python -m infinigen_examples.generate_nature \
  --seed 0 \
  --task render \
  --output_folder outputs/my_animal_scene \
  infinigen_examples/configs_nature/animals_plants.gin
```

## Scene Types Available

Infinigen includes several pre-built scene configurations:

### Nature Scenes
- `desert.gin` - Desert environments with cacti and desert plants
- `plain.gin` - Open grasslands and plains  
- `forest.gin` - Dense forest environments
- `arctic.gin` - Snow and ice environments
- `cliff.gin` - Rocky cliff and mountain scenes
- `cave.gin` - Underground cave systems
- `coastal.gin` - Beach and coastal environments

### Custom Configurations
- `animals_plants.gin` - Scenes with creatures and vegetation (optimized for 16GB RAM)
- `custom_desert.gin` - Custom desert setup with specific parameters

## Pipeline Configurations

### blender_gt.gin
Standard rendering pipeline that generates:
- ✅ RGB images
- ✅ Depth maps  
- ✅ Segmentation masks
- ✅ Surface normals
- ❌ No 3D bounding boxes

```bash
python -m infinigen_examples.generate_nature \
  --seed 0 --task coarse \
  --output_folder outputs/blender_scene \
  infinigen_examples/configs_nature/animals_plants.gin \
  infinigen/datagen/configs/gt_options/blender_gt.gin
```

### opengl_gt.gin  
Advanced pipeline with customgt (Linux + GPU required) that generates:
- ✅ RGB images
- ✅ Depth maps
- ✅ Segmentation masks  
- ✅ Surface normals
- ✅ **3D bounding boxes** (key advantage)

```bash
# Requires Linux with EGL/GPU support
python -m infinigen_examples.generate_nature \
  --seed 0 --task coarse \
  --output_folder outputs/opengl_scene \
  infinigen_examples/configs_nature/animals_plants.gin \
  infinigen/datagen/configs/gt_options/opengl_gt.gin
```

### blender_gt_savemesh.gin
Custom pipeline configuration for mesh extraction workflows.

## Resolution and Performance Tuning

### Key Parameters

```bash
# High quality (slow, high memory)
--gin_param="generate_resolution=(1920, 1080)"
--gin_param="num_samples=512" 

# Balanced (recommended for 16GB systems)
--gin_param="generate_resolution=(1280, 720)"  
--gin_param="num_samples=256"

# Fast preview (quick iterations)
--gin_param="generate_resolution=(640, 480)"
--gin_param="num_samples=64"
```

### Memory Optimization for 16GB Systems

The `animals_plants.gin` config includes these RAM-saving optimizations:

```gin
# Disable memory-heavy features
monocots.MomocotsFactory.factory_seed_map = {}
caves.CavesFactory.factory_seed_map = {} 
caves_chance = 0.0

# Reduce vegetation density
populate_plants.PlantPopulationParams.density_per_km2 = 10000  # reduced from default

# Disable hair rendering for creatures (major RAM saver)
hair = False
```

## Extracting 3D Bounding Boxes

### Official Tool (Recommended)
Use Infinigen's built-in bounding box extraction:

```bash
# Extract bounding boxes for all objects in frame 0
python -m infinigen.tools.ground_truth.bounding_boxes_3d \
  outputs/my_scene \
  0 \
  --query "all"

# Extract bounding boxes for specific object types
python -m infinigen.tools.ground_truth.bounding_boxes_3d \
  outputs/my_scene \
  0 \
  --query "creatures"

python -m infinigen.tools.ground_truth.bounding_boxes_3d \
  outputs/my_scene \
  0 \
  --query "plants"
```

### Requirements
- Only works with scenes generated using `opengl_gt.gin` pipeline
- Requires the customgt rendering backend
- Linux with EGL/GPU support recommended

## Available Creature Factories

Infinigen includes several creature types you can populate scenes with:

### Mammals
- `CarnivoreFactory` - Predators like wolves, big cats
- `HerbivoreFactory` - Prey animals like deer, rabbits

### Birds  
- `BirdFactory` - Ground birds and general avian species
- `FlyingBirdFactory` - Birds with flight animations

### Other Creatures
- `SnakeFactory` - Various snake species
- `BeetleFactory` - Insect creatures
- `CrabFactory` - Crustaceans

### Enabling Creatures in Custom Configs

```gin
# Enable specific creature types
creatures_chance = 1.0
CarnivoreFactory.factory_seed_map = {0: 23}
HerbivoreFactory.factory_seed_map = {0: 24, 1: 25}
BirdFactory.factory_seed_map = {0: 26}
```

## Key Configuration Overrides

### For macOS Users
```bash
--gin_param="caves_chance=0.0"  # Avoid cave generation compatibility issues  
--gin_param="hair=False"        # Disable hair rendering (reduces crashes)
```

### Common Overrides
```bash
# Increase creature population
--gin_param="creatures_chance=1.0"

# Adjust lighting
--gin_param="lighting_energy=2.0"

# Control terrain size  
--gin_param="terrain_scale=10.0"

# Set specific weather
--gin_param="weather='cloudy'"
```

## Platform Notes

- **Linux (recommended)**: Use `opengl_gt.gin` for full 3D bounding box support via customgt + EGL.
- **macOS**: customgt has `GL_LINES_ADJACENCY` compatibility issues with the Metal translation layer. Use `blender_gt.gin` on macOS (no 3D bbox). For 3D bbox, run on Linux.

## Example Commands

### Generate a Desert Scene with Animals
```bash
python -m infinigen_examples.generate_nature \
  --seed 42 \
  --task coarse \
  --output_folder outputs/desert_animals \
  infinigen_examples/configs_nature/custom_desert.gin \
  infinigen_examples/configs_nature/animals_plants.gin \
  --gin_param="generate_resolution=(1280, 720)" \
  --gin_param="creatures_chance=1.0" \
  --gin_param="hair=False"
```

### Generate with 3D Bounding Boxes (Linux Only)
```bash
# Step 1: Generate scene structure
python -m infinigen_examples.generate_nature \
  --seed 100 \
  --task coarse \
  --output_folder outputs/bbox_scene \
  infinigen_examples/configs_nature/animals_plants.gin \
  infinigen/datagen/configs/gt_options/opengl_gt.gin

# Step 2: Render with ground truth
python -m infinigen_examples.generate_nature \
  --seed 100 \
  --task render \
  --output_folder outputs/bbox_scene \
  infinigen_examples/configs_nature/animals_plants.gin \
  infinigen/datagen/configs/gt_options/opengl_gt.gin

# Step 3: Extract bounding boxes
python -m infinigen.tools.ground_truth.bounding_boxes_3d \
  outputs/bbox_scene \
  0 \
  --query "all"
```

### Quick Preview Generation
```bash
python -m infinigen_examples.generate_nature \
  --seed 1 \
  --task coarse \
  --output_folder outputs/preview \
  infinigen_examples/configs_nature/animals_plants.gin \
  --gin_param="generate_resolution=(640, 480)" \
  --gin_param="num_samples=64" \
  --gin_param="hair=False"
```

## Troubleshooting

### Memory Issues
- Reduce `generate_resolution`
- Lower `num_samples` 
- Set `hair=False`
- Use `animals_plants.gin` (pre-optimized)
- Set `caves_chance=0.0`

### Rendering Crashes
- Try `blender_gt.gin` instead of `opengl_gt.gin`
- On macOS: always use `hair=False` and `caves_chance=0.0`
- Ensure adequate GPU memory for high resolutions

### Missing 3D Bounding Boxes
- Verify you're using `opengl_gt.gin` pipeline
- Confirm scene was generated with customgt enabled
- Check that extraction tool targets correct scene folder

## Output Structure

After generation, your output folder will contain:

```
outputs/my_scene/
├── fine/
│   ├── Image0001.png          # RGB render
│   ├── Depth0001.exr          # Depth map  
│   ├── Segmentation0001.png   # Object masks
│   ├── SurfaceNormals0001.exr # Normal maps
│   └── bbox_3d.npz           # 3D bounding boxes (opengl_gt only)
├── coarse/
│   └── scene.blend           # Blender scene file
└── logs/
    └── generate.log          # Generation logs
```

For more advanced usage, see the official Infinigen documentation and the examples in `infinigen_examples/`.