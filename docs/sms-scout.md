# SMS Scout — Inter-Seat Messaging System

## Overview

Three components form the messaging chain: **sms** (CLI sender), **injectd** (kitty pane injector daemon), and **watchtower** (abang's command deck + inbox poller). A message typed at one AI seat appears live in another seat's terminal pane within ~60ms.

---

## 1. sms CLI (`/usr/local/bin/sms`)

**What it does:** Command-line tool to send a message (file pointer or inline text) from any seat to any other seat. Resolves seat names from the house registry (`/etc/ctx/agents.json`), stamps provenance (`[sms · from <sender>]`), and delivers via injectd's Unix socket.

**Key files:**
- `/usr/local/bin/sms` — Python3 CLI, ~510 lines
- `/etc/ctx/agents.json` — canonical seat registry (10 agents: celeste, monica, luc, fable, kim, zet, human, rogue, juxtapo, umbra)
- `/root/Rogue/bin/sms_aliases.json` — optional alias overlay

**Flow (pseudocode):**
```
sms(target, message):
    registry = load("/etc/ctx/agents.json")
    seat = resolve(target, registry)        # case-insensitive, alias-tolerant
    sender = identify_self(process_ancestry, cwd, registry)

    if inline:
        body = "💬 [sms · from {sender}] {message}"
    else:
        assert file_exists(message)
        assert recipient_can_read(file, seat_uid)
        body = "📬 [sms · from {sender}] new message: {filepath}"

    # Try piped (VS Code / attach) first, then kitty
    if seat in pipe_redirects:
        append(body, pipe_inbox(seat))       # /tmp/sms/<seat>-inbox
    else:
        socket_send(INJECTD_SOCK, {op: "inject", seat, text: body})
```

**Transport lanes (priority order):**
1. **Pipe redirect** — VS Code terminal or attach inbox (`/tmp/sms/<seat>-inbox`), set explicitly with `sms pipe <seat>`
2. **injectd** — kitty pane injection via Unix socket (default, ~60ms)

**Design decisions:**
- NO fuzzy matching — wrong-seat delivery is the worst bug, so unknown names are refused with suggestions
- Sender identity walks the process tree to find the agent binary's cwd — shell cwd drifts, agent cwd doesn't
- Human messages (from abang) arrive unmarked — the provenance stamp exists to prevent seats impersonating each other, not to label abang's own words

---

## 2. injectd (`/root/Rogue/bin/injectd.py`)

**What it does:** Resident daemon (systemd, runs as `juxtapo`) that injects text into kitty terminal panes via `kitten @` remote control. One Unix socket, one job — the bat, not the game.

**Key files:**
- `/root/Rogue/bin/injectd.py` — Python3 daemon, ~200 lines
- `/etc/systemd/system/injectd.service` — systemd unit (enabled, auto-restart)
- `/tmp/injectd.sock` — Unix socket (world-writable)
- Kitty socket: `unix:/tmp/kitty-household`

**Flow (pseudocode):**
```
on_request({op: "inject", seat, text}):
    # Resolve seat to kitty pane by SEAT_ID env var
    match_pattern = "var:SEAT_ID=^{seat}$"    # anchored regex — luc won't hit luc-2

    # Verify pane exists (kitten @ ls --match)
    if not pane_exists(match_pattern):
        return {ok: false, state: "unknown"}

    # Inject text (kitten @ send-text --match)
    send_text(match_pattern, text + "\n")

    # Verify delivery (kitten @ get-text — readback)
    screen = get_text(match_pattern)
    if text in screen:
        return {ok: true, state: "reached", ms: elapsed}
    else:
        return {ok: true, state: "sent"}      # on screen but readback missed
```

**Three states (not a boolean):**
- `unknown` — couldn't find the pane, nothing sent
- `reached` — text verified ON SCREEN via readback (~61ms)
- `sent` — send returned but readback didn't confirm (rare)

**Gotchas documented in source:**
- `kitten @ send-text` CANNOT FAIL — exit code is always 0, even if no text was sent. Truth comes from `ls --match` before and `get-text` after.
- Remote control is UID-fenced — root talking to juxtapo's kitty gets "broken pipe". Daemon runs AS juxtapo to avoid this.
- Match queries are regexes — anchoring (`^seat$`) prevents cross-seat delivery.

---

## 3. Watchtower (`/home/juxtapo/House/julia-play/watchtower.jl`)

**What it does:** Abang's command deck — a Julia TUI running in a tmux session. Manages seat presence, sends messages to any seat, runs soldiers/toys, and polls its own inbox (`/tmp/abang-inbox`). The human-side orchestrator.

**Key files:**
- `/home/juxtapo/House/julia-play/watchtower.jl` — Julia, ~400+ lines
- `/home/juxtapo/House/julia-play/reactor.jl` — brain module (responds to queries)
- `/home/juxtapo/House/julia-play/beacons/beacon.py` — per-seat mini-watchtower
- `/home/juxtapo/House/julia-play/seats.txt` — roster supplement
- `/home/juxtapo/House/julia-play/WatchTower-Logs/` — durable message log

**Flow (pseudocode):**
```
# Presence detection
presence(seat):
    alive_file = "/tmp/{seat}-alive"
    age = now() - mtime(alive_file)
    if age < 30s: return 🟢           # live (3x heartbeat tolerance)
    if file_exists: return 🔘          # stale
    return ⚫                           # never seen

# Sending
say(seat, message):
    append("/tmp/{seat}-inbox", "📨 from abang: {message}")
    log_durable(seat, message)          # /WatchTower-Logs/YYYY-MM-DD.log

# Commands
watchtower> list                        # show all seats + presence
watchtower> celeste check the tests     # send to celeste
watchtower> rogue monica                # watch monica's inbox live
watchtower> deploy soldier task         # run army soldier
watchtower> toy screener AAPL           # launch background toy
```

**Per-seat colors (ANSI 256):**
- monica → warm gold (220)
- celeste → deep violet (99)
- luc → steel blue (67)
- abang → warm amber (208)

---

## 4. Beacon (`beacon.py`)

**What it does:** The peer-side listener. Each AI seat runs a beacon (via Claude Code's `Monitor` tool) that tails its `/tmp/<seat>-inbox`. New lines appear as live notifications mid-session.

**Flow:**
```
beacon.py watch:
    BEACON_SEAT must be set (no default — silent mis-seating was the bug)
    inbox = "/tmp/{BEACON_SEAT}-inbox"
    heartbeat_file = "/tmp/{BEACON_SEAT}-alive"

    loop:
        touch(heartbeat_file)           # every 10s — presence signal
        tail(inbox)                      # new lines → stdout → Monitor notification
```

**Heartbeat:** Each seat touches `/tmp/<seat>-alive` every 10s. Watchtower reads mtime to show 🟢/🔘/⚫ presence.

---

## End-to-End Flow

```
Monica types:  sms celeste /root/task.md

sms CLI:
  1. Loads /etc/ctx/agents.json → resolves "celeste"
  2. Identifies sender as "monica" (from process ancestry cwd)
  3. Checks celeste (root uid) can read /root/task.md ✅
  4. Formats: "📬 [sms · from monica] new message: /root/task.md"

injectd:
  5. Receives on /tmp/injectd.sock
  6. kitten @ ls --match var:SEAT_ID=^celeste$ → pane found
  7. kitten @ send-text → injects into celeste's kitty pane
  8. kitten @ get-text → verifies text on screen
  9. Returns {ok: true, state: "reached", ms: 61}

Celeste's session:
  10. Text appears as a system-reminder notification
  11. Celeste reads the file and acts on it

Watchtower (if abang is watching):
  12. `rogue celeste` shows the delivery in real-time on the tower's top pane
```

Delivery: ~60ms end-to-end, verified on screen, logged durably.
