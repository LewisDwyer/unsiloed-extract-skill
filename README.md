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

## Troubleshooting (Hermes)

**The agent "runs" the skill instantly and describes results, but no output file exists.**
It's role-playing: it has no execution tools, so it narrates tool calls in prose and invents plausible results. Check `agent.disabled_toolsets` in `~/.hermes/config.yaml` — the skill needs `skills` (to load at all) and a shell (`code_execution` or `terminal`) enabled; `clarify` (interactive questions) and `vision` (field suggestions from the document) make the flow work as designed. You can confirm from `~/.hermes/logs/agent.log`: a turn with a tiny `in=` token count and `tool_turns=0` never called any tools.

**How to tell a run was real:** the agent asks the field/format/HITL questions before extracting, quotes a `job_id`, the extraction takes ~30 s to several minutes, and the raw response JSON it cites actually exists on disk.

**The agent can't find the API key.** Put `UNSILOED_API_KEY` in Hermes's secrets store (`hermes setup`, or its `.env` file in the Hermes home directory) — Hermes passes it through to its execution sandboxes. There is no per-skill config file.

**Install blocked by the security scanner.** Versions before 1.1.0 tripped Hermes's `env_exfil_curl` rule (secret-named variable on a curl line). Current versions scan `safe`; if you see a `dangerous` verdict, make sure you're installing the latest commit.

## Related

- [`LewisDwyer/unsiloed-skill`](https://github.com/LewisDwyer/unsiloed-skill) — general-purpose Unsiloed skill (parse / extract / classify / split) for OpenClaw
- [Unsiloed cookbook](https://github.com/Unsiloed-AI/unsiloed-cookbook) — recipes and examples for building with the Unsiloed API
