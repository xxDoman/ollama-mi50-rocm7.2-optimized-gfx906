# Optimized Ollama for AMD Instinct MI50 (gfx906)

Update: v0.32.14 (Date: 26.08.2026) — MTP / speculative decoding works, new benchmarks
 Update: v0.24.0 (Date: 17.05.2026)
 Update: v0.30.0-rc17 (Date: 17.05.2026) (add -e OLLAMA_LLM_LIBRARY=rocm)
 Update: v0.23.0 (Date: 05.05.2026)

This repository contains a high-performance Docker image for **Ollama (v0.32.14)**, specifically optimized for the **AMD Instinct MI50 (32GB HBM2)**.

The build utilizes **ROCm 7.2** to fix critical issues found in standard deployments, such as text corruption ("garbage output") in recurrent models like **Qwen 3.5**.

## 🌟 Key Improvements

- **Text Stability**: Fixed the recurrent layer (SSM) bugs that caused corrupted output.
- **Memory Management**: Optimized to keep models entirely within the 32GB HBM2 VRAM, avoiding slow system RAM spillover.
- **Advanced Features**: Flash Attention enabled and KV Cache set to `q8_0` for best speed/precision balance.
- **Multi-Token Prediction (MTP)**: Gemma 4 models run with speculative decoding out of the box (see below).

## 🚀 Performance (MI50 32GB)

Tested on a clean GPU (nothing else occupying VRAM), Ollama v0.32.14, KV cache `q8_0`, flash attention on, all layers fully offloaded:

| model | long generation | short answers |
|---|---|---|
| Qwen 3.5 35B (Q6_K) | ~52 tok/s | ~541 tok/s prompt eval |
| gemma4:26b without MTP (Q4_0) | ~53 tok/s | — |
| gemma4:26b official (Q4_K_M, auto MTP) | ~65-68 tok/s | ~95 tok/s |
| bvassie/gemma4:26b-a4b-it-qat-mtp (QAT Q4_0) | ~78 tok/s | ~115 tok/s |

With `sudo rocm-smi --setpoweroverdrive 160` (SCLK 1143 → 1606 MHz): add another +8-9%.

## 🔮 Multi-Token Prediction (MTP) — it just works

Gemma 4 models ship with an MTP draft head included in the official Ollama tags (a separate `image.draft` manifest layer). Ollama ≥0.32.x detects it automatically and launches llama-server with `--spec-type draft-mtp`. No Modelfile needed:

```
ollama pull gemma4:26b
ollama run gemma4:26b --verbose
```

Verify in container logs — you should see:

```
cmd="... --spec-type draft-mtp --spec-draft-n-max 3 ... --spec-draft-model ..."
spec common_specu: adding speculative implementation 'draft-mtp'
```

Measured speedup on gfx906: **+20-28%** (draft acceptance 0.48-0.91 depending on text difficulty).

To attach an MTP head manually to a model that doesn't have one in its manifest, use `DRAFT` (NOT `ADAPTER` — that registers a LoRA and crashes with `Gemma4Assistant requires ctx_other`) with local file paths in the Modelfile. Requires Ollama ≥0.32.x.

### ⚠️ Benchmarking pitfall

If anything else is using VRAM (e.g. a standalone llama.cpp server), Ollama silently offloads part of the layers to CPU — throughput drops ~3x and MTP looks broken. Always check `ollama ps` (PROCESSOR column must be 100%) before comparing numbers.

## ⚡ Pro tip: power limits

Stock VBIOS power cap is usually 225-250W (design TDP is 300W), but you can run the card well below it with no performance loss during LLM inference:

```
sudo rocm-smi --setpoweroverdrive 160    # SMI limit 160W (stock: depends on VBIOS)
```

Optionally, a permanent TDC limit via UPP (`TdcLimitGfx=150A` in pp_table). With these set, the card draws only ~113-155W during decoding while boosting SCLK from 1143 to 1606 MHz — free performance at low power and temperature (junction stays under ~85°C).

## 🛠️ Usage

To run the container with full hardware acceleration on your MI50:

(you can use `-e OLLAMA_KV_CACHE_TYPE=q4_0` for minimum VRAM usage or `q8_0` for higher precision cache):

```
docker rm -f ollama-mi50
docker run -d --name ollama-mi50 \
  --device=/dev/kfd \
  --device-cgroup-rule='c 226:* rmw' \
  --network ai-network \
  -v /dev/dri:/dev/dri \
  -v ollama_models:/models \
  -p 11434:11434 \
  -e OLLAMA_MODELS=/models \
  -e OLLAMA_LLM_LIBRARY=rocm \
  -e OLLAMA_NUM_PARALLEL=1 \
  -e OLLAMA_KV_CACHE_TYPE=q8_0 \
  -e OLLAMA_FLASH_ATTENTION=1 \
  -e HSA_OVERRIDE_GFX_VERSION=9.0.6 \
  -e LD_LIBRARY_PATH="/usr/lib/ollama/rocm" \
  xxdoman/ollama-mi50:latest
```

