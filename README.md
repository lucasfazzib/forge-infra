# 🔥 Forge — Local AI Lab Infrastructure

Infrastructure-as-code and documentation for **Forge**, a self-hosted local AI laboratory built to run local LLMs, containerized applications, AI services, RAG pipelines, agents and experimental software.

The goal of Forge is to provide a reproducible, persistent and security-conscious environment for building AI applications locally.

> **Current milestone:** LAB v0.1 — Local LLM inference with a containerized web interface.

---

## 🏗️ Current Architecture

```text
                    FORGE
                      │
              Ubuntu Linux Host
                      │
          ┌───────────┴───────────┐
          │                       │
       Docker                  Ollama
          │                       │
     Open WebUI               Local LLMs
          │                       │
          └──────────┬────────────┘
                     │
                  NVIDIA GPU
```

Current request flow:

```text
Browser
   │
   ▼
127.0.0.1:3000
   │
   ▼
Open WebUI
[Docker Container]
   │
   ▼
Docker Bridge
172.30.0.0/24
   │
   ▼
Ollama API
Host :11434
   │
   ├── Hermes 3 3B
   └── Gemma 3 4B
```

The web interface is bound to localhost and is therefore not intentionally exposed to the local network.

---

# 📦 Storage Architecture

Forge separates the operating system from persistent LAB data.

The Linux operating system resides on the system disk while AI-related data is stored under:

```text
/data
```

Current organization:

```text
/data/
├── artifacts/
├── backups/
├── databases/
├── datasets/
├── documents/
├── experiments/
├── logs/
├── models/
├── ollama/
└── projects/
```

Infrastructure repositories live under:

```text
/data/projects/
```

This repository:

```text
/data/projects/forge-infra/
```

Docker's data directory was also moved away from the system disk:

```text
/data/docker
```

This prevents container images, layers and runtime data from unnecessarily consuming the operating-system filesystem.

---

# 🐳 Docker

Forge uses **Docker Engine** with the Docker Compose plugin.

Docker was configured with:

```json
{
  "data-root": "/data/docker",
  "log-driver": "local"
}
```

Configuration location:

```text
/etc/docker/daemon.json
```

The `local` logging driver is used to reduce the risk of uncontrolled container log growth.

Verify Docker:

```bash
sudo systemctl status docker
sudo docker version
sudo docker compose version
```

Verify Docker storage:

```bash
sudo docker info | grep -E "Docker Root Dir|Logging Driver"
```

Expected:

```text
Logging Driver: local
Docker Root Dir: /data/docker
```

---

# 🤖 Ollama

Ollama runs directly on the Ubuntu host as a `systemd` service.

Check its status:

```bash
systemctl status ollama
```

List installed models:

```bash
ollama list
```

Current models:

```text
hermes3:3b
gemma3:4b
```

Ollama models are persisted under:

```text
/data/ollama
```

The service override is located at:

```text
/etc/systemd/system/ollama.service.d/override.conf
```

Current relevant configuration:

```ini
[Service]
Environment="OLLAMA_MODELS=/data/ollama"
Environment="OLLAMA_HOST=0.0.0.0:11434"
```

The Ollama API can be tested locally with:

```bash
curl http://127.0.0.1:11434/api/tags
```

---

# 🌐 Open WebUI

Open WebUI runs as a Docker container and communicates with Ollama running on the host.

The service is managed through:

```text
compose.yaml
```

Start:

```bash
sudo docker compose up -d
```

Stop:

```bash
sudo docker compose down
```

Restart:

```bash
sudo docker compose restart open-webui
```

Check status:

```bash
sudo docker compose ps
```

View logs:

```bash
sudo docker logs open-webui --tail 100
```

Open WebUI is available locally at:

```text
http://127.0.0.1:3000
```

The Docker port mapping intentionally binds to:

```text
127.0.0.1
```

instead of:

```text
0.0.0.0
```

to avoid exposing the interface to the LAN.

---

# 🔌 Docker → Ollama Networking

Forge uses a dedicated Docker bridge network:

```text
forge-net
```

Subnet:

```text
172.30.0.0/24
```

This subnet is explicitly configured instead of relying on an automatically assigned Docker network.

Open WebUI accesses Ollama through:

```text
host.docker.internal:11434
```

with Docker's:

```text
host-gateway
```

mapping.

Connectivity can be tested from inside the container:

```bash
sudo docker exec open-webui \
  curl --max-time 5 -sS \
  http://host.docker.internal:11434/api/tags
```

A successful response should return the locally installed Ollama models.

---

# 🔐 Firewall

Ubuntu UFW is enabled.

Ollama listens on port:

```text
11434/tcp
```

