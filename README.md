# 📧 ecom-klaviyo-email

> Turn one product or collection link into a finished, on-brand campaign email — deployed to Klaviyo as a ready-to-review draft. Built as a Claude skill.

---

## ⚡ Quick Setup (the lazy way)

**Copy the `SKILL.md` file. Paste it into Claude. Say:**

> *"Help me set this skill up. Here's the file — walk me through what keys I need and store them safely on my machine."*

Claude will:

1. Tell you the 4 API keys you need (links below).
2. Save them into your shell config (`~/.zshrc` on Mac, `~/.bashrc` on Linux) so every session picks them up.
3. Tell you to restart once, and you're done.

**⚠️ Security note — treat keys like passwords.** Add them yourself in your own terminal, never paste keys into a chat. Rotate them from each provider's dashboard after setup, and never commit `.zshrc`, `.bashrc`, or any env file to Git.

---

## 🔑 The 4 Keys You'll Need

| # | Provider | What it does | Env var | Get key |
| --- | --- | --- | --- | --- |
| 1 | **Klaviyo** | Reads past performance + creates the draft campaign | `KLAVIYO_API_KEY` | [klaviyo.com](https://www.klaviyo.com) → Settings → API Keys |
| 2 | **Google AI Studio** | Hero image generation (Nano Banana 2 / Gemini) | `GEMINI_API_KEY` | [aistudio.google.com/apikey](https://aistudio.google.com/apikey) |
| 3 | **Tavily** | Customer research + reference imagery | `TAVILY_API_KEY` | [tavily.com](https://tavily.com) |
| 4 | **Scrape Creators** | Competitor email/ad research (Facebook Ads Library) | `SCRAPE_CREATORS_API_KEY` | [scrapecreators.com](https://scrapecreators.com) |

**Klaviyo** and **Gemini** are the workhorses. Tavily and Scrape Creators sharpen the research but **fall back to Claude's built-in web search** if you skip them. If a Klaviyo MCP/connector is already connected, the skill prefers it and you may not need the raw Klaviyo key.

---

## 🧑‍🍳 What this actually does

You give it **one link** — a product page (PDP) or a collection page (PLP) — plus the email type. It runs a mini email team:

- **Brand director** learns your look and voice (from a style guide, Figma, or by scraping your site) — and remembers it.
- **Strategist** reads the page, then mines Reddit, Amazon reviews, and TikTok for the real customer language and angle.
- **Analyst** studies your own Klaviyo history to learn what's worked — best subjects, send times, top content — so this email aims to **beat your past performance**.
- **Competitive analyst** scrapes competitor emails/ads to find the gap nobody's claiming.
- **Copywriter + art director** writes the email and generates an AI hero image (using your real product photo as reference) alongside real product shots.
- **Email developer** builds the email as an editable Klaviyo drag-and-drop template (single product or a shop-the-collection grid) — with your **universal header & footer auto-inserted**, so every draft is already on-brand and has a compliant unsubscribe footer.
- **Ops** loads it into Klaviyo as a **draft** with a clearly-marked placeholder audience — so nothing ever sends by accident — and hands you two ready-to-paste audience prompts (one targeted, one broad).

All from one link and one sentence.

---

## 🚀 Running it

```
Build a campaign email for my daily send, featuring this product:
https://yourstore.com/products/your-product
```

Or for a collection:

```
/ecom-klaviyo-email https://yourstore.com/collections/bestsellers — weekly newsletter
```

What you get back:

- A **draft campaign** in your Klaviyo account (audience = placeholder; you set the real one before sending).
- **Two A/B subject lines** + a preheader.
- **Two audience prompts** (targeted + broad) to paste into Klaviyo's audience builder.
- A **recommended send time** and the **benchmarks to beat**, both pulled from your account history.
- Local files in `email-workspace/<timestamp>/` (the HTML, the images, and the briefs).

---

## 🔧 Manual install (if you'd rather do it yourself)

```
mkdir -p ~/.claude/skills/ecom-klaviyo-email
curl -sL https://raw.githubusercontent.com/jasoncarrigan/ecom-klaviyo-email-skill/main/SKILL.md \
  -o ~/.claude/skills/ecom-klaviyo-email/SKILL.md
mkdir -p ~/.claude/skills/ecom-klaviyo-email/references
for f in caching klaviyo-deploy email-html image-generation; do
  curl -sL "https://raw.githubusercontent.com/jasoncarrigan/ecom-klaviyo-email-skill/main/references/$f.md" \
    -o "~/.claude/skills/ecom-klaviyo-email/references/$f.md"
done
```

Then add to `~/.zshrc` (or `~/.bashrc`):

```
export KLAVIYO_API_KEY="..."
export GEMINI_API_KEY="..."
export TAVILY_API_KEY="..."            # optional
export SCRAPE_CREATORS_API_KEY="..."   # optional
```

Restart Claude (the **Code** tab of Claude Desktop, or the `claude` CLI). Run the skill.

> **Note:** this skill runs in **Claude Code** (the Code tab of the Claude desktop app, or the CLI) — it needs network access to reach Gemini, Tavily, and Scrape Creators. It won't run in Cowork's sandbox.

---

## 🧠 What it remembers

To avoid redoing slow work every run, the skill caches two things in a local `brand/` folder **in your working directory** (never in this repo):

- `brand/brand-profile.md` — your colors, fonts, logo, voice, and default sender. Validated with you each run.
- `brand/klaviyo-benchmarks.json` — your performance benchmarks + audience prompts. Refreshed every 30 days.
- `brand/universal-blocks.json` — your Klaviyo universal header & footer, captured once so they're auto-inserted into every email.

These are personal to your store and are **git-ignored** — they never get committed.

---

## 💸 Cost to run

- Claude Pro/Max: required to run skills.
- Gemini image generation: free tier covers casual use, then ~$0.03 / image.
- Tavily: free tier (1,000 searches/mo) is plenty.
- Scrape Creators: ~$29/mo (optional).
- Klaviyo: your existing plan.

---

## 📁 Works for any store

Apparel, supplements, cookware, RV parts, pet food, beauty — the skill reads your page and your account and infers the rest. Drop in a link, generate, review, send.

## ⚠️ Safety

This skill **never sends or schedules** anything. It stops at a Klaviyo draft with a placeholder audience, and you choose the real recipients and hit send yourself.