---

My Portainer (Stack)

```
services:
  ollama-mi50:
    image: xxdoman/ollama-mi50:v0.32.14
    container_name: ollama-mi50
    restart: unless-stopped
    shm_size: 16g
    devices:
      - /dev/kfd:/dev/kfd
      - /dev/dri:/dev/dri
    device_cgroup_rules:
      - 'c 226:* rmw'
    ports:
      - "11430:11434"
    environment:
      - OLLAMA_MODELS=/models
      - OLLAMA_NUM_PARALLEL=1
      - OLLAMA_KV_CACHE_TYPE=q8_0
      - OLLAMA_FLASH_ATTENTION=1
      - OLLAMA_LLM_LIBRARY=rocm
      - HSA_OVERRIDE_GFX_VERSION=9.0.6
      - LD_LIBRARY_PATH=/usr/lib/ollama/rocm
    volumes:
      - /home/models/ollama/:/models
      - /home/ollama_data:/root
      - /home/ai_work:/workspace
    networks:
      - ai-net
networks:
  ai-net:
    external: true
```

### 📂 Model Storage Configuration

To ensure Ollama uses your existing models and doesn't download them inside the container, you must map your host directory correctly.

#### Method 1: Docker Compose (Recommended)

Edit the `volumes` section in your `docker-compose.yml` to match your local path:

```
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

```
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

```
FROM /models/your-model-file.gguf
```

2. Import it into Ollama:

```
docker exec -it ollama-mi50 ollama create my-custom-model -f /models/Modelfile
```

### ⚠️ Important Note

- **Path Alignment**: The host path (e.g., `/home/models/ollama`) must exist and contain your models before starting the container.
- **Environment Sync**: You **must** set `-e OLLAMA_MODELS=/models` so the application knows to look in the mounted directory instead of the default location.

### Explain Importing .gguf models

Place models in the volume directory (e.g., `/home/models/ollama`) on the host (e.g `Qwen3.5-27B.Q4_K_M.gguf` - [link](https://huggingface.co/Jackrong/Qwen3.5-27B-Claude-4.6-Opus-Reasoning-Distilled-GGUF?show_file_info=Qwen3.5-27B.Q4_K_M.gguf)

Create new directory `modelfiles` and file `.Modelfile` (e.g `/home/models/ollama/modelfiles/Qwen35.Modelfile`)

Set contents with path to .gguf as in the container mounted path

```
FROM /models/Qwen3.5-27B.Q4_K_M.gguf
TEMPLATE """{{ .Prompt }}"""
PARAMETER temperature 0.7
```

Enter docker container and create ollama `qwen35` model from Modelfile

```
docker run -it <hash> /bin/bash
#check files exist
ls /models && ls /models/modelfiles
ollama create qwen35 -f /models/modelfiles/Qwen35.Modelfile
```

Now your .gguf model will appear as the `qwen35` in the list in connected OpenWebUI or other interface

### Critical Environment Variables:

- **`HSA_OVERRIDE_GFX_VERSION=9.0.6`**: Mandatory for MI50 (gfx906) to be recognized by the ROCm stack.
- **`OLLAMA_KV_CACHE_TYPE=q8_0`**: Reduces VRAM footprint for long contexts.
- **`OLLAMA_FLASH_ATTENTION=1`**: Enables optimized attention kernels for significant speedup.

## 🌍 Hardware Support & Dockerfile Overview

- **Broad Architecture Support:** While explicitly tuned for **AMD Instinct MI50 (gfx906)**, the integrated `rocBLAS` library includes pre-compiled kernels for multiple GPUs. It features out-of-the-box support for:
  - **Instinct Series:** MI100 (`gfx908`), MI200/250 (`gfx90a`), MI300 (`gfx942`)
  - **Radeon RX 6000 (RDNA 2):** e.g., `gfx1030`
  - **Radeon RX 7000 (RDNA 3):** e.g., `gfx1100`, `gfx1101`, `gfx1150`
  - **Next-Gen (RDNA 4):** `gfx1200`, `gfx1201`
- **Pre-compiled for Vega 20:** Includes specialized `LLAMA_HIP` libraries targeting the `gfx906` instruction sets.
- **Optimized for 32GB HBM2:** Memory reporting and buffer management leverage the MI50's specific memory layout.
⚠️ **Disclaimer:** Support for architectures other than `gfx906` is provided "as-is" based on library availability and has not been rigorously tested by the author.

### License

This project is licensed under the MIT License.