Access to this port is restricted to the dedicated Forge Docker network:

```text
172.30.0.0/24
```

Current intended rule:

```text
11434/tcp ALLOW IN 172.30.0.0/24
```

Check firewall rules:

```bash
sudo ufw status numbered
```

This allows container-to-Ollama communication without intentionally opening the Ollama API to the rest of the local network.

---

# 🔒 Security Principles

Forge follows several basic security principles.

### Local-first

Services should remain local unless remote access is explicitly required.

### Least privilege

Administrative privileges are used only where necessary.

Docker commands currently require:

```bash
sudo docker ...
```

The normal user is intentionally not automatically added to the `docker` group because Docker daemon access can effectively provide root-level privileges.

### No secrets in Git

The repository must never contain:

* passwords
* API keys
* access tokens
* private SSH keys
* `.env` files containing secrets
* credentials
* private databases
* private datasets
* backups
* personal documents

The `.gitignore` provides an additional safeguard, but every commit should still be reviewed.

Before committing:

```bash
git status
git diff --cached
```

---

# 🔑 GitHub Authentication

Forge authenticates to GitHub using a dedicated Ed25519 SSH key.

Conceptually:

```text
Forge
  │
  ▼
Dedicated SSH Key
  │
  ▼
GitHub
```

The private key remains exclusively on Forge.

Only the **public key** is registered with GitHub.

The private key must never be committed to this repository.

Test authentication:

```bash
ssh -T git@github.com
```

---

# 💻 Development Environment

Development is performed primarily using Visual Studio Code.

The main workspace is:

```text
/data/projects
```

Relevant tooling currently includes:

```text
Visual Studio Code
Git
Python tooling
Docker tooling
YAML tooling
```

Example:

```bash
code /data/projects
```

---

# ⚙️ Service Lifecycle

Docker and Ollama run as background services.

Closing:

```text
VS Code
Terminal
Firefox
```

does **not** stop the LAB.

Quick health check:

```bash
systemctl is-active ollama
systemctl is-active docker
sudo docker compose -f /data/projects/forge-infra/compose.yaml ps
```

Expected state:

```text
ollama     active
docker     active
open-webui Up
```

---

# 📂 Repository Structure

Current repository:

```text
forge-infra/
├── .gitignore
├── compose.yaml
└── README.md
```

As the LAB evolves, infrastructure configuration and documentation will remain version-controlled here.

Runtime data should remain outside Git.

---

# 🚀 Roadmap

## LAB v0.1 — Infrastructure Foundation

* [x] Ubuntu host configured
* [x] Dedicated `/data` storage
* [x] Ollama installed
* [x] Persistent Ollama model storage
* [x] Local LLM inference
* [x] Visual Studio Code
* [x] Git
* [x] Dedicated GitHub SSH authentication
* [x] Docker Engine
* [x] Docker Compose
* [x] Docker storage moved to `/data`
* [x] Open WebUI containerized
* [x] Dedicated Docker network
* [x] Docker → Ollama connectivity
* [x] UFW restriction for Ollama
* [x] Infrastructure version-controlled

## LAB v0.2 — AI Application Layer

Planned:

* [ ] Pin container image versions
* [ ] Improve secrets management
* [ ] Infrastructure health checks
* [ ] Build a Forge API
* [ ] FastAPI service
* [ ] Local LLM gateway
* [ ] Structured logging
* [ ] Application configuration
* [ ] First reusable AI service

## LAB v0.3 — Knowledge Layer

Planned:

* [ ] Document ingestion
* [ ] Embeddings
* [ ] Vector database
* [ ] Retrieval-Augmented Generation (RAG)
* [ ] Local knowledge bases
* [ ] Evaluation pipeline

## LAB v0.4 — Agents & Automation

Planned:

* [ ] Tool-enabled agents
* [ ] Sandboxed execution
* [ ] Human approval for critical actions
* [ ] Workflow automation
* [ ] Scheduled jobs
* [ ] Observability and audit logs

---

# 🎯 Long-Term Direction

Forge is intended to evolve from a local LLM server into a reusable AI engineering platform:

```text
Applications
     │
     ▼
Forge APIs
     │
 ┌───┼──────────────┐
 │   │              │
RAG Agents       Workers
 │   │              │
 └───┼──────────────┘
     ▼
LLM Gateway
     │
 ┌───┴─────────────┐
 │                 │
Local Models    External APIs
 │
 ▼
Ollama
```

The infrastructure itself is not the final product.

It is the foundation for building, testing and operating **AI applications, data pipelines, RAG systems, agents, automation and reusable software**.

---

## Status

**Forge LAB v0.1: ONLINE 🔥**
