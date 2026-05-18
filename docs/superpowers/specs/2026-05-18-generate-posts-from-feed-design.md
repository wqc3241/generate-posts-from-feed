# Design: `/generate-posts-from-feed` + `/lovart-video`

**Date:** 2026-05-18
**Status:** Approved design — ready for implementation plan
**Repo homes:** two dedicated single-skill repos — `generate-posts-from-feed`
and `lovart-video` (matching the existing `claude-init-project` pattern).

## Summary

Two new global, user-level Claude Code command-skills for the NLP Performance
store (`nlpperformance.com`, Shopify):

1. **`/lovart-video`** — a reusable wrapper that turns a prompt (+ optional
   reference images) into a finished video file, driven by the OpenClaw
   `lovart-skill`. Fully generic; no store-specific logic.
2. **`/generate-posts-from-feed`** — the main skill. Given product link(s)
   from the store, it identifies the product and its corresponding vehicle,
   generates an 8-second vehicle-aware promotional video via `/lovart-video`,
   generates full social post content, and writes a manifest for
   `/zoho-social-batch` to schedule.

This is the feed-driven counterpart to the existing
`/generate-posts-from-reviews` skill. Where that skill scrapes Yotpo customer
reviews and renders video via fal.ai Seedance, this skill takes product links
directly and renders video via Lovart (through OpenClaw).

## Background and constraints

- "Commands" in this environment *are* skills — flat self-contained `.md`
  files installed under `~/.claude/commands/`, invocable from any project
  (global / user-level). Existing examples: `generate-posts-from-reviews`,
  `generate-post`, `zoho-social-post`, `zoho-social-batch`.
- The parent skill `/generate-posts-from-reviews` lives at
  `~/.claude/commands/generate-posts-from-reviews.md`. It is a live revenue
  skill and must not be modified.
- `~/.claude` itself is not version-controlled. The established pattern for
  publishing a global skill is a **dedicated GitHub repo per skill** — e.g.
  `github.com/wqc3241/claude-init-project` holds one skill (`init-project.md`)
  plus a README and LICENSE. The two new skills follow this: one repo each.
- Lovart integration facts (verified 2026-05-18):
  - There is **no official Lovart MCP server or CLI**.
  - There **is** an OpenClaw skill `lovart-skill`, installed via
    `npx skills add lovartai/lovart-skill`. It registers as `lovart-skill`
    and auto-discovers in OpenClaw.
  - The skill ships a Python entrypoint (`agent_skill.py`) runnable directly:
    `python3 agent_skill.py chat --prompt "..." --json --download`.
  - Commands: `chat` (send + wait), `send`, `watch`, `result`, `status`,
    `confirm` (confirm high-cost ops), `upload`, `download`, plus project
    management (`projects`, `project-add`, ...).
  - Auth: requires **both** `LOVART_ACCESS_KEY` (`ak_…`) and
    `LOVART_SECRET_KEY` (`sk_…`) as environment variables. Both are now set
    in `~/.openclaw/.env` (private, git-ignored).
  - **Duration and aspect ratio are expressed in natural language inside the
    prompt** — Lovart exposes no flags for them. Optional `--prefer-models`
    (e.g. `{"VIDEO":["generate_video_kling_v3"]}`) and `--mode fast|thinking`.
  - OpenClaw CLI `v2026.5.7` is installed at
    `~/.nvm/versions/node/v24.14.0/bin/openclaw`.
- `gh` is authenticated as `wqc3241`. Both new repos will be created on that
  account and pushed.

## Architecture

### Repository layout (two repos)

Each skill gets its own repo, mirroring `claude-init-project`:

```
generate-posts-from-feed/
  generate-posts-from-feed.md          # the skill (repo root)
  README.md
  LICENSE
  docs/superpowers/specs/
    2026-05-18-generate-posts-from-feed-design.md   (this file)

lovart-video/
  lovart-video.md                      # the skill (repo root)
  README.md
  LICENSE
```

The skill `.md` files are the source of truth in their repos and are
installed into `~/.claude/commands/` via **symlink**, so edits in a repo stay
in sync with the live global skill. This design spec lives in the
`generate-posts-from-feed` repo (the primary deliverable); the `lovart-video`
README links to it.

### Unit 1 — `/lovart-video`

