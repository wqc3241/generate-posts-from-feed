# Generate Posts From Feed — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build two global Claude Code command-skills — `/lovart-video` (a reusable OpenClaw `lovart-skill` wrapper) and `/generate-posts-from-feed` (Shopify product link → vehicle-aware 8s Lovart promo video → social post content → `/zoho-social-batch` manifest) — each in its own GitHub repo, symlinked into `~/.claude/commands/`.

**Architecture:** Each skill is a single self-contained Markdown command file (the established pattern: see `~/.claude/commands/generate-posts-from-reviews.md`). Claude executes the embedded bash/curl/python instructions when the slash command is invoked. `/generate-posts-from-feed` calls `/lovart-video` for the video step. Source of truth lives in two git repos under `~/gitrepos/`; the live copies in `~/.claude/commands/` are symlinks.

**Tech Stack:** Markdown skill files; bash, `curl`, `jq`, `python3` + Pillow, `ffmpeg`; OpenClaw CLI `v2026.5.7` + `lovart-skill`; `gh` CLI (account `wqc3241`).

**Verification note:** These deliverables are Markdown instruction files, not a code library — there is no pytest suite. Each task's "verify" steps use concrete shell checks (file exists, JSON/markdown well-formed, smoke-test of a non-paid code path). A full paid end-to-end run (real Lovart generation + Zoho scheduling) is explicitly a **user-gated manual step** (Task 14), never run automatically.

## Plan Amendments (post-Task-1, 2026-05-18)

Task 1 probed `lovartai/lovart-skill` and found the plan's Lovart assumptions wrong. **These amendments override Task 2, Task 4, and Task 11 wherever they conflict:**

1. **No `openclaw skills list` / `npx skills add` reliance.** `npx skills add lovartai/lovart-skill` registers the skill *project-local* as `lovart-api` under `.agents/skills/`, does **not** appear in `openclaw skills list`, and — critically — **does not copy the `agent_skill.py` entrypoint**. It is unusable for a global skill.
2. **`/lovart-video` bootstraps `agent_skill.py` itself.** The entrypoint is a single self-contained ~1000-line Python CLI. `/lovart-video` preflight fetches it once to a stable cache: `~/.cache/lovart-video/agent_skill.py` from `https://raw.githubusercontent.com/lovartai/lovart-skill/main/skills/lovart-skill/agent_skill.py`. No `npx`, no `openclaw skills` calls.
3. **Reference flag is `--attachments <URL>...`** (CDN URLs, not local paths). Flow: `python3 agent_skill.py upload --file <localpath> --json` → read `.url` → pass URLs to `chat --attachments <url1> <url2> ...`.
4. **Confirmed `agent_skill.py` interface:** subcommands include `chat upload confirm result status download projects ...`. `chat` flags: `--prompt --mode {fast,thinking} --attachments --prefer-models --json --download --output-dir --thread-id --project-id`. JSON output keys: `upload` → `result["url"]`; `chat`/`result` downloaded video → `result["downloaded"][i]["local_path"]`; success → `result["generation_succeeded"]`; high-cost gate → `result["final_status"] == "pending_confirmation"`.
5. **Task 11 `.gitignore`** for `generate-posts-from-feed` must also ignore `.agents/` and `skills-lock.json` (npx-installer side-effects).

The implementer dispatches for Tasks 2, 4, and 11 carry the corrected content.

**Pre-existing state (already done during brainstorming):**
- `~/gitrepos/lovart-video/` and `~/gitrepos/generate-posts-from-feed/` exist, each `git init`-ed on `main`.
- The design spec is committed in `generate-posts-from-feed` at `docs/superpowers/specs/2026-05-18-generate-posts-from-feed-design.md`.
- `LOVART_ACCESS_KEY` and `LOVART_SECRET_KEY` are set in `~/.openclaw/.env`.

---

## File Structure

**Repo `~/gitrepos/lovart-video/`:**
- `lovart-video.md` — the skill (repo root). Sole responsibility: prompt + refs → video file.
- `README.md` — what the skill does, install instructions.
- `LICENSE` — MIT.
- `.gitignore` — ignore OS cruft.

**Repo `~/gitrepos/generate-posts-from-feed/`:**
- `generate-posts-from-feed.md` — the skill (repo root). Sole responsibility: orchestrate feed → posts pipeline.
- `README.md` — what the skill does, install instructions, dependency on `/lovart-video`.
- `LICENSE` — MIT.
- `.gitignore` — ignore OS cruft.
- `docs/superpowers/specs/2026-05-18-generate-posts-from-feed-design.md` — already present.
- `docs/superpowers/plans/2026-05-18-generate-posts-from-feed.md` — this file.

**Live install (symlinks, created by Tasks 7 and 13):**
- `~/.claude/commands/lovart-video.md` → `~/gitrepos/lovart-video/lovart-video.md`
- `~/.claude/commands/generate-posts-from-feed.md` → `~/gitrepos/generate-posts-from-feed/generate-posts-from-feed.md`

