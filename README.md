# OpenClaw Installer

> **WIP -  The local deployer works great. K8s and SSH modes are not yet implemented.**

Deploy [OpenClaw](https://github.com/openclaw) from your browser. One script, one command, one click.

## Quick Start

```bash
curl -fsSLo run.sh https://raw.githubusercontent.com/sallyom/claw-installer/main/run.sh
chmod +x run.sh

export ANTHROPIC_API_KEY=sk-...   # or OPENAI_API_KEY=sk-...
./run.sh
```

That's it. The script detects your platform, pulls what it needs, and starts the installer. Open http://localhost:3000 in your browser.

**Requirements:** podman or docker. On macOS with podman, Node.js is also needed (`brew install node`).

### Alternative: clone and run

```bash
git clone https://github.com/sallyom/claw-installer.git
cd claw-installer
./run.sh                    # or: npm install && npm run dev
```

### Then what?

1. Open the installer in your browser
2. Pick **"This Machine"**
3. Fill in a **name prefix** (e.g., `sally`) and **agent name** (e.g., `lynx`)
4. Add your API key if not already detected from the environment
5. Hit **Deploy OpenClaw**
6. Go to the **Instances** tab to manage your deployment — copy the gateway token, view the run command, open the UI, stop/start the container, or delete the data volume

The installer pulls the image, provisions your agent with a default identity and security guidelines, starts the container, and streams logs in real time. Your OpenClaw instance will be running at `http://localhost:18789`.

## What You Get

Every deploy creates a fully configured OpenClaw instance with:

- A registered agent in `openclaw.json` with the right model for your provider
- Workspace files (`AGENTS.md`, `agent.json`) with identity, security rules, and style guidelines
- Skills directory watching enabled
- Gateway configured for local access (no device pairing needed)

## Launcher Script

`run.sh` abstracts all the platform-specific container plumbing:

```bash
./run.sh                              # Pull image and start
./run.sh --build                      # Build from source instead of pulling
./run.sh --port 8080                  # Custom port (default: 3000)
./run.sh --runtime docker             # Force docker (default: auto-detect)
ANTHROPIC_API_KEY=sk-... ./run.sh     # Anthropic
OPENAI_API_KEY=sk-... ./run.sh        # OpenAI
```

| Platform | Runtime | What the script does |
|----------|---------|---------------------|
| macOS | podman | Extracts app from image, runs natively with Node.js |
| macOS | docker | Runs as a container with Docker socket |
| Linux | podman | Runs as a container with rootless podman socket |
| Linux | docker | Runs as a container with `/var/run/docker.sock` |

To stop: `Ctrl+C` (macOS/podman) or `podman stop claw-installer` / `docker stop claw-installer`.

## Model Providers

The installer supports multiple model providers. Set your API key and the model is auto-detected:

| Provider | Default Model | API Key |
|----------|---------------|---------|
| Anthropic | `claude-sonnet-4-6` | `ANTHROPIC_API_KEY` |
| OpenAI | `openai/gpt-5` | `OPENAI_API_KEY` |
| Vertex AI (Gemini) | `google-vertex/gemini-2.5-pro` | GCP credentials |
| Vertex AI (Claude) | `anthropic-vertex/claude-sonnet-4-6` | GCP credentials |
| Self-hosted | `openai/default` | `MODEL_ENDPOINT` |

You can override the model in the deploy form. Examples:

- `claude-opus-4-6`
- `openai/gpt-5.3`

## Customizing Your Agent

After the first deploy, your agent files are saved to `~/.openclaw-installer/agents/` on the host:

```
~/.openclaw-installer/agents/
└── workspace-sally_lynx/
    ├── AGENTS.md      # Agent identity, instructions, security rules
    └── agent.json     # Metadata (name, emoji, color, capabilities)
```

Edit `AGENTS.md` to change your agent's personality, instructions, or capabilities. Re-deploy to apply changes.

### Adding Skills

Create a skill directory with a `SKILL.md`:

```
~/.openclaw-installer/agents/
├── workspace-sally_lynx/
│   ├── AGENTS.md
│   └── agent.json
└── skills/
    └── my-skill/
        └── SKILL.md
```

Skills are copied into the container and loaded automatically by the gateway.

## Features

- **One-click deploy** — pull image, provision agent, start container
- **Agent provisioning** — default agent with identity, security guidelines, and workspace files
- **Custom agents and skills** — edit files on the host, re-deploy to apply
- **Instance management** — stop, start, view token, see run command, delete data
- **Instance discovery** — finds all OpenClaw containers via labels and image name
- **Model selection** — explicit or auto-detected from provider config
- **Saved configs** — `.env` and gateway token saved per instance, auto-loaded on next run
- **.env upload** — upload a `.env` file to pre-fill the deploy form
- **Server env fallback** — API keys from the server environment used automatically

## Host Filesystem

```
~/.openclaw-installer/
├── agents/                              # Agent source files (mounted on deploy)
│   ├── workspace-<prefix>_<name>/
│   │   ├── AGENTS.md
│   │   └── agent.json
│   └── skills/
│       └── <skill-name>/
│           └── SKILL.md
├── openclaw-<prefix>-<name>/            # Per-instance config (auto-saved)
│   ├── .env                             # Instance variables
│   └── gateway-token                    # Gateway auth token
└── ...
```

## Environment Variables

Pass these to `run.sh` or `npm run dev` to set server-side defaults:

| Variable | Purpose |
|----------|---------|
| `ANTHROPIC_API_KEY` | Anthropic API key (users can leave the form field blank) |
| `OPENAI_API_KEY` | OpenAI API key |
| `MODEL_ENDPOINT` | OpenAI-compatible model endpoint for self-hosted models |
| `OPENCLAW_IMAGE` | Default container image |
| `OPENCLAW_PREFIX` | Default name prefix |

## Remote Access

Running the installer on a remote machine:

```bash
# On the remote machine
ANTHROPIC_API_KEY=sk-... ./run.sh --build

# On your laptop
ssh -L 3000:localhost:3000 user@remote-host
# Open http://localhost:3000
```

## Architecture

```
┌─────────────────────────────────────────┐
│           Browser (React + Vite)        │
│  ┌────────────┬──────────┬───────────┐  │
│  │ DeployForm │ LogStream│ Instances │  │
│  └─────┬──────┴────┬─────┴─────┬─────┘  │
│        │ REST      │ WebSocket │ REST   │
└────────┼───────────┼───────────┼────────┘
         ▼           ▼           ▼
┌─────────────────────────────────────────┐
│        Express + WebSocket Server       │
│  ┌──────────┐  ┌──────────────────────┐ │
│  │ Deployers│  │ Services             │ │
│  │  local   │  │  container discovery │ │
│  │  k8s  *  │  │  (podman / docker)   │ │
│  │  ssh  *  │  │                      │ │
│  └──────────┘  └──────────────────────┘ │
└─────────────────────────────────────────┘
              * = planned
```

## Container Details

The installer launches OpenClaw containers with:

- `-p <port>:18789` — port mapping (works on macOS and Linux)
- `--bind lan` — gateway listens on `0.0.0.0` (required for port mapping)
- Labels: `openclaw.managed=true`, `openclaw.prefix=<prefix>`, `openclaw.agent=<name>`
- Volume: `openclaw-<prefix>-data` at `/home/node/.openclaw`

## Manual Setup

If `run.sh` doesn't work for your setup:

```bash
# Linux (podman)
podman run -d --name claw-installer \
  --security-opt label=disable \
  -p 3000:3000 \
  -v /run/user/$(id -u)/podman/podman.sock:/run/podman/podman.sock \
  -v ~/.openclaw-installer:/home/node/.openclaw-installer:Z \
  -e ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY \
  quay.io/sallyom/claw-installer:latest

# Docker (any platform)
docker run -d --name claw-installer \
  -p 3000:3000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v ~/.openclaw-installer:/home/node/.openclaw-installer \
  -e ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY \
  quay.io/sallyom/claw-installer:latest

# macOS with podman (run from source — socket forwarding not supported)
git clone https://github.com/sallyom/claw-installer.git
cd claw-installer && npm install && npm run dev
```

## API

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/health` | GET | Runtime detection, version, server defaults |
| `/api/deploy` | POST | Start a deployment (streams logs via WebSocket) |
| `/api/configs` | GET | List saved instance configs |
| `/api/instances` | GET | List all discovered instances |
| `/api/instances/:name/start` | POST | Start a stopped instance |
| `/api/instances/:name/stop` | POST | Stop and remove container (volume preserved) |
| `/api/instances/:name/token` | GET | Get the gateway auth token |
| `/api/instances/:name/command` | GET | Get the run command |
| `/api/instances/:name/data` | DELETE | Delete the data volume |
| `/ws` | WebSocket | Subscribe to deploy logs |

## Project Structure

```
claw-installer/
├── run.sh                        # Launcher script (macOS + Linux, podman + docker)
├── Dockerfile                    # Multi-stage build (node + podman CLI)
├── src/
│   ├── server/
│   │   ├── index.ts              # Express + WS server
│   │   ├── ws.ts                 # WebSocket log streaming
│   │   ├── routes/
│   │   │   ├── deploy.ts         # POST /api/deploy
│   │   │   ├── status.ts         # Instance discovery and lifecycle
│   │   │   └── agents.ts         # Agent browsing
│   │   ├── deployers/
│   │   │   ├── types.ts          # Deployer interface
│   │   │   └── local.ts          # Podman/docker deployer + agent provisioning
│   │   └── services/
│   │       └── container.ts      # Runtime detection, container/volume discovery
│   └── client/
│       ├── App.tsx               # Tabs: Deploy | Instances | Agents
│       ├── components/
│       │   ├── DeployForm.tsx     # Config form + .env upload
│       │   ├── LogStream.tsx      # Real-time deploy output
│       │   ├── InstanceList.tsx   # Manage running instances
│       │   └── AgentBrowser.tsx   # Browse agents
│       └── styles/theme.css      # Dark theme
└── package.json
```

## Roadmap

- [x] Local deployer (podman + docker, macOS + Linux)
- [x] Launcher script with platform auto-detection
- [x] Instance discovery and lifecycle
- [x] Gateway token access from UI
- [x] Saved configs and .env upload
- [x] Multi-provider support (Anthropic, OpenAI, Vertex AI)
- [x] Model selection (explicit or auto-detect)
- [x] Default agent provisioning with workspace files
- [x] Custom agent/skill provisioning from host directory
- [ ] Cron job provisioning from JOB.md files
- [ ] Agent/skill import from git repos
- [ ] Kubernetes deployer (K8s API)
- [ ] SSH deployer (remote host)
