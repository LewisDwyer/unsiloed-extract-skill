---
name: unsiloed-extract
description: Guided document data extraction with human-in-the-loop review, powered by Unsiloed AI. Use when the user points at a document (PDF, scan, photo, DOCX, XLSX) and wants specific fields pulled out of it — names, totals, dates, ID numbers, line items — especially when they want to choose the fields interactively, pick an output format (JSON, CSV, Markdown), or verify low-confidence values against an annotated copy of the document before trusting them. Trigger phrases include "extract the data from this document", "pull the fields from this invoice", "get the totals and dates out of this PDF", "extract this with verification".
version: 1.0.0
required_environment_variables:
  - name: UNSILOED_API_KEY
    prompt: Unsiloed API key
    help: Get a key from the Unsiloed dashboard at https://unsiloed.ai
    required_for: all API calls
metadata:
  hermes:
    tags: [documents, extraction, ocr, human-in-the-loop]
    category: documents
---

# Unsiloed Extract — guided extraction with human review

Walk the user from "here's a document" to verified structured data in five moves: agree the fields, agree the output format, ask about human-in-the-loop (HITL) review, run the extraction against the Unsiloed API, then — if HITL is on — review every low-confidence value against an annotated image of the original document.

This skill is pure instructions: everything runs through `curl` and whatever image tooling the machine already has. No bundled scripts.

## Requirements

- `curl` and `jq` on the PATH.
- `UNSILOED_API_KEY` in the environment. If unset, check for a `.env` in the working directory and source it; otherwise ask the user for the key. Never echo the key back.
- For the HITL annotated image (optional, with graceful fallback): any one of Python 3 with `pymupdf`/`Pillow`, `pdftoppm` (poppler), or ImageMagick.

API base is `https://prod.visionapi.unsiloed.ai`. Every call sends the key as an `api-key` header. Accepted inputs: PDF, PNG, JPEG, TIFF, BMP, DOCX, XLSX, PPTX.

## When to Use

Use this skill when the user wants **named fields as structured data** out of a document and the interaction is conversational — you can ask them questions and wait for answers. If they just want to *read* the document ("what does this say?"), or classify or split it, use the general `unsiloed` skill instead.

Do not read the document with your own vision and answer from that. Vision-reading fabricates content on handwriting and dense layouts and produces no confidence signal — the confidence scores and citations from the API are the entire point of this workflow. If the API call fails, report the failure; do not fall back to vision. (One carve-out: glancing at the document to *suggest field names* in Step 1 is fine — the extracted values must always come from the API.)

**Non-interactive contexts** (cron jobs, batch pipelines, no human on the other end): skip every question below. Use the fields given in the invocation, JSON output, HITL off. If no fields were given, stop and report that this skill needs either a field list or a human to ask.

## Procedure

Ask the Step 1–3 questions with the framework's native question mechanism if it has one (e.g. AskUserQuestion in Claude Code; a plain chat message in Hermes). Ask, then wait. Never invent the user's answer, and never run the extraction before the fields are agreed.

### Step 1 — Fields

Ask the user which fields they want extracted, and offer to suggest some. Do not build suggestions before they ask — no API calls, no document reads; the first move is the question.

If they ask for suggestions, build the list by looking at the document directly (open the file with your own vision/read tooling) and derive candidate fields from its headings and structure — column headings, form labels, letterhead fields. This is the one place vision is fine: you're proposing field *names* for the user to choose from, not extracting values, so speed matters and fabrication risk is near zero. If the type is obvious without looking (filename, or the user said "this invoice"), the standard sets work too — invoice: vendor name, invoice number, date, due date, line items, subtotal, tax, total; receipt: merchant, date, currency, total; ID document: full name, ID number, date of birth, expiry date; contract: parties, effective date, term, governing law.

Present suggestions as a numbered list so the user can reply "1, 3, 4 plus purchase order number" — always invite additions of their own. Confirm the final field list back in one line before moving on. The flow is strict: prompt for fields → user asks for suggestions → list suggested fields → user selects and/or adds their own → only then build the schema and extract.

### Step 2 — Output format

Ask where the values should land. Sensible menu (pick defaults by shape of the data):

1. **JSON** — the default; matches the API's native shape.
2. **CSV** — best when the fields are rows of a table (line items, register entries). One row per record.
3. **Markdown table** — best for a quick human read in chat.
4. **YAML** — for config-ish downstream use.

