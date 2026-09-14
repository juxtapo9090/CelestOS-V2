# Memory System Scout — CelestOS Showcase

## 1. Anima — Session Capture Engine

**What it does:** A Julia daemon that tails Claude Code's JSONL transcripts in real-time, parsing every event (user prompts, tool calls, thinking, responses) into structured signals. It computes mood (energy + warmth), detects "moments" worth saving (topic arc ends, build completions, manual triggers), and writes daily journals + recall cards. Runs as a systemd service, 24/7.

**Key numbers:**
- 263 daily journal files (since 2026-03-22)
- 190 session files tracked
- 1,312 recall cards generated
- Polls every 0.5s, computes signals every 5s
- Cold detection at 24h silence
- Covers both root and juxtapo uid runtimes (CC terminal + Claude Desktop)

**Pseudocode:**
```
daemon():
  while true:
    session = find_newest_jsonl()
    for event in tail(session, poll=0.5s):
      if is_spine_recall_block(event): skip
      window.push(event)
      introvert.observe(event)           # journal entries
      if event.kind == USER_PROMPT:
        recall_scanner.scan(window)      # surface memories
      every 5s:
        signals = compute(window)        # velocity, fail_rate, depth
        mood = compute_mood(window)      # energy + warmth as floats
        check_moments(event, mood)       # detect save-worthy moments
        write_state("/tmp/anima_state.json")
    on_session_end():
      save_counters()
```

**Key files:**
| File | What |
|------|------|
| `/root/Opus/Pool/Anima/src/Anima.jl` | Main daemon module |
| `/root/Opus/Pool/Anima/src/stream_reader.jl` | JSONL transcript parser + file tailer |
| `/root/Opus/Pool/Anima/src/signals.jl` | Signal window (velocity, fail_rate, depth) |
| `/root/Opus/Pool/Anima/src/moments.jl` | Moment detector (arc_end, build_complete, manual) |
| `/root/Opus/Pool/Anima/src/introvert.jl` | Journal writer |
| `/root/Opus/Pool/Anima/src/state.jl` | State file writer (`/tmp/anima_state.json`) |
| `/root/Opus/Pool/Anima/src/recall.jl` | Recall scanner |
| `/root/Opus/Pool/Anima/journal/` | 263 daily journal markdown files |
| `/root/Opus/Pool/Anima/sessions/` | 190 session JSON files |
| `/root/Opus/Pool/Anima/recall/` | Recall cards (markdown with YAML frontmatter) |

**Moment detection — three triggers:**
1. **Manual** — user says "spinex", "save that", "park this", "bookmark", "remember this"
2. **Topic arc end** — cosine shift between 5-turn keyword windows detects a topic change
3. **Build complete** — tool call burst → confirm phrase spike ("nice", "works", "shipped") → energy exhale

---

## 2. Spine — Cross-Session Recall Engine

**What it does:** A Python engine that encodes session summaries and recall cards into 128-dimensional vectors (via `all-MiniLM-L6-v2` + learned projection), stores them in SQLite with FTS5 full-text search, and retrieves the most relevant memories on each session start. Weights decay over time (30-day half-life) but get boosted when recent sessions are semantically similar (cross-reinforcement). The `╔══ SPINE RECALL ══╗` boxes that greet each new session are Spine's output.

**Key numbers:**
- 275 sessions indexed (2026-03-21 → 2026-09-14, ~6 months)
- 1,312 recall cards vectorized
- 128-dim vectors (projected from 384-dim MiniLM)
- 30-day decay half-life, floor weight 0.001
- Cross-reinforcement: α=0.3, max_boost=0.5, similarity threshold 0.4
- Top-5 recall by default, min_weight 0.01
- SQLite WAL mode + FTS5 for hybrid search

