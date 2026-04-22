# YouTube Intelligence Agent — Weekly Competitive Brief Automation

A Cowork-native agent that produces a branded, McKinsey-style PDF intelligence brief every Sunday, analyzing competitor YouTube activity in a niche real estate vertical. Delivered via Gmail draft for human review, archived automatically to Google Drive, and announced through a Google Calendar event so the operator knows the brief is ready to review and send.

> **Client context (anonymized).** A boutique cross-border real estate consultancy helping European investors (€100K–€1M capital band) evaluate Brazilian real estate opportunities. The founder is a sole content creator on YouTube, competing against better-resourced channels — the brief helps him see market moves fast and film the right video each week.

---

## The problem

The founder needed to:

- Monitor 10+ competitor YouTube channels every week without watching them manually
- Identify topic gaps where his positioning could cut through
- Score 10 film-able video ideas each week, ranked by opportunity
- Receive all of this in a format he could read on Sunday night and act on Monday morning

Done by hand, this is 4–6 hours of work per week. The agent does it in ~10 minutes of machine time plus ~30 seconds of human review.

---

## Architecture

```
Sunday 19:00 BRT
    │
    ▼
┌──────────────────────┐    ┌──────────────────────┐
│  Cowork scheduled    │    │  YouTube Data API v3 │
│  task wakes up       │───▶│  (10 channels, 7 d.) │
└──────────────────────┘    └──────────┬───────────┘
    │                                  │
    │                                  ▼
    │              ┌──────────────────────┐
    │              │ youtube-transcript-  │
    │              │ api  →  transcripts  │
    │              └──────────┬───────────┘
    │                         │
    │                         ▼
    │              ┌──────────────────────┐
    │              │ Claude analyzes:     │
    │              │  · topic coverage    │
    │              │  · gaps & saturation │
    │              │  · audience sentiment│
    │              │  · 10 scored ideas   │
    │              └──────────┬───────────┘
    │                         │
    │                         ▼
    │              ┌──────────────────────┐
    │              │ WeasyPrint: HTML →   │
    │              │ A4 PDF (14 pages)    │
    │              └──────────┬───────────┘
    │                         │
    │           ┌─────────────┴──────────────┐
    │           ▼                            ▼
    │  ┌─────────────────┐        ┌─────────────────┐
    │  │ Google Drive    │        │ Gmail MCP       │
    │  │ upload (archive)│        │ create_draft    │
    │  └────────┬────────┘        └────────┬────────┘
    │           └──────────┬────────────────┘
    │                      ▼
    │           ┌──────────────────────┐
    │           │ Google Calendar MCP  │
    │           │ event: "Brief ready" │
    │           └──────────┬───────────┘
    │                      ▼
    │           Operator reviews draft,
    │           sends it from Gmail
    ▼
Next Sunday
```

---

## Stack

| Layer              | Choice                                           | Why                                                                         |
| ------------------ | ------------------------------------------------ | --------------------------------------------------------------------------- |
| Orchestrator       | Claude in Cowork (scheduled task)                | Lives where the intelligence does; no separate infra to maintain            |
| Data collection    | YouTube Data API v3 (plain `urllib`, no SDK)     | Deterministic, quota-friendly, zero dependencies                            |
| Transcripts        | `youtube-transcript-api`                         | Fallback chain: manual → auto-generated → translated                        |
| Analysis           | Claude (same orchestrator)                       | Judgment-heavy step; cheap LLM calls are the right tool                     |
| PDF rendering      | WeasyPrint (HTML + CSS with `@page` rules)       | Print-quality typography, brand compliance, selectable text                 |
| Email              | Gmail MCP connector — `create_draft`             | Human-in-the-loop review gate (no auto-send from the agent)                 |
| File archive       | Google Drive MCP                                 | Stable URLs, searchable filenames, client can bookmark                      |
| Notification       | Google Calendar MCP — `create_event`             | A calendar event the operator can't miss — beats a silent inbox draft       |
| Secrets            | `.env` file                                      | Simple, appropriate scope                                                   |

---

## Repository structure

```
.
├── .env.example                    # placeholders for YOUTUBE_API_KEY, RECIPIENT_EMAIL
├── 01_system_prompt.md             # Claude's operating instructions (the methodology)
├── assets/
│   └── report_template.html        # canonical PDF design (McKinsey-style, brand palette)
└── agent/
    ├── RUN_PROMPT.md               # self-contained prompt the scheduled task executes
    ├── config/
    │   └── channels.json           # tiered competitor list (T0 own / T1 direct / T2 adjacent / T3 macro)
    ├── memory/
    │   └── history.json            # rolling 8-week memory + acted_on + suppressed_topics
    ├── scripts/
    │   ├── fetch_youtube_data.py   # YouTube Data API pull
    │   ├── fetch_transcripts.py    # transcript extraction, multi-language fallbacks
    │   ├── render_pdf.py           # WeasyPrint wrapper
    │   └── send_email.py           # SMTP fallback (unused once Gmail MCP is connected)
    └── output/
        └── <YYYY-Www>/             # per-week artifacts
            ├── raw.json
            ├── transcripts.json
            ├── brief.json
            ├── report.html
            ├── report.pdf
            └── draft_body.html
```

---

## Key design decisions

### 1 · Claude orchestrates, Python does the plumbing

Data collection, transcript extraction, PDF rendering, and the Gmail/Drive calls are deterministic — Python scripts own them. Claude's weekly job is the judgment-heavy part: ranking videos, identifying saturation vs. gap, writing a compelling "observation of the week," scoring 10 content opportunities on four dimensions.

