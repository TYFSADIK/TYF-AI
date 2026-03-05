```md
# TYF-AI — Self-Hosted AI Assistant Platform (Public)

TYF-AI is a **self-hosted AI assistant platform** built by **MD. Taki Yasir Faraji Sadik (Taki)**.  
It runs a local LLM behind an **OpenAI-compatible API** (llama.cpp), provides a modern chat UI (Open WebUI), enables **web search** (SearXNG), and securely exposes everything through **Cloudflare Tunnel** — all managed as **systemd services**.

> ⚠️ Note on “model”: TYF-AI is the platform and deployment. You can **bring your own GGUF model file** (weights). This repo documents the full build + hosting pipeline.

---

## Features

- ✅ **Local LLM server (OpenAI compatible)** via `llama.cpp` (`/v1/chat/completions`, `/v1/models`)
- ✅ **Open WebUI** (public sign-up, admin controls)
- ✅ **Web Search** via **SearXNG** integration
- ✅ **Cloudflare Tunnel** (no open ports on router, safe public domain)
- ✅ **systemd services** for:
  - llama.cpp
  - docker compose stack (Open WebUI + SearXNG)
  - cloudflared tunnel
- ✅ **Branding**: custom favicon + custom Open WebUI build (optional)

---

## Architecture

```