---

## Phase 1 — `/lovart-video`

### Task 1: Install and probe the OpenClaw `lovart-skill`

This task gathers the concrete facts the `/lovart-video` skill file needs: the entrypoint path and the exact `chat` output shape. No files created — it produces notes used in Task 2.

**Files:** none (investigation only).

- [ ] **Step 1: Install the lovart-skill**

Run:
```bash
npx -y skills add lovartai/lovart-skill 2>&1 | tail -20
```
Expected: success message; the skill registers as `lovart-skill`.

- [ ] **Step 2: Confirm registration**

Run:
```bash
openclaw skills list 2>/dev/null | grep -i lovart
```
Expected: a row containing `lovart-skill` with status `ready`.

- [ ] **Step 3: Locate the `agent_skill.py` entrypoint**

Run:
```bash
find ~/.openclaw ~/.claude -name 'agent_skill.py' -path '*lovart*' 2>/dev/null
```
Expected: one path, e.g. `~/.openclaw/plugin-skills/lovart-skill/agent_skill.py`.
Record this path — Task 2's skill file uses this exact `find` command at runtime so it stays robust to install location.

- [ ] **Step 4: Capture the help output**

Run:
```bash
SKILL=$(find ~/.openclaw ~/.claude -name 'agent_skill.py' -path '*lovart*' 2>/dev/null | head -1)
python3 "$SKILL" chat --help 2>&1 | tail -40
```
Expected: the `chat` subcommand's flags. Confirm `--prompt`, `--json`, `--download`, `--mode`, `--prefer-models`, `--ref`/`--image` (note the exact reference-file flag name — the skill file in Task 2 must use whatever this prints). If a separate `upload` subcommand exists, also run `python3 "$SKILL" upload --help`.

- [ ] **Step 5: Record findings**

Append a short note to the bottom of the design spec under a new `## Implementation notes` heading: the resolved entrypoint path, the exact reference-file flag name, and the `--json` output structure (which key holds the downloaded file path). Commit:
```bash
cd ~/gitrepos/generate-posts-from-feed
git add docs/superpowers/specs/2026-05-18-generate-posts-from-feed-design.md
git commit -m "docs: record lovart-skill entrypoint and chat output shape"
```

---

### Task 2: Write the `/lovart-video` skill file

**Files:**
- Create: `~/gitrepos/lovart-video/lovart-video.md`

- [ ] **Step 1: Write the skill file**

Create `~/gitrepos/lovart-video/lovart-video.md` with this exact content (substitute the reference-file flag name confirmed in Task 1 Step 4 wherever `<REF_FLAG>` appears):

````markdown
# /lovart-video

Generate a video from a text prompt (and optional reference images) using the
OpenClaw `lovart-skill`. Returns the path to a finished video file.

This is a low-level building block — it knows nothing about products, stores,
or social posts. Other skills call it for their video step.

## Usage

```
/lovart-video --prompt "<description>" [--ref <path>]... [--out <path>] [--mode fast|thinking] [--prefer-models '<json>']
```

| Arg | Required | Default | Meaning |
|-----|----------|---------|---------|
| `--prompt` | yes | — | Video description. State duration and aspect ratio here in plain words (Lovart has no flags for them), e.g. "8-second vertical 9:16 clip". |
| `--ref` | no | — | Local image/video reference file. Repeatable. |
| `--out` | no | `/tmp/lovart-video/<timestamp>.mp4` | Destination path for the finished video. |
| `--mode` | no | `fast` | Lovart reasoning depth: `fast` or `thinking`. |
| `--prefer-models` | no | — | JSON soft model preference, e.g. `{"VIDEO":["generate_video_kling_v3"]}`. |

## Workflow

### Step 1 — Preflight

```bash
# 1. Locate the lovart-skill entrypoint.
SKILL=$(find ~/.openclaw ~/.claude -name 'agent_skill.py' -path '*lovart*' 2>/dev/null | head -1)
if [ -z "$SKILL" ]; then
  echo "lovart-skill not installed — installing..."
  npx -y skills add lovartai/lovart-skill
  SKILL=$(find ~/.openclaw ~/.claude -name 'agent_skill.py' -path '*lovart*' 2>/dev/null | head -1)
fi
[ -z "$SKILL" ] && { echo "ERROR: lovart-skill entrypoint not found after install."; exit 1; }

# 2. Verify credentials are present.
set -a; [ -f ~/.openclaw/.env ] && . ~/.openclaw/.env; set +a
if [ -z "$LOVART_ACCESS_KEY" ] || [ -z "$LOVART_SECRET_KEY" ]; then
  echo "ERROR: LOVART_ACCESS_KEY / LOVART_SECRET_KEY not set in ~/.openclaw/.env or shell."
  exit 1
fi
```

If preflight fails, STOP and tell the user exactly which prerequisite is
missing. Do not attempt generation.

### Step 2 — Upload references

For each `--ref` path, upload it and collect the returned CDN URL:

