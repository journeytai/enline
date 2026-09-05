# enline — Implementation Plan

## Project
LINE chat summarization and digest system. Daily/periodic English digests of LINE group chats with translation, summarization, mention detection, attachment processing, and a TailScale-backed web UI.

- **GitHub**: https://github.com/journeytai/enline
- **Linear**: https://linear.app/jatj/project/enline-2b67fb99eae8
- **Team**: Apaise Software Factory (ASF)

---

## Architecture

```
LINE Desktop App
    │
    ▼ (AppleScript + cliclick)
line-desktop-skill (enline/line-desktop-skill/)
    │
    ▼ (raw messages)
scripts/digest.py (cron job)
    │
    ├─► omniroute API → translate + summarize
    │
    ├─► Apple Vision OCR → attachment text
    │
    ├─► config.json → filter muted groups, detect mentions
    │
    ├─► Discord DM (rich embed, push)
    │
    └─► enline/data/digests/ → web UI (static HTML)
                              │
                              ▼
                         tailscale serve → TailScale URL
```

---

## Resolved Decisions

| Ticket | Decision |
|--------|----------|
| ASF-180 | LINE data access via `line-desktop-skill` (AppleScript + cliclick) |
| ASF-181 | Hermes omniroute (cloud API) for translation and summarization |
| ASF-182 | Discord DM push + TailScale web UI for detail |
| ASF-183 | Static HTML/CSS/JS served via `python3 -m http.server` + `tailscale serve` |
| ASF-184 | Clipboard copy for text, screenshot + Apple Vision OCR for attachments |
| ASF-185 | Hermes cronjob tool, 4 daily runs (8am, 10am, 2pm, 6pm) |
| ASF-186 | JSON config + CLI/web UI for mute, keyword filters, mention detection |

---

## Phase 1: Foundation (Week 1)

### 1.1 Set up environment
- [ ] Grant Accessibility permission to Terminal (System Settings → Privacy & Security → Accessibility)
- [ ] Confirm LINE desktop app is installed and logged in
- [ ] Confirm `cliclick` is installed (`brew install cliclick` — already done)
- [ ] Confirm `ollama` is running and `qwen3.5:4b` is available
- [ ] Set up Discord bot in Hermes (if not already done)

### 1.2 Test LINE reader
- [ ] Run `line-desktop-skill` manually against a test group
- [ ] Verify message extraction works (scroll, copy, parse)
- [ ] Identify message format: timestamp, sender name, text, attachment indicators
- [ ] Handle edge cases: E2EE chats, deleted messages, long scroll depths

### 1.3 Build config system
- [ ] Create `enline/config.json` with default structure
- [ ] Create `enline/scripts/config.py` CLI for mute/unmute/list
- [ ] Test: mute a group, verify it's excluded from digest

### 1.4 Build translation pipeline
- [ ] Create `enline/scripts/translate.py` — takes raw messages, outputs JSON
- [ ] Test with sample Chinese messages
- [ ] Verify omniroute output quality (translation accuracy, summary conciseness)

---

## Phase 2: Core Pipeline (Week 2)

### 2.1 Build digest generator
- [ ] Create `enline/scripts/digest.py` — main pipeline:
  1. Read LINE messages since last run (from `enline/data/state.json`)
  2. Filter out user's own messages
  3. Filter out muted groups and keyword-filtered messages
  4. Translate + summarize via omniroute API
  5. Detect mentions (user's LINE name + aliases)
  6. Format as Discord embed JSON
  7. Save digest to `enline/data/digests/YYYY-MM-DD_HHMM.json`
  8. Update `enline/data/state.json` with last_run timestamp

### 2.2 Build web UI
- [ ] Create `enline/web/index.html` — static page with:
  - Group cards (collapsible)
  - Message list per group (sender, timestamp, original + translation)
  - Mention highlighting (red border on mentioned messages)
  - Attachment thumbnails
  - Mute toggle per group
  - Search box
  - Filter by date/group
- [ ] Create `enline/web/style.css` — dark theme, responsive
- [ ] Create `enline/web/app.js` — load digest JSON, render, handle interactions
- [ ] Test locally: `python3 -m http.server 8080` in `enline/web/`

### 2.3 Test end-to-end
- [ ] Run digest manually against a small set of groups
- [ ] Verify Discord embed delivery
- [ ] Verify web UI rendering
- [ ] Check attachment processing (screenshot + OCR)

---

## Phase 3: Scheduling & Delivery (Week 3)

### 3.1 Set up cron jobs
- [ ] Create Hermes cron jobs:
  - Morning: `0 8 * * *` → run `scripts/digest.py`
  - Mid-morning: `0 10 * * *`
  - Mid-afternoon: `0 14 * * *`
  - Evening: `0 18 * * *` (optional)
- [ ] Configure delivery to Discord DM
- [ ] Test one cron run manually

### 3.2 Set up TailScale hosting
- [ ] Run `tailscale serve --bg 8080` to expose web UI
- [ ] Verify access via TailScale URL
- [ ] Set up auto-start (launchd plist or Hermes cron)

### 3.3 Polish
- [ ] Add error handling (LINE app not running, Ollama unavailable, etc.)
- [ ] Add logging to `enline/data/logs/`
- [ ] Add digest history view (past digests in web UI)
- [ ] Add settings page in web UI (mute groups, edit keywords, change schedule)

---

## Phase 4: Production (Week 4)

### 4.1 Hardening
- [ ] Handle LINE app updates (UI changes break AppleScript)
- [ ] Add fallback screenshot+OCR path (line-desktop-mcp approach)
- [ ] Add rate limiting (don't overwhelm LINE server)
- [ ] Add attachment size limits and cleanup
- [ ] Security review: ensure no credentials in repo, data stays local

### 4.2 Monitoring
- [ ] Add health check (last run time, success/failure status)
- [ ] Add notification on failure (Discord DM alert)
- [ ] Add digest statistics (messages per group, top senders, etc.)

### 4.3 Documentation
- [ ] README with setup instructions
- [ ] Config reference
- [ ] Troubleshooting guide

---

## File Structure

```
enline/
├── PLAN.md
├── config.json
├── line-desktop-skill/       (cloned from kaosensei)
├── scripts/
│   ├── digest.py             (main pipeline)
│   ├── translate.py          (translation + summarization)
│   ├── config.py             (CLI for config management)
│   └── line_reader.py        (wraps line-desktop-skill)
├── web/
│   ├── index.html
│   ├── style.css
│   └── app.js
├── data/
│   ├── digests/              (generated digest JSONs)
│   ├── attachments/          (processed attachments)
│   ├── state.json            (last_run, stats)
│   └── logs/                 (run logs)
└── .gitignore
```

---

## Prerequisites

- [ ] macOS with LINE desktop app installed and logged in
- [ ] Accessibility permission granted to Terminal
- [ ] `cliclick` installed (`brew install cliclick`)
- [ ] `ollama` running with `qwen3.5:4b` model
- [ ] Discord bot configured in Hermes
- [ ] TailScale installed and running
- [ ] GitHub repo: https://github.com/journeytai/enline
- [ ] Linear project: enline (ASF team)

---

## Open Questions

1. What is Joshua's LINE display name for mention detection?
2. Which groups should be muted by default?
3. What keywords indicate ads/sales in his groups?
4. Should evening digest be enabled?
5. Any specific attachment types to prioritize (PDFs, images, etc.)?
