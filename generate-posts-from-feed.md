# /generate-posts-from-feed

Generate vehicle-aware promotional social posts from NLP Performance store
product links. For each product: identify the product and its fitment
vehicle, generate an 8-second Lovart promo video, brand it, write post
content, and queue everything for `/zoho-social-batch`.

Feed-driven counterpart to `/generate-posts-from-reviews` (which scrapes Yotpo
reviews). This skill takes product links directly and renders video via
Lovart instead of fal.ai Seedance.

## Usage

```
/generate-posts-from-feed <input> [--limit N] [--video-duration 8] [--aspect 9:16] [--schedule-start <date>] [--schedule-interval 24] [--schedule-time 21:00]
```

`<input>` is one of:
- one or more `https://nlpperformance.com/products/{handle}` URLs (space/newline separated)
- a `https://nlpperformance.com/collections/{handle}` collection URL
- a path or URL to a product feed file (`.xml`, `.csv`, or `.json`)

| Arg | Default | Meaning |
|-----|---------|---------|
| `--limit` | all | Cap number of products processed. |
| `--video-duration` | `8` | Target video length (seconds). |
| `--aspect` | `9:16` | Video aspect ratio. |
| `--schedule-start` | next free day | First post date. |
| `--schedule-interval` | `24` | Hours between posts. |
| `--schedule-time` | `21:00` | Time of day (9:00 PM EDT). |

## Strict rules (carried from /generate-posts-from-reviews)

- **Cadence: 1 post/day at 9:00 PM EDT.** Never use Zoho "Add to Queue".
- Before scheduling, check the Zoho Scheduled Posts page for the last
  scheduled date; start the new batch the next day at 9 PM EDT.
- **Never fabricate discount percentages.** No "30% off" unless the user
  explicitly provides a real promotion.
- Append `Follow us for more promotions and content.` to every caption
  (between body and hashtags) and every first comment (new line after the link).

## Step 1 — Resolve input to a product list

Determine the input type and build a list of product handles.

- **Product URLs** (single or multiple): extract `{handle}` from each
  `.../products/{handle}`.
- **Collection URL:** page through
  `https://nlpperformance.com/collections/{handle}/products.json?limit=250&page=N`,
  incrementing `N` from 1, until a page returns an empty `products` array;
  collect every handle. Each page:

  ```bash
  curl -s "https://nlpperformance.com/collections/{handle}/products.json?limit=250&page=N" | jq -r '.products[].handle'
  ```
- **Feed file (`.json`):** `jq` for product handles/urls.
- **Feed file (`.xml`):** parse `<link>`/`<g:link>` entries for `/products/{handle}`.
- **Feed file (`.csv`):** the column holding the product URL or handle.

Dedupe the handle list. Apply `--limit` (keep the first N). If the input
matches none of the patterns, STOP and tell the user the input was not
recognized.

## Step 2 — Identify each product

For each handle, fetch the Shopify product JSON:

```bash
curl -s "https://nlpperformance.com/products/{handle}.json" \
  | jq '{title:.product.title, vendor:.product.vendor, product_type:.product.product_type, tags:.product.tags, body_html:.product.body_html, images:[.product.images[].src], handle:.product.handle}'
```

Store per product: `title`, `vendor`, `product_type`, `tags`, `body_html`,
`images[]`, `handle`, and `product_url = https://nlpperformance.com/products/{handle}`.
If the fetch fails or returns no `title`, skip that product and note it in
the run summary.

## Step 3 — Identify the vehicle (priority chain, first hit wins)

For each product, determine `{vehicle}` and record `vehicle_source`:

1. **Title parse** (`vehicle_source = title`). Match a year range plus a
   known make and model in `title`, e.g. `16-23 Toyota Tacoma V6-3.5L`.
   Known makes: Toyota, Ford, Honda, Subaru, Chevrolet, Chevy, Jeep, RAM,
   Dodge, GMC, Nissan, Lexus, Mazda, Volkswagen, VW, BMW, Audi, Mercedes,
   Mercedes-Benz, Hyundai, Kia, Tesla, Acura, Infiniti, Porsche, Mitsubishi,
   Mini, Scion, Pontiac, Cadillac, Buick, Chrysler, Lincoln, Land Rover,
   Range Rover, Genesis, Volvo, Jaguar, Fiat, Alfa Romeo, Maserati, Suzuki.
   Capture `{year_range} {make} {model}` (drop the engine code from
   `{vehicle}` but keep make+model).