```bash
python3 "$SKILL" upload --file "<ref_path>" --json
```

Parse the JSON for the CDN URL. Collect all URLs into `$REF_URLS`. If there
are no `--ref` args, skip this step.

### Step 3 — Generate

```bash
python3 "$SKILL" chat \
  --prompt "<prompt text>. Reference images: <REF_URLS joined by spaces>" \
  --mode "<mode>" \
  --json --download
```

Add `--prefer-models '<json>'` only if the caller passed it. The `chat`
command sends the prompt, waits for completion, and downloads artifacts.

### Step 4 — Confirm high-cost operations

If the Step 3 response indicates a confirmation is required (a
`needs_confirmation` / `confirm` status with a thread id and a cost figure):

1. Report the cost figure to the user in your text output.
2. Run the confirm command (the user invoked `/lovart-video` intentionally):
   ```bash
   python3 "$SKILL" confirm --thread-id "<thread_id>" --json
   ```
3. Then poll for the result:
   ```bash
   python3 "$SKILL" result --thread-id "<thread_id>" --json --download
   ```

### Step 5 — Resolve the output file

Parse the final `--json` payload for the downloaded video artifact path
(the key identified in the design spec's Implementation notes). Then:

```bash
mkdir -p "$(dirname "<out_path>")"
mv "<downloaded_path>" "<out_path>"
echo "<out_path>"
```

Print the absolute output path on the LAST line of your response so callers
can capture it. If generation failed, print an error and exit non-zero —
never print a fake path.

## Notes

- Duration/aspect ratio must be in the `--prompt` text — there are no flags.
- Credentials live in `~/.openclaw/.env` (`LOVART_ACCESS_KEY`, `LOVART_SECRET_KEY`).
- Each call may incur Lovart cost; the `confirm` gate surfaces it.
````

- [ ] **Step 2: Verify the file is well-formed**

Run:
```bash
test -f ~/gitrepos/lovart-video/lovart-video.md && head -1 ~/gitrepos/lovart-video/lovart-video.md
```
Expected: prints `# /lovart-video`.

- [ ] **Step 3: Confirm no dangling placeholders**

Run:
```bash
grep -nE 'TBD|TODO|FIXME|<REF_FLAG>' ~/gitrepos/lovart-video/lovart-video.md || echo "clean"
```
Expected: `clean`. (If `<REF_FLAG>` still appears, replace it with the real flag name from Task 1 Step 4.)

- [ ] **Step 4: Commit**

```bash
cd ~/gitrepos/lovart-video
git add lovart-video.md
git commit -m "feat: add /lovart-video skill"
```

---

### Task 3: Scaffold the `lovart-video` repo metadata

**Files:**
- Create: `~/gitrepos/lovart-video/README.md`
- Create: `~/gitrepos/lovart-video/LICENSE`
- Create: `~/gitrepos/lovart-video/.gitignore`

- [ ] **Step 1: Write `.gitignore`**

Create `~/gitrepos/lovart-video/.gitignore`:
```
.DS_Store
*.log
/tmp/
```

- [ ] **Step 2: Write `LICENSE`**

Create `~/gitrepos/lovart-video/LICENSE` — standard MIT License text, copyright line:
```
MIT License

Copyright (c) 2026 Qichao Wang
```
(Use the full standard MIT body after the copyright line.)

- [ ] **Step 3: Write `README.md`**

Create `~/gitrepos/lovart-video/README.md`:
```markdown
# /lovart-video — Claude Code Skill

A reusable Claude Code command-skill that generates a video from a text
prompt (and optional reference images) via the OpenClaw `lovart-skill`.

It is a low-level building block: it knows nothing about products or social
posts, so any skill that needs an AI-generated video can call it.

## Requirements

- OpenClaw CLI installed, with the `lovart-skill` (`npx skills add lovartai/lovart-skill`).
- `LOVART_ACCESS_KEY` and `LOVART_SECRET_KEY` set in `~/.openclaw/.env`.

## Install

Symlink the skill into your Claude Code commands directory:

```bash
ln -sf "$(pwd)/lovart-video.md" ~/.claude/commands/lovart-video.md
```

## Usage

```
/lovart-video --prompt "8-second vertical 9:16 product clip of ..." --ref ./photo.jpg --out ./out.mp4
```

See `lovart-video.md` for the full argument reference.
```

- [ ] **Step 4: Verify all three files exist**

Run:
```bash
ls ~/gitrepos/lovart-video/{README.md,LICENSE,.gitignore}
```
Expected: all three paths listed, no errors.

- [ ] **Step 5: Commit**

```bash
cd ~/gitrepos/lovart-video
git add README.md LICENSE .gitignore
git commit -m "chore: add repo metadata (README, LICENSE, gitignore)"
```

---

### Task 4: Install and smoke-test `/lovart-video`

**Files:**
- Create (symlink): `~/.claude/commands/lovart-video.md`

- [ ] **Step 1: Create the symlink**

Run:
```bash
ln -sf ~/gitrepos/lovart-video/lovart-video.md ~/.claude/commands/lovart-video.md
ls -l ~/.claude/commands/lovart-video.md
```
Expected: a symlink pointing at the repo file.

- [ ] **Step 2: Smoke-test the preflight block (no paid generation)**

Run the Step 1 preflight block from the skill verbatim:
```bash
SKILL=$(find ~/.openclaw ~/.claude -name 'agent_skill.py' -path '*lovart*' 2>/dev/null | head -1)
set -a; [ -f ~/.openclaw/.env ] && . ~/.openclaw/.env; set +a
echo "entrypoint: ${SKILL:-MISSING}"
echo "access key set: $([ -n "$LOVART_ACCESS_KEY" ] && echo yes || echo NO)"
echo "secret key set: $([ -n "$LOVART_SECRET_KEY" ] && echo yes || echo NO)"
```
Expected: a real entrypoint path, and both keys `yes`. If anything is `MISSING`/`NO`, fix it before proceeding — do NOT continue.

- [ ] **Step 3: Commit (repo unchanged — nothing to commit)**

No commit needed; the symlink lives outside the repo. Confirm `git status` in `~/gitrepos/lovart-video` is clean.

---

### Task 5: Publish the `lovart-video` repo to GitHub

**Files:** none (remote operation).

- [ ] **Step 1: Create the GitHub repo and push**

Run:
```bash
cd ~/gitrepos/lovart-video
gh repo create lovart-video --public --source=. --remote=origin --push
```
Expected: repo created at `github.com/wqc3241/lovart-video`, `main` pushed.

- [ ] **Step 2: Verify**

Run:
```bash
gh repo view wqc3241/lovart-video --json url,defaultBranchRef -q '.url'
git -C ~/gitrepos/lovart-video status -sb
```
Expected: the repo URL prints; local `main` is up to date with `origin/main`.

---

## Phase 2 — `/generate-posts-from-feed`

### Task 6: Write the skill header and Steps 1–2 (input resolution + product identification)

**Files:**
- Create: `~/gitrepos/generate-posts-from-feed/generate-posts-from-feed.md`

- [ ] **Step 1: Write the file header and Steps 1–2**

Create `~/gitrepos/generate-posts-from-feed/generate-posts-from-feed.md` with this content:

````markdown
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

```bash
# Single/multiple product URLs:
#   extract {handle} from each .../products/{handle}
# Collection URL:
curl -s "https://nlpperformance.com/collections/{handle}/products.json?limit=250&page=1" | jq -r '.products[].handle'
#   repeat with page=2,3,... until an empty array is returned.
# Feed file (.json): jq for product handles/urls.
# Feed file (.xml): parse <link>/<g:link> entries for /products/{handle}.
# Feed file (.csv): the column holding the product URL or handle.
```

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
````

- [ ] **Step 2: Verify the file starts correctly**

Run:
```bash
head -1 ~/gitrepos/generate-posts-from-feed/generate-posts-from-feed.md
```
Expected: `# /generate-posts-from-feed`.

- [ ] **Step 3: Smoke-test the Step 2 product fetch against a real product**

Run (uses a real store endpoint, read-only, free):
```bash
curl -s "https://nlpperformance.com/collections/all/products.json?limit=1" | jq -r '.products[0].handle' \
 | xargs -I{} curl -s "https://nlpperformance.com/products/{}.json" | jq '.product.title, .product.vendor'
```
Expected: a real product title and vendor print. This confirms the `.json` endpoint shape used by Step 2.

- [ ] **Step 4: Commit**

```bash
cd ~/gitrepos/generate-posts-from-feed
git add generate-posts-from-feed.md
git commit -m "feat: add /generate-posts-from-feed skill (header + input resolution)"
```

---

### Task 7: Append Step 3 (vehicle identification) and Step 4 (reference media)

**Files:**
- Modify: `~/gitrepos/generate-posts-from-feed/generate-posts-from-feed.md` (append)

- [ ] **Step 1: Append Steps 3 and 4**

Append this content to `generate-posts-from-feed.md`:

````markdown
## Step 3 — Identify the vehicle (priority chain, first hit wins)

For each product, determine `{vehicle}` and record `vehicle_source`:

1. **Title parse** (`vehicle_source = title`). Match a year range plus a
   known make and model in `title`, e.g. `16-23 Toyota Tacoma V6-3.5L`.
   Known makes: Toyota, Ford, Honda, Subaru, Chevrolet, Chevy, Jeep, RAM,
   Dodge, GMC, Nissan, Lexus, Mazda, Volkswagen, VW, BMW, Audi, Mercedes,
   Hyundai, Kia, Ford, Tesla, Acura, Infiniti. Capture `{year_range} {make}
   {model}` (drop the engine code from `{vehicle}` but keep make+model).
2. **Fitment table in `body_html`** (`vehicle_source = fitment-table`).
   If the title yields nothing, parse `body_html` for an HTML `<table>` whose
   header row contains Year/Make/Model columns; take the make+model (if
   multiple rows, use the most frequent make, and "multiple models" or the
   single model). Use `python3` with the standard library `html.parser`.
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
````

- [ ] **Step 2: Verify the append succeeded**

Run:
```bash
grep -c '^## Step' ~/gitrepos/generate-posts-from-feed/generate-posts-from-feed.md
```
Expected: `4` (Steps 1, 2, 3, 4 present).

- [ ] **Step 3: Smoke-test the vehicle title parser**

Run:
```bash
python3 - <<'PY'
import re
makes = r"Toyota|Ford|Honda|Subaru|Chevrolet|Chevy|Jeep|RAM|Dodge|GMC|Nissan|Lexus|Mazda|Volkswagen|VW|BMW|Audi|Mercedes|Hyundai|Kia|Tesla|Acura|Infiniti"
pat = re.compile(rf"(\d{{2}}-\d{{2}})\s+({makes})\s+([A-Za-z0-9-]+)")
for t in ["K&N 16-23 Toyota Tacoma V6-3.5L Intake Kit", "Whiteline Sway Bar"]:
    m = pat.search(t)
    print(t, "->", f"{m.group(1)} {m.group(2)} {m.group(3)}" if m else "universal")
PY
```
Expected: first line resolves to `16-23 Toyota Tacoma`, second line `universal`.

- [ ] **Step 4: Commit**

```bash
cd ~/gitrepos/generate-posts-from-feed
git add generate-posts-from-feed.md
git commit -m "feat: add vehicle identification and reference media steps"
```

---

### Task 8: Append Step 5 (Lovart video generation)

**Files:**
- Modify: `~/gitrepos/generate-posts-from-feed/generate-posts-from-feed.md` (append)

- [ ] **Step 1: Append Step 5**

Append this content:

````markdown
## Step 5 — Generate the 8-second promo video

For each product, build a highlight-reel prompt and call `/lovart-video`.

**`product_short_name`** = `title` with the leading SKU/year/model prefix
stripped (e.g. "K&N 16-23 Toyota Tacoma V6-3.5L Elevated Intake Kit (Snorkel)"
→ "K&N Tacoma Elevated Intake (Snorkel)").

**`product_kind`** = a short category noun derived from `product_type`/title
(e.g. "cold air intake", "sway bar", "coilovers", "fuel pump", "exhaust").

**`ambience_cue`** by category:
- intake/exhaust → `engine idle with a short confident rev`
- suspension/wheels → `tire grip on pavement, distant workshop sounds`
- brakes → `brake disc rotation hiss, subtle impact clicks`
- other/accessory → `subtle workshop ambience, tools on metal`

**Prompt template** (keep under 500 chars; include the vehicle only when
`vehicle_source != universal`):

```
Cinematic 8-second vertical 9:16 product promo of {product_short_name}
{for the {vehicle}}. Open on a clean studio hero shot of just the
{product_kind} isolated on a seamless dark gradient backdrop, then transition
through 3-4 rolling close-up detail shots from multiple angles. Soft
cinematic lighting, subtle golden rim light, shallow depth of field,
photorealistic, premium confident energy. {ambience_cue}. No text overlays,
no logos, no watermarks — added in post.
```

Invoke the video skill, passing any downloaded reference images:

```
/lovart-video --prompt "<engineered prompt>" --ref /tmp/feed-media/product-{handle}/media-0.jpg [--ref ...] --out /tmp/feed-media/product-{handle}/raw.mp4
```

Capture the path printed on `/lovart-video`'s last line. If it fails for a
product, record the failure and skip that product's post (do not abort the
whole run).
````