Internet
│
└── Cloudflare Tunnel (cloudflared)
│
└── Open WebUI (Docker)  [http://127.0.0.1:3000](http://127.0.0.1:3000)
│
├── Web Search -> SearXNG (Docker) [http://127.0.0.1:8080](http://127.0.0.1:8080)
│
└── LLM API -> llama.cpp server (systemd) [http://127.0.0.1:4545/v1](http://127.0.0.1:4545/v1)

````

---

## Requirements

### Hardware (recommended)
- Linux server (tested on Debian)
- NVIDIA GPU recommended (works on CPU too, just slower)
- Enough RAM/VRAM for your chosen GGUF model

### Software
- Docker + Docker Compose v2
- `systemd`
- `cloudflared` (Cloudflare Tunnel)
- `curl`, `git`

---

## Quick Start (Copy/Paste)

### 1) Install Docker + Compose
```bash
sudo apt update
sudo apt install -y docker.io docker-compose
sudo systemctl enable --now docker

# optional: run docker without sudo
sudo groupadd docker 2>/dev/null || true
sudo usermod -aG docker "$USER"
newgrp docker
````

---

## Deploy Open WebUI + SearXNG (Docker Compose)

### 2) Create the stack

```bash
mkdir -p ~/ai-stack && cd ~/ai-stack

cat > docker-compose.yml <<'YAML'
services:
  searxng:
    image: docker.io/searxng/searxng:latest
    container_name: searxng
    restart: unless-stopped
    ports:
      - "127.0.0.1:8080:8080"
    volumes:
      - ./searxng:/etc/searxng:rw
      - searxng-data:/var/cache/searxng:rw

  open-webui:
    # Use official image OR your own custom image:
    # image: ghcr.io/open-webui/open-webui:main
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    restart: unless-stopped
    ports:
      - "127.0.0.1:3000:8080"
    volumes:
      - open-webui:/app/backend/data
    extra_hosts:
      - "host.docker.internal:host-gateway"
    environment:
      # Connect Open WebUI to your llama.cpp OpenAI-compatible server (must include /v1)
      OPENAI_API_BASE_URL: "http://host.docker.internal:4545/v1"
      OPENAI_API_KEY: "sk-no-key"

      # Prefer env config (prevents DB overrides)
      ENABLE_PERSISTENT_CONFIG: "False"

      # Enable Web Search via SearXNG
      ENABLE_RAG_WEB_SEARCH: "True"
      RAG_WEB_SEARCH_ENGINE: "searxng"
      RAG_WEB_SEARCH_RESULT_COUNT: "5"
      RAG_WEB_SEARCH_CONCURRENT_REQUESTS: "10"
      SEARXNG_QUERY_URL: "http://searxng:8080/search?q=<query>&format=json"

      # Public auth (turn off if private)
      WEBUI_AUTH: "True"
      ENABLE_SIGNUP: "True"

volumes:
  searxng-data:
  open-webui:
YAML

docker compose up -d
```

### 3) Test services

```bash
curl -I http://127.0.0.1:3000 | head
curl -s "http://127.0.0.1:8080/search?q=test&format=json" | head -c 200; echo
```

---

## Install & Build llama.cpp

### 4) Build llama.cpp (CUDA optional)

```bash
cd ~
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp

# CUDA build (NVIDIA). If you want CPU-only, remove -DGGML_CUDA=ON
cmake -B build -DGGML_CUDA=ON
cmake --build build -j
```

---

## Run llama.cpp as a systemd service

### 5) Create `llama.service`

> Replace `YOUR_MODEL.gguf` with your GGUF model file path.

```bash
sudo tee /etc/systemd/system/llama.service >/dev/null <<'EOF'
[Unit]
Description=TYF-AI llama.cpp Server (OpenAI-compatible)
After=network.target

[Service]
User=ai
WorkingDirectory=/home/ai/llama.cpp
ExecStart=/home/ai/llama.cpp/build/bin/llama-server \
  --model /home/ai/Downloads/YOUR_MODEL.gguf \
  --alias TYF-AI \
  --port 4545 \
  --host 0.0.0.0 \
  --ctx-size 4096 \
  --n-gpu-layers 12 \
  --threads 8 \
  --no-mmap \
  --cache-ram 0 \
  --n-predict 512 \
  --chat-template-kwargs '{"enable_thinking":false}'

Restart=always
RestartSec=5
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now llama.service
sudo systemctl status llama.service --no-pager
```

### 6) Test llama.cpp OpenAI-compatible endpoints

```bash
curl -s http://127.0.0.1:4545/v1/models | head -c 400; echo
```

---

## Run Docker stack as a systemd service

### 7) Create `ai-stack.service`

```bash
sudo tee /etc/systemd/system/ai-stack.service >/dev/null <<'EOF'
[Unit]
Description=TYF-AI Stack (Open WebUI + SearXNG)
After=docker.service network-online.target
Wants=docker.service network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/home/ai/ai-stack
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now ai-stack.service
sudo systemctl status ai-stack.service --no-pager
```

---

## Expose Publicly with Cloudflare Tunnel

### 8) Install cloudflared

```bash
cd /tmp
curl -L -o cloudflared-linux-amd64.deb \
  https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared-linux-amd64.deb
cloudflared --version
```

### 9) Login + create tunnel + route DNS

```bash
cloudflared tunnel login
cloudflared tunnel create ai-tyfsadik
cloudflared tunnel route dns ai-tyfsadik ai.tyfsadik.org
```

### 10) Create `/etc/cloudflared/config.yml`

> Replace the UUIDs with your real tunnel ID and credential file.

```bash
sudo mkdir -p /etc/cloudflared

sudo tee /etc/cloudflared/config.yml >/dev/null <<'EOF'
tunnel: YOUR_TUNNEL_UUID
credentials-file: /home/ai/.cloudflared/YOUR_TUNNEL_UUID.json

ingress:
  - hostname: ai.tyfsadik.org
    service: http://localhost:3000
  - service: http_status:404
EOF
```

### 11) Install cloudflared as a service

```bash
sudo cloudflared service install
sudo systemctl enable --now cloudflared
sudo systemctl status cloudflared --no-pager
```

### 12) (Recommended) Ensure cloudflared starts AFTER ai-stack

```bash
sudo systemctl edit cloudflared
```

Paste:

```ini
[Unit]
After=ai-stack.service
Wants=ai-stack.service
```

Then:

```bash
sudo systemctl daemon-reload
sudo systemctl restart cloudflared
```

---

## Branding (Optional)

### Custom favicon

If you build a custom Open WebUI image, replace:

* `static/favicon.png`

Example:

```bash
cd ~/openwebui-brand/open-webui
curl -L -o static/favicon.png "https://YOUR_IMAGE_URL.png"
docker build -t open-webui-tyfai:latest .
```

Then set in `~/ai-stack/docker-compose.yml`:

```yaml
image: open-webui-tyfai:latest
```

Restart:

```bash
cd ~/ai-stack
docker compose down
docker compose up -d
```

Hard refresh the browser (favicons cache aggressively).

---

## Security Notes (Recommended for Public Hosting)

* Keep Open WebUI bound to `127.0.0.1` (done in compose) and expose only through Cloudflare Tunnel.
* Use Cloudflare Zero Trust / WAF / Rate limiting if public.
* Consider disabling signups if you want private access:

  * `ENABLE_SIGNUP: "False"`

---

## Troubleshooting

### Open WebUI logs

```bash
docker logs --tail 200 open-webui
```

### SearXNG test

```bash
curl -s "http://127.0.0.1:8080/search?q=test&format=json" | head -c 300; echo
```

### llama.cpp status/logs

```bash
sudo systemctl status llama.service --no-pager
journalctl -u llama.service -n 100 --no-pager
```

### cloudflared status/logs

```bash
sudo systemctl status cloudflared --no-pager
sudo journalctl -u cloudflared -n 100 --no-pager
```

---

## Credits / Acknowledgements

* Open WebUI (UI)
* llama.cpp (OpenAI-compatible local inference server)
* SearXNG (web search engine)
* Cloudflare Tunnel (secure public access)

---

## Author

TYF Sadik

---

## License

This repository contains deployment/configuration instructions and scripts.
Respect upstream licenses for Open WebUI, llama.cpp, and any model weights you use.

```
```
