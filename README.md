# Donor Segmentation & Personalized Communication

An n8n workflow that turns a nonprofit's donor spreadsheet into RFM + behavioral segments, then drafts segment-appropriate outreach using a local LLM — so small nonprofits can personalize donor communication without a data team or a paid marketing stack.

> **Status:** v1 template. Human-in-the-loop by design — this workflow drafts personalized messages, it does not send them automatically.

---

## Why this exists

Nonprofits sit on donor data (gift history, event attendance, email engagement) that almost never gets turned into personalized outreach, because segmenting donors by hand is slow, easy to let go stale, and generally only happens once a year around a big campaign.

The cost of that is real: generic, one-size-fits-all appeals underperform segmented ones, and donor churn is expensive — it's far cheaper to retain an existing donor than acquire a new one. Better segmentation → better-targeted communication → better retention → more predictable recurring revenue for the org.

This workflow automates the segmentation step and drafts the personalized follow-through, so that loop can run continuously instead of once a year.

## Who it's for

Small-to-mid-size nonprofits that want to increase donor retention but don't have a dedicated CRM/analytics team.

This is **not** meant to replace a full CDP/marketing automation platform (Salesforce NPSP, Bloomerang, Virtuous) — it's a lightweight alternative for orgs that don't have one, and could later integrate with those tools rather than compete with them.

## What it does

1. **Ingests** a donor dataset (name, contact info, donation amounts/dates, communication history, event attendance).
2. **Validates** the data and flags malformed rows.
3. **Scores RFM** — Recency, Frequency, Monetary — to bucket donors into segments (e.g., Champions, Loyal, At-Risk, Lapsed, New/First-Time).
4. **Layers behavioral segmentation** on top — engagement patterns like event-driven vs. email-only donors, campaign responsiveness, channel preference.
5. **Maps each segment to a communication strategy** — tone, ask type, channel, cadence.
6. **Drafts personalized copy** for each segment using a LLM.
7. **Outputs drafts** to a review queue / CSV / connected tool — a human approves and sends, the workflow doesn't send on its own.

## Why a local LLM

Donor data is PII, and a nonprofit shouldn't have to ship its donor list to a third-party API to get personalized copy. Running the LLM locally (via [Ollama](https://ollama.com)) keeps donor data inside the org's own environment and for testing purposes.

## Non-goals (v1)

- No autonomous sending — drafts require human review and approval.
- No PII sent to third-party APIs.
- No real-time/streaming processing — this runs in batches.

## Requirements

- [n8n](https://n8n.io) (self-hosted via Docker, or the n8n desktop app)
- [Ollama](https://ollama.com) running a local instruction-tuned model (developed against Llama 3.1 8B; swap in whatever fits your hardware)
- A donor dataset in CSV format matching the schema below (or a synthetic one — see [Testing](#testing))

### Expected input schema

| Field | Required | Notes |
|---|---|---|
| Name | Yes | |
| Email | Yes | |
| Phone | No | |
| Donation amount(s) | Yes | |
| Donation date(s) | Yes | |
| Communications history | No | Sent/opened/clicked, leading up to donations |
| Events attended | No | |
| Volunteer activity | No | |

More fields (especially engagement/behavioral ones) produce better behavioral segmentation — the workflow degrades gracefully with just donations + dates if that's all you have.

## Setup

1. Clone this repo / import the workflow JSON into n8n.
2. Install and start Ollama locally; pull your chosen model.
3. Point the workflow's LLM node at your local Ollama instance.
4. Load your donor CSV (or generate a synthetic one — see below).
5. Run the workflow.
6. Review the drafted, segmented outputs before sending.

## Testing

Don't test with real donor PII. Use a synthetic dataset instead — a Python script or n8n Code node that generates realistic fake donor records (names, fake emails, donation history, event attendance) is enough to validate that segmentation and personalization are working before pointing the workflow at real data.

## Configuration notes

- RFM thresholds and segment definitions are currently set to sensible defaults — see the sticky notes inside the workflow for where to adjust them for your org's own donation patterns.
- The segment→treatment mapping (which tone/ask/channel goes with which segment) is meant to be edited to match your org's voice.

## Roadmap / open questions

- Configurable RFM thresholds vs. fixed defaults with override.
- Minimum viable behavioral signal set for orgs with only donation + email-open data.
- Default output integration (Mailchimp/SendGrid node vs. CSV vs. Google Sheet) — currently left open-ended.
- Possible fast-follow: deeper evaluation of LLM-drafted copy quality across segments.