- [ ] **Step 2: Verify**

Run:
```bash
grep -q 'Step 5 — Generate the 8-second promo video' ~/gitrepos/generate-posts-from-feed/generate-posts-from-feed.md && echo ok
```
Expected: `ok`.

- [ ] **Step 3: Commit**

```bash
cd ~/gitrepos/generate-posts-from-feed
git add generate-posts-from-feed.md
git commit -m "feat: add Lovart video generation step"
```

---

### Task 9: Append Step 6 (branding overlay — self-contained)

**Files:**
- Modify: `~/gitrepos/generate-posts-from-feed/generate-posts-from-feed.md` (append)

- [ ] **Step 1: Append Step 6**

Append this content. It is fully self-contained (no reference to `/generate-post`):

````markdown
## Step 6 — Branding overlay composite

Composite three overlays onto `raw.mp4` so they appear on frame 1 and every
frame: NLP logo (top-right), brand logo (top-left), product-name label
(bottom-center). Output `/tmp/feed-media/product-{handle}/{handle}.mp4`.

### 6a. Resolve the asset cache

```bash
mkdir -p ~/.claude/assets/brand-logos
```

**NLP logo** — `~/.claude/assets/nlp-logo.png`. If missing, fetch the store
logo once:
```bash
[ -f ~/.claude/assets/nlp-logo.png ] || \
  WebFetch "https://nlpperformance.com" "Return the URL of the site header logo image." \
  # then: curl -sL "<logo_url>" -o ~/.claude/assets/nlp-logo.png ; verify with `file`.
```
If it still cannot be obtained, skip the NLP overlay (do not abort).

