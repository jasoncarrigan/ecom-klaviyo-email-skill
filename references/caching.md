# Caching: brand profile & performance benchmark

Two artifacts are expensive to produce and change slowly, so cache them instead of rebuilding every run.

## Where cached files live

A `brand/` folder in the user's **current working directory** (the store project they run the skill from) — **not**
inside the skill repo. These files contain store-specific data and must never be committed to the public skill repo.
The repo ships a `.gitignore` that excludes `brand/`.

```
brand/
├── brand-profile.md          # colors, fonts, logo URL, voice, sender defaults
├── klaviyo-benchmarks.json   # performance stats + the two audience prompts, with a saved_at timestamp
└── universal-blocks.json     # the store's universal header + footer sections (full JSON + universal_id)
```

If the user runs the skill from a fresh directory with no `brand/`, treat the brand and benchmarks as unknown and
generate them (Phases 1 and 4), then save them here for next time.

## Brand profile — validate, don't assume

The brand changes rarely, but you should never silently reuse stale branding. On each run, if `brand-profile.md`
exists, surface it and ask the user to confirm:

> "I found brand reference documents here: `brand/brand-profile.md` (saved 2026-05-02). Is this what you want me to use,
> or should I refresh it?"

Only regenerate if they ask you to, or if the file is missing. Include a `saved_at` line at the top of the file.

## Benchmark — refresh every 30 days

`klaviyo-benchmarks.json` carries a `saved_at` ISO timestamp. On each run:

- If `saved_at` is **less than 30 days** ago → reuse it; print the summary; skip the deep Klaviyo pull.
- If it's **30+ days old or missing** → re-run the Phase 4 analysis and overwrite the file with a fresh timestamp.

Suggested shape:

```json
{
  "saved_at": "2026-06-16T12:00:00Z",
  "window": "last 20 campaigns / 90 days",
  "benchmarks": { "avg_open_rate": 0.24, "avg_click_rate": 0.019, "revenue_per_recipient": 0.41 },
  "top_patterns": { "subjects": ["..."], "best_send": "Thursday 9am CT", "offers": ["..."] },
  "audiences": {
    "targeted": "Plain-English segment description for Klaviyo's builder ...",
    "broad": "Plain-English segment description for Klaviyo's builder ..."
  }
}
```

Keep the two audience prompts in the cache too, so a reused benchmark still ships them in Phase 10. If the store's
strategy or list changes a lot, the 30-day refresh picks it up; the user can also force a refresh by asking.

## Universal blocks — capture once, validate, reuse

`universal-blocks.json` holds the **full section JSON** (content + `universal_id`) for the store's universal header and
footer, captured once from an existing Klaviyo DnD template (see `klaviyo-dnd.md` for how to find and confirm them):

```json
{
  "saved_at": "2026-06-16T12:00:00Z",
  "header": { /* the full header section object, incl. its universal_id */ },
  "footer": { /* the full footer section object, incl. its universal_id */ }
}
```

These rarely change, so reuse them on every run; just validate like the brand profile ("I'll use your saved
header/footer — still right?"). Because each section keeps its original `universal_id`, Klaviyo links it to the saved
universal block automatically when the new email is created. Refresh only if the user updates their header/footer or
asks. Like the rest of `brand/`, this file is git-ignored and never committed.
