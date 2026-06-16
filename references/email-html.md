# Email HTML that survives real inboxes

Email clients (especially Outlook, which renders with Word's engine) are years behind browsers. These rules exist
because of that — follow them and the email looks right everywhere; ignore them and it breaks in ways a browser preview
won't show.

## Core rules

- **Layout in tables, not divs/flex/grid.** Use nested `<table role="presentation">`. Flexbox and grid are unreliable
  or unsupported in major clients.
- **Inline every style.** Most clients strip `<head><style>`. Put CSS on the element via `style="..."`. A small
  `<style>` block for `@media` mobile tweaks is fine as progressive enhancement, never as a dependency.
- **600px max content width, single column.** Wider breaks on mobile; multi-column needs fragile hacks. The collection
  grid below stays safe by using a 2-up table that stacks on narrow screens.
- **Set `width` and `display:block` on images**, and always give descriptive **`alt` text** — many opens have images
  off by default, so the email must read with no images loaded.
- **Buttons are bulletproof table buttons, not images.** Image buttons vanish when images are blocked.
- **Web-safe font stacks** with the brand font first and a system fallback
  (`font-family:'BrandFont',Helvetica,Arial,sans-serif;`).
- **Don't strip Klaviyo's footer/unsubscribe** — it's merged in and legally required (CAN-SPAM).
- **Hero text lives in HTML, not baked into the image**, so it stays sharp, accessible, and readable with images off.
- **Always include the social-proof block** — a real review/rating near the CTA. Never fabricate a quote or rating.

## Accessibility & deliverability quick list

- `lang` on `<html>`; a semantic heading for the main headline.
- Sufficient contrast on text and the CTA.
- Real, current price/promo only.
- A meaningful preheader (35–90 chars), hidden with the standard preheader span.
- Keep total size well under Gmail's ~102KB clipping threshold where possible.

---

## Scaffold A — single product (PDP)

Fill colors, fonts, copy, image URLs, and links from the brand profile. Starting point, not a straitjacket — vary the
body for the product and goal.

```html
<!DOCTYPE html>
<html lang="en" xmlns="http://www.w3.org/1999/xhtml">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <title>{{ email_title }}</title>
</head>
<body style="margin:0; padding:0; background:#f4f4f4;">
  <div style="display:none; max-height:0; overflow:hidden; opacity:0;">{{ preheader }}</div>
  <table role="presentation" width="100%" cellpadding="0" cellspacing="0" style="background:#f4f4f4;">
    <tr><td align="center" style="padding:24px 12px;">
      <table role="presentation" width="600" cellpadding="0" cellspacing="0" style="width:600px; max-width:600px; background:#ffffff;">

        <tr><td align="center" style="padding:20px;">
          <img src="{{ logo_url }}" width="160" alt="{{ brand_name }}" style="display:block; border:0;">
        </td></tr>

        <tr><td>
          <a href="{{ cta_url }}"><img src="{{ hero_url }}" width="600" alt="{{ hero_alt }}"
             style="display:block; width:100%; max-width:600px; height:auto; border:0;"></a>
        </td></tr>
        <tr><td align="center" style="padding:28px 24px 8px;">
          <h1 style="margin:0; font-family:'BrandFont',Helvetica,Arial,sans-serif; font-size:28px; line-height:1.2; color:#1a1a1a;">{{ headline }}</h1>
        </td></tr>
        <tr><td align="center" style="padding:8px 32px 20px; font-family:'BrandFont',Helvetica,Arial,sans-serif; font-size:16px; line-height:1.5; color:#444;">{{ subhead }}</td></tr>

        <!-- bulletproof CTA -->
        <tr><td align="center" style="padding:8px 24px 24px;">
          <table role="presentation" cellpadding="0" cellspacing="0"><tr>
            <td align="center" bgcolor="{{ brand_accent }}" style="border-radius:6px;">
              <a href="{{ cta_url }}" style="display:inline-block; padding:14px 32px; font-family:'BrandFont',Helvetica,Arial,sans-serif; font-size:16px; font-weight:bold; color:#ffffff; text-decoration:none;">{{ cta_label }}</a>
            </td>
          </tr></table>
        </td></tr>

        <!-- social proof (always include) -->
        <tr><td align="center" style="padding:4px 32px 24px;">
          <div style="font-size:16px; color:#f5a623; letter-spacing:2px;">&#9733;&#9733;&#9733;&#9733;&#9733;</div>
          <p style="margin:8px 0 0; font-family:'BrandFont',Helvetica,Arial,sans-serif; font-style:italic; font-size:15px; line-height:1.5; color:#444;">&ldquo;{{ review_quote }}&rdquo;</p>
          <p style="margin:6px 0 0; font-family:'BrandFont',Helvetica,Arial,sans-serif; font-size:13px; color:#888;">&mdash; {{ review_author }}</p>
        </td></tr>

        <!-- product block -->
        <tr><td align="center" style="padding:0 24px 24px;">
          <img src="{{ product_url }}" width="300" alt="{{ product_alt }}" style="display:block; width:100%; max-width:300px; height:auto; border:0;">
          <div style="padding:12px 0; font-family:'BrandFont',Helvetica,Arial,sans-serif; color:#1a1a1a;">
            <strong style="font-size:18px;">{{ product_name }}</strong><br>
            <span style="font-size:16px; color:#c0392b;">{{ price }}</span>
          </div>
        </td></tr>

        <tr><td align="center" style="padding:20px; font-family:'BrandFont',Helvetica,Arial,sans-serif; font-size:12px; color:#888;">{{ brand_name }} · {{ brand_signoff }}</td></tr>
      </table>
    </td></tr>
  </table>
</body>
</html>
```

---

## Scaffold B — collection grid (PLP)

Same shell, but replace the single product block with a **2-up grid** of 3–6 product cards. Each card is a 50%-width
table cell that stacks to full width on mobile. Repeat the card `<td>` pair per row.

```html
<!-- collection grid: one row = two cards -->
<tr><td style="padding:0 16px 8px;">
  <table role="presentation" width="100%" cellpadding="0" cellspacing="0"><tr>

    <!-- card 1 -->
    <td width="50%" valign="top" style="padding:8px;">
      <a href="{{ p1_url }}"><img src="{{ p1_img }}" width="260" alt="{{ p1_alt }}" style="display:block; width:100%; max-width:260px; height:auto; border:0;"></a>
      <div style="padding:8px 0; font-family:'BrandFont',Helvetica,Arial,sans-serif; text-align:center;">
        <strong style="font-size:15px; color:#1a1a1a;">{{ p1_name }}</strong><br>
        <span style="font-size:14px; color:#c0392b;">{{ p1_price }}</span>
      </div>
    </td>

    <!-- card 2 -->
    <td width="50%" valign="top" style="padding:8px;">
      <a href="{{ p2_url }}"><img src="{{ p2_img }}" width="260" alt="{{ p2_alt }}" style="display:block; width:100%; max-width:260px; height:auto; border:0;"></a>
      <div style="padding:8px 0; font-family:'BrandFont',Helvetica,Arial,sans-serif; text-align:center;">
        <strong style="font-size:15px; color:#1a1a1a;">{{ p2_name }}</strong><br>
        <span style="font-size:14px; color:#c0392b;">{{ p2_price }}</span>
      </div>
    </td>

  </tr></table>
</td></tr>
```

For a clean mobile stack, add this to the optional `<style>` block (progressive enhancement only):

```css
@media only screen and (max-width:480px){
  .stack { display:block !important; width:100% !important; }
}
```

…and put `class="stack"` on the card `<td>`s. Keep one shared hero + headline + primary CTA above the grid, and a
single "Shop the collection" CTA below it. The `{{ ... }}` tokens are placeholders you fill in (or swap for Klaviyo
personalization tags) before creating the template.