**Brand logo** — slug = `vendor` lowercased, non-alphanumeric → `-`
(e.g. "K&N" → `k-n`). Path `~/.claude/assets/brand-logos/{slug}.png`.
If missing: one `WebSearch` `"{vendor} logo png"`, `WebFetch` the top result
for the image URL, `curl` it, verify with `file`. If unobtainable, skip the
brand overlay (do not abort).

### 6b. Render the product-name label PNG (Pillow)

ImageMagick is not installed — use Pillow. Word-wrap so the label never
exceeds 85% of the 720px canvas width:

```python
from PIL import Image, ImageDraw, ImageFont
text = "<product_short_name>"
canvas_w, font_size, max_lines = 720, 44, 3
max_text_w = int(canvas_w * 0.85)  # 612px
# font: try Inter Bold, fall back to Arial Bold, then PIL default.
font = None
for p in ["/Library/Fonts/Inter-Bold.ttf", "/System/Library/Fonts/Supplemental/Arial Bold.ttf"]:
    try: font = ImageFont.truetype(p, font_size); break
    except Exception: pass
if font is None: font = ImageFont.load_default()
probe = ImageDraw.Draw(Image.new("RGBA", (1, 1)))
def measure(s): b = probe.textbbox((0, 0), s, font=font); return (b[2]-b[0], b[3]-b[1])
def wrap(t, max_w, max_lines=3):
    words, lines, cur = t.split(), [], ""
    for i, w in enumerate(words):
        trial = (cur + " " + w).strip()
        if measure(trial)[0] <= max_w or not cur:
            cur = trial
        else:
            lines.append(cur); cur = w
            if len(lines) == max_lines - 1:
                rest = " ".join(words[i:])
                while measure(rest)[0] > max_w and len(rest) > 3:
                    rest = rest[:-1]
                if measure(rest)[0] > max_w: rest = rest.rstrip() + "…"
                lines.append(rest); return lines
    if cur: lines.append(cur)
    return lines
lines = wrap(text, max_text_w, max_lines)
line_h = max(measure(l)[1] for l in lines) + 8
pad = 20
img_w = canvas_w
img_h = line_h * len(lines) + pad * 2
img = Image.new("RGBA", (img_w, img_h), (0, 0, 0, 0))
d = ImageDraw.Draw(img)
y = pad
for l in lines:
    w = measure(l)[0]; x = (img_w - w) // 2
    d.text((x, y), l, font=font, fill=(255,255,255,255),
           stroke_width=3, stroke_fill=(0,0,0,255))
    y += line_h
img.save("/tmp/feed-media/product-{handle}/label.png")
```

