# OpenClaw Installer

> **WIP — may be broken. The local (this machine) deployer works. Cluster and SSH modes are not yet implemented.**

A web-based installer for [OpenClaw](https://github.com/openclaw). Deploy and manage OpenClaw instances from a browser — on your laptop.
Deploys containers. Tested with Podman. Docker should work, too.

## Deployment Modes

| Mode | Status | What It Does |
|------|--------|-------------|
| **This Machine** | Working | Runs OpenClaw in podman/docker on localhost |
| **Kubernetes / OpenShift** | Planned | Deploys to a cluster via K8s API |
| **Remote Host** | Planned | Deploys via SSH to a Linux machine |

## Quick Start

```bash
npm install
npm run dev
# Open http://localhost:3001
```

The UI opens in your browser. Pick "This Machine", fill in your prefix and agent name, configure your model provider, and hit Deploy. The installer pulls the OpenClaw image, starts a container, and streams logs in real time.

Your agent is provisioned automatically with a default identity, workspace files, and security guidelines. After the first deploy, editable agent files are saved to `~/.openclaw-installer/agents/` — customize them and re-deploy to update.

For remote access (e.g., running on a NUC over SSH):

```bash
# On the remote machine
npm install && npm run dev

# On your laptop
ssh -L 3001:localhost:3001 user@remote-host
# Open http://localhost:3001
```

## Features

- **Deploy** — Pull image, create volume, provision agent, start container with one click
- **Agent provisioning** — Default agent with identity, security guidelines, and workspace files; auto-configures `openclaw.json` with the right model provider
- **Custom agents** — Drop `AGENTS.md` and skills into `~/.openclaw-installer/agents/` and they're picked up on deploy
- **Instance discovery** — Finds all OpenClaw containers (including manually launched ones) via podman/docker labels and image name
- **Stop / Start** — Lifecycle management; volumes preserve state across restarts
- **Gateway token** — View and copy the gateway auth token from the UI
- **Run command** — See the exact podman/docker command used to launch each instance
- **Delete data** — Remove the data volume when you're done
- **Model providers** — Anthropic (Claude), OpenAI (GPT-5), Google Vertex AI (Gemini or Claude via Vertex), OpenAI-compatible endpoints
- **.env upload** — Upload a `.env` file to pre-fill the deploy form (handy for fleet provisioning)
- **Saved configs** — Each deploy saves `.env` and `gateway-token` to `~/.openclaw-installer/<container-name>/`; loaded automatically on next run
- **Server env fallback** — API keys set on the server (e.g., `ANTHROPIC_API_KEY`) are used automatically if not provided in the form

## Agent Provisioning

On first deploy, the installer:

1. Writes `openclaw.json` with the agent registered, model auto-detected from your provider config, skills directory watching, and cron enabled
2. Creates a workspace at `/home/node/.openclaw/workspace-<prefix>_<agentName>/` with `AGENTS.md` (identity, security, style guidelines) and `agent.json` (metadata)
3. Saves the agent files to `~/.openclaw-installer/agents/` on the host for editing

### Customizing Your Agent

After the first deploy, you'll find your agent files at:

```
~/.openclaw-installer/agents/
└── workspace-<prefix>_<agentName>/
    ├── AGENTS.md      # Agent identity, instructions, security rules
    └── agent.json     # Metadata (name, emoji, color, capabilities)
```

Edit these files and re-deploy — the installer mounts this directory and copies your customizations into the volume.

### Adding Skills

Create a skill directory with a `SKILL.md`:

```
~/.openclaw-installer/agents/
├── workspace-<prefix>_<agentName>/
│   ├── AGENTS.md
│   └── agent.json
└── skills/
    └── my-skill/
        └── SKILL.md    # Skill instructions (the agent reads this)
```

Skills are copied into `~/.openclaw/skills/` inside the container. The gateway watches this directory and loads new skills automatically.

### Custom Agent Source Directory

By default, the installer looks for agent files in `~/.openclaw-installer/agents/`. You can override this in the deploy form with any host directory path. The expected structure:

```
my-agents/
├── agents/                          # Copied into ~/.openclaw/ in the container
│   └── workspace-<prefix>_<name>/
│       ├── AGENTS.md
│       └── agent.json
└── skills/                          # Copied into ~/.openclaw/skills/
    └── some-skill/
        └── SKILL.md
```

### Model Selection

You can set the model explicitly in the deploy form, or leave it blank for auto-detection based on your provider config:

| Provider | Default Model |
|----------|---------------|
| Anthropic API key | `claude-sonnet-4-6` |
| OpenAI API key | `openai/gpt-5` |
| Vertex AI (Google) | `google-vertex/gemini-2.5-pro` |
| Vertex AI (Anthropic) | `anthropic-vertex/claude-sonnet-4-6` |
| OpenAI-compatible endpoint | `openai/default` |

To use a different model, type it in the Model field. Examples:

- `claude-opus-4-6` — Anthropic Opus
- `openai/gpt-5.3` — OpenAI GPT-5.3

The model is saved in the instance `.env` as `AGENT_MODEL` and restored on the next deploy.

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

Instances are discovered directly from podman/docker — no state files needed. Running containers are found via `podman ps` (filtered by label `openclaw.managed=true` or image name). Stopped instances are discovered via orphaned `openclaw-*-data` volumes.

## API

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/health` | GET | Runtime detection, version, server environment defaults |
| `/api/deploy` | POST | Start a deployment (returns deployId, streams logs via WS) |
| `/api/configs` | GET | List saved instance configs from `~/.openclaw-installer/` |
| `/api/instances` | GET | List all discovered instances with live status |
| `/api/instances/:name/start` | POST | Start a stopped instance (re-creates container from volume) |
| `/api/instances/:name/stop` | POST | Stop and remove container (volume preserved) |
| `/api/instances/:name/token` | GET | Get the gateway auth token |
| `/api/instances/:name/command` | GET | Get the podman/docker run command |
| `/api/instances/:name/data` | DELETE | Delete the data volume |
| `/api/agents/local` | GET | List agents from a local repo |
| `/api/agents/browse?repo=...` | GET | List agents from a public git repo |
| `/ws` | WebSocket | Subscribe to deploy logs by deployId |

## Container Details

The installer launches OpenClaw containers with:

- `-p <port>:18789` — Port mapping from host to gateway (works on both macOS and Linux)
- `--bind lan` — Gateway listens on `0.0.0.0` inside the container (required for port mapping)
- Labels: `openclaw.managed=true`, `openclaw.prefix=<prefix>`, `openclaw.agent=<name>`
- Volume: `openclaw-<prefix>-data` mounted at `/home/node/.openclaw`
- Image: `quay.io/sallyom/openclaw:latest` (configurable)

## Host Filesystem Layout

```
~/.openclaw-installer/
├── agents/                              # Agent source files (auto-mounted on deploy)
│   ├── workspace-<prefix>_<name>/
│   │   ├── AGENTS.md                    # Agent identity and instructions
│   │   └── agent.json                   # Agent metadata
│   └── skills/
│       └── <skill-name>/
│           └── SKILL.md                 # Skill instructions
├── openclaw-<prefix>-<name>/            # Per-instance config (auto-saved)
│   ├── .env                             # Instance variables (loaded on next run)
│   └── gateway-token                    # Gateway auth token
└── ...
```

## Running as a Container

Build and run the installer itself as a container. The installer needs access to the host's container runtime socket so it can manage OpenClaw containers on your behalf.

> **Note:** Running containerized is tested on Linux. On macOS, podman runs inside a VM which may cause socket access issues. Use `npm run dev` directly on Mac instead.

```bash
# Build
podman build -t claw-installer .

# Run (podman)
podman run -p 3000:3000 \
  -v /run/user/$(id -u)/podman/podman.sock:/run/podman/podman.sock \
  -v ~/.openclaw-installer:/home/node/.openclaw-installer:Z \
  -e ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY \
  claw-installer

# Run (docker)
docker run -p 3000:3000 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v ~/.openclaw-installer:/home/node/.openclaw-installer:Z \
  -e ANTHROPIC_API_KEY=$ANTHROPIC_API_KEY \
  claw-installer
```

Then open `http://localhost:3000`.

You can pass server-side defaults as environment variables:

| Variable | Purpose |
|----------|---------|
| `ANTHROPIC_API_KEY` | Pre-fill Anthropic API key (users can leave the field blank) |
| `OPENAI_API_KEY` | Pre-fill OpenAI API key |
| `MODEL_ENDPOINT` | Default model endpoint (OpenAI-compatible) |
| `OPENCLAW_IMAGE` | Default container image |
| `OPENCLAW_PREFIX` | Default name prefix |

## Project Structure

```
claw-installer/
├── src/
│   ├── server/
│   │   ├── index.ts              # Express + WS server, serves static frontend
│   │   ├── ws.ts                 # WebSocket log streaming
│   │   ├── routes/
│   │   │   ├── deploy.ts         # POST /api/deploy
│   │   │   ├── status.ts         # Instance discovery and lifecycle
│   │   │   └── agents.ts         # Agent browsing (local + git)
│   │   ├── deployers/
│   │   │   ├── types.ts          # Deployer interface
│   │   │   └── local.ts          # podman/docker deployer + agent provisioning
│   │   └── services/
│   │       └── container.ts      # Runtime detection, container/volume discovery
│   └── client/
│       ├── App.tsx               # Tabs: Deploy | Instances | Agents
│       ├── components/
│       │   ├── DeployForm.tsx     # Mode selector + config form + .env upload
│       │   ├── LogStream.tsx      # Real-time deploy output
│       │   ├── InstanceList.tsx   # Manage running instances
│       │   └── AgentBrowser.tsx   # Browse agents from local repo or git
│       └── styles/theme.css      # Dark theme matching OpenClaw UI
├── Dockerfile
└── package.json
```

## Roadmap

- [x] Local deployer (podman/docker on this machine)
- [x] Instance discovery and lifecycle (stop, start, delete data)
- [x] Gateway token access from UI
- [x] .env file upload and per-instance export
- [x] Vertex AI / multi-provider support
- [x] Default agent provisioning with workspace files
- [x] Custom agent/skill provisioning from host directory
- [x] Model selection (explicit or auto-detect from provider)
- [x] OpenAI provider support (GPT-5)
- [ ] Cron job provisioning from JOB.md files
- [ ] Agent/skill import from git repos
- [ ] Kubernetes deployer (K8s API)
- [ ] SSH deployer (remote host)
- [ ] Private git repo auth (PAT)