I resisted the temptation to let Claude do everything (too expensive, too non-deterministic for API calls) or to script everything (no room for nuance or competitive insight). The split follows a simple rule: **deterministic → code, judgment → LLM.**

### 2 · Draft, don't send

Even with Gmail OAuth authorized, the agent creates a draft in the operator's Gmail and leaves it there. Three reasons:

- Early-run quality calibration — catches hallucinated metrics before the client sees them
- Respects the relationship (the brief goes from an operator to a founder — both real people, not a generic inbox)
- Opens the door to human-in-the-loop tweaks (tone, adding a note, removing a sensitive finding)

To make sure the draft isn't missed, the agent also creates a Google Calendar event titled "Weekly brief ready to review" for Monday morning, with the Drive link in the description. A draft alone is easy to overlook; a calendar event forces a touchpoint.

### 3 · Separate the methodology prompt from the execution prompt

`01_system_prompt.md` defines the methodology — scoring bands, output structure, memory schema, constraints. It changes rarely.

`agent/RUN_PROMPT.md` is the weekly execution spec — step-by-step orchestration, file paths, tool calls. It changes as the pipeline evolves.

This split keeps the methodology editable by non-engineers (the founder can refine scoring criteria directly) while keeping pipeline plumbing in engineer hands.

### 4 · Consulting-style PDF, not a digest email

The PDF is 14 pages, A4, navy/orange brand palette, numbered sections §01 through §08, full-bleed navy cover and closing pages. It's meant to feel like a weekly brief from a consulting firm, not a marketing newsletter.

The email is deliberately *not* a digest — it's a ~90-word teaser that creates curiosity and points to the Drive link. The visual craft lives in the PDF; the email is a door.

### 5 · Memory as a first-class artifact

`history.json` stores the last 8 weeks of briefs (older ones roll into an `archive[]` array), plus:

- `acted_on` — which ideas the founder actually filmed, tracked over time to improve future scoring
- `suppressed_topics` — clusters we've covered recently, deprioritized in the next week's idea ranking

This is what turns a series of independent briefs into an agent that learns from its own output.

---

## Weekly flow

1. **Sunday 19:02 local time** — scheduled task fires
2. Read system prompt, memory, channel list
3. Fetch last 7 days of videos + top comments from all monitored channels (YouTube Data API)
4. Extract transcripts (manual → auto-generated → translated)
5. Analyze: top 5 videos, topic coverage heatmap, audience sentiment per channel, 10 scored content opportunities, observation of the week
6. Render HTML → A4 PDF (14 pages, brand-compliant)
7. Upload the PDF to the Google Drive archive folder with filename `DD-MM-YYYY · <Theme>.pdf` (theme is derived from the week's §08 observation)
8. Create a Gmail draft with a short teaser body and the Drive link
9. Create a Google Calendar event for Monday morning — "Weekly brief ready to review" — with the Drive link in the description
10. Update memory (rolling 8-week window, suppressed topics, acted-on tracking)
11. Post a completion summary (week tag, PDF path, Drive URL, draft ID, calendar event ID, video counts, warnings)

---

## Safety & guardrails

- The agent is **prohibited from editing** the PDF template — design changes require human review
- `.env` values are never echoed in logs, emails, or notification channels
- Gmail is draft-only — the agent never sends mail directly
- Insufficient-signal weeks (fewer than 3 competitor videos) trigger a short "low-signal" draft instead of a fabricated brief
- Network failures abort gracefully and explain the cause in a draft, rather than silently retrying

---

## Limitations & tradeoffs

- **Sandbox network allowlist.** Early runs hit `403 Forbidden` on `googleapis.com`. Resolving it is a workspace-level config, not code.
- **No attachments in Gmail drafts.** The Gmail MCP connector exposes body/subject/recipients but no attachment field. Solved by routing the PDF through Drive and linking from the email — which arguably makes emails lighter anyway.
- **Cost.** Each weekly run consumes Claude tokens (analysis step) + YouTube API quota (~2–3% of the free daily quota) + local WeasyPrint CPU (free). Cost scales with channel count, not with brief complexity.
- **Single-persona focus.** The brief is calibrated for one founder's voice, thesis, and audience. Not a general-purpose tool.

---

## Roadmap

- [ ] Title-pattern A/B tracking — feed back into opportunity scoring
- [ ] Transcript-level quote mining for commenter sentiment (beyond comment counts)
- [ ] Second-language output (PT-BR) for local market briefs
- [ ] Slack notification when the draft lands, as an alternative to the calendar event

---

## Lessons learned

Building an agent that a non-engineer uses weekly taught me three things:

1. **The review gate matters more than automation speed.** My first instinct was to auto-send everything. The right call was to leave a 30-second review window — it saved the project when early runs had wrong data.

2. **Design is an agent constraint, not decoration.** The client cared visibly more about the PDF looking like a consulting brief than about any specific metric. Making the design template non-editable by the agent protected the thing the client cared about most.

3. **Iterate with screenshots, not specs.** Four of the five design changes came from screenshots the client annotated — "this chart: I don't understand what the numbers mean." Writing a perfect spec up front would have shipped the wrong thing; reading annotated screenshots shipped the right one.

---

_Built with Claude, Cowork scheduled tasks, YouTube Data API v3, WeasyPrint, and Gmail, Drive & Calendar MCP connectors._
