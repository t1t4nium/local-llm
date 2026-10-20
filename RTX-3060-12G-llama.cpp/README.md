# RTX 3060 12G llama.cpp setup

This is a viable setup for agentic tasks and coding assistance with short context.

Hardware:
- GPU: NVIDIA GeForce RTX 3060 12G VRAM
- CPU: i5-10600K @ 4.10GHz 6 cores * 2 threads
- RAM: 4 * 16G DDR4-3600 @ 2133 MT/s

OS: [Debian 13-amd64](https://www.debian.org/distrib/)

## Model

- Name: unsloth/Qwen3.8-27B-GGUF
- Version: UD-IQ3_XXS
- Filename: Qwen3.8-27B-UD-IQ3_XXS.gguf
- sha256sum: `c0b7c3038681ed2e3040456c1dd45f9858b6c2290bed172c70388a94874f3eee`
- [HF page](https://huggingface.co/unsloth/Qwen3.8-27B-GGUF?show_file_info=Qwen3.8-27B-UD-IQ3_XXS.gguf)

Download:

```bash
# As root
mkdir /models
chmod 755 /models
apt install -y wget
wget https://huggingface.co/unsloth/Qwen3.8-27B-GGUF/resolve/main/Qwen3.8-27B-UD-IQ3_XXS.gguf?download=true -O /models/Qwen3.8-27B-UD-IQ3_XXS.gguf
chmod 644 /models/Qwen3.8-27B-UD-IQ3_XXS.gguf
```

## llama.cpp

- Website: https://llama.app/
- Docs: https://llama.app/docs
- GitHub: https://github.com/ggml-org/llama.cpp
- Build with CUDA: https://github.com/ggml-org/llama.cpp/blob/master/docs/build.md#cuda

### Install

Install required packages:

```bash
apt install -y build-essential cmake git nvidia-cuda-toolkit pkg-config
```

Create dedicated user to llama-server:

```bash
useradd --system --no-create-home --shell /usr/sbin/nologin llama
```

Create directories:

```bash
mkdir -p /opt/llama
mkdir -p /etc/llama/models
chown -R llama:llama /etc/llama
```

Link model:

```bash
ln -s /models/Qwen3.8-27B-UD-IQ3_XXS.gguf /etc/llama/models/qwen3.8-27b-q3.gguf
```

Clone llama.cpp repository:

```bash
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp
```

Checkout the last build:

```bash
git checkout `git tag -l 'b[0-9]*' | sort -V | tail -n 1`
```

Build:

```bash
cmake -B build \
  -DBUILD_SHARED_LIBS=OFF \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_CUDA_ARCHITECTURES=86 \
  -DGGML_CUDA=ON \
  -DGGML_CUDA_GRAPHS=ON \
  -DGGML_NATIVE=ON \
  -DLLAMA_BUILD_SERVER=ON \
  -DLLAMA_BUILD_UI=OFF \
  -DLLAMA_USE_PREBUILT_UI=OFF
cmake --build build -j $((`nproc` - 2))
```

| Parameter | Description | Value |
|---|---|---|
| `BUILD_SHARED_LIBS` | Build shared libraries | `OFF`. Static linking keeps the binaries self-contained. |
| `CMAKE_BUILD_TYPE` | Compiler optimization level | `Release`. Enables -O2/-O3, strips debug symbols. |
| `CMAKE_CUDA_ARCHITECTURES` | Target GPU architecture | `86`. RTX 3060 is compute capability 8.6 (Ampere). Compiles CUDA kernels for the exact chip instead of relying on just-in-time translation. |
| `GGML_CUDA` | GPU acceleration backend | `ON`. Without it, nothing runs on the GPU. |
| `GGML_CUDA_GRAPHS` | CUDA Graph capture/replay | `ON`. Records sequences of CUDA operations and replays them, cutting kernel launch overhead by about 5%. |
| `GGML_NATIVE` | CPU instruction sets | `ON`. Passes -march=native to the compiler (AVX2, FMA, etc.). Speeds up tokenization and whatever the CPU still handles. |
| `LLAMA_BUILD_SERVER` | Build the HTTP server binary | `ON`. Agent talks to the server through its OpenAI-compatible API. |
| `LLAMA_BUILD_UI` | Build the web interface | `OFF`. The server is not driven via a browser. |
| `LLAMA_USE_PREBUILT_UI` | Download prebuilt web UI assets | `OFF`. No web UI is used. |
| `-j` | Parallel build jobs | `$(nproc) - 2`. Leaves two cores free so the machine stays responsive while building. |

Install:

```bash
# As root
cmake --install build --prefix /opt/llama
```

Create llama-server systemd service `/etc/systemd/system/llama-server.service`:

```
[Unit]
Description=llama.cpp server
After=network.target

[Service]
Environment="GGML_CUDA_ENABLE_UNIFIED_MEMORY=1"
User=llama
Group=llama
WorkingDirectory=/etc/llama
ExecStart=/opt/llama/bin/llama-server \
  --host 127.0.0.1 \
  --port 8080 \
  --timeout 600 \
  --parallel 1 \
  --jinja \
  --metrics \
  --sleep-idle-seconds 600 \
  --ctx-checkpoints 64 \
  --checkpoint-min-step 512 \
  -fa auto \
  -ngl 99 \
  -c 65536 \
  -ctk q4_0 \
  -ctv q4_0 \
  -m /etc/llama/models/qwen3.8-27b-q3.gguf
Restart=on-failure
RestartSec=10
LimitMEMLOCK=infinity
NoNewPrivileges=true

[Install]
WantedBy=default.target
```

| Parameter | Description | Value |
|---|---|---|
| `GGML_CUDA_ENABLE_UNIFIED_MEMORY` | CUDA unified memory (env var) | `1`. Lets pages move between GPU and system RAM as needed. If cache and model ever push past 12 GB VRAM, pages spill to RAM instead of crashing. |
| `--host` | Bind address | `127.0.0.1`. No access from outside the machine. |
| `--port` | Listen port | `8080`. Agent connects here. |
| `--timeout` | Idle connection timeout | `600` (10 minutes). Covers the gap between agent requests without holding connections open forever. |
| `--parallel` | Concurrent slots | `1`. A second slot would double the KV cache. With 12 GB VRAM that means thrashing. |
| `--jinja` | Jinja chat template | On. Hermes Agent uses the template for tool-calling. |
| `--metrics` | Prometheus-style metrics | On. Exposes `/metrics` for checking VRAM, tokens, and speed while debugging. |
| `--sleep-idle-seconds` | Unload model on idle | `600` (10 minutes). Frees VRAM when nothing is happening. The model reloads on the next request. |
| `--ctx-checkpoints` | KV cache checkpoints | `64`. Saves the context at regular intervals. When a new request shares a prefix with an earlier one, the server picks up from the nearest checkpoint instead of recomputing attention from the start. |
| `--checkpoint-min-step` | Checkpoint interval | `512`. A checkpoint every 512 tokens. An agent that sends many short requests hits a recent checkpoint more often. |
| `-fa` | Flash attention | `auto`. llama.cpp picks the best implementation for the hardware. Cuts VRAM use and speeds up attention. |
| `-ngl` | Layers offloaded to GPU | `99`. Every layer stays on the GPU. If even one layer ran on CPU, generation speed would tank. |
| `-c` | Context window size | `65536` (64k). |
| `-ctk` | Key cache quantization | `q4_0`. 4-bit KV cache, about 1.1 GB at 64k context. A full f16 cache would be about 4 GB, pushing model plus cache over 12 GB VRAM and cutting generation speed from ~13.2 t/s to ~1.4 t/s. |
| `-ctv` | Value cache quantization | `q4_0`. Same deal as `-ctk`: 4-bit keeps the value cache small enough to fit. |
| `-m` | Model path | `/etc/llama/models/qwen3.8-27b-q3.gguf`. |

Enable and start llama-server:

```bash
systemctl daemon-reload
systemctl enable llama-server.service
systemctl start llama-server.service
```

### Update

Stop service and uninstall:

```bash
# As root
systemctl stop llama-server.service
cd llama.cpp/build
xargs rm < install_manifest.txt
cd ..
rm -rf build
```

Fetch new commits:

```bash
cd llama.cpp
git fetch
git checkout master
git pull
```

Then, from the install process, replay:
1. Checkout on the last build
2. Build
3. Install
4. Start llama-server service