# unsiloed-extract

A Markdown-only agent skill for guided document data extraction with human-in-the-loop review, powered by the [Unsiloed AI](https://unsiloed.ai) document-processing API.

Point the agent at a document (PDF, scan, photo, DOCX, XLSX) and the skill walks you through:

1. **Fields** — you list the fields you want, or ask for suggestions derived from the document's own headings.
2. **Output format** — JSON, CSV, Markdown table, or YAML.
3. **Human-in-the-loop review** — optionally review every field scoring below a confidence threshold (default 0.97) against an annotated copy of the document, with numbered bounding boxes drawn on the exact values to check, then accept or amend each one. If more than 10 fields fall below the threshold, the skill warns that the document is low quality and helps you pick a workable threshold.

The skill follows the [agentskills.io](https://agentskills.io) open standard: a single `SKILL.md`, no bundled scripts. Everything runs through `curl` plus whatever image tooling the machine already has.

## Install

**Hermes:**

```bash
hermes skills install LewisDwyer/unsiloed-extract-skill/skills/unsiloed-extract
```

or in a chat session: `/skills install LewisDwyer/unsiloed-extract-skill/skills/unsiloed-extract`

Manual: copy `skills/unsiloed-extract/` into `~/.hermes/skills/documents/`.

**Claude Code:**

```bash
mkdir -p ~/.claude/skills/unsiloed-extract
curl -fsSL https://raw.githubusercontent.com/LewisDwyer/unsiloed-extract-skill/main/skills/unsiloed-extract/SKILL.md \
  -o ~/.claude/skills/unsiloed-extract/SKILL.md
```

## Requirements

- `curl` and `jq` on the PATH
- `UNSILOED_API_KEY` — get one from the [Unsiloed dashboard](https://unsiloed.ai). Hermes prompts for it on first load; otherwise export it or put it in a `.env`.
- Optional, for the annotated review image: Python 3 with `pymupdf`/`Pillow`, or `pdftoppm`, or ImageMagick (the skill falls back to text descriptions without them)

## Related

- [`LewisDwyer/unsiloed-skill`](https://github.com/LewisDwyer/unsiloed-skill) — general-purpose Unsiloed skill (parse / extract / classify / split) for OpenClaw
- [Unsiloed cookbook](https://github.com/Unsiloed-AI/unsiloed-cookbook) — recipes and examples for building with the Unsiloed API