### 6c. Composite with ffmpeg

Build the inputs and filtergraph from whichever overlays are available. With
all three present:

```bash
ffmpeg -y -i /tmp/feed-media/product-{handle}/raw.mp4 \
  -i ~/.claude/assets/nlp-logo.png \
  -i ~/.claude/assets/brand-logos/{slug}.png \
  -i /tmp/feed-media/product-{handle}/label.png \
  -filter_complex "
    [1:v]scale=180:-1[nlp];
    [2:v]scale=180:-1[brand];
    [0:v][brand]overlay=20:20[a];
    [a][nlp]overlay=W-w-20:20[b];
    [b][3:v]overlay=(W-w)/2:H-h-80[v]
  " -map "[v]" -map 0:a? -c:v libx264 -pix_fmt yuv420p -c:a copy \
  /tmp/feed-media/product-{handle}/{handle}.mp4
```

If the NLP logo is missing, drop input `-i nlp-logo.png` and the `[nlp]` /
`overlay=W-w-20:20` nodes. If the brand logo is missing, drop input
`-i brand-logos/{slug}.png` and the `[brand]` / `overlay=20:20` nodes,
re-chaining the remaining overlays. The label PNG is always present.
````

- [ ] **Step 2: Verify the Pillow snippet runs**

Run (proves the wrap/render code is valid Python and produces a PNG):
```bash
mkdir -p /tmp/feed-media/product-smoketest
python3 - <<'PY'
text="DeatschWerks DW430C 430LPH Compact Fuel Pump"
# paste the Step 6b script body, with the save path /tmp/feed-media/product-smoketest/label.png
PY
file /tmp/feed-media/product-smoketest/label.png
```
Expected: `label.png` reported as PNG image data. Fix the snippet if Python errors.

- [ ] **Step 3: Verify the ffmpeg filtergraph syntax**

