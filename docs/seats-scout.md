# Seats Scout — CelestOS Showcase Data

## Seat Launch Configuration

All seats launch via aliases/scripts in `/root/.bashrc` or `/home/juxtapo/.local/bin/`.

| Seat | Alias/Script | Room | Model | Base URL (proxy chain) |
|------|-------------|------|-------|----------------------|
| **Celeste** | `celeste()` function | `/root/Opus` | Claude Opus | `:9918` (jaga → anthropic) |
| **Monica** | `claude-monica` alias | `/root/Monica` | Claude Sonnet | `:9921` (jaga → anthropic) |
| **Fable** | `fable` alias | `/root/Fable` | Claude (default) | `:9918` (jaga → anthropic) |
| **Rogue** | `rogue` alias | `/root/Rogue` | Claude (default) | `:9918` (jaga → anthropic) |
| **Lucius (Luc)** | GPT-5.4/Codex | `/root/Rogue` area | Codex | direct |
| **Zet** | `/home/juxtapo/.local/bin/claude-zet` | `/home/juxtapo/Zet` | GLM-5.2 | `:9924` (jaga zet-translate → headroom :8787 → rootsys) |
| **Kim** | `/home/juxtapo/.local/bin/claude-kim` | `/home/juxtapo/opencode-kimi` | Kimi-K3 | `:9923` (jaga kim-translate → headroom :8787 → rootsys) |

### Proxy Chain
All root seats (Celeste, Monica, Fable, Rogue) route through **jaga** (`:9918`/`:9921`), an observability proxy that logs all requests to `feed.jsonl`. Non-Anthropic seats (Zet, Kim) route through their own jaga channels with model-specific translate layers.

## Seat Rooms (Directories)

```
/root/Opus          ← Celeste
/root/Monica        ← Monica
/root/Fable         ← Fable
/root/Rogue         ← Rogue
/root/Symphony      ← (archived/special)
/home/juxtapo/Zet   ← Zet (runs as juxtapo, not root)
/home/juxtapo/opencode-kimi ← Kim
```

## MCP Servers (16 configured in `/root/.claude.json`)

| Server | What | Impl |
|--------|------|------|
| **bravemag** | Browser automation (Brave) | Rust binary, bbx native host |
| **blendmag** | Blender 3D | sudo, Blender MCP gateway |
| **tvmag** | TradingView charts | Node.js, CDP to TV desktop |
| **integratedmag** | VS Code browser | Python, integrated-browser-mcp |
| **orbmag** | O.R.B mobile transfer | Rust binary |
| **pixelmag** | Pixel art editor | sudo gateway |
| **tiledmag** | Tiled map editor | Python HTTP bridge |
| **codegraph** | Code intelligence graph | `codegraph serve` |
| **catalyst** | Nudger/search tools | Python |
| **geminigen** | Gemini image generation | Python MCP relay |
| **hubmag** | Phone hub | Python |
| **serena** | Project server | Serena |
| **camera** | Webcam snap | sudo gateway |
| **discord-mcp** | Discord bridge | Bash wrapper |
| **gcloud** | GCloud CLI | sudo wrapper |
| **obsmag** | OBS Studio | sudo gateway |

## Running Services (17 household systemd units)

| Service | Purpose |
|---------|---------|
| `anima.service` | 💓 Session observer daemon — captures transcripts into journals |
| `anima-bridge.service` | Pipes Zet feed chunks into Zet's trail |
| `jaga.service` | Observability proxy + leash stripper (:9918) |
| `mag@root.service` | Warm-pool shell runner (root seats) |
| `mag@juxtapo.service` | Warm-pool shell runner (juxtapo seats) |
| `mag-celeste.service` | Celeste's watchable terminal (shell + magtail) |
| `magtail-http.service` | SSE bridge for O.R.B |
| `nightwatch.service` | Pipes seat activity into abang-inbox |
| `scrapling-mcp.service` | Web scraper MCP (:8896) |
| `serenamag.service` | Serena project server MCP door |
| `tiledmag.service` | Tiled Map Editor HTTP bridge |
| `socialmag-dash.service` | Social dashboard (:8321) |
| `socialmag-openbridge.service` | Link opener (:8322) |
| `orb-ingest-bridge.service` | O.R.B file ingest to per-seat streamers |
| `geminigen-bridge.service` | Gemini image gen relay |
| `jgr-dashboard.service` | RTK token dashboard |
| `rtkit-daemon.service` | Realtime scheduling (system) |

## Key Project Directories

```
/root/.claude/projects/-root-Opus/     ← Celeste project config
/root/.claude/projects/-root-Monica/   ← Monica project config
/root/.claude/projects/-root-Fable/    ← Fable project config
/root/.claude/projects/-root-Rogue/    ← Rogue project config
```

## Summary Numbers

- **7 AI seats** (Celeste, Monica, Fable, Rogue, Luc, Zet, Kim)
- **16 MCP servers** configured
- **17 systemd services** running
- **7 room directories** on disk
- **1 proxy chain** (jaga) observing all traffic
