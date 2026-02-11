# OpenClaw Docker

Run an OpenClaw gateway in Docker with browser automation support (Chromium + Playwright).

## Prerequisites

- Docker and Docker Compose
- An `openclaw:local` base image (the OpenClaw OCI image)
- API keys for at least one LLM provider (Anthropic and/or OpenAI)
- Massive proxy credentials ([sign up](https://partners.joinmassive.com/create-account-clawpod))

## Quick start

```bash
# 1. Copy the example env file and fill in your credentials
cp .env.example .env

# 2. Build and start the gateway
docker compose up -d --build openclaw

# 3. Pair your device — open the gateway URL shown in the logs
docker compose logs -f openclaw
```

## Configuration

### Environment variables

All secrets live in `.env` (never committed to git). See `.env.example` for the full list:

| Variable | Required | Description |
|----------|----------|-------------|
| `ANTHROPIC_API_KEY` | Yes* | Anthropic API key |
| `OPENAI_API_KEY` | No | OpenAI API key |
| `MASSIVE_PROXY_USERNAME` | Yes | Massive proxy username |
| `MASSIVE_PROXY_PASSWORD` | Yes | Massive proxy password |
| `OPENCLAW_GATEWAY_TOKEN` | No | Gateway auth token (default: `clawpod-dev`) |

\* At least one LLM provider key is required.

### Model selection

The default model is configured in `openclaw.json` and mounted read-only into the container. Edit it to change the model:

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "anthropic/claude-sonnet-4-5-20250929"
      }
    }
  }
}
```

Verify with:

```bash
docker compose exec openclaw node openclaw.mjs models status
```

### Device pairing

Device pairing is interactive and cannot be automated. After starting the gateway, open the pairing URL from the logs and approve devices manually.

## Common commands

```bash
# Start the gateway
docker compose up -d --build openclaw

# View logs
docker compose logs -f openclaw

# Check model config
docker compose exec openclaw node openclaw.mjs models status

# Stop and remove containers
docker compose down

# Stop and remove containers + volumes (full reset)
docker compose down -v

# Run the ClawPod test suite
docker compose run --rm clawpod-test
```

## Architecture

The `openclaw` service uses `Dockerfile.openclaw` which extends the `openclaw:local` base image with:

- Chromium browser (via Playwright)
- `agent-browser` for browser automation
- `clawhub` for skill management

### Volumes

| Volume | Path in container | Purpose |
|--------|-------------------|---------|
| `openclaw-devices` | `~/.openclaw/devices` | Paired device data (persists across rebuilds) |
| `openclaw-config` | `~/.openclaw/agents` | Agent configuration |
| `openclaw.json` (bind mount) | `~/.openclaw/openclaw.json` | Model config (read-only) |

### Ports

| Port | Protocol | Description |
|------|----------|-------------|
| 18789 | WebSocket | Gateway endpoint for device connections |

The canvas UI is also served at `http://localhost:18789/__openclaw__/canvas/`.

## Troubleshooting

**Container exits immediately**
Check logs with `docker compose logs openclaw`. Most likely a missing or invalid API key in `.env`.

**"Missing or invalid Sec-WebSocket-Key"**
Port 18789 is a WebSocket server, not HTTP. Use a WebSocket client or the OpenClaw app to connect.

**Model not configured after rebuild**
Verify `openclaw.json` exists in the project root and the volume mount is present in `docker-compose.yml`.

**Device pairing doesn't persist**
The `openclaw-devices` volume stores paired devices. Running `docker compose down -v` deletes it and you'll need to re-pair.