Run:
```bash
ffmpeg -hide_banner -f lavfi -i color=c=black:s=720x1280:d=1 \
  -f lavfi -i color=c=red:s=200x80:d=1 \
  -filter_complex "[1:v]scale=180:-1[b];[0:v][b]overlay=20:20[v]" \
  -map "[v]" -t 1 -y /tmp/feed-media/ffmpeg-smoketest.mp4 2>&1 | tail -3
test -f /tmp/feed-media/ffmpeg-smoketest.mp4 && echo "ffmpeg overlay OK"
```
Expected: `ffmpeg overlay OK`.

- [ ] **Step 4: Commit**

```bash
cd ~/gitrepos/generate-posts-from-feed
git add generate-posts-from-feed.md
git commit -m "feat: add self-contained branding overlay step"
```

---

### Task 10: Append Step 7 (post content) and Steps 8–10 (manifest, approval, schedule)

**Files:**
- Modify: `~/gitrepos/generate-posts-from-feed/generate-posts-from-feed.md` (append)

- [ ] **Step 1: Append Steps 7–10 and caveats**

Append this content:

````markdown
## Step 7 — Generate post content

Product/vehicle-led (there is no customer quote).

**Caption** (drop the "for your {vehicle}" clause when `universal`):

```
Upgrade your {vehicle} 🔧 {product_short_name} — {value_prop}.

Follow us for more promotions and content.

{hashtags}
```

- `value_prop` — pick ONE: "best deal on the market", "average 2-day
  shipping to major US states", "built to last". Do NOT use
  "customer-approved" or "verified 5-star" (no review evidence here).
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

## Step 9 — Present for approval

Display the Markdown queue and ask the user:
"N posts generated. Review `.claude/plans/feed-post-queue-{date}.md`. Approve
all, or specify post numbers (e.g. `approve 1,3,5-10`)."

## Step 10 — Schedule via /zoho-social-batch

On approval, run:
```
/zoho-social-batch .claude/plans/feed-post-queue-{date}.json
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
````

- [ ] **Step 2: Verify all 10 steps are present**

Run:
```bash
grep -E '^## Step' ~/gitrepos/generate-posts-from-feed/generate-posts-from-feed.md
```
Expected: Steps 1 through 10 listed in order.

- [ ] **Step 3: Confirm no placeholders**

Run:
```bash
grep -nE 'TBD|TODO|FIXME|XXX' ~/gitrepos/generate-posts-from-feed/generate-posts-from-feed.md || echo "clean"
```
Expected: `clean`.

- [ ] **Step 4: Commit**

```bash
cd ~/gitrepos/generate-posts-from-feed
git add generate-posts-from-feed.md
git commit -m "feat: add post content, manifest, approval, and scheduling steps"
```

---

### Task 11: Scaffold the `generate-posts-from-feed` repo metadata

**Files:**
- Create: `~/gitrepos/generate-posts-from-feed/README.md`
- Create: `~/gitrepos/generate-posts-from-feed/LICENSE`
- Create: `~/gitrepos/generate-posts-from-feed/.gitignore`

- [ ] **Step 1: Write `.gitignore`**

Create `~/gitrepos/generate-posts-from-feed/.gitignore`:
```
.DS_Store
*.log
/tmp/
```

- [ ] **Step 2: Write `LICENSE`**

Create `~/gitrepos/generate-posts-from-feed/LICENSE` — standard MIT License,
copyright `Copyright (c) 2026 Qichao Wang`.

- [ ] **Step 3: Write `README.md`**

Create `~/gitrepos/generate-posts-from-feed/README.md`:
```markdown
# /generate-posts-from-feed — Claude Code Skill

Generate vehicle-aware promotional social posts from NLP Performance store
product links. For each product it identifies the product and its fitment
vehicle, generates an 8-second Lovart promo video, brands it, writes post
content, and queues everything for `/zoho-social-batch`.

Feed-driven counterpart to `/generate-posts-from-reviews`.

## Requirements

