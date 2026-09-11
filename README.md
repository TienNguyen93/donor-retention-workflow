# Donor Segmentation & Personalized Communication

An n8n workflow that turns a nonprofit's donor spreadsheet into RFM + behavioral segments, then drafts segment-appropriate outreach using a local LLM — so small nonprofits can personalize donor communication without a paid marketing stack, cutting acknowledgment turnaround time by **50%** while maintaining on-premise data privacy.

### Link to workflow: 

https://n8n.io/workflows/19358-segment-donors-and-generate-personalized-messages-with-google-sheets-and-ollama/

> **Status:** v1 template. Human-in-the-loop by design — this workflow drafts personalized messages, it does not send them automatically.

<img width="1426" height="322" alt="donor-workflow" src="https://github.com/user-attachments/assets/ef00cd83-b458-49c3-8795-3e19900963f4" />

---

## Problem Statement

Nonprofit fundraising teams often face massive operational overhead when processing recurring donations. Manually reviewing donor histories, calculating gift frequencies, and drafting individual thank-you notes takes dozens of hours each month, delaying follow-ups and reducing donor retention.

They sit on donor data (gift history, event attendance, email engagement) that almost never gets turned into personalized outreach, because segmenting donors by hand is slow, easy to let go stale, and generally only happens once a year around a big campaign. On average, staff spent **4-8 hours** every week manually cross-referencing intake spreadsheets and typing custom emails. High-value donors often received generic auto-replies while staff worked through the backlogs, resulting in missed stewardship opportunities.

The cost of that is real: generic, one-size-fits-all appeals underperform segmented ones, and donor churn is expensive — it's far cheaper to retain an existing donor than acquire a new one. Better segmentation → better-targeted communication → better retention → more predictable recurring revenue for the org.

## Who it's for

Small-to-mid-size nonprofits that want to increase donor retention but don't have a dedicated CRM/analytics team.

This is **not** meant to replace existing donor management and segmentation systems (e.g, Salesforce NPSP, Bloomerang, etc) — it's a lightweight alternative for organizations that don't have one, and could later integrate with those tools rather than compete with them.

## What it does

1. **Runs** when you manually execute the workflow to start processing donor data from a Google Sheets document.
2. **Validates** all donor rows, normalizes key fields (email, donation amount, donation date), deduplicates exact duplicate rows, and flags each row as valid or invalid.
3. **Appends** or updates invalid rows in a separate “Invalid Rows” sheet for follow-up.
4. For valid donors, 
    * **Scores RFM** — Recency, Frequency, Monetary — to bucket donors into segments (e.g., Champions, Loyal, At-Risk, Lapsed, New/First-Time), assigns an RFM segment label via quintile scoring

      **Filtering & Aggregation**
         - **Lookback Window**: Scans all donation records within a set period (default: 730 days / 2 years). Donations older than the cutoff are excluded from RFM scoring.
         - **Donor Aggregation**: Groups valid transactions by donor email to construct three raw values per donor:
           - **Recency**: Number of days between today and their most recent donation.
           - **Frequency**: Total number of donations made within the lookback window.
           - **Monetary**: Sum total dollar amount donated within the lookback window.
         - Relative Quintile Scoring (1 to 5)
           - Each donor's raw metrics are compared against the entire donor pool to compute a percentile rank.
           - Ranks are mapped to a 1–5 score using quintiles:
             - **Frequency & Monetary**: Higher values yield higher scores (1 = lowest, 5 = highest).
             - **Recency**: Inverted so that fewer days since last gift yields a higher score (1 = least recent/worst, 5 = most recent/best).
    * **Layers behavioral segmentation** on top — engagement patterns like event-driven vs. email-only donors, campaign responsiveness, channel preference.
       - Based on individual $R$, $F$, and $M$ scores (or their sum), donors are assigned one of six segments
         - **Champions**: High scores across all three areas ($R \ge 4$, $F \ge 4$, $M \ge 4$).
         - **New / First-Time**: Recent donors who have only made a single gift ($R \ge 4$, $F = 1$).
         - **At-Risk**: Formerly frequent donors who haven't given recently ($R \le 2$, $F \ge 3$).
         - **Lapsed**: Inactive donors with low recent activity and low overall gift counts ($R \le 2$, $F \le 2$).
         - **Loyal**: Highly engaged donors with a combined score total $\ge 9$ that didn't meet "Champion" thresholds.
         - **Standard**: Donors who do not fit into any of the above conditional categories.
5. **Treatment mapping**
   
   Combines rfm_segment + behavioral_tag into a treatment spec for prompt constructing
   
   Lookup Matrix (TREATMENT_TABLE)

    | Lookup Key (Segment Tag) | Tone | Ask Type | Channel | Cadence |
    | :--- | :--- | :--- | :--- | :--- |
    | Champions - event_driven | Warm, celebratory | Invite to next event / ambassador ask | Email + Event Invite | Quarterly |
    | Champions  - email_only | Warm, personal thank-you | Impact story, soft upsell | Email | Quarterly |
    | Champions - mixed_channel | Warm, personal | Impact story + event invite | Email + Event Invite | Quarterly |
    | Loyal - email_only | Appreciative | Renewal reminder | Email | Semi-annual |
    | At-Risk - campaign_responsive | Concerned, re-engaging | Reactivation appeal (campaign-tied) | Email | Immediate |
    | At-Risk - steady_giver | Gentle check-in | Impact update, no hard ask | Email | Immediate |
    | Lapsed - event_driven | Nostalgic, inviting | Invite back to an event | Email | Immediate |
    | New - First-Time - mixed_channel | Welcoming | Onboarding / thank-you, no ask | Email | Within 1 week |

7. **LLM Prompt Construction**

Prompt Structure Template
```
Write a short, warm donor communication. Donor first name: <firstName>
Segment: <rfm_segment> / <behavioral_tag>
Tone: <treatment.tone>
Ask type: <treatment.ask_type>
Last donation amount: <_donation_amount>
Last donation date: <_donation_date>
Keep it under 150 words. Do not invent facts not given above. Do not include placeholder tags like curly braces in the output.
```

Example Generated Prompt
```
Dear George,
We're grateful for your continued support, especially your recent gift of $383.52 on February 2nd.
Your investment makes a tangible difference in the lives of those we serve.
One story that stands out is of a family who benefited from our programs.
They were struggling to make ends meet, but with our help, they were able to access essential resources and start rebuilding their lives.
Stories like these remind us of the impact we can have together.
We're hosting an event soon, where you'll have the opportunity to meet some of the individuals we've helped and hear more about our work.
It's a chance to connect with others who share your passion for creating positive change. We'd love for you to join us.

Sincerely,

[Your Name]
```

9.  **Outputs drafts** to initial donor Google Sheet for review. A human approves and sends, the workflow doesn't send on its own.
    * Sends each prompt to Ollama (Llama 3.1) through LLM Chain (deterministic, ideal for specific task) to draft a short donor message, validates the draft, and falls back to a segment-based template when the model output fails checks.
    * Updates the Google Sheets “Donor Data” rows under “Personalized Message” field for HTIL

## Why a local LLM

Donor data is PII, and a nonprofit shouldn't have to ship its donor list to a third-party API to get personalized copy. Running the LLM locally (via [Ollama](https://ollama.com)) keeps donor data inside the org's own environment and for testing purposes.

## Non-goals (v1)

- No autonomous sending — drafts require human review and approval.
- No PII sent to third-party APIs.
- No real-time/streaming processing — this runs in batches.

## Requirements

- [n8n](https://n8n.io) (self-hosted via Docker, or the n8n desktop app)
- [Ollama](https://ollama.com) running a local instruction-tuned model (developed against Llama 3.1 8B; swap in whatever fits your hardware)
- Google Drive API and Google Sheet API enablement
- A Google Sheet donor dataset matching the schema below (or a synthetic one — see [Testing](#testing))

### Expected input schema

| Field | Required | Notes |
|---|---|---|
| Name | Yes | |
| Email | Yes | |
| Phone | Yes | |
| Donation amount(s) | Yes | |
| Donation date(s) | Yes | |
| Communications history | Yes | Sent/opened/clicked, leading up to donations |
| Events attended | Yes | |
| Volunteer activity | Yes | |
| Personalized Message | Yes | |

More fields (especially engagement/behavioral ones) produce better behavioral segmentation — the workflow degrades gracefully with just donations + dates if that's all you have.

## Setup

1. Connect Google Sheets OAuth credentials and set the correct spreadsheet ID and sheet names for “Donor Data” and “Invalid Rows.”
2. Ensure the “Donor Data” sheet includes columns specified in the input schema for the write-back.
3. Connect an Ollama credential, confirm the **llama3.1:latest** model is available on your Ollama instance, and adjust model options (context, temperature, token limit) as needed.
4. Customize the segment-to-treatment table and fallback templates in the JavaScript steps to match your organization’s messaging guidelines.

## Testing

Don't test with real donor PII. Use a synthetic dataset instead — a Python script or n8n Code node that generates realistic fake donor records (names, fake emails, donation history, event attendance) is enough to validate that segmentation and personalization are working before pointing the workflow at real data.

## Configuration notes

- RFM thresholds and segment definitions are currently set to sensible defaults
- The segment→treatment mapping (which tone/ask/channel goes with which segment) is meant to be edited to match your org's voice.

## Roadmap / open questions

- Configurable RFM thresholds vs. fixed defaults with override.
- Minimum viable behavioral signal set for orgs with only donation + email-open data.
- Modify default output integration, currently in Google Sheet.
- Possible fast-follow: deeper evaluation of LLM-drafted copy quality across segments.