2. **Fitment table in `body_html`** (`vehicle_source = fitment-table`).
   If the title yields nothing, parse `body_html` for an HTML `<table>` whose
   header row contains Year/Make/Model columns; take the make+model (if
   multiple rows, use the most frequent make, and "multiple models" or the
   single model). Pipe `body_html` into this `python3` script (standard
   library `html.parser` only):

   ```python
   import sys
   from html.parser import HTMLParser
   from collections import Counter
   class TableExtractor(HTMLParser):
       def __init__(self):
           super().__init__()
           self.tables=[]; self._rows=None; self._cells=None; self._cur=[]; self._in=False
       def handle_starttag(self,t,a):
           if t=='table': self._rows=[]
           elif t=='tr' and self._rows is not None: self._cells=[]
           elif t in('td','th') and self._cells is not None: self._in=True; self._cur=[]
       def handle_endtag(self,t):
           if t=='table' and self._rows is not None: self.tables.append(self._rows); self._rows=None
           elif t=='tr' and self._cells is not None: self._rows.append(self._cells); self._cells=None
           elif t in('td','th') and self._in: self._cells.append(''.join(self._cur).strip()); self._in=False
       def handle_data(self,d):
           if self._in: self._cur.append(d)
   p=TableExtractor(); p.feed(sys.stdin.read())
   make=model=None
   for rows in p.tables:
       if not rows: continue
       hdr=[c.lower() for c in rows[0]]
       if 'make' in hdr and 'model' in hdr:
           mi,mo=hdr.index('make'),hdr.index('model')
           makes=[r[mi] for r in rows[1:] if len(r)>max(mi,mo) and r[mi]]
           models=[r[mo] for r in rows[1:] if len(r)>max(mi,mo) and r[mo]]
           if makes:
               make=Counter(makes).most_common(1)[0][0]
               uniq=list(dict.fromkeys(models))
               model=uniq[0] if len(uniq)==1 else 'multiple models'
           break
   print(f"{make} {model}" if make else "")
   ```
3. **Product tags** (`vehicle_source = tags`). If still nothing, scan `tags`
   for any value matching a known make; pair with a model tag if present.
4. **Else** `{vehicle}` is empty and `vehicle_source = universal`.

## Step 4 — Gather reference media (priority chain, first hit wins)

Create a staging dir and walk the chain; stop at the first source that yields
a usable image. Record `image_source`.

```bash
mkdir -p /tmp/feed-media/product-{handle}/
```

### 4a. Shopify product images (`image_source = shopify`)

Download the first 1–3 `images[]` entries:
```bash
curl -sL "<image_src>" -o "/tmp/feed-media/product-{handle}/media-{n}.jpg"
file "/tmp/feed-media/product-{handle}/media-{n}.jpg" | grep -q "image data"
```
Keep only files that verify as real image data.

### 4b. Web-search fallback (`image_source = web-search`)

Only if the product has no `images[]` or none verified. Strict depth-1 budget:
**at most 1 `WebSearch` + 3 `WebFetch` calls per product.**
1. `WebSearch`: `"{title} product photo"`.
2. Take the top 3 organic URLs.
3. For each (max 3, stop at first success): `WebFetch` with prompt
   "Find the single primary product image URL on this page. Return just the
   URL." If the result is a valid image URL, `curl -sL` it to
   `media-0.jpg` and verify with `file`. Stop on first success.
Never re-search, never multi-hop crawl.

### 4c. None (`image_source = none`)

If 4a and 4b both fail, the Lovart prompt runs text-only.

## Step 5 — Generate the 8-second promo video

For each product, build a three-act prompt and call `/lovart-video`. Lovart
renders the full 8 seconds of clean footage; the branded title-card text is
overlaid afterward in Step 6.

**`product_short_name`** = `title` with the leading SKU/year/model prefix
stripped (e.g. "K&N 16-23 Toyota Tacoma V6-3.5L Elevated Intake Kit (Snorkel)"
→ "K&N Tacoma Elevated Intake (Snorkel)").

