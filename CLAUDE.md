# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**RKLLama** is an Ollama alternative server for running LLM models optimized for Rockchip RK3588(S) and RK3576 platforms with NPU acceleration. It supports:
- Model inference on Rockchip NPU (6 TOPS)
- Ollama API compatibility
- Partial OpenAI API compatibility
- Multimodal models (Qwen2VL, MiniCPMV4, InternVL3.5)
- Image generation (Stable Diffusion RKNN)
- Text-to-Speech and Speech-to-Text
- GGUF models via llama.cpp fork

**Target Hardware:**
- Orange Pi 5 (Pro/Plus/Max) - RK3588S
- Radxa Rock 4D - RK3576
- **OS:** Ubuntu 24.04 arm64 or Armbian

**Python:** 3.9–3.12

## Architecture

```
src/rkllama/
├── server/       Flask REST API server (port 8080 by default)
├── api/          Request processing, model management, workers
│   ├── rkllm.py      NPU inference via RKLLM runtime
│   ├── rknnlite.py   RKNN model inference
│   ├── worker.py     WorkerManager for concurrent model execution
│   ├── model_utils.py Model detection & HuggingFace integration
│   ├── format_utils.py API format conversion (Ollama ↔ OpenAI)
│   └── classes.py    Request/Response dataclasses
├── config/       Configuration management from INI files
├── client/       Python client library
└── lib/          Pre-compiled C++ libraries (librkllmrt.so, librknnrt.so)
```

**Key Components:**
- **WorkerManager** (api/worker.py): Manages concurrent model loading/unloading, memory management, and FIFO request queuing
- **Request Pipeline** (api/process.py): Handles Ollama and OpenAI API requests with format conversion
- **Model Auto-detection**: Distinguishes RKLLM vs GGUF models; auto-selects platform (RK3588 or RK3576)
- **Prompt Cache**: Automatic per-session cache for fast inference in multi-turn conversations
- **Streaming**: Supports both streaming and non-streaming responses

## Development Workflow

**Branch Strategy:**
- `main` — Upstream (NotPunchnox/rkllama)
- `minhas-customizacoes` — Your customization branch
- Set up as Fork + Upstream to pull improvements without losing changes

```bash
git fetch upstream
git rebase upstream/main minhas-customizacoes
```

## Installation & Running

**Install from source:**
```bash
pip install -e .  # Installs with editable mode
```

**Run the server:**
```bash
rkllama_server --models ./models --llamacpp ./rk-llama.cpp/build/bin
```

**Server options:**
```
--debug              Enable debug logging
--models <path>      Path to models directory
--llamacpp <path>    Path to llama.cpp binary
--port <port>        Server port (default 8080)
```

**Check server health:**
```bash
curl http://localhost:8080/api/version
curl http://localhost:8080/api/tags     # List loaded models
```

## Docker Build & Deployment

**Build for ARM64 (Orange Pi):**
```bash
docker build -t rkllama:latest .
docker run -p 8080:8080 --privileged -v $(pwd)/models:/opt/rkllama/models rkllama:latest
```

**Pipeline (GitHub Actions):**
- Trigger: Push to `minhas-customizacoes` with version change in `pyproject.toml`
- Flow: Check version → Build Docker (ARM64 via QEMU) → Create release & tag → Push to Docker Hub
- Only runs if `version` in `pyproject.toml` differs from last git tag

**Manual release:**
1. Update `version = "1.0.X"` in `pyproject.toml`
2. Push to `minhas-customizacoes`
3. Pipeline auto-creates tag, release, and pushes to Docker Hub

## Configuration

**Config file:** `config/rkllama.ini`

Key sections:
- `[server]` — Port, host, debug mode
- `[model]` — Default parameters (temperature, context length, penalties)
- `[prompt_cache]` — Cache TTL (default 7 days), auto-cleanup

Models are dynamically loaded on-demand and auto-unloaded after inactivity (default 30 min).

## Common Tasks

### Adding an API endpoint
Edit `src/rkllama/server/server.py`:
- Add route: `@app.route('/api/endpoint', methods=['POST'])`
- Request validation in `api/classes.py`
- Format conversions in `api/format_utils.py`

### Debugging model issues
Enable debug mode:
```bash
rkllama_server --debug
```
Logs go to `logs/rkllama_server.log`

Model loading flow (troubleshoot):
1. `api/model_utils.py` → detect type (RKLLM/GGUF)
2. `api/rkllm.py` (RKLLM) or `api/rknnlite.py` (GGUF) → load runtime
3. `api/worker.py` → queue & execute inference

### Testing models locally
```bash
from rkllama.client import Client

client = Client(base_url="http://localhost:8080")
response = client.generate(
    model="qwen2.5:3b",
    prompt="Hello",
    stream=False
)
print(response.response)
```

## Key Dependencies

- **Flask 2.3.2** — REST API framework
- **transformers 4.57.6** — Model tokenization & utilities
- **torch 2.8.0** — Deep learning framework (CPU-only on build)
- **pillow, opencv** — Image processing
- **rknn-toolkit-lite2** — Local RKNN inference (ARM64 wheels only)
- **huggingface_hub** — Model downloading

Platform-specific wheels (rknn-toolkit-lite2, etc.) are in `src/rkllama/lib/` and matched to Python version in `pyproject.toml`.

## Release Checklist

Before bumping version:
1. Update `version = "X.Y.Z"` in `pyproject.toml`
2. Test changes locally on Orange Pi or emulated ARM64
3. Push to `minhas-customizacoes` — pipeline handles the rest
4. Monitor: https://github.com/abel-cabral/rkllama/actions

## Important Notes

- **ARM64-only:** Docker image builds for ARM64 (RK3588/RK3576) via QEMU emulation. Not for x86_64.
- **NPU runtime:** Requires actual RK3588/RK3576 hardware or emulation to fully test inference.
- **Models directory:** Git-ignored (`models/**/*`). Add RKLLM/GGUF models manually.
- **Prompt cache:** Auto-saved per model/session for fast multi-turn inference; auto-cleanup after 7 days.
- **Streaming:** Both Ollama and OpenAI APIs support streaming; format conversion happens automatically.
