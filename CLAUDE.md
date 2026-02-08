# CLAUDE.md

## Project Overview

This repository contains **ComfyUI provisioning scripts** for RunPod cloud GPU containers. Each shell script is a self-contained configuration profile that automatically downloads and installs ComfyUI with specific custom nodes, ML models, and workflows during container initialization.

The scripts are designed to be **sourced** (not executed) by the [Runpod-init](https://github.com/MushroomFleet/Runpod-init) framework's `init.sh` during container startup on the [ai-dock](https://github.com/ai-dock/comfyui) ComfyUI Docker image.

## Repository Structure

```
ai-comfyui/
├── comfyui-generation.sh   # FLUX.1 image gen + Wan 2.2 i2v + GGUF models
├── comfyui-wan.sh          # Wan 2.1 video generation (t2v, i2v, 720p/480p)
├── comfyui-temp-pod.sh     # Temporary pod with FLUX.1 + ControlNet
├── comfyui-training.sh     # Training-focused with ControlNet + ESRGAN
├── default.sh              # Full node set, minimal models (base config)
└── CLAUDE.md               # This file
```

**Language:** 100% Bash shell scripts. No build system, package manager, tests, or linting.

## Script Architecture

Every script follows the same two-part structure:

### Part 1: Configuration Arrays (editable)

Declarative arrays defining what to install. Located at the top of each file:

| Array | Target Path | Purpose |
|---|---|---|
| `APT_PACKAGES` | System | APT packages to install |
| `PIP_PACKAGES` | ComfyUI venv | Python packages via pip |
| `NODES` | `/opt/ComfyUI/custom_nodes/` | ComfyUI custom node repos (GitHub URLs) |
| `WORKFLOWS` | `/opt/ComfyUI/user/default/workflows/` | Workflow repos (GitHub URLs) |
| `CHECKPOINT_MODELS` | `$WORKSPACE/storage/.../models/ckpt` | Stable Diffusion checkpoints |
| `UNET_MODELS` | `$WORKSPACE/storage/.../models/unet` | UNet/diffusion models |
| `CLIP_MODELS` | `$WORKSPACE/storage/.../models/clip` | Text encoder models |
| `LORA_MODELS` | `$WORKSPACE/storage/.../models/lora` | LoRA fine-tuning models |
| `VAE_MODELS` | `$WORKSPACE/storage/.../models/vae` | VAE models |
| `ESRGAN_MODELS` | `$WORKSPACE/storage/.../models/esrgan` | Upscaling models |
| `CONTROLNET_MODELS` | `$WORKSPACE/storage/.../models/controlnet` | ControlNet models |

Some scripts also define `CLIPVISION_MODELS`, `TEXT_MODELS`, `DIFFUSION_MODELS`, and `PIP_UPGRADE_PACKAGES`.

### Part 2: Provisioning Functions (do not edit casually)

Below the `### DO NOT EDIT BELOW HERE ###` marker. Core functions shared across scripts:

- `provisioning_start()` - Main entry point; detects environment, sources ai-dock, orchestrates everything
- `provisioning_get_nodes()` - Clones custom nodes with `--recursive`, installs their `requirements.txt`
- `provisioning_get_models()` - Downloads models via `wget` to target directories
- `provisioning_download()` - Handles auth tokens for HuggingFace/CivitAI downloads
- `provisioning_has_valid_hf_token()` / `provisioning_has_valid_civitai_token()` - Token validation via API
- `pip_install()` - Wrapper that routes through venv pip or micromamba depending on environment

## Provisioning Flow

```
Container start
  └─> init.sh sources the chosen .sh script
        └─> provisioning_start()
              ├─> Detect environment (MAMBA_BASE vs venv)
              ├─> Source ai-dock environment
              ├─> Check HF_TOKEN → conditionally add gated models
              ├─> Install APT packages
              ├─> Clone/update custom nodes + install requirements
              ├─> Install pip packages
              ├─> Download all model categories to storage paths
              ├─> Clone workflow repositories
              └─> Print completion message
```

## Key Differences Between Script Variants

| Script | Focus | Nodes | Key Models | HF Token Behavior |
|---|---|---|---|---|
| `comfyui-generation.sh` | Image gen + video | 17 active | FLUX.1 GGUF, Chroma1-HD, Wan 2.2 i2v | Adds GGUF/Chroma/Wan 2.2 models; falls back to schnell |
| `comfyui-wan.sh` | Video generation | 13 active | Wan 2.1 t2v/i2v 14B fp16 | No HF token branching; always downloads Wan models |
| `comfyui-temp-pod.sh` | Temporary usage | 5 active | FLUX.1 + ControlNet | Adds FLUX.1-dev; falls back to schnell |
| `comfyui-training.sh` | Training workflows | 10 active | FLUX.1 + ControlNet | Adds FLUX.1-dev; falls back to schnell |
| `default.sh` | Base/full nodes | 19 active | FLUX.1 + ControlNet | Models mostly commented out |

## Environment Variables

| Variable | Purpose |
|---|---|
| `HF_TOKEN` | HuggingFace API token for gated model access |
| `CIVITAI_TOKEN` | CivitAI API token for model downloads |
| `AUTO_UPDATE` | Set to `"false"` to skip updating existing nodes |
| `WORKSPACE` | Base workspace path (set by ai-dock) |
| `DISK_GB_ALLOCATED` / `DISK_GB_REQUIRED` | Disk space checking (set by ai-dock) |
| `MAMBA_BASE` | Auto-set when `/opt/environments/python` doesn't exist |

## Conventions for Editing

### Adding a new custom node
Add the GitHub URL to the `NODES` array. The provisioning system will `git clone --recursive` it and install its `requirements.txt` automatically.

### Adding a new model
Add the HuggingFace/CivitAI URL to the appropriate model array (`CHECKPOINT_MODELS`, `UNET_MODELS`, etc.). For gated models requiring authentication, add them inside the `provisioning_has_valid_hf_token` conditional block in `provisioning_start()`.

### Disabling an entry
Comment it out with `#` prefix. Do not delete entries that might be re-enabled later.

### Creating a new variant
Copy an existing script and modify the configuration arrays at the top. The provisioning functions below the `### DO NOT EDIT BELOW HERE ###` marker should be kept as-is unless adding new model categories (which also requires adding a new `provisioning_get_models` call in `provisioning_start()`).

### Indentation
The codebase uses mixed indentation (tabs and spaces). When editing, match the surrounding style within each file.

### Model URL format
- HuggingFace: `https://huggingface.co/<org>/<repo>/resolve/main/<filename>`
- CivitAI: Direct download URLs with CivitAI token auth
- Note: `comfyui-wan.sh` uses some `/blob/` URLs instead of `/resolve/` — this is inconsistent with the other scripts.

## What NOT to Change

- Do not modify functions below `### DO NOT EDIT BELOW HERE ###` without understanding the full provisioning pipeline
- Do not remove the `provisioning_start` call at the end of each script — it is the entry point
- Do not change the model storage path structure (`$WORKSPACE/storage/stable_diffusion/models/...`) as it must align with ai-dock's storage mapping
- The `comfyui-wan.sh` script adds custom storage mappings via `provisioning_add_mapping_line()` for `diffusion_models`, `text_encoders`, and `clip_vision` — these are needed because Wan models use directories not in the default ai-dock mapping