Also ask (or infer) whether output goes into the chat, into a file next to the source document (`<docname>-extracted.json` etc.), or both. Default: file plus a short summary in chat.

### Step 3 — Human-in-the-loop review

Ask whether they want HITL verification: after extraction, any field scoring below a confidence threshold (**default 0.97**) gets shown to them with an annotated image of the document so they can accept or correct each value.

Offer to make their answer the default for future runs so they aren't re-asked. If the framework has persistent skill config or memory, store it there (Hermes: skill config; Claude Code: memory); otherwise remember it for the session and note it in your reply. Let them override the 0.97 threshold up front if they mention one.

### Step 4 — Run the extraction

Build a JSON Schema from the agreed fields:

- One property per field; every property gets a short `description` telling the model exactly what to grab ("total as printed, digits only", "name as written, do not expand abbreviations"). The descriptions steer the extraction — use them.
- Repeating records (line items, table rows) are an `array` whose `items` is an **object** with one property per column — not an array of strings. Per-row objects get you per-cell confidence scores.
- `additionalProperties: false` on every object level.
- Mark a field `required` only when it is reliably present on this kind of document. A `required` field the model can't find pressures it to guess — leave optional fields out of `required` so a missing value comes back `null` instead of invented.

Then submit and poll. Always pass `enable_citations=true` (it adds per-field bounding boxes and real nested scores) and default to `model=gamma`:

```bash
SCHEMA='<SCHEMA-JSON>'   # single-quote so the schema's double quotes survive

JOB=$(curl -s -X POST https://prod.visionapi.unsiloed.ai/v2/extract \
  -H "api-key: $UNSILOED_API_KEY" \
  -F "pdf_file=@<FILE-PATH>" \
  -F "schema_data=$SCHEMA" \
  -F "model=gamma" \
  -F "enable_citations=true" | jq -r '.job_id')

for _ in $(seq 60); do   # 60 × 5s = 300s cap
  R=$(curl -s "https://prod.visionapi.unsiloed.ai/extract/$JOB" -H "api-key: $UNSILOED_API_KEY")
  S=$(echo "$R" | jq -r '.status')
  [ "$S" = "completed" ] && break
  [ "$S" = "failed" ] && { echo "$R" | jq -r '.error' >&2; break; }
  sleep 5
done
echo "$R" > <DOCNAME>-extract-raw.json   # keep the full raw response as the audit trail
```

`pdf_file=@...` accepts images too, despite the name. If the input is a **large photo or scan image** (roughly 9 megapixels or more), downscale it to ~1500 px on the long side first (`magick in.jpg -resize 1500x1500 out.jpg`, or `sips --resampleWidth 1500` on macOS, or Pillow) — oversized images make the API silently return empty values with score 0 and no error.

### Step 5 — Read the result

Every leaf in `.result` comes back as:

```json
{
  "value": "national treasury",
  "score": {"grounding_score": 0.96, "extraction_score": 0.48},
  "citation": {"bbox": [47, 55, 82, 63], "page": 1, "page_width": 792.0, "page_height": 612.0}
}
```

Reading rules (each of these has silently corrupted a real run before):

