# /generate-posts-from-feed — Claude Code Skill

Generate vehicle-aware promotional social posts from NLP Performance store
product links. For each product it identifies the product and its fitment
vehicle, generates an 8-second Lovart promo video, brands it, writes post
content, and queues everything for `/zoho-social-batch`.

Feed-driven counterpart to `/generate-posts-from-reviews`.

## Requirements

- The `/lovart-video` skill installed (https://github.com/wqc3241/lovart-video).
- `LOVART_ACCESS_KEY` / `LOVART_SECRET_KEY` set in `~/.openclaw/.env`.
- `jq`, `ffmpeg`, `python3` with Pillow.

## Install

Symlink the skill into your Claude Code commands directory (run from the repo root):

```bash
ln -sf "$(pwd)/generate-posts-from-feed.md" ~/.claude/commands/generate-posts-from-feed.md
```

## Usage

```
/generate-posts-from-feed https://nlpperformance.com/products/<handle> --limit 10
```

See `generate-posts-from-feed.md` for the full argument reference, and
`docs/superpowers/specs/` for the design spec.