**Purpose:** prompt + optional reference media → one finished video file.
Single clear responsibility; no knowledge of products, vehicles, or social
posts. Reusable by `/generate-posts-from-feed` and, later, `/generate-post`.

**Arguments:**

| Arg | Required | Default | Meaning |
|-----|----------|---------|---------|
| `--prompt "<text>"` | yes | — | Video description. Duration + aspect ratio stated in this text. |
| `--ref <path>` | no | — | Local image/video reference. Repeatable. |
| `--out <path>` | no | temp path | Destination for the finished video. |
| `--mode fast\|thinking` | no | `fast` | Lovart reasoning depth. |
| `--prefer-models '<json>'` | no | — | Passthrough soft model preference. |

**Flow:**

1. **Preflight.**
   - Confirm `lovart-skill` is installed (`openclaw skills list`). If missing,
     run `npx skills add lovartai/lovart-skill`.
   - Confirm `LOVART_ACCESS_KEY` and `LOVART_SECRET_KEY` are set (shell env or
     `~/.openclaw/.env`). If either is missing, stop and ask the user — do not
     attempt generation.
   - Locate the installed skill's `agent_skill.py` entrypoint.
2. **Upload references.** For each `--ref`, run the skill's `upload` command;
   collect the returned CDN URLs.
3. **Generate.** Run `chat --prompt "<prompt + ref URL references>" --json
   --download --mode <mode>` (with `--prefer-models` if given).
4. **Confirm high-cost.** If Lovart returns a confirmation-required response,
   run `confirm` for the thread (the user invoked generation intentionally)
   and capture the reported cost.
5. **Output.** Parse the `--json` result for the downloaded artifact path,
   move it to `--out`, and print the absolute path plus any reported cost.

**Output contract:** prints the absolute path of the finished video on the
last line so callers can capture it. Non-zero exit on failure.

### Unit 2 — `/generate-posts-from-feed`

**Purpose:** product link(s) → vehicle-aware 8s promo videos + post content +
`/zoho-social-batch` manifest.

**Arguments:**

| Arg | Default | Meaning |
|-----|---------|---------|
| positional input | — | One or more product URLs, a collection URL, or a feed file path/URL. |
| `--limit N` | all | Cap the number of products processed. |
| `--video-duration N` | `8` | Target video length in seconds. |
| `--aspect <ratio>` | `9:16` | Video aspect ratio. |
| `--schedule-start <date>` | next free day | First post date. |
| `--schedule-interval <hours>` | `24` | Hours between posts. |
| `--schedule-time <HH:MM>` | `21:00` | Time of day (9:00 PM EDT). |

**Workflow:**

**Step 1 — Resolve input to a product list.**
- `…/products/{handle}` → that product. Multiple URLs → multiple products.
- `…/collections/{handle}` → fetch `{collection_url}/products.json`,
  paginated, → all products in the collection.
- A feed file/URL (`.xml` / `.csv` / `.json`) → parse product handles/URLs.
- Dedupe; apply `--limit`.

**Step 2 — Identify each product.** Fetch
`https://nlpperformance.com/products/{handle}.json` → `title`, `vendor`,
`product_type`, `tags`, `body_html`, `images[]`, `handle`.

**Step 3 — Identify the vehicle (priority chain, first hit wins).**
1. **Title parse** — regex for year-range + make + model + engine
   (e.g. `16-23 Toyota Tacoma V6-3.5L`). Known-make list (Toyota, Ford,
   Honda, Subaru, Chevrolet, Jeep, RAM, GMC, Nissan, Dodge, ...).
2. **Fitment table in `body_html`** — parse embedded compatibility `<table>`
   (Year / Make / Model columns); take the make/model.
3. **Product `tags[]`** — scan tags for vehicle make/model values.
4. **Else** — `universal` (caption + prompt go vehicle-agnostic).
   Record `vehicle_source` (`title` | `fitment-table` | `tags` | `universal`).

**Step 4 — Gather reference media (priority chain, first hit wins).**
1. Shopify product `images[]` — download the first 1–3.
2. Depth-1 web-search fallback (1 `WebSearch` + ≤3 `WebFetch`) if the product
   has no images — identical budget rules to the parent skill.
3. None — Lovart prompt runs text-only.
   Record `image_source` (`shopify` | `web-search` | `none`).

