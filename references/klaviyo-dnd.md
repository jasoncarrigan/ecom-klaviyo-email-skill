# Building Klaviyo drag-and-drop (DnD) templates

Deploy the email as a **drag-and-drop template** (`editor_type: SYSTEM_DRAGGABLE`), not a CODE/HTML template. DnD
templates stay editable in Klaviyo's visual editor and carry the store's real header/footer content and brand type
styles. (CODE templates work too and are simpler — keep them as a fallback if DnD fails.)

## ⚠️ Hard-won connector constraints — read these first

These were learned the hard way against the Klaviyo MCP connector. Ignore them and `create_dnd_email_template` /
`update_dnd_email_template` will reject your payload repeatedly.

1. **Never send `id` or `data_id` anywhere in the definition.** Not on the body, sections, rows, columns, blocks,
   subblocks, or the `styles[]` entries. Klaviyo assigns them on create. Sending any → `400 "id is not allowed to be
   specified on create"` (and the same on update). Build the whole definition with **none** of them.

2. **You cannot set `universal_id` either — on create OR update.** Both reject `"universal_id is not allowed to be
   specified"`. This means **the API cannot create or preserve a universal/saved-block link.** The "paste the universal
   section verbatim so Klaviyo re-links it" idea does **not** work through this connector. Clone-and-update fails too:
   `update_dnd_email_template` *fully replaces* the definition, so re-sending the header/footer forces their
   `universal_id` back in → rejected; strip it and they stop being universal.
   → **Therefore: capture the header/footer for their CONTENT and embed them as ordinary sections.** True universal
   linkage is a one-time **manual step the user does in Klaviyo's visual editor** (open the email → select the header
   section → Universal Content → Replace with the saved block; repeat for the footer). Tell the user this; don't try to
   automate it.

3. **Native `image` blocks require a valid Klaviyo `asset_id`.** A `src` with no `asset_id` → `"Static image blocks with
   a src must also have an asset_id"`. And a *foreign* asset_id (e.g. one copied from another template) →
   `"Asset with id ... not found"`. So for AI heroes / product photos / any externally-hosted image (catbox, cloudfront,
   Shopify CDN), **use an `html` block with a plain `<img src="...">` instead** — html blocks have no asset_id
   requirement and render the external image fine. Only keep a native `image` block if you genuinely uploaded the asset
   to this account and have its real asset_id.

4. **Build the definition with a script, write it to a file, then read it back to inline into the tool call.** The
   definition is ~14 KB; assembling it by hand is error-prone. Write a small python step that builds the dict, strips any
   `id`/`data_id`/`universal_id`, validates with `json.load`, and saves it; then Read that file and pass it as the
   `definition` argument.

## Definition structure (annotated skeleton)

```
definition = {
  "body": {
    "properties": { "css_class": "root-container" },      # NO "id" here
    "styles": { "background_color": "#FFFFFF", "width": 600 },
    "sections": [
      <HEADER SECTION>,    # embedded from cache (content only — no universal_id)
      <body section(s)>,   # your generated content
      <FOOTER SECTION>     # embedded from cache (carries {% unsubscribe %})
    ]
  },
  "styles": [ <base-styles>, <text-styles>, <link-styles>, <heading-1..4-styles>, <mobile-styles> ]   # NO "id" on each
}
```

A **section** → has `rows` → each row sets `data.styles.column_layout` (`"1-column-full-width"`,
`"2-columns-equal-width"`, `"3-columns-equal-width"`, …) → each column has `blocks`. None of these carry `id`/`data_id`.

```
section = { "content_type":"section", "type":"section",
            "data":{"properties":{},"display_options":{},"styles":{}},
            "rows":[ { "data":{"styles":{"column_layout":"1-column-full-width"}},
                      "columns":[ {"data":{}, "blocks":[ ... ]} ] } ] }
```

## Block types you'll use for the body

- **text** — rich HTML: `{"content_type":"block","type":"text","data":{"content":"<h1>…</h1>","display_options":{},"styles":{…}}}`. Headline, body copy, price, the social-proof quote. Inline-style the HTML for color/size; the top-level `styles[]` array sets the defaults (pull font/colors from the brand profile — e.g. this store is **DM Sans**, link `#1d429a`).
- **html** — `{"content_type":"block","type":"html","data":{"content":"<a href=…><img src=… ></a>","display_options":{}}}`. **Use this for every externally-hosted image** (AI hero, product photos) to avoid the asset_id requirement. Also fine for the logo if you only have its URL.
- **button** — `{"content_type":"block","type":"button","data":{"content":"SHOP NOW","display_options":{},"properties":{"href":"<url>"},"styles":{"background_color":"#…","color":"#ffffff","border_radius":6,"text_align":"center","font_size":17,"font_weight":"bold","inner_padding_top":14,"inner_padding_bottom":14,"inner_padding_left":32,"inner_padding_right":32}}}`. Bulletproof; use for CTAs.
- **horizontal_rule**, **social** — dividers / social icons (usually already inside the cached footer).

For a **collection email**, build the grid with rows using `"2-columns-equal-width"` (or `3-`), one image/text block
per column, repeated for 3–6 products, under a shared hero + headline + "Shop the collection" button.

## Capturing the header & footer (one-time, then cached)

1. `list_email_templates` → find a `SYSTEM_DRAGGABLE` template (this store has **"Standard Campaign Template"**).
2. `get_email_template(id, additional_fields_template=["definition"])`.
3. In `definition.body.sections`, the sections carrying a `universal_id` are the universal blocks: the one near the
   **top** (logo / nav) is the header; the one near the **bottom** (`{% unsubscribe %}` / social) is the footer. Also
   grab the top-level `styles[]` array — it has the brand's real type (font family, heading sizes, link color).
4. **Confirm with the user** which is header and which is footer.
5. Cache the **full JSON** of both sections + the base styles to `brand/universal-blocks.json` (see `caching.md`).

When you build an email, paste those sections in as the first and last sections — but **strip their `id`, `data_id`,
and `universal_id`** first (constraints #1, #2). They'll render identically as ordinary branded sections. If the cached
header contains a native `image` logo block with an `asset_id` that fails on create, swap it for an `html` block with
the logo's `<img src>` (constraint #3).

## Creating the template

`create_dnd_email_template(name, definition)` with the assembled, fully-id-stripped definition. Then create the draft
campaign (A/B subjects, sender, placeholder audience) and attach it — see `klaviyo-deploy.md`. `render_email_template`
can sanity-check the render before attaching.

If anything about DnD fails and you're blocked, fall back to a **CODE** template (`create_email_template`, full inline
HTML with the header/footer baked in) — it's less editable but ships reliably.