- The `/lovart-video` skill installed (https://github.com/wqc3241/lovart-video).
- OpenClaw `lovart-skill` + `LOVART_ACCESS_KEY` / `LOVART_SECRET_KEY`.
- `jq`, `ffmpeg`, `python3` with Pillow.

## Install

```bash
ln -sf "$(pwd)/generate-posts-from-feed.md" ~/.claude/commands/generate-posts-from-feed.md
```

## Usage

```
/generate-posts-from-feed https://nlpperformance.com/products/<handle> --limit 10
```

See `generate-posts-from-feed.md` for the full argument reference, and
`docs/superpowers/specs/` for the design spec.
```

- [ ] **Step 4: Verify**

Run:
```bash
ls ~/gitrepos/generate-posts-from-feed/{README.md,LICENSE,.gitignore}
```
Expected: all three listed.

- [ ] **Step 5: Commit**

```bash
cd ~/gitrepos/generate-posts-from-feed
git add README.md LICENSE .gitignore
git commit -m "chore: add repo metadata (README, LICENSE, gitignore)"
```

---

### Task 12: Install and smoke-test `/generate-posts-from-feed`

**Files:**
- Create (symlink): `~/.claude/commands/generate-posts-from-feed.md`

- [ ] **Step 1: Create the symlink**

Run:
```bash
ln -sf ~/gitrepos/generate-posts-from-feed/generate-posts-from-feed.md ~/.claude/commands/generate-posts-from-feed.md
ls -l ~/.claude/commands/generate-posts-from-feed.md
```
Expected: symlink pointing at the repo file.

- [ ] **Step 2: Dry smoke-test Steps 1–4 on one real product (no paid calls)**

Pick one real product and run input resolution → product fetch → vehicle
parse → Shopify image download, stopping BEFORE Step 5 (Lovart). Confirm:
- a handle is resolved,
- product JSON returns a title/vendor,
- the vehicle parser returns a vehicle or `universal`,
- at least one image downloads and verifies as image data.

Expected: all four succeed. No Lovart call, no Zoho call.

- [ ] **Step 3: Verify the symlink resolves and repo is clean**

Run:
```bash
readlink ~/.claude/commands/generate-posts-from-feed.md
git -C ~/gitrepos/generate-posts-from-feed status -sb
```
Expected: symlink target prints; repo working tree clean.

---

### Task 13: Publish the `generate-posts-from-feed` repo to GitHub

**Files:** none (remote operation).

- [ ] **Step 1: Create the GitHub repo and push**

Run:
```bash
cd ~/gitrepos/generate-posts-from-feed
gh repo create generate-posts-from-feed --public --source=. --remote=origin --push
```
Expected: repo created at `github.com/wqc3241/generate-posts-from-feed`, `main` pushed (spec + plan + skill + metadata all included).

- [ ] **Step 2: Verify**

Run:
```bash
gh repo view wqc3241/generate-posts-from-feed --json url -q '.url'
git -C ~/gitrepos/generate-posts-from-feed status -sb
```
Expected: repo URL prints; local `main` up to date with `origin/main`.

---

### Task 14: User-gated full end-to-end validation

This task spends real Lovart credits and writes a real Zoho schedule — it is
**not** run automatically. Present it to the user and run only on explicit
approval.

**Files:** none (produces `/tmp/feed-media/...` artifacts + a manifest).

- [ ] **Step 1: Ask the user to approve a paid test run**

Tell the user: "Both skills are built, installed, and pushed. A full
end-to-end test will generate one real Lovart video (incurs cost) for a
single product. Approve a 1-product test run? Provide a product URL, or I can
pick one."

- [ ] **Step 2: Run a 1-product end-to-end on approval**

Run:
```
/generate-posts-from-feed <product-url> --limit 1
```
Stop at Step 9 (the manifest approval gate) — do NOT auto-approve scheduling.

- [ ] **Step 3: Verify the artifacts**

Confirm: `raw.mp4` exists, `{handle}.mp4` exists with branding overlays
visible on frame 1, and `.claude/plans/feed-post-queue-{date}.{md,json}` are
written with correct `vehicle`, `vehicle_source`, `image_source`, and
`video_engine: "lovart"` fields.

- [ ] **Step 4: Hand the scheduling decision to the user**

Show the manifest and let the user decide whether to proceed to Step 10
(`/zoho-social-batch`). Do not schedule without explicit approval.

---

## Self-Review

**Spec coverage:**
- `/lovart-video` (args, preflight, upload, generate, confirm, output) → Tasks 1–2.
- Two-repo layout, README/LICENSE, symlink install → Tasks 3–4, 11–12.
- GitHub publish on `wqc3241` → Tasks 5, 13.
- Input resolution (URL/collection/feed, single + multiple) → Task 6 Step 1.
- Product identification via `.json` → Task 6 Step 2.
- Vehicle chain (title → fitment-table → tags → universal) → Task 7.
- Reference media chain (shopify → web-search → none) → Task 7.
- Lovart video step (engineered prompt, 8s, refs) → Task 8.
- Branding overlay (NLP + brand logo + Pillow label + ffmpeg) → Task 9.
- Post content rules (caption, hashtags, first comment, YT title, no fake discounts, footer line) → Task 10.
- Manifest dual output with added fields → Task 10.
- Approval gate + `/zoho-social-batch` schedule → Task 10, Task 14.
- Spec already committed; plan committed via repo push → Task 13.

All spec sections map to a task. No gaps.

**Placeholder scan:** No TBD/TODO/FIXME in task content. `<REF_FLAG>` in
Task 2 is an explicit substitution resolved by Task 1 Step 4 and checked in
Task 2 Step 3 — not a placeholder left dangling.

**Type/name consistency:** `vehicle_source` values (`title`,
`fitment-table`, `tags`, `universal`), `image_source` values (`shopify`,
`web-search`, `none`), `product_short_name`, `product_kind`, `ambience_cue`,
and the `/tmp/feed-media/product-{handle}/` paths (`raw.mp4`, `label.png`,
`{handle}.mp4`) are used consistently across Tasks 6–10 and the manifest in
Task 10.
