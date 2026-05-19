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
({product_kind}) floating on a dark gradient backdrop. Seconds 1-4:
ultra-realistic multi-angle orbiting overview of {scene_vehicle}. Seconds 4-8:
smooth close-up multi-angle shots of the {product_kind} already installed on
{scene_vehicle} at its correct mounting location. {ambience_cue}. No text, no
logos, no watermarks — added in post. Render with the Seedance 2.0 Fast video
model.
```

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

# shop logo, centered in the top band
if os.path.isfile(logo_path):
    try:
        logo = Image.open(logo_path).convert("RGBA")
        lw = 300
        logo = logo.resize((lw, max(1, round(logo.height * lw / logo.width))))
        img.alpha_composite(logo, ((W - lw) // 2, 70))
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

**With the NLP logo:**

```bash
ffmpeg -y -i /tmp/feed-media/product-{handle}/raw.mp4 \
  -i /tmp/feed-media/product-{handle}/titlecard.png \
  -i ~/.claude/assets/nlp-logo.png \
  -filter_complex "
    [0:v]scale=720:1280:force_original_aspect_ratio=increase,crop=720:1280,setsar=1[base];
    [1:v]format=rgba,fade=t=in:st=0:d=0.2:alpha=1,fade=t=out:st=0.85:d=0.15:alpha=1[tc];
    [base][tc]overlay=0:0:enable='between(t,0,1)'[a];
    [2:v]scale=96:-2[wm];
    [a][wm]overlay=W-w-24:24[v]
  " -map "[v]" -map 0:a? -t 8 -c:v libx264 -pix_fmt yuv420p -c:a copy \
  /tmp/feed-media/product-{handle}/{handle}.mp4
```

**Without the NLP logo (Step 6a could not obtain it)** — drop the third input
and the watermark nodes; the title-card overlay output is named `[v]` directly:

```bash
ffmpeg -y -i /tmp/feed-media/product-{handle}/raw.mp4 \
  -i /tmp/feed-media/product-{handle}/titlecard.png \
  -filter_complex "
    [0:v]scale=720:1280:force_original_aspect_ratio=increase,crop=720:1280,setsar=1[base];
    [1:v]format=rgba,fade=t=in:st=0:d=0.2:alpha=1,fade=t=out:st=0.85:d=0.15:alpha=1[tc];
    [base][tc]overlay=0:0:enable='between(t,0,1)'[v]
  " -map "[v]" -map 0:a? -t 8 -c:v libx264 -pix_fmt yuv420p -c:a copy \
  /tmp/feed-media/product-{handle}/{handle}.mp4
```

## Step 7 — Generate post content

Product/vehicle-led (there is no customer quote).

**Caption** (drop the "for your {vehicle}" clause when `universal`):

```
Upgrade your {vehicle} 🔧 {product_short_name} — {value_prop}.

Follow us for more promotions and content.

{hashtags}
```

- `value_prop` — pick ONE: "best deal on the market", "average 2-day
  shipping to major US states", "built to last". Choose by category: if
  `product_type` or title indicates suspension, coilovers, sway bar, or
  brakes → `built to last`; otherwise → `best deal on the market`. Use
  `average 2-day shipping to major US states` only when the user explicitly
  asks for shipping-focused copy. Do NOT use "customer-approved" or
  "verified 5-star" (no review evidence here).
- `hashtags` — 10–15 tags: brand tag from `vendor` (e.g. K&N → `#KNFilters`,
  Whiteline → `#Whiteline`), category tag from `product_type` (e.g. Intake →
  `#ColdAirIntake`), vehicle tag(s) (`#Tacoma`, `#ToyotaTacoma`), then always
  append `#CarCommunity #ModdedCars #AutoParts #PerformanceParts #NLPPerformance`.

**First comment:**
```
Shop {product_short_name} 👇 {product_url}

Follow us for more promotions and content.
```

**YouTube title** (≤100 chars): `{product_short_name} for {vehicle} — Promo`
(drop "for {vehicle}" when `universal`).

Before finalizing, scan generated content and ERROR if it contains a
fabricated discount percentage.

## Step 8 — Write the manifest

Write BOTH files (dual output):
- `.claude/plans/feed-post-queue-{YYYYMMDD}.md` — human-readable queue.
- `.claude/plans/feed-post-queue-{YYYYMMDD}.json` — for `/zoho-social-batch`.

JSON shape:
```json
{
  "schedule_mode": "scheduled",
  "posts": [
    {
      "schedule_at": "2026-05-19 21:00 EDT",
      "video": "/tmp/feed-media/product-{handle}/{handle}.mp4",
      "caption": "...",
      "hashtags": "#... #...",
      "first_comment": "Shop ... 👇 https://...",
      "yt_title": "... — Promo",
      "vehicle": "16-23 Toyota Tacoma",
      "vehicle_source": "title",
      "image_source": "shopify",
      "video_engine": "lovart"
    }
  ]
}
```

The Markdown file lists each post with schedule time, vehicle, vehicle/image
source, video path, caption, first comment, and YouTube title.

The `caption` field contains the full caption text including hashtags (as
built in Step 7). The separate `hashtags` field is a convenience copy for
`/zoho-social-batch`; do not strip hashtags out of `caption`.

## Step 9 — Present for approval

Display the Markdown queue and ask the user:
"N posts generated. Review `.claude/plans/feed-post-queue-{YYYYMMDD}.md`. Approve
all, or specify post numbers (e.g. `approve 1,3,5-10`)."

## Step 10 — Schedule via /zoho-social-batch

On approval, run:
```
/zoho-social-batch .claude/plans/feed-post-queue-{YYYYMMDD}.json
```
with `schedule_mode: scheduled` so each post lands at its `schedule_at`.
Cadence is 1/day at 9 PM EDT, starting the day after the last already-
scheduled Zoho post. Never publish immediately; never use "Add to Queue".

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