- **Score is a nested object, not a float.** A field's confidence for the threshold check is `min(grounding_score, extraction_score)`; if `extraction_score` is `null`, use `grounding_score` alone.
- **`value: null` with scores near 0 means "not found".** Treat not-found fields as review items in HITL (the human confirms the field genuinely isn't on the document), not as silent omissions.
- **Array and object nodes are wrapped too.** A schema field that is an array arrives as `{"score": ..., "value": [...], "citation": ...}` — the rows live at `.result.<field>.value`, not `.result.<field>` (a naive `length` on the wrapper counts its three keys, not your rows). Array-of-*object* row fields sit directly on each row object.
- **Array-of-string items are double-wrapped.** Each item arrives as `{"__value__": {"value": ..., "score": ..., "citation": ...}}` — the string is at `item.__value__.value`, not `item.value`. Always unwrap `__value__` when present.
- **Blank cells come back as empty string with score ~0 and a citation near (not on) the blank.** That's a legitimate "nothing printed here" — put it in the HITL review so the human confirms the cell really is blank, and don't treat the slightly-offset box as a bug.
- **Flat 0.95 everywhere is a placeholder, not a score.** If every field reports grounding exactly 0.95 with null extraction scores, the scoring pass didn't run. Don't present those as real confidences — resubmit once, and if it persists, tell the user the scores are unavailable and treat every field as needing review.
- **`bbox` is `[x1, y1, x2, y2]`**, top-left origin, in units where the page is `page_width × page_height` (PDF points).

If HITL is **off**: convert the values to the agreed format (strip the score/citation wrappers), deliver, and you're done. Mention in passing if anything scored conspicuously low, but don't run the review loop.

### Step 6 — HITL review

Collect every field with confidence below the threshold (default 0.97), including nulls/not-founds. For array rows, identify each item by its row ("line_items[3].amount").

**If none:** tell the user every field cleared the threshold and deliver the output.

**If more than 10:** stop before reviewing. Warn the user that the document appears to be low quality (or the threshold is stricter than this scan can support), show the score distribution in one line (e.g. "31 fields total: 12 ≥ 0.97, 11 between 0.90–0.97, 8 below 0.90"), and ask whether they want to lower the threshold — suggest the highest value that leaves ≤10 fields to review — or press on and review everything. Recompute the review set with whatever they choose.

**Review table.** Show the low-confidence fields as a numbered table: `# | field | extracted value | confidence | page`.

**Annotated document.** Produce a copy of the cited pages with a numbered box per low-confidence field so the user can eyeball each value in place. The recipe, with any tooling available (Python + pymupdf/Pillow shown; `pdftoppm` + ImageMagick work the same way):

1. Rasterize each page that has at least one low-confidence citation (150 DPI is plenty).
2. For each citation on that page, scale: `px = x * rendered_width / citation.page_width` (same for y with heights). Use the citation's own `page_width`/`page_height` — do not assume the page size.
3. Draw a red rectangle at `[x1, y1, x2, y2]`, padded ~3 points on every side, with the review-table number drawn next to it.
4. Save as `<docname>-review-p<page>.png` next to the source document, and show/attach the image(s) however the framework displays files. Fields with `citation: null` get no box — say so in the table ("not located on the document").

If no rendering tooling exists on the machine, fall back to text: give each field's page number and describe roughly where the box sits on the page (from the bbox relative to the page size, e.g. "top-left letterhead area").

**Accept or amend.** Ask the user to go through the numbered items and either accept or correct each — invite the compact reply form: "1 ok, 2 = 4,500.00, 3 ok, 4 not on doc". Apply their corrections, then rebuild the final output in the agreed format. In your closing summary state which fields were human-amended, which were human-confirmed, and which were auto-accepted at or above the threshold. Do not put review metadata inside the output file unless the user asks for it.

## Pitfalls

- **`model=alpha` can return silently empty on scanned photos** (score 0, no error) — that's why this skill defaults to `gamma`. If gamma comes back empty on a document that clearly has the data, retry once with `alpha` before reporting failure.
- **Oversized images fail silently.** ~9 MP and up returns empty values with no error. Downscale first (Step 4).
- **Don't mix coordinate systems.** `/parse` segment/word boxes use a different scale (144 DPI, left/top/width/height, word boxes relative to their segment) than `/v2/extract` citations (PDF points, x1/y1/x2/y2). This skill only needs the citation convention.
- **The score is a triage gate, not a lie detector.** Genuinely wrong values have scored 0.95–0.98 (confident-and-wrong). The threshold catches most cleanly-attributable errors; it does not guarantee the rest. Where the data allows a deterministic cross-check (line items sum to the total, dates in range), run it and flag mismatches regardless of score. If the user needs full recall on catchable errors, offer a 0.99 threshold and say the review burden will rise.
- **Shell-quoting the schema.** Wrap `SCHEMA` in single quotes; a double-quoted schema gets its inner quotes eaten and the API rejects it.
- **Multi-page documents go through whole.** No pre-splitting needed. Use the citation's `page` to annotate the right page.

## Verification

Before calling the task done, confirm:

1. The job reached `completed` and the full raw response is saved next to the source document.
2. Every agreed field appears in the delivered output (with `null` — not a guess — where nothing was found).
3. Scores were read from the nested object and any `__value__` wrappers were unwrapped (no silently-null values that the raw JSON shows as populated).
4. If HITL was on: every below-threshold field was either confirmed or amended by the user, and the delivered output reflects the amendments.
5. The delivered format matches what the user picked in Step 2.