**Step 5 — Generate the 8s promo video** via `/lovart-video`.
Engineer a highlight-reel prompt per product: names `product_short_name` and
the vehicle, specifies "8-second vertical 9:16," includes a category-based
`ambience_cue`, and includes the negative-overlay clause ("no text overlays,
no logos — added in post"). Downloaded product images are passed as `--ref`.
Output to `/tmp/feed-media/product-{handle}/raw.mp4`.

**Step 6 — Branding overlay composite.** Identical ffmpeg overlay step from
`/generate-posts-from-reviews` Step 9: NLP logo top-right, brand logo (from
`vendor` slug) top-left, word-wrapped product-name text bottom-center via
Pillow. Produces the final `{handle}.mp4`. Reuses the shared asset cache
`~/.claude/assets/nlp-logo.png`, `~/.claude/assets/brand-logos/{slug}.png`,
and Inter Bold font.

**Step 7 — Generate post content** (product/vehicle-led; no customer quote).
- Caption:
  ```
  Upgrade your {vehicle} 🔧 {product_short_name} — {value_prop}.

  Follow us for more promotions and content.

  {hashtags}
  ```
  For `universal` products the vehicle clause is dropped.
- `value_prop` — one of: "best deal on the market", "average 2-day shipping
  to major US states", "built to last". ("customer-approved" / "verified
  5-star" are NOT used here — there is no review evidence.)
- Hashtags (10–15): brand tag from `vendor`, category tag from
  `product_type`, vehicle tag(s), plus the fixed set
  `#CarCommunity #ModdedCars #AutoParts #PerformanceParts #NLPPerformance`.
- First comment: `Shop {product_short_name} 👇 {product_url}` followed by
  `Follow us for more promotions and content.`
- YouTube title (≤100 chars): `{product_short_name} for {vehicle} — Promo`.

**Step 8 — Write manifest.** Dual output (same as parent):
`.claude/plans/feed-post-queue-{YYYYMMDD}.md` (human review) and `.json`
(for `/zoho-social-batch`). The JSON per-post object adds: `vehicle`,
`vehicle_source`, `image_source`, `video_engine: "lovart"`.

**Step 9 — Approval gate.** Display the Markdown queue; user approves all or
a subset (`approve 1,3,5-10`).

**Step 10 — Schedule.** On approval, run
`/zoho-social-batch .claude/plans/feed-post-queue-{date}.json` with
`schedule_mode: scheduled` — 1 post/day at 9:00 PM EDT, starting the day after
the last already-scheduled post. Parent skill's strict schedule rules apply.

## Content rules carried over from `/generate-posts-from-reviews`

- **Never fabricate discount percentages.**
- Always append `Follow us for more promotions and content.` to every caption
  (between body and hashtags) and every first comment.
- `product_short_name` strips the SKU year/model prefix from the title.
- Brand-logo cache and download flow, font resolution, vendor slug rules, and
  the ffmpeg overlay command are reused verbatim.

## Differences from the parent skill

| Aspect | `/generate-posts-from-reviews` | `/generate-posts-from-feed` |
|--------|-------------------------------|------------------------------|
| Input | Yotpo review scrape (Playwright, 2FA) | Product link(s) / collection / feed |
| PII handling | Strict stripping required | Not applicable (no customer data) |
| Caption basis | Customer review quote | Product + vehicle |
| Video engine | fal.ai Seedance (HTTP) | Lovart via OpenClaw `lovart-skill` |
| Video duration | 5s (8s for reference reels) | 8s |
| Cost reporting | Fixed per-endpoint table | From Lovart's `confirm` step |

## Out of scope

- Modifying `/generate-posts-from-reviews`.
- Migrating `/generate-post` to Lovart (future; `/lovart-video` makes it easy).
- Configuring a Lovart provider inside OpenClaw's `capability video` system
  (the `lovart-skill` path is used instead).

## Delivery

- Two new GitHub repos are created on the `wqc3241` account via `gh repo
  create` and pushed: `generate-posts-from-feed` and `lovart-video`.
- Each repo contains its skill `.md` at the repo root, a README, and a
  LICENSE. The `generate-posts-from-feed` repo additionally holds this spec.
- Each skill `.md` is symlinked into `~/.claude/commands/` for live global use.

## Implementation notes

_Recorded by Task 1 probe (2026-05-18). These are facts observed from the actual
installed skill — not inferred from docs._

### Install outcome

`npx -y skills add lovartai/lovart-skill` succeeded but registered the skill as
**`lovart-api`** (not `lovart-skill`), installed to:

```
./.agents/skills/lovart-api/
```

Only markdown files were copied by the installer (README.md, README_CN.md,
README_JA.md, README_TW.md, SKILL.md). The Python client `agent_skill.py` was
**not** copied despite being present in the GitHub repo at
`skills/lovart-skill/agent_skill.py`. It was manually downloaded from:

```
https://raw.githubusercontent.com/lovartai/lovart-skill/main/skills/lovart-skill/agent_skill.py
```

`openclaw skills list` returned no output (empty — `lovart-api` is a project-level
skill installed to `.agents/skills/`, not a global openclaw plugin-skill).

### Resolved entrypoint path

```
/Volumes/Storage/gitrepos/generate-posts-from-feed/.agents/skills/lovart-api/agent_skill.py
```

In the SKILL.md, `{baseDir}` resolves to the skill directory:
`.agents/skills/lovart-api/` relative to the project root.

For the `/lovart-video` skill file (which lives in its own repo), the entrypoint
will be wherever `lovart-api` is installed in that repo:
`.agents/skills/lovart-api/agent_skill.py`.

### `chat` subcommand — exact flags

```
python3 agent_skill.py chat
  --prompt PROMPT            (required)
  --project-id PROJECT_ID
  --thread-id THREAD_ID
  --attachments [URL ...]    (reference images/videos, CDN URLs)
  --json
  --download
  --output-dir OUTPUT_DIR
  --prefer-models JSON       e.g. '{"VIDEO":["generate_video_kling_3_0"]}'
  --include-tools [TOOL ...] e.g. upscale_image
  --exclude-tools [TOOL ...]
  --mode {thinking,fast}
```

**There is no `--ref`, `--image`, or `--images` flag.** Reference files are
passed as CDN URLs via `--attachments` after uploading with the `upload` subcommand.

### `upload` subcommand — exact invocation and JSON output

```bash
python3 agent_skill.py upload --file /path/to/file.png
```

Returns (printed to stdout):
```json
{"url": "https://assets-persist.lovart.ai/img/{user_uuid}/xxx.png"}
```

The CDN URL is in the top-level `"url"` key.

### `chat --json` output structure

```json
{
  "thread_id": "xxx",
  "status": "done",
  "project_id": "xxx",
  "final_status": "done",
  "generation_succeeded": true,
  "items": [
    {"type": "assistant", "text": "Agent's reply"},
    {"type": "generator", "name": "artifacts", "artifacts": [
      {"type": "image", "content": "https://assets-persist.lovart.ai/artifacts/agent/xxx.png"},
      {"type": "video", "content": "https://assets-persist.lovart.ai/artifacts/agent/xxx.mp4"}
    ]}
  ],
  "downloaded": [
    {"type": "video", "url": "https://...", "local_path": "/tmp/openclaw/lovart_ab12cd.mp4", "new": true}
  ]
}
```

Key facts for Task 2:
- Downloaded video path: `result["downloaded"][i]["local_path"]` (string, absolute path)
- CDN URL of artifact: `result["items"][i]["artifacts"][j]["content"]`
- Success check: `result["generation_succeeded"]` (bool; `False` + `result["warning"]` on silent failure)
- High-cost gate: `result["final_status"] == "pending_confirmation"` → must call `confirm --thread-id`

### Top-level subcommands (from `--help`)

`chat`, `send`, `watch`, `create-project`, `upload-artifact`, `upload`,
`confirm`, `set-mode`, `query-mode`, `status`, `result`, `download`, `config`,
`projects`, `project-add`, `project-switch`, `project-rename`, `project-remove`,
`threads`, `thread-remove`

### Action required for Task 2

When writing the `/lovart-video` skill, the entrypoint invocation pattern is:

```bash
python3 {baseDir}/agent_skill.py chat --prompt "..." --attachments CDN_URL [...] \
  --json --download --output-dir /tmp/openclaw
```

where `{baseDir}` is replaced by openclaw with the absolute path to the skill
directory at runtime. Reference images must be uploaded first (`upload --file`)
to obtain CDN URLs, then passed to `--attachments`.