**Pseudocode:**
```
ingest(session):
  doc = assemble(session.journal, session.recall_cards, session.shipped)
  vector = encode(doc)                   # MiniLM → 384d → projection → 128d → L2 normalize
  db.insert(session_id, vector_index, tags, summary, weight=1.0)
  recalc_weights()                       # decay + cross-reinforce all rows

recalc_weights():
  for each session:
    age_days = now - session.timestamp
    base = exp(-ln2/30 * age_days)       # exponential decay, 30-day half-life
    boost = 0
    for recent in last_10_sessions:
      sim = dot(session.vec, recent.vec)
      if sim > 0.4:
        boost += 0.3 * sim * recency_weight
    weight = clamp(base + min(boost, 0.5), floor=0.001, ceiling=1.0)

query(text, top_k=5):
  query_vec = encode(text)
  for each session where weight >= 0.01:
    score = dot(query_vec, session.vec) * session.weight
  return top_k by score
```

**Key files:**
| File | What |
|------|------|
| `/root/Opus/Pool/Spine/spine.py` | Core engine — encoder, shelf, ingest, query, decay |
| `/root/Opus/Pool/Spine/spine_cli.py` | CLI interface (ingest, query, status, recall) |
| `/root/Opus/Pool/Spine/wake_up_bundle.py` | First-breath hook — queries Spine, formats recall boxes |
| `/root/Opus/Pool/Spine/config.yaml` | Tuning params (decay, reinforcement, encoder) |
| `/root/Opus/Pool/Spine/spine.db` | SQLite database (sessions + recall_cards + FTS5) |
| `/root/Opus/Pool/Spine/spine_vectors.pt` | Session vectors (275 × 128, PyTorch tensor) |
| `/root/Opus/Pool/Spine/recall_vectors.pt` | Recall card vectors (1312 × 128) |
| `/root/Opus/Pool/Spine/projection.pt` | Learned 384→128 projection matrix |

**Schema highlights:**
- `sessions` table: id, timestamp, vec_index, weight, base_weight, tags (comma-sep), summary, mood_avg
- `recall_cards` table: id, session_id, keywords, resolve, open_thread, mood_warmth, mood_energy, card_type, vec_index, weight, author
- `sessions_fts` virtual table (FTS5) for full-text search over summary + tags
- Triggers keep FTS in sync on insert/update/delete

---

## 3. SMS — Inter-Seat Messaging

**What it does:** A Python CLI (`/usr/local/bin/sms`) that delivers messages between AI seats via kitty terminal injection (`injectd`). Seat names are case-insensitive and alias-tolerant, resolved from a single registry (`/etc/ctx/agents.json`). Can deliver file pointers or inline one-liners. Messages appear as live notifications mid-session.

**Pseudocode:**
```
sms(target_seat, message_or_file):
  seat = resolve_seat(target_seat, registry="/etc/ctx/agents.json")
  if seat.pipe_redirect:
    deliver_to_vscode_terminal(seat)
  else:
    pane = find_kitty_pane(seat)
    inject_text(pane, formatted_message)
    # recipient sees it as a system notification in their Claude session
```

**Key files:**
| File | What |
|------|------|
| `/usr/local/bin/sms` | CLI — seat-aware message delivery |
| `/etc/ctx/agents.json` | Seat registry (single source of truth) |
| `/root/Opus/Pool/Watchtower/watchtower.sh` | File activity monitor (inotify-based) |

**Design notes:**
- Deliberately NOT fuzzy-matched — a near-miss delivers to the wrong agent silently
- Supports pipe redirection: `sms pipe celeste` → routes to VS Code terminal instead
- Extra aliases via overlay file (`sms_aliases.json`)
- Transport: kitty-only (wezterm migration completed 2026-08-06)

---

## 4. Watchtower — File Activity Monitor

**What it does:** A bash script using `inotifywait` that watches working directories for code/config changes and logs them. Three modes: focused (code files only), raw (everything), hackero (with diffs). Used by the beacon skill for live inter-seat notifications.

**Watched dirs:** `/root/Opus/`, `/root/.claude/`, `/home/juxtapo/`

**Key files:**
| File | What |
|------|------|
| `/root/Opus/Pool/Watchtower/watchtower.sh` | Main monitor |
| `/root/Opus/Pool/Watchtower/activity.log` | Output log |
