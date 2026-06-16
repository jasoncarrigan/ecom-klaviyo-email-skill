# Building Klaviyo drag-and-drop (DnD) templates + reusing universal blocks

Deploy the email as a **drag-and-drop template** (`editor_type: SYSTEM_DRAGGABLE`), not a CODE/HTML template. Only DnD
templates support universal/saved content blocks and stay editable in Klaviyo's visual editor. CODE templates lock the
email to the HTML editor, where universal blocks can't be inserted.

## The big idea: universal blocks = sections with a `universal_id`

In a DnD `definition`, the email is `body.sections[]`. A **universal/saved block** is simply a `section` that carries a
`universal_id` field. When two templates contain a section with the same `universal_id`, Klaviyo treats them as the same
saved block and keeps them in sync. So to put the store's universal header and footer into a new email, you reuse those
exact sections (content **and** their `universal_id`) — Klaviyo links them automatically.

The store's header and footer already exist as universal sections inside its older DnD templates. Capture them once,
cache them, and wrap every generated email between them.

## Definition structure (annotated skeleton)

```
definition = {
  "body": {
    "properties": { "id": "bodyTable", "css_class": "root-container" },
    "styles": { "background_color": "#FFFFFF", "width": 600 },
    "id": "<unique-hex>",
    "sections": [
      <UNIVERSAL HEADER SECTION>,      # reused from cache (has its own universal_id)
      <generated body section 1>,      # your content
      <generated body section 2>,
      ...
      <UNIVERSAL FOOTER SECTION>       # reused from cache (carries {% unsubscribe %})
    ]
  },
  "styles": [ <base-styles>, <text-styles>, <link-styles>, <heading-1..4-styles>, <mobile-styles> ]
}
```

A **section** → has `rows` → each row has `columns` (via `data.styles.column_layout`, e.g. `"1-column-full-width"`,
`"2-columns-equal-width"`, `"3-columns-equal-width"`) → each column has `blocks`.

```
section = { "content_type":"section", "type":"section",
            "data":{"properties":{},"display_options":{},"styles":{}},
            "id":"<hex>", "data_id":"<hex>",
            "rows":[ { "data":{"styles":{"column_layout":"1-column-full-width"}},
                      "id":"<hex>", "data_id":"<hex>",
                      "columns":[ {"id":"<hex>","data_id":"<hex>","data":{},"blocks":[ ... ]} ] } ] }
```

**Every `id` and `data_id` must be unique** across the definition. Generate fresh 32-char hex strings (e.g.
`uuid4().hex`) for each section/row/column/block you create. Do **not** reuse ids — except inside a cached universal
section, which you paste verbatim (its ids and `universal_id` stay as-is so Klaviyo recognizes it).

## Block types you'll use for the body

- **text** — rich content as HTML: `{"content_type":"block","type":"text","data":{"content":"<h1>...</h1>","styles":{}},"id":...,"data_id":...}`. Use for headline, body copy, the social-proof quote, price.
- **image** — `data.properties` = `{"dynamic":false,"alt_text":"...","href":"<click url>","src":"<hosted url>"}`, `data.styles` = `{"width":600,"height":...}`. Use for the AI hero and product photos. `src` must be a hosted URL (host via Klaviyo image upload first — see `klaviyo-deploy.md`).
- **button** — `data` = `{"content":"SHOP NOW","properties":{"href":"<url>"},"styles":{"background_color":"#213482"}}`. Bulletproof by default; use for CTAs.
- **split** — two side-by-side `subblocks` (`table_image` + `table_text`); handy for a product card (image beside copy).
- **product** — Klaviyo's dynamic product block, if pulling from a catalog feed.
- **horizontal_rule**, **social** — dividers and social icons (usually these live in the universal footer already).

For a **collection email**, build the grid with rows using `"2-columns-equal-width"` (or `3-`), one product image+text
block per column, repeated for 3–6 products, under a shared hero + headline + "Shop the collection" button.

Pull colors, fonts, and the heading styles in the top-level `styles` array from the brand profile so type and color
match the store. (Base text/heading/link styles control default block typography.)

## Capturing the universal header & footer (one-time, then cached)

1. List the store's templates and find a `SYSTEM_DRAGGABLE` one (`list_email_templates`, look for `editor_type`).
2. Pull its definition: `get_email_template(id, additional_fields_template=["definition"])`.
3. Scan `definition.body.sections` for sections that have a `universal_id`. The one near the **top** containing the logo
   / nav is the **header**; the one near the **bottom** containing `{% unsubscribe %}` / social icons is the **footer**.
4. **Confirm with the user** which section is the header and which is the footer (show a short description of each
   candidate — there may be several universal sections, e.g. a rewards or referral block — you only want header +
   footer unless the user wants more).
5. Cache the **full JSON** of the chosen header and footer sections (content + their `universal_id`) to
   `brand/universal-blocks.json`. See `caching.md`.

On later runs, reuse from cache (validate like the brand profile: "I'll use your saved header/footer — still right?").
If the cache is missing and you can't capture them, fall back to leaving clearly-marked empty sections at top and bottom
(e.g. a text block reading "⬆ add your Universal Header") so the user inserts them in the editor.

## Creating the template

Prefer the connector: **`create_dnd_email_template(name, definition)`** with the assembled definition (header + body +
footer). Then proceed exactly as before — create the draft campaign (A/B subjects, sender, placeholder audience) and
attach this template. See `klaviyo-deploy.md`.

The REST `templates` endpoint historically creates CODE templates only; DnD creation is best done through the connector
tool. If you must use REST and it won't accept a DnD definition, say so and create the template via the connector
instead of falling back to a CODE template (which would defeat the universal-blocks fix).

After creating, you can call `render_email_template` to sanity-check the render before attaching it to the campaign.
