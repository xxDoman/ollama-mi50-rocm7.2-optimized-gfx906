# Optimized Ollama for AMD Instinct MI50 (gfx906)

This repository contains a high-performance Docker image for **Ollama (v0.20.3)**, specifically optimized for the **AMD Instinct MI50 (32GB HBM2)**.

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

To run the container with full hardware acceleration on your MI50, use the following command 

(you can also use `-e OLLAMA_KV_CACHE_TYPE=q8_0` for higher precision cache):
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
  xxdoman/ollama-mi50:latest

```
---

### 📂 Model Storage Configuration

To ensure Ollama uses your existing models and doesn't download them inside the container, you must map your host directory correctly.

#### Method 1: Docker Compose (Recommended)

Edit the `volumes` section in your `docker-compose.yml` to match your local path:

```yaml
services:
  ollama-mi50:
    ...
    volumes:
      - /home/models/ollama:/models # Path on your host : Path in container
    environment:
      - OLLAMA_MODELS=/models # Tells Ollama where to find the files

```

#### Method 2: Docker Run (Manual CLI)

If you are running the command manually, you must use the `-v` flag for the mapping and `-e` for the environment variable:

```bash
docker run -d --name ollama-mi50 \
  --device=/dev/kfd \
  --device-cgroup-rule='c 226:* rmw' \
  -v /dev/dri:/dev/dri \
  -v /home/models/ollama:/models \
  -e OLLAMA_MODELS=/models \
  -e HSA_OVERRIDE_GFX_VERSION=9.0.6 \
  ...

```
## 📥 Importing custom GGUF models
To use a custom `.gguf` model downloaded directly from HuggingFace, create a `Modelfile`:

1. Create a text file named `Modelfile`:

```dockerfile
FROM /models/your-model-file.gguf
```

2. Import it into Ollama:

```bash
docker exec -it ollama-mi50 ollama create my-custom-model -f /models/Modelfile
```
### ⚠️ Important Note

* **Path Alignment**: The host path (e.g., `/home/models/ollama`) must exist and contain your models before starting the container.
* **Environment Sync**: You **must** set `-e OLLAMA_MODELS=/models` so the application knows to look in the mounted directory instead of the default location.

### Explain  Importing .gguf models 

Place models in the volume directory (e.g., `/home/models/ollama`) on the host (e.g `Qwen3.5-27B.Q4_K_M.gguf` - [link](https://huggingface.co/Jackrong/Qwen3.5-27B-Claude-4.6-Opus-Reasoning-Distilled-GGUF?show_file_info=Qwen3.5-27B.Q4_K_M.gguf)

Create new directory `modelfiles` and file `.Modelfile` (e.g `/home/models/ollama/modelfiles/Qwen35.Modelfile`)

Set contents with path to .gguf as in the container mounted path

```yaml
FROM /models/Qwen3.5-27B.Q4_K_M.gguf
TEMPLATE """{{ .Prompt }}"""
PARAMETER temperature 0.7
```

Enter docker container and create ollama `qwen35` model from Modelfile

```bash
docker run -it <hash> /bin/bash
#check files exist
ls /models && ls /models/modelfiles
ollama create qwen35 -f /models/modelfiles/Qwen35.Modelfile
```

Now your .gguf model will appear as the `qwen35` in the list in connected OpenWebUI or other interface


### Critical Environment Variables:

* **`HSA_OVERRIDE_GFX_VERSION=9.0.6`**: Mandatory for MI50 (gfx906) to be recognized by the ROCm stack.
* **`OLLAMA_KV_CACHE_TYPE=q4_0`**: Reduces VRAM footprint for long contexts.
* **`OLLAMA_FLASH_ATTENTION=1`**: Enables optimized attention kernels for significant speedup.

## 🌍 Hardware Support & Dockerfile Overview

* **Broad Architecture Support:** While explicitly tuned for **AMD Instinct MI50 (gfx906)**, the integrated `rocBLAS` library includes pre-compiled kernels for multiple GPUs. It features out-of-the-box support for:
  * **Instinct Series:** MI100 (`gfx908`), MI200/250 (`gfx90a`), MI300 (`gfx942`)
  * **Radeon RX 6000 (RDNA 2):** e.g., `gfx1030`
  * **Radeon RX 7000 (RDNA 3):** e.g., `gfx1100`, `gfx1101`, `gfx1150`
  * **Next-Gen (RDNA 4):** `gfx1200`, `gfx1201`
* **Pre-compiled for Vega 20:** Includes specialized `LLAMA_HIP` libraries targeting the `gfx906` instruction sets.
* **Optimized for 32GB HBM2:** Memory reporting and buffer management leverage the MI50’s specific memory layout.

⚠️ **Disclaimer:** Support for architectures other than `gfx906` is provided "as-is" based on library availability and has not been rigorously tested by the author.

### License
This project is licensed under the MIT License.