**`product_kind`** = a short category noun derived from `product_type`/title
(e.g. "cold air intake", "sway bar", "coilovers", "fuel pump", "wheel",
"exhaust system", "brake kit").

**`scene_vehicle`** = `{vehicle}` when `vehicle_source != universal`;
otherwise the literal phrase `a matching performance vehicle`.

**`ambience_cue`** by category:
- intake/exhaust → `engine idle with a short confident rev`
- suspension/wheels → `tire grip on pavement, distant workshop sounds`
- brakes → `brake disc rotation hiss, subtle impact clicks`
- other/accessory → `subtle workshop ambience, tools on metal`

**Three-act prompt template** (keep under 500 chars). This template is fixed —
only `product_short_name`, `product_kind`, `scene_vehicle`, and `ambience_cue`
are substituted, so it works for any kind of vehicle part:

```
Vertical 9:16 720p 8-second cinematic product reel, set in one ultra-realistic
modern performance shop garage, very smooth continuous camera motion,
photorealistic. Seconds 0-1: clean centered hero shot of {product_short_name}
({product_kind}) shown at modest scale, occupying only the central third of
the vertical frame with generous empty space above and below it, on a dark
gradient backdrop. Seconds 1-4: ultra-realistic multi-angle orbiting overview
of {scene_vehicle}. Seconds 4-8: smooth close-up multi-angle shots of the
{product_kind} already installed on {scene_vehicle} at its correct mounting
location. {ambience_cue}. No text, no logos, no watermarks — added in post.
Render with the Seedance 2.0 Fast video model.
```

The "central third" framing in seconds 0–1 is deliberate: it leaves clear
space at the top for the title-card logo and below for the product name.

The final sentence — `Render with the Seedance 2.0 Fast video model.` — is
part of the prompt on purpose: the Lovart agent resolves the video model by
name from the prompt text, which is the reliable channel. Keep it verbatim.

**Lovart settings (fixed for this skill).** Every `/lovart-video` call this
skill makes passes these options, so the generation runs in the right Lovart
project with the intended model and reasoning depth:

- `--mode thinking` — deep structured reasoning.
- `--project-id e38875ee330f4140bf024d880d644ab3` — the NLP Performance
  promo-video project in the Lovart workspace.
- `--prefer-models '{"VIDEO":["generate_video_seedance_v2"]}'` — a secondary
  soft hint toward Seedance. `prefer-models` needs an internal tool identifier
  that Lovart does not expose, so this is best-effort only; the in-prompt
  "Seedance 2.0 Fast" sentence above is the channel that actually selects the
  model.

Invoke the video skill, passing any downloaded reference images:

```
/lovart-video --prompt "<engineered prompt>" \
  --ref /tmp/feed-media/product-{handle}/media-0.jpg [--ref ...] \
  --out /tmp/feed-media/product-{handle}/raw.mp4 \
  --mode thinking \
  --prefer-models '{"VIDEO":["generate_video_seedance_v2"]}' \
  --project-id e38875ee330f4140bf024d880d644ab3
```

`/lovart-video` accepts `--out` and prints the resulting absolute path on its
last line — capture that printed path (it equals the `--out` value). If it
fails for a product, record the failure and skip that product's post (do not
abort the whole run).

## Step 6 — Title-card + branding overlay

Lovart's `raw.mp4` is clean footage. This step overlays the branded title
card on the first second and a small persistent shop logo, producing
`/tmp/feed-media/product-{handle}/{handle}.mp4`.

The title card sits on top of Lovart's first-second product hero, in this
layout: the NLP shop logo across the top, the product name in the
upper-middle, and a tagline near the bottom — the dark gradient backdrop
comes from the Lovart footage itself.

### 6a. Resolve the NLP logo

```bash
mkdir -p ~/.claude/assets
```

**NLP logo** — `~/.claude/assets/nlp-logo.png`. If absent, use the WebFetch
tool on `https://nlpperformance.com` with a prompt asking for the site header
logo image URL, then `curl -sL "<logo_url>" -o ~/.claude/assets/nlp-logo.png`
and verify with `file`. If it cannot be obtained, the title card and the
watermark render without the logo (text only) — do not abort.

### 6b. Render the title-card overlay PNG (Pillow)

