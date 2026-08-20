---
description: "Global durable memory - environment, conventions and lessons learned"
tags:
  - "global"
  - "memory"
  - "environment"
  - "wsl"
created: "2026-08-19"
updated: "2026-08-19"
---

# MEMORY

## Environment

- **Platform: WSL** — This environment runs on Windows Subsystem for Linux (WSL).
- **Windows drive mount:** The actual Windows drive is at `/mnt/c` (not `C:/` or `/c`). Always translate Windows paths like `C:\Users\...` to `/mnt/c/Users/...` when accessing from WSL.
- When working with files on the Windows filesystem, use `/mnt/c/...` as the base prefix.
- **Original skills (Windows):** `C:\Users\n0ne\.zcode\skills\` == `/mnt/c/Users/n0ne/.zcode/skills/` — `music-caption-rewriter`, `z-music3-song`, `z-image-food-photo` (food), etc. ComfyUI Windows at `/mnt/c/ComfyUI` with `python_embeded`.

## Skills — WSL refactors (2026-08-19)

- **WSL-native skills** live in `~/.pi/agent/skills/` (auto-discovered by pi):
  - `music-caption-rewriter` — copy of Windows templates/references to `~/.pi/agent/skills/music-caption-rewriter/` (no `/mnt/c` dependency). Same workflow/validation, paths now relative.
  - `z-image` — generic image generator, refactored from `z-image-food-photo` (removed recipe/food prompt logic, ingredient lists, FR_EN_STEPS). Same Z-Image-Turbo INT8 workflow (8 steps, res_multistep, 1024x1024, ~4s) but prompt is now open-ended `subject/style/composition/lighting`. Script: `~/.pi/agent/skills/z-image/scripts/generate.py` (uses `COMFY_URL=http://127.0.0.1:8188`, `~/ComfyUI/venv/bin/python`).
  - `z-music3-song` — WSL port of `z-music3-song`. Models same (minimax_music3_dit_int8 + text encoder + dav vae), builder at `~/.pi/agent/skills/z-music3-song/scripts/build_music_workflow.py`. Python now `~/ComfyUI/venv/bin/python`, ComfyUI `~/ComfyUI` (symlink models to `/mnt/c/ComfyUI/models` to reuse 60GB).
- **ComfyUI WSL install:** `~/ComfyUI-WSL-SETUP.sh` — clones `~/ComfyUI`, venv cu130, symlinks models/output/user, smoke test. GPU: RTX 3080 Ti 12GB sm_86 (INT8 speedup, no FP8 cores), CUDA 13.3 / driver 610.74, `nvcc 13.3` present. Launch: `~/ComfyUI/venv/bin/python ~/ComfyUI/main.py --listen 127.0.0.1 --port 8188` (shares localhost with Windows ComfyUI on 8188).




## ComfyUI — Dual install (2026-08-19 evening, validated)

- **Windows (original, primary when up):** `C:\ComfyUI` (`/mnt/c/ComfyUI`) — `run_3080ti.bat` → `http://127.0.0.1:8188` — `win32 0.33.0 / python 3.13.12 / torch 2.13.0+cu130` — models at `/mnt/c/ComfyUI/models` (61GB diffusion, 44GB TE, 6.3GB vae)
- **WSL (fallback, restartable from WSL):** `~/ComfyUI` (`/home/n0ne/ComfyUI`) — venv `~/ComfyUI/venv` (python 3.12.3, torch 2.13.0+cu130) — symlinked `models → /mnt/c/ComfyUI/models`, `output → /mnt/c/ComfyUI/output`, `user → /mnt/c/ComfyUI/user` — **no 60GB re-download**
  - Launch: `bash ~/ComfyUI/run_wsl.sh` (auto picks 8188 if free else 8189) or `bash ~/ComfyUI/run_wsl.sh 8188` / `8189`
  - Stop: `bash ~/ComfyUI/stop_wsl.sh`
  - Logs: `/tmp/comfy_wsl_8188.log` / `/tmp/comfy_wsl_8189.log`
  - Validated 2026-08-19 21:14: both running side-by-side — WIN 8188 + WSL 8189, each generated `z-image` live correctly (`/tmp/zimage_win_8188_final.png` 172KB + `/tmp/zimage_wsl_8189_final.png` 191KB, 512², 8 steps, ~6s). WSL startup ~15s + prestart 15.6s for Manager.
  - Skills auto-discover WSL: `z-image` / `z-music3-song` accept `--comfy-url http://127.0.0.1:8189` for WSL fallback; default `http://127.0.0.1:8188` for Windows primary.

