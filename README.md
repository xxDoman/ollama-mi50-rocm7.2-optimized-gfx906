# Optimized Ollama for AMD Instinct MI50 (gfx906)

This repository contains a high-performance Docker image for **Ollama (v0.17.7)**, specifically optimized for the **AMD Instinct MI50 (32GB HBM2)**.

The build utilizes **ROCm 7.2** to fix critical issues found in standard deployments, such as text corruption ("garbage output") in recurrent models like **Qwen 3.5**.

## 🌟 Key Improvements

* **Text Stability**: Fixed the recurrent layer (SSM) bugs that caused corrupted output.
* **Memory Management**: Optimized to keep models entirely within the 32GB HBM2 VRAM, avoiding slow system RAM spillover.
* **Advanced Features**: Flash Attention enabled and KV Cache set to `q4_0` for maximum speed.

## 🚀 Performance (MI50 32GB)

Tested on **Qwen 3.5 35B (Q6_K)**:

* **Prompt Eval**: ~541 tokens/s.
* **Generation**: ~31.5 tokens/s.

## 🛠️ Usage

To run the container with full hardware acceleration on your MI50, use the following command:

```bash
docker rm -f ollama-mi50
docker run -d --name ollama-mi50 \
  --device=/dev/kfd \
  --device-cgroup-rule='c 226:* rmw' \
  --network ai-network \
  -v /dev/dri:/dev/dri \
  -v ollama_models:/models \
  -p 11434:11434 \
  -e OLLAMA_MODELS=/models \
  -e OLLAMA_NUM_PARALLEL=1 \
  -e OLLAMA_KV_CACHE_TYPE=q4_0 \
  -e OLLAMA_FLASH_ATTENTION=1 \
  -e HSA_OVERRIDE_GFX_VERSION=9.0.6 \
  -e LD_LIBRARY_PATH="/usr/lib/ollama/rocm" \
  xxdoman/ollama-mi50:v0.17.7

```

### Critical Environment Variables:

* **`HSA_OVERRIDE_GFX_VERSION=9.0.6`**: Mandatory for MI50 (gfx906) to be recognized by the ROCm stack.
* **`OLLAMA_KV_CACHE_TYPE=q4_0`**: Reduces VRAM footprint for long contexts.
* **`OLLAMA_FLASH_ATTENTION=1`**: Enables optimized attention kernels for significant speedup.

### 🐳 Dockerfile Overview

* **Exclusive Architecture**: This image is strictly built and compiled for **AMD Instinct MI50 (gfx906)**.
* **No Multi-GPU Bloat**: To maximize performance and reduce image size, support for other architectures (like gfx10xx or gfx11xx) has been intentionally excluded.
* **Pre-compiled for Vega 20**: Includes specialized `LLAMA_HIP` libraries and kernels specifically tuned for **gfx906** instruction sets.
* **Optimized for 32GB HBM2**: Memory reporting and buffer management are hard-coded to leverage the MI50’s specific memory layout.

> **Warning**: If you are trying to run this on a different GPU architecture, it likely won't work or will perform poorly. Use at your own risk.

---