Render one full-canvas 720×1280 transparent PNG carrying the shop logo (top),
the product name (upper-middle, word-wrapped), and the tagline (lower). The
center is left transparent so Lovart's product hero shows through.
ImageMagick is not installed — use Pillow.

The tagline is `For your {vehicle}` when `vehicle_source != universal`,
otherwise the literal `Performance. Installed.`

```bash
HANDLE="<handle>"                 # current product handle
SHORT_NAME="<product_short_name>"
TAGLINE="<tagline>"               # per the rule above
LOGO="$HOME/.claude/assets/nlp-logo.png"   # may be absent — handled in-script
python3 - "$SHORT_NAME" "$TAGLINE" "$LOGO" "/tmp/feed-media/product-$HANDLE/titlecard.png" <<'PY'
import sys, os
from PIL import Image, ImageDraw, ImageFont
short_name, tagline, logo_path, out_path = sys.argv[1], sys.argv[2], sys.argv[3], sys.argv[4]
W, H = 720, 1280
img = Image.new("RGBA", (W, H), (0, 0, 0, 0))
d = ImageDraw.Draw(img)

def load_font(size):
    for p in ["/Library/Fonts/Inter-Bold.ttf",
              "/System/Library/Fonts/Supplemental/Arial Bold.ttf"]:
        try: return ImageFont.truetype(p, size)
        except Exception: pass
    return ImageFont.load_default()

def measure(s, fnt):
    b = d.textbbox((0, 0), s, font=fnt); return b[2]-b[0], b[3]-b[1]

def wrap(t, fnt, max_w, max_lines):
    words, lines, cur = t.split(), [], ""
    for i, w in enumerate(words):
        trial = (cur + " " + w).strip()
        if measure(trial, fnt)[0] <= max_w or not cur:
            cur = trial
        else:
            lines.append(cur); cur = w
            if len(lines) == max_lines - 1:
                rest = " ".join(words[i:])
                while measure(rest, fnt)[0] > max_w and len(rest) > 3:
                    rest = rest[:-1]
                if measure(rest, fnt)[0] > max_w: rest = rest.rstrip() + "…"
                lines.append(rest); return lines
    if cur: lines.append(cur)
    return lines

def draw_centered(lines, fnt, y, gap=12):
    for ln in lines:
        w, h = measure(ln, fnt)
        d.text(((W - w) // 2, y), ln, font=fnt, fill=(255, 255, 255, 255),
               stroke_width=3, stroke_fill=(0, 0, 0, 235))
        y += h + gap
    return y

# shop logo on a contrast panel, centered in the top band.
# The logo PNG has a transparent background; a dark logo would vanish on the
# dark first-second footage, so it always sits on a rounded contrast panel:
# light panel for a dark logo, dark panel for a light logo.
if os.path.isfile(logo_path):
    try:
        logo = Image.open(logo_path).convert("RGBA")
        lw = 280
        lh = max(1, round(logo.height * lw / logo.width))
        logo = logo.resize((lw, lh))
        vis = [(r, g, b) for r, g, b, a in logo.getdata() if a > 40]
        lum = sum(0.299*r + 0.587*g + 0.114*b for r, g, b in vis) / len(vis) if vis else 255
        pad, radius, top = 28, 22, 60
        pw, ph = lw + pad * 2, lh + pad * 2
        px0, py0 = (W - pw) // 2, top
        panel = (255, 255, 255, 235) if lum < 110 else (16, 16, 22, 210)
        ImageDraw.Draw(img).rounded_rectangle(
            [px0, py0, px0 + pw, py0 + ph], radius=radius, fill=panel)
        img.alpha_composite(logo, (px0 + pad, py0 + pad))
    except Exception:
        pass

# product name, upper-middle
draw_centered(wrap(short_name, load_font(52), int(W * 0.86), 2), load_font(52), 330)
# tagline, lower
draw_centered(wrap(tagline, load_font(40), int(W * 0.86), 1), load_font(40), 1080)

img.save(out_path)
PY
```

### 6c. Composite with ffmpeg

The title-card PNG shows only during seconds 0–1 (fading in then out); a small
shop logo stays top-right for the whole clip. The raw video is first
normalized to exactly 720×1280. Output is capped at 8 seconds.

