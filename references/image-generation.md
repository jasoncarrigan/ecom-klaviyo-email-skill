# Generating the email hero with Gemini / Nano Banana

The email needs **one** strong hero image — a branded lifestyle shot, not a plain product-on-white. The real product
photo is a non-negotiable reference so the product renders accurately. Model: `gemini-3-pro-image-preview` (Nano Banana
2), key `GEMINI_API_KEY`.

## The one rule that matters most

**The real product photo (from Phase 2) is the first reference image in the generation call.** Without it the model
invents the packaging/shape. Tell the model explicitly:

> "REFERENCE IMAGE 1 is the ACTUAL product. Reproduce it with photographic accuracy — exact shape, materials, colors,
> branding, and any text/logos must match. Do not redesign it."

Add 1–2 style references (from Tavily) as REFERENCE IMAGE 2, 3 to anchor the mood and color world.

## How to write the prompt

Nano Banana rewards narrative, art-direction prompts — brief it like a creative director on a real shoot, not a keyword
list.

1. **Role-set** — "You are a world-class commercial photographer and art director shooting a hero image for a premium
   [brand] email campaign."
2. **Lead with the concept** — one sentence describing the scene and the feeling, tied to the email's angle.
3. **Commit to the brand color world** — name the exact brand hex and command the scene to live in it (props,
   background, lighting), with the product as the focal contrast.
4. **Camera + lens** for photographic control — e.g. lifestyle/set: "Fujifilm GFX 100S, 80mm f/1.7, medium format";
   product macro: "Canon EOS R5, 100mm macro, f/2.8".
5. **Lighting as mood** — "soft diffused editorial light" / "warm golden-hour outdoor light" / "bright even Apple-style
   product light," matched to the brand tone.
6. **Texture callouts** — real materials, legible product label, true-to-life surfaces.
7. **People, if any** — cast and wardrobe inside the brand world and the ICP; direct genuine emotion ("mid-laugh, eyes
   crinkled — not a stock smile"), never a forced pose.
8. **Composition for email** — wide hero framing (≈1200×600), product clearly placed, and **leave clean negative space
   where the HTML headline/CTA will sit** (remember: no baked-in text — keep the image text-free).
9. **Label the references** — "REFERENCE 1: actual product, match exactly. REFERENCE 2: aesthetic/style I want."
10. **Anti-generic close** — "This must not look like a generic stock photo or a product-on-countertop shot. It should
    look like a bold, high-budget brand campaign."

## API pattern

```python
import json, urllib.request, base64, ssl, os

api_key = os.environ['GEMINI_API_KEY']
ctx = ssl.create_default_context()

def load_ref(path):
    with open(path, 'rb') as f:
        return base64.b64encode(f.read()).decode()

product   = load_ref('email-workspace/RUN/images/product-original.png')   # REQUIRED, first
style_ref = load_ref('email-workspace/RUN/images/style-ref.jpg')          # optional

prompt = """YOUR DETAILED ART-DIRECTION PROMPT HERE"""

payload = json.dumps({
    'contents': [{'parts': [
        {'text': prompt},
        {'inlineData': {'mimeType': 'image/png',  'data': product}},
        {'inlineData': {'mimeType': 'image/jpeg', 'data': style_ref}}
    ]}],
    'generationConfig': {
        'responseModalities': ['TEXT', 'IMAGE'],
        'imageConfig': {'aspectRatio': '16:9'}   # wide hero
    }
}).encode()

url = f'https://generativelanguage.googleapis.com/v1beta/models/gemini-3-pro-image-preview:generateContent?key={api_key}'
req = urllib.request.Request(url, data=payload, headers={'Content-Type': 'application/json'})
data = json.loads(urllib.request.urlopen(req, timeout=180, context=ctx).read())

for part in data.get('candidates', [{}])[0].get('content', {}).get('parts', []):
    if 'inlineData' in part:
        with open('email-workspace/RUN/images/hero.png', 'wb') as f:
            f.write(base64.b64decode(part['inlineData']['data']))
```

After generating, **host the hero** (Klaviyo image upload — see `klaviyo-deploy.md`) and use the returned URL in the
HTML. Model names and config fields evolve; if a call is rejected, check the current Gemini image API and adjust the
model id or `generationConfig` shape — the workflow (product-as-first-reference → art-directed prompt → host → embed)
stays the same.

If `GEMINI_API_KEY` is unset, skip the hero, use a strong real product photo in its place, and tell the user.
