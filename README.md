# YouTube Intelligence Agent — Weekly Competitive Brief Automation

This agent replaced 4–6 hours of weekly manual competitor research with an automated, consulting-grade PDF brief delivered every Sunday. After adoption, the client's first data-driven video reached 8× his average view count and the channel grew 70% in new subscribers.

> **Client context (anonymized).** A consultancy helping European investors evaluate real estate and investment opportunities in Brazil. The founder runs the company's YouTube channel as a strategic growth lever — but needed a way to track what competitors were publishing, spot topic gaps, and decide what to film next, without spending half a Sunday doing it manually.

**What the agent does.** Monitors 10+ competitor YouTube channels weekly — videos, transcripts, and top comments — then scores 10 filmable content opportunities ranked by topic gap, audience demand, and competitive saturation. Delivered as a branded 14-page PDF via Gmail draft, archived to Google Drive, with a Calendar event so the brief never goes unread.

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

| Layer              | Choice                                                              | Why                                                                   |
| ------------------ | -------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| Orchestrator       | Claude's Cowork runtime (scheduled tasks, sandboxed execution)       | Runs on schedule with no separate infrastructure to maintain           |
| Data collection    | YouTube Data API v3, called directly with `urllib` (no SDK)          | Deterministic, quota-friendly, zero extra dependencies                 |
| Transcripts        | `youtube-transcript-api`                                              | Fallback chain: manual → auto-generated → translated captions          |
| Analysis           | Claude — runs the analysis step (topic gaps, scoring, competitive insight) | Judgment-heavy work; a cheap LLM call is the right tool for it    |
| PDF rendering      | WeasyPrint — converts HTML/CSS into print-quality A4 PDFs             | Real typography, brand compliance, and selectable text                 |
| Email              | Gmail integration via MCP — creates a draft, never sends directly     | Keeps a human in the loop before anything reaches the client           |
| File archive       | Google Drive integration via MCP                                      | Stable URLs and searchable filenames the client can bookmark           |
| Notification       | Google Calendar integration via MCP — creates an event                | A calendar event is harder to miss than a silent inbox draft           |
| Secrets            | `.env` file                                                            | Simple, and the right scope for a single-operator agent                |

---

## Repository structure

The methodology (`01_system_prompt.md`) is kept separate from the weekly execution spec (`agent/RUN_PROMPT.md`), and both are separate from the deterministic Python scripts under `agent/scripts/` that fetch data, extract transcripts, and render the PDF. Each week's raw data, transcripts, brief, and rendered PDF are saved under `agent/output/<week>/`, and the brand template lives at `assets/report_template.html`.

---

## Design tradeoffs

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

`01_system_prompt.md` defines the methodology: scoring bands, output structure, memory schema, constraints. It changes rarely.

`agent/RUN_PROMPT.md` is the weekly execution spec: step-by-step orchestration, file paths, tool calls. It changes as the pipeline evolves.

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

## Lessons learned

Building an agent that a non-engineer relies on every week forced three judgment calls — each one could have gone the wrong way:

1. **Review gate over automation speed.** My first instinct was to auto-send everything. I built in a 30-second human review window instead — and it saved the project the first time an early run pulled wrong data. Automation should earn the right to skip a checkpoint, not start there.

2. **Design is a constraint, not decoration.** Every round of client feedback was about how the brief looked — layout, chart clarity, hierarchy — never about the scoring logic underneath. Making the PDF template non-editable by the agent protected exactly the layer that got reviewed.

3. **Iterate from screenshots, not specs.** Four of five design changes came from screenshots the client annotated by hand — "this chart: I don't understand what the numbers mean." A perfect spec written up front would have shipped the wrong thing; reading real, annotated confusion shipped the right one.

---

## What this demonstrates

- **Multi-API integration** — YouTube Data API, transcript extraction, and Gmail/Drive/Calendar via MCP, all coordinated in a single scheduled pipeline
- **Prompt engineering for competitive analysis** — turning raw video and transcript data into ranked, scored insight instead of a generic summary
- **Human-in-the-loop guardrail design** — a draft-only email flow and a non-editable design template, both deliberate limits on what the agent is allowed to do

---

## Safety & guardrails

- PDF template is off-limits to the agent — design changes require human review
- `.env` values are never echoed in logs, emails, or notifications
- Gmail is draft-only — the agent never sends mail directly
- Weeks with fewer than 3 competitor videos get a short "low-signal" draft instead of a fabricated brief
- Network failures abort gracefully, with the cause explained in the draft rather than a silent retry

---

## Limitations & tradeoffs

- **No attachments in Gmail drafts.** The Gmail MCP connector exposes body/subject/recipients but no attachment field. Solved by routing the PDF through Drive and linking from the email, which arguably makes emails lighter anyway.
- **Cost.** Each weekly run consumes Claude tokens (analysis step), YouTube API quota (~2–3% of the free daily quota), and local WeasyPrint CPU (free). Cost scales with channel count, not with brief complexity.
- **Single-persona focus.** The brief is calibrated for one founder's voice, thesis, and audience. Not a general-purpose tool.

---

## Roadmap

- [ ] Title-pattern A/B tracking — feed back into opportunity scoring
- [ ] Transcript-level quote mining for commenter sentiment (beyond comment counts)
- [ ] Second-language output (PT-BR) for local market briefs
- [ ] Slack notification when the draft lands, as an alternative to the calendar event

---

_Built with Claude, Cowork scheduled tasks, YouTube Data API v3, WeasyPrint, and Gmail, Drive & Calendar MCP connectors._