`-loop 1` on the title-card input is required: it turns the still PNG into a
continuous stream so the `fade` filter animates over time. Without it, `fade`
evaluates the single frame at t=0 (alpha 0) and the whole overlay renders
invisible.

**With the NLP logo:**

```bash
ffmpeg -y -i /tmp/feed-media/product-{handle}/raw.mp4 \
  -loop 1 -i /tmp/feed-media/product-{handle}/titlecard.png \
  -i ~/.claude/assets/nlp-logo.png \
  -filter_complex "
    [0:v]scale=720:1280:force_original_aspect_ratio=increase,crop=720:1280,setsar=1[base];
    [1:v]format=rgba,fade=t=in:st=0:d=0.2:alpha=1,fade=t=out:st=0.85:d=0.15:alpha=1[tc];
    [base][tc]overlay=0:0:enable='between(t,0,1)'[a];
    [2:v]scale=96:-2[wm];
    [a][wm]overlay=W-w-24:24[v]
  " -map "[v]" -map "0:a?" -t 8 -c:v libx264 -pix_fmt yuv420p -c:a copy \
  /tmp/feed-media/product-{handle}/{handle}.mp4
```

**Without the NLP logo (Step 6a could not obtain it)** — drop the third input
and the watermark nodes; the title-card overlay output is named `[v]` directly:

```bash
ffmpeg -y -i /tmp/feed-media/product-{handle}/raw.mp4 \
  -loop 1 -i /tmp/feed-media/product-{handle}/titlecard.png \
  -filter_complex "
    [0:v]scale=720:1280:force_original_aspect_ratio=increase,crop=720:1280,setsar=1[base];
    [1:v]format=rgba,fade=t=in:st=0:d=0.2:alpha=1,fade=t=out:st=0.85:d=0.15:alpha=1[tc];
    [base][tc]overlay=0:0:enable='between(t,0,1)'[v]
  " -map "[v]" -map "0:a?" -t 8 -c:v libx264 -pix_fmt yuv420p -c:a copy \
  /tmp/feed-media/product-{handle}/{handle}.mp4
```

## Step 7 — Generate post content

Product/vehicle-led and feature-forward (there is no customer quote). Produce
five fields per product: `caption`, `hashtags`, `yt_title`, `yt_description`,
`first_comment`.

### 7a. Gather feature highlights

Collect 2–4 short feature highlights for the product:

1. **From `body_html`** — if it carries a real description or bullet list,
   pull the 2–4 most concrete selling points (strip HTML; ignore the fitment
   table).
2. **Sparse-description fallback** — many catalog products carry only a
   fitment table. Derive 2–3 category-true highlights from `vendor`,
   `product_type`, and title keywords: coilovers → adjustable ride height +
   road/track-tuned damping; intake → increased airflow + bolt-on install;
   brakes → stronger, more consistent stopping power; wheels → lightweight +
   strength; exhaust → improved flow + sound.

**Never fabricate numeric specs** — no "30-way adjustable", "+15 hp", and no
discount percentage — unless the figure literally appears in the product data.

### 7b. `caption` — social post body (Facebook / Instagram / TikTok)

Trending, scroll-stopping style: a hook, the highlights as short emoji
bullets, the value line, then the footer. Drop the `for your {vehicle}`
clause when `universal`. **The caption does NOT contain hashtags** —
`/zoho-social-batch` appends the `hashtags` field to the post message itself.

```
{hook} — {product_short_name} for your {vehicle}.

🔧 {highlight 1}
⚡ {highlight 2}
✅ {highlight 3}

{value_prop}.

Follow us for more promotions and content.
```

`hook` = a short punchy line (`Dial in your {vehicle}`, `Built to perform`,
`Upgrade day`). `value_prop` — pick ONE: "best deal on the market",
"average 2-day shipping to major US states", "built to last". Choose by
category: suspension / coilovers / sway bar / brakes → `built to last`;
otherwise → `best deal on the market`. Never use "customer-approved" or
"verified 5-star" (no review evidence here).

### 7c. `hashtags` — tags for all social posts

