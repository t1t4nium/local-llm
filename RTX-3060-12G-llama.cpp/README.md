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