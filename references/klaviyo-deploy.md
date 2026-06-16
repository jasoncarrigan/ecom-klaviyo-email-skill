# Klaviyo: reading performance + deploying the draft

Two jobs in this skill touch Klaviyo: **reading** account performance (Phase 4, read-only) and **writing** the draft
campaign (Phase 9). **Prefer a connected Klaviyo MCP/connector** if one is present (no raw key handling, simpler
calls). Otherwise use the REST API with `KLAVIYO_API_KEY`.

Detect a connector by looking for tools whose names contain `klaviyo` (e.g. `get_campaigns`, `get_campaign_report`,
`create_email_template`, `create_campaign`, `get_lists`, `upload_image_from_url`). If present, use them; else use REST.

Never call anything that sends or schedules a campaign.

---

## Phase 4 — reading performance (read-only)

Goal: benchmarks to beat + what's working. Pull, don't modify.

**Via connector:** `get_campaigns` (recent, email channel) → for the top/recent ones, `get_campaign_report` for open
rate, click rate, and conversion/revenue per recipient. `get_lists` and `get_segments` for audience names + sizes.
`query_metric_aggregates` / `get_metrics` can give placed-order and revenue metrics if reports don't.

**Via REST:** base `https://a.klaviyo.com/api/`, headers below.
- `GET /campaigns/?filter=equals(messages.channel,'email')` — recent campaigns.
- `GET /campaign-values-reports/` (POST a report query) — open/click/conversion stats per campaign.
- `GET /lists/`, `GET /segments/` — audiences and sizes.

Summarize: average open/click/revenue-per-recipient, top performers and their shared traits (subject style, send
day/time, offers), and the best send window. Save to `brand/klaviyo-benchmarks.json` (see `caching.md`).

---

## Phase 9 — creating the draft

### Via connector (preferred)

1. **Host images.** `upload_image_from_url` for each generated/local image → hosted URLs. Put them in the image blocks
   before assembling the definition.
2. **DnD template.** `create_dnd_email_template` with `name` + the assembled `definition` (universal header + body
   sections + universal footer — see `klaviyo-dnd.md`). Capture the template id. (`render_email_template` can sanity-
   check the render.) **Do not** use `create_email_template`/CODE here — a CODE template can't use universal blocks,
   which is the whole reason for this approach.
3. **Placeholder audience.** Find or create one reusable segment named `⚠️ PLACEHOLDER — REPLACE BEFORE SENDING`
   scoped to a harmless test profile (the owner's own email). Reuse it across runs — don't create a new one each time.
4. **Draft campaign.** `create_campaign` with: a name, email channel, the placeholder segment as the audience, the
   **two subject lines as an A/B test**, the preheader, and the default sender. Klaviyo creates campaigns in **draft**
   by default — do not call any send/schedule tool. If the connector can't configure A/B subjects, use the stronger
   single subject and tell the user to add the variant in Klaviyo.
5. **Attach the template** to the campaign message (`assign_template_to_campaign_message`).
6. **Verify + return** (`get_campaign`) — confirm draft status and return the id/link.

### Via REST (fallback, uses `KLAVIYO_API_KEY`)

Headers on every request:

```
Authorization: Klaviyo-API-Key ${KLAVIYO_API_KEY}
revision: 2024-10-15            # use a current Klaviyo API revision date
accept: application/json
content-type: application/json
```

Sequence mirrors the connector path:
- `POST /image-upload/` (or `/images/`) — host each image, get a URL.
- **Template:** the REST `/templates/` endpoint creates CODE/HTML templates, which can't carry universal blocks. For
  this skill, create the **DnD** template through the connector's `create_dnd_email_template` instead. If only REST is
  available and it won't accept a DnD definition, tell the user rather than silently shipping a CODE template that
  breaks the universal-header/footer feature.
- Ensure the placeholder segment exists (`GET /segments/?filter=...`; create via `POST /segments/` only if missing),
  scoped to the test profile.
- `POST /campaigns/` — audience = the placeholder segment id, the email message with subject/preview_text/from_email/
  from_label, A/B subject variations if the revision supports them, send strategy left unset so it stays a **draft**.
- Assign the template to the campaign message.
- Do **not** call `POST /campaign-send-jobs/` — that is the send action and is out of scope.

Klaviyo revises its API periodically; if a call is rejected, confirm the current revision date and request shapes from
Klaviyo's docs and adjust. The sequence (host images → template → placeholder audience → draft campaign → attach →
leave as draft) holds regardless of revision.

---

## Why a placeholder audience (and why it's safe)

The user defines who the email goes to — that's their call, and a wrong audience is costly. So the skill never targets a
real list. It attaches a single, loudly-named placeholder segment scoped to a test profile, so the draft is complete and
previewable but a careless "send" reaches no real customers. The real audience gets set by the user — the two audience
prompts shipped in Phase 10 give them a ready-made starting point for Klaviyo's segment builder.