10–15 tags: brand tag from `vendor` (K&N → `#KNFilters`, Öhlins → `#Ohlins`),
category tag(s) from `product_type` (Intake → `#ColdAirIntake`), vehicle
tag(s) (`#Tacoma`, `#ToyotaTacoma`), then always append
`#CarCommunity #ModdedCars #AutoParts #PerformanceParts #NLPPerformance`.

### 7d. `yt_title` — YouTube Shorts title (≤100 chars)

Keyword-front-loaded and punchy:
`{product_short_name} — {vehicle} | NLP Performance` (drop the vehicle when
`universal`). Trim to ≤100 characters.

### 7e. `yt_description` — YouTube Shorts description

A fuller, trending description that highlights the features and links to the
product (hashtags ARE included here — they belong in a YouTube description):

```
{product_short_name} — upgrade your {vehicle}.

✅ {highlight 1}
✅ {highlight 2}
✅ {highlight 3}

🛒 Shop now: {product_url}

{value_prop}. Follow us for more promotions and content.

{hashtags}
```

### 7f. `first_comment`

```
Shop {product_short_name} 👇 {product_url}

Follow us for more promotions and content.
```

Before finalizing, scan all generated content and ERROR if it contains a
fabricated discount percentage or a fabricated numeric spec.

## Step 8 — Write the manifest

Write BOTH files (dual output):
- `.claude/plans/feed-post-queue-{YYYYMMDD}.md` — human-readable queue.
- `.claude/plans/feed-post-queue-{YYYYMMDD}.json` — for `/zoho-social-batch`.

The JSON uses `schedule_mode: spaced` so `/zoho-social-batch` resolves the
**next available 21:00 EDT slot** automatically — it reads the last scheduled
Zoho post and starts the day after. Do not hand-compute `schedule_at` dates.

JSON shape:
```json
{
  "schedule_mode": "spaced",
  "interval_hours": 24,
  "time_of_day": "21:00",
  "timezone": "America/New_York",
  "posts": [
    {
      "video": "/tmp/feed-media/product-{handle}/{handle}.mp4",
      "caption": "...",
      "hashtags": "#... #...",
      "first_comment": "Shop ... 👇 https://...",
      "yt_title": "... | NLP Performance",
      "yt_description": "...",
      "vehicle": "16-23 Toyota Tacoma",
      "vehicle_source": "title",
      "image_source": "shopify",
      "video_engine": "lovart"
    }
  ]
}
```

The Markdown file lists each post with vehicle, vehicle/image source, video
path, caption, hashtags, first comment, YouTube Shorts title, and YouTube
description.

`caption`, `hashtags`, and `yt_description` are kept as separate fields
because `/zoho-social-batch` composes them per channel: the social post
message is `caption + "\n\n" + hashtags`, so **`caption` must not already
contain hashtags**. `yt_description` is used verbatim as the YouTube Shorts
description and already includes hashtags (Step 7e).

## Step 9 — Present for approval

Display the Markdown queue and ask the user:
"N posts generated. Review `.claude/plans/feed-post-queue-{YYYYMMDD}.md`. Approve
all, or specify post numbers (e.g. `approve 1,3,5-10`)."

## Step 10 — Schedule via /zoho-social-batch

On approval, run:
```
/zoho-social-batch .claude/plans/feed-post-queue-{YYYYMMDD}.json
```
The manifest's `schedule_mode: spaced` makes `/zoho-social-batch` place each
post in the next free 9 PM EDT slot (1/day), starting the day after the last
already-scheduled Zoho post. Never publish immediately; never use
"Add to Queue".

## Caveats

- **Video cost:** each product triggers one Lovart generation via
  `/lovart-video`; the cost is surfaced by Lovart's `confirm` gate, not a
  fixed table. The approval step (Step 9) is the cost checkpoint.
- **Universal products:** when no vehicle is found, captions/prompts/titles
  drop the vehicle clause gracefully.
- **Partial failures:** a product that fails video generation or media
  gathering is skipped and noted in the run summary; the run continues.
- **Web-search budget:** strict depth-1 — max 1 `WebSearch` + 3 `WebFetch`
  per product in Step 4b.

## Related skills

- `/lovart-video` — the video generation building block this skill calls.
- `/generate-posts-from-reviews` — Yotpo-review-driven counterpart.
- `/zoho-social-batch` — downstream batch scheduler.
