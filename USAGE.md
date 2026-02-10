# 3D Detection Data Generation Guide

Generate diverse outdoor scenes with 3D bounding boxes for trees, animals, and large vegetation.

## Setup

```bash
conda activate infinigen
cd ~/projects/infinigen
```

Requires Linux with GPU + EGL for `opengl_gt` (3D bounding boxes).

## Generate a Single Scene

```bash
python -m infinigen.datagen.manage_jobs \
    --output_folder outputs/det3d_plain/0 \
    --num_scenes 1 --specific_seed 0 \
    --configs plain.gin detection_3d.gin \
    --pipeline_configs local_256GB.gin monocular.gin opengl_gt.gin \
    --pipeline_overrides LocalScheduleHandler.use_gpu=True
```

Replace `plain.gin` with any land scene type.

## Generate All Scene Types (Batch)

```bash
#!/bin/bash
# generate_all.sh
# Generates N scenes per scene type with 3D bbox ground truth.

N=10  # scenes per type
GPU=True
OUTPUT_BASE=outputs/det3d

LAND_SCENES=(plain forest desert mountain snowy_mountain cliff canyon cave river coast arctic)

for scene in "${LAND_SCENES[@]}"; do
    echo "=== Generating $scene ($N scenes) ==="
    python -m infinigen.datagen.manage_jobs \
        --output_folder ${OUTPUT_BASE}/${scene} \
        --num_scenes $N \
        --configs ${scene}.gin detection_3d.gin \
        --pipeline_configs local_256GB.gin monocular.gin opengl_gt.gin \
        --pipeline_overrides LocalScheduleHandler.use_gpu=${GPU} \
        --cleanup big_files \
        --warmup_sec 2000
done

echo "=== Done ==="
```

This generates 11 scene types x 10 seeds = 110 scenes.

For underwater scenes (separate camera config):
```bash
WATER_SCENES=(under_water coral_reef kelp_forest)
for scene in "${WATER_SCENES[@]}"; do
    python -m infinigen.datagen.manage_jobs \
        --output_folder ${OUTPUT_BASE}/${scene} \
        --num_scenes $N \
        --configs ${scene}.gin \
        --pipeline_configs local_256GB.gin monocular.gin opengl_gt.gin \
        --pipeline_overrides LocalScheduleHandler.use_gpu=${GPU}
done
```

Underwater scenes use their own default camera settings (no aerial views).

## Multi-Frame per Scene

To render multiple viewpoints per scene (more efficient):
```bash
python -m infinigen.datagen.manage_jobs \
    --output_folder outputs/det3d_plain_multi \
    --num_scenes 5 \
    --configs plain.gin detection_3d.gin noisy_video.gin \
    --pipeline_configs local_256GB.gin stereo_video.gin opengl_gt.gin \
    --pipeline_overrides \
        LocalScheduleHandler.use_gpu=True \
        iterate_scene_tasks.frame_range=[1,50] \
        iterate_scene_tasks.view_block_size=1000 \
        iterate_scene_tasks.cam_block_size=25
```

This renders 50 frames per scene from different camera positions along a trajectory.

## Extracting 3D Bounding Boxes

After generation, extract bboxes from opengl_gt output:

```bash
# List all objects with 3D bbox
python -m infinigen.tools.ground_truth.bounding_boxes_3d outputs/det3d/plain/0 48

# Filter by category
python -m infinigen.tools.ground_truth.bounding_boxes_3d outputs/det3d/plain/0 48 --query herbivore
python -m infinigen.tools.ground_truth.bounding_boxes_3d outputs/det3d/plain/0 48 --query tree
```

Target categories for detection:
- **Animals**: herbivore, carnivore, bird, flyingbird, snake, crab, crustacean
- **Plants**: tree (GenericTreeFactory), bush (BushFactory), cactus (CactusFactory)
- **Skip**: rock, boulder, grass, monocot, leaf, terrain, atmosphere

## detection_3d.gin Summary

**Camera intrinsics (focal length)**:
| Weight | Type | Focal (mm) | FOV |
|--------|------|-----------|-----|
| 50% | Standard | 24-50 | 35-53 deg |
| 20% | Wide | 18-30 | 56-77 deg |
| 20% | Telephoto | 55-120 | 15-30 deg |
| 10% | Ultra-wide | 14-20 | 77-95 deg |

**Camera extrinsics (altitude)**:
| Weight | Type | Height |
|--------|------|--------|
| 60% | Eye-level | 0.5-3m |
| 20% | Low aerial | 5-40m |
| 10% | Ground | 0.15-0.5m |
| 10% | High aerial | 30-80m |

**Other settings**:
- Resolution: 1280x720, 1024 samples
- inview_distance: 100m
- Creatures: ground 100%, flying 70%, hair disabled
- Small vegetation disabled (monocots, ferns, leaves, twigs, flowers)
- Volume scatter disabled (corrupts depth map)

## Scene Type x Animal Registry

| Scene | Ground Creatures | Flying Creatures |
|-------|-----------------|------------------|
| plain | Carnivore, Herbivore, Bird, Snake | Dragonfly, FlyingBird |
| forest | Snake | Dragonfly, FlyingBird |
| desert | Snake | - |
| mountain | - | FlyingBird |
| snowy_mountain | - | FlyingBird |
| cliff | Carnivore, Herbivore, Bird | FlyingBird |
| canyon | Snake | - |
| cave | Carnivore, Herbivore, Bird | Dragonfly |
| river | Carnivore, Herbivore, Snake | Dragonfly, FlyingBird |
| coast | Bird, Crab | FlyingBird |
| arctic | Herbivore, Bird | - |
| under_water | Crustacean | - |

## Hardware Requirements

- **Linux + GPU**: Required for opengl_gt (customgt + EGL)
- **RAM**: 64GB+ recommended for inview_distance=100m. 256GB for large scenes.
- **GPU VRAM**: 8GB+ for Cycles rendering with GPU
- **Disk**: ~2-5GB per scene (before cleanup)

## Troubleshooting

- **OOM during coarse**: Reduce `compose_nature.inview_distance` or disable more vegetation
- **Cave generation hangs**: Add `scene.caves_chance=0.0` override
- **Creature hair crash**: Already disabled in detection_3d.gin (`hair=False`)
- **Blank customgt output on macOS**: Use Linux -- macOS Metal doesn't support GL_LINES_ADJACENCY
