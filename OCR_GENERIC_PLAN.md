# Plan: one generic OCR pipeline, forms defined in JSON

Notebook: `Meenakshi_All_Forms_OCR_not_MIS_v2.ipynb` · Model: Qwen3-VL (Hugging Face `generate`) · Status: proposal

## 1. Goals

1. **Every page is read.** A page that matches no form is extracted anyway, never dropped.
2. **Forms are data, not code.** Adding a form means adding one JSON file in `forms/`.
3. **Matching is safe.** A form is chosen only on strong evidence; an unclear page is flagged, never sent to the closest wrong template.
4. **Every value has a confidence score**, so low-confidence cells go to review instead of silently into the database.
5. **One database layout for all forms.** New forms need no new tables.

## 2. Pipeline

```
PDF page
  ├─ A. Prepare page        deskew, rotate, normalise, ruled lines           (existing preprocessing)
  ├─ B. Read HEADER         title lines, doc no, every "label: value" pair    generic
  ├─ C. Read TABLE SKELETON header bands, column labels, row labels           generic
  ├─ D. MATCH FORM          score page against forms/*.json
  │        ├─ matched  → load spec (keys, types, cell format, rules)
  │        └─ no match → generic keys from printed labels + draft spec, status "new_form"
  ├─ E. Read BODY           column strips / boxed row strips, prompt built from labels + spec
  ├─ F. Confidence          token probabilities of each value (section 4)
  ├─ G. Normalise + check   numbers, dates, "—", ditto; row/column totals; spec rules
  └─ H. Write               page JSON (section 5) → database (section 6) + CSV / Excel
```

### Form matching (step D)

| Signal | Weight | Example |
|---|---|---|
| Doc no (exact after normalising) | high | `MPPL/F/QAD/39/CR/QG-02` vs `MPPL/F/QAD/35/PL` |
| Column-label overlap (fuzzy Jaccard) | high | `7 TO 9, 9 TO 11 …` vs `CR AAEL, CR AAD …` |
| Title similarity | medium | `WELDSHOP REWORK & REJECTION SHEET` |
| Orientation, column count | low | landscape, 9 columns |

- **Matched:** best score ≥ 0.75 and ≥ 0.15 ahead of the runner-up.
- **Ambiguous:** two forms within 0.15. Extracted generically and flagged.
- **New form:** best score < 0.5. Extracted generically and a draft spec is written.
- `TEMPLATE_OVERRIDE` and `FOLDER_TEMPLATES` stay as manual overrides that always win.

## 3. Tally-mark cells (QG-2 weld checksheets)

**Rule: read only the written count; never count strokes.**

| How the cell is written | Value read | `source` |
|---|---|---|
| `|| — ②` (count in a circle) | 2 | `circled` |
| `||| - 3` (count after the dash) | 3 | `after_dash` |
| `||||` or a bundle of 5 with a slash, no number | `null` | `no_count_written` |
| `—` or empty | `null` | `empty` |

- The prompt tells the model to return the number after the dash or inside the circle, to ignore the strokes, and to return `null` when no number is written.
- No stroke counting is ever used as a value or as a fallback.
- **Consequence:** on sheets written with strokes only (Cr. AAEL robotic, main stand ACK, Cr. AAD manual, pedal brake AAE), the defect cells come out `null` with status `no_count`. Their digit cells (TOTAL column, Total Production, OK, Defect qty, %) are read normally.
- **Rows that overlap:** the handwriting is taller than the row, so circles spill into the row below. Strategy:
  1. Main pass: one call per time-slot column, returning an ordered list of counts top to bottom.
  2. Repair pass, only for rows whose totals don't add up: the row is cropped with half a row of context above and below, the target row is outlined in red, and the prompt says "read only the row inside the red box".
- Spec setting: `"cell_format": "tally_with_count"`.

## 4. Confidence score per cell

### Where it comes from

| Component | How it's computed | Range |
|---|---|---|
| `model` | Generation runs with `output_scores=True, return_dict_in_generate=True`. The value's characters are mapped to their generated tokens, and the score is the **lowest** probability among those tokens (one weak digit makes the whole value weak). | 0–1 |
| `alternatives` | For each digit token, the other tokens with probability ≥ 0.05 (e.g. ① read as 1 with p=0.81, 7 with p=0.14). | list |
| `agreement` | When the cell was read twice (column pass + boxed-row repair pass): 1 if both readings match, 0 if not, `null` if read once. | 0/1/null |
| `checks` | Row total, column total and spec rules this cell takes part in: `pass`, `fail` or `n/a`. | per rule |

The model is **not** asked to rate its own confidence. Self-rated scores from a VLM are not calibrated; token probabilities are measured.

### Final score and status

```
final = model
if any check this cell takes part in passed:   final = max(final, 0.95)  # arithmetic confirms the value
if any check failed:                            final = min(final, 0.50)
if agreement == 0:                              final = min(final, 0.40)
```

| Status | Rule |
|---|---|
| `ok` | final ≥ 0.90 |
| `review` | 0.60 ≤ final < 0.90 |
| `low` | final < 0.60 |
| `repaired` | a total check failed, and swapping in one `alternative` for one cell made it pass (old value kept in `raw`) |
| `no_count` | tally cell with no written count |
| `empty` | blank or dash |

Thresholds (0.90 / 0.60) are starting values. They get tuned once ground-truth CSVs exist, so that `ok` cells are ≥ 99% correct.

## 5. JSON output (one file per page)

```json
{
  "schema_version": "2.0",
  "document": {"file": "Fwd_ Fwd_ IOT WELD SHOP/Daily Cr. AAD Robotic st. pass checksheet.pdf",
               "sha256": "9f2c…", "page": 1, "pages_total": 1},
  "run": {"run_id": "2026-10-12T10:41:07Z-a1b2", "model_id": "Qwen/Qwen3-VL-4B-Instruct",
          "pipeline_version": "2.0.0"},
  "form": {"form_id": "weld_qg2_checksheet", "form_version": 1,
           "match": {"status": "matched", "score": 0.91, "runner_up": {"form_id": "mis_fi_visual_pc", "score": 0.38},
                     "evidence": {"doc_no": 1.0, "columns": 0.86, "title": 0.78}}},
  "header": {
    "title":     {"value": "QGFI CHECK SHEET- QUALITY GATE-2(VISUAL INSPECTION-100%), WELD SHOP", "confidence": 0.97, "status": "ok"},
    "part_name": {"value": "RR CARRIER-AAD (ROBOTIC WELDING)", "confidence": 0.95, "status": "ok"},
    "shift":     {"value": "A", "raw": "(A)", "confidence": 0.93, "status": "ok"},
    "line":      {"value": "ROBOTIC 1,2,3,5", "confidence": 0.71, "status": "review"},
    "date":      {"value": "2026-09-29", "raw": "29/09/2026", "confidence": 0.96, "status": "ok"}
  },
  "tables": [{
    "table_no": 1,
    "kind": "matrix",
    "columns": [
      {"key": "slot_07_09", "label": "7 TO 9",  "type": "int", "cell_format": "tally_with_count"},
      {"key": "slot_09_11", "label": "9 TO 11", "type": "int", "cell_format": "tally_with_count"},
      {"key": "total",      "label": "TOTAL",   "type": "int"}
    ],
    "rows": [{
      "row_idx": 1, "row_type": "defect",
      "labels": {"s_no": "1", "section": "PIPE B&C ASSY", "description": "BLOW HOLE ROUND WELD PIPE B"},
      "cells": {
        "slot_07_09": {"value": 1, "raw": "| — ①", "source": "circled",
                       "confidence": {"model": 0.81, "final": 0.95, "agreement": null,
                                      "alternatives": [{"value": 7, "p": 0.14}]},
                       "checks": {"row_total": "pass"}, "status": "ok",
                       "bbox": [412, 958, 588, 1001], "crop": "debug/p1/r01_slot_07_09.png"},
        "slot_09_11": {"value": null, "raw": "|||", "source": "no_count_written",
                       "confidence": null, "checks": {}, "status": "no_count",
                       "bbox": [588, 958, 764, 1001], "crop": "debug/p1/r01_slot_09_11.png"},
        "total":      {"value": 11, "raw": "11", "source": "digit",
                       "confidence": {"model": 0.98, "final": 0.98, "agreement": null, "alternatives": []},
                       "checks": {"row_total": "pass"}, "status": "ok",
                       "bbox": [1640, 958, 1760, 1001], "crop": "debug/p1/r01_total.png"}
      }
    }]
  }],
  "checks": [
    {"rule": "row_total", "scope": "row 1", "expression": "total == sum(slot_*)",
     "expected": 11, "actual": 11, "result": "pass"}
  ],
  "summary": {"cells": 252, "ok": 201, "review": 22, "low": 6, "repaired": 3, "no_count": 0, "empty": 20,
              "needs_review": true}
}
```

- `value` is normalised (int, ISO date, trimmed text); `raw` is exactly what was read.
- `confidence` is `null` for `empty` and `no_count` cells: there's no value to be confident about.
- `bbox` is in page pixels after preprocessing; `crop` is the debug image, so review can show the cell.

## 6. Database

PostgreSQL (SQLite works the same for local runs, with `TEXT` in place of `JSONB`). Every form uses the same tables: one row per cell, with typed value columns. A new form adds rows, never tables.

```sql
-- form definitions (copy of forms/*.json, versioned)
CREATE TABLE form_spec (
  form_id       TEXT,
  form_version  INT,
  name          TEXT NOT NULL,
  spec          JSONB NOT NULL,
  created_at    TIMESTAMPTZ DEFAULT now(),
  PRIMARY KEY (form_id, form_version)
);

-- one row per pipeline run
CREATE TABLE ocr_run (
  run_id            TEXT PRIMARY KEY,
  model_id          TEXT NOT NULL,
  pipeline_version  TEXT NOT NULL,
  started_at        TIMESTAMPTZ NOT NULL,
  finished_at       TIMESTAMPTZ
);

-- one row per scanned PDF (sha256 stops the same scan being loaded twice)
CREATE TABLE document (
  document_id  BIGSERIAL PRIMARY KEY,
  file_name    TEXT NOT NULL,
  sha256       TEXT UNIQUE NOT NULL,
  pages_total  INT,
  uploaded_at  TIMESTAMPTZ DEFAULT now()
);

-- one row per page per run
CREATE TABLE page (
  page_id        BIGSERIAL PRIMARY KEY,
  document_id    BIGINT REFERENCES document,
  run_id         TEXT REFERENCES ocr_run,
  page_no        INT NOT NULL,
  form_id        TEXT,                    -- NULL when match_status = 'new_form'
  form_version   INT,
  match_status   TEXT NOT NULL,           -- matched | ambiguous | new_form | override
  match_score    REAL,
  form_date      DATE,                    -- copied from header for fast filtering
  shift          TEXT,
  needs_review   BOOLEAN NOT NULL,
  result_json    JSONB NOT NULL,          -- the full page JSON (section 5), kept for audit
  UNIQUE (document_id, page_no, run_id)
);

-- header fields, key/value
CREATE TABLE header_field (
  page_id     BIGINT REFERENCES page ON DELETE CASCADE,
  key         TEXT,
  value_text  TEXT,
  raw         TEXT,
  confidence  REAL,
  status      TEXT,
  PRIMARY KEY (page_id, key)
);

-- every table cell of every form
CREATE TABLE cell (
  cell_id        BIGSERIAL PRIMARY KEY,
  page_id        BIGINT REFERENCES page ON DELETE CASCADE,
  table_no       INT NOT NULL,
  row_idx        INT NOT NULL,
  row_type       TEXT,                    -- defect | summary | entry
  row_label      TEXT,                    -- e.g. "BLOW HOLE ROUND WELD PIPE B"
  section        TEXT,                    -- e.g. "PIPE B&C ASSY"
  col_key        TEXT NOT NULL,           -- e.g. "slot_07_09"
  col_label      TEXT,                    -- as printed, e.g. "7 TO 9"
  value_num      NUMERIC,                 -- filled for int / number cells
  value_text     TEXT,                    -- filled for text cells
  value_date     DATE,                    -- filled for date cells
  raw            TEXT,
  source         TEXT,                    -- digit | circled | after_dash | text | no_count_written | empty
  conf_model     REAL,
  conf_final     REAL,
  agreement      SMALLINT,
  alternatives   JSONB,
  checks         JSONB,
  status         TEXT NOT NULL,           -- ok | review | low | repaired | no_count | empty
  bbox           INT[4],
  crop_path      TEXT,
  UNIQUE (page_id, table_no, row_idx, col_key)
);
CREATE INDEX cell_review_idx ON cell (status) WHERE status IN ('review', 'low', 'repaired');
CREATE INDEX cell_lookup_idx ON cell (col_key, row_label);

-- results of totals / spec rules
CREATE TABLE check_result (
  page_id     BIGINT REFERENCES page ON DELETE CASCADE,
  rule        TEXT,
  scope       TEXT,
  expected    NUMERIC,
  actual      NUMERIC,
  result      TEXT,                       -- pass | fail | n/a
  PRIMARY KEY (page_id, rule, scope)
);

-- human corrections; the OCR value is never overwritten
CREATE TABLE cell_correction (
  correction_id  BIGSERIAL PRIMARY KEY,
  cell_id        BIGINT REFERENCES cell ON DELETE CASCADE,
  old_value      TEXT,
  new_value      TEXT NOT NULL,
  corrected_by   TEXT NOT NULL,
  corrected_at   TIMESTAMPTZ DEFAULT now()
);

-- the value everyone reads: latest correction if any, otherwise OCR
CREATE VIEW cell_current AS
SELECT c.*,
       COALESCE(cc.new_value, COALESCE(c.value_num::TEXT, c.value_text, c.value_date::TEXT)) AS final_value,
       (cc.correction_id IS NOT NULL) AS corrected
FROM cell c
LEFT JOIN LATERAL (
  SELECT * FROM cell_correction x WHERE x.cell_id = c.cell_id
  ORDER BY corrected_at DESC LIMIT 1
) cc ON true;
```

### Typical queries

```sql
-- review queue: cells that need a person, worst first
SELECT p.form_id, d.file_name, p.page_no, c.row_label, c.col_label, c.raw, c.value_num, c.conf_final, c.crop_path
FROM cell c JOIN page p USING (page_id) JOIN document d USING (document_id)
WHERE c.status IN ('low', 'review', 'repaired')
ORDER BY c.conf_final NULLS FIRST;

-- defects per day for QG-2 checksheets, only trusted values
SELECT p.form_date, c.row_label, SUM(c.final_value::NUMERIC) AS defects
FROM cell_current c JOIN page p USING (page_id)
WHERE p.form_id = 'weld_qg2_checksheet' AND c.col_key = 'total'
  AND (c.status = 'ok' OR c.corrected)
GROUP BY 1, 2;
```

Wide per-form tables (the current CSV / Excel layout) become pivot views over `cell_current`. They're no longer stored separately.

### Loading

1. Notebook writes `<output>/<file>/page_<n>.json` (section 5).
2. Loader cell (or script) upserts `ocr_run` → `document` (by sha256) → `page` → `header_field` / `cell` / `check_result` in one transaction per page.
3. Re-running a file creates a new `page` row under the new `run_id`. Earlier runs stay for comparison; the latest run per `(document, page_no)` is the current one.

## 7. Form spec format: `forms/<form_id>.json`

```json
{
  "form_id": "weld_qg2_checksheet",
  "version": 1,
  "name": "QGFI Check Sheet – Quality Gate-2 (Weld Shop)",
  "match": {
    "title_any":   ["QGFI CHECK SHEET", "QUALITY GATE-2"],
    "title_none":  ["FI VISUAL INSPECTION 100%(POWDER COATING)"],
    "doc_no_any":  ["MPPL/F/QAD/39/CR/QG-02"],
    "columns_any": ["7 TO 9", "9 TO 11", "11 TO 1", "1 TO 3", "3 TO 5", "5 TO 7", "TOTAL"]
  },
  "header": {
    "part_name": ["PART NAME"], "shift": ["SHIFT"], "line": ["LINE", "LINE NO"],
    "date": {"labels": ["DATE"], "type": "date"}
  },
  "layout": {
    "kind": "matrix",
    "row_label_cols": ["S.NO", "OPERATION NAME", "DEFECT'S DESCRIPTION", "WELDER NAME"],
    "rows": "read",
    "summary_rows": ["TOTAL PRODUCTION", "OK QTY (FIRST TIME RIGHT)", "DEFECT QTY (REJECTION)",
                     "REWORK OK", "STRAIGHT PASS", "SCRAP QTY", "INSPECTION/CHIPPING PENDING QTY"]
  },
  "columns": [
    {"key": "slot_07_09", "labels": ["7 TO 9"],  "type": "int", "cell_format": "tally_with_count"},
    {"key": "slot_09_11", "labels": ["9 TO 11"], "type": "int", "cell_format": "tally_with_count"},
    {"key": "slot_11_01", "labels": ["11 TO 1"], "type": "int", "cell_format": "tally_with_count"},
    {"key": "slot_01_03", "labels": ["1 TO 3"],  "type": "int", "cell_format": "tally_with_count"},
    {"key": "slot_03_05", "labels": ["3 TO 5"],  "type": "int", "cell_format": "tally_with_count"},
    {"key": "slot_05_07", "labels": ["5 TO 7"],  "type": "int", "cell_format": "tally_with_count"},
    {"key": "total",      "labels": ["TOTAL"],   "type": "int"}
  ],
  "rules": ["total == slot_07_09 + slot_09_11 + slot_11_01 + slot_01_03 + slot_03_05 + slot_05_07"],
  "ground_truth": "weld_qg2_checksheet/"
}
```

- `layout.kind`: `record` (list of entries), `matrix` (row labels × column labels), `block_grid` (repeating block of N rows, e.g. straight pass with `block_rows: 8` and `column_groups: ["A","B","TOTAL"]`).
- `rows`: `"read"` or a fixed list (a fixed list catches skipped rows).
- `rules`: plain expressions, evaluated with a restricted `eval`. A rule is skipped (`n/a`) when any of its cells is `null`.
- `cell_format`: `plain` (default) or `tally_with_count` (section 3).

### Forms covered

| Form | Spec | Kind |
|---|---|---|
| Supplier lot rejection / rejection / rework / segregation | 4 specs | record |
| Line rejection (press, weld), penetration rejection report | `line_rejection` | record |
| Scrap note | `scrap_note` | record |
| MIS FI visual / plating / PC QA / PC daily | 4 specs | matrix |
| Straight pass weld1/2, monthly handwritten | `st_pass_monitoring` | block_grid |
| Weld shop rework & rejection sheet | `weld_rework_rejection` | block_grid (22 fixed parts × shift A/B) |
| QG-2 checksheets (AAD robotic/manual, AAEL, main stand AAE/ACK, pedal brake) | `weld_qg2_checksheet` | matrix, `tally_with_count` |

## 8. Adding a new form

1. Drop the PDF in any input folder and run the notebook. No setup.
2. The page matches nothing, so it's extracted generically with keys made from the printed labels. Status `new_form`; red row in the debug report. Cells still get confidence scores and are loaded into the database with `form_id = NULL`.
3. The notebook writes `forms/_drafts/<slug>.json` with title lines, doc no, header labels, column and row labels, detected `layout.kind`, block size, and suggested keys (`"OK QTY (FIRST TIME RIGHT)"` → `ok_qty_first_time_right`).
4. A person reviews the draft: rename keys, set types and `cell_format`, add rules, choose `rows: "read"` or a fixed list.
5. Move the file to `forms/`. Optionally add a ground-truth CSV in `Dataset/csv/<form_id>/`. Re-run with `ONLY_FILES` set to that PDF.
6. The page now matches; rules run; validation reports accuracy. The spec is copied into `form_spec`, and earlier `new_form` pages can be re-run to pick up the form.

A helper `draft_form("path.pdf")` does steps 2–3 for one PDF.

## 9. Notebook changes

| Section | Change |
|---|---|
| Configuration | `FORMS_DIR`, `MATCH_THRESHOLDS`, `CONF_THRESHOLDS`, `WRITE_DRAFTS`, `DB_URL` |
| Templates | replaced by `load_forms()` reading `forms/*.json`, with schema checks |
| Model | `qwen_generate` / `qwen_generate_batch` return per-token probabilities and top alternatives |
| Template decision | header prompt reads every label/value pair; `score_form()` replaces `classify_text`; `detect_mis_variant` removed |
| Record / MIS / straight-pass extractors | merged into `extract_table(page, skeleton, spec)` with `record`, `matrix`, `block_grid` handlers, reusing the current grid code |
| new | `read_skeleton()`, `read_tally_counts()`, `value_confidence()`, `draft_form()` |
| Page driver | header → skeleton → match → body → confidence → checks → JSON |
| Excel tables | built from the cell-level output |
| new | DB loader (section 6) |
| Validation | unchanged, plus accuracy per confidence band (are `ok` cells really ≥ 99% right?) |

## 10. Rollout

1. **Generic path + confidence, behind `MODE="generic"`.** Run on the 9 IOT Weld Shop PDFs. Done when the QG-2 sheets with written counts (Cr. AAD robotic, main stand AAE) and the rework sheet come out as readable tables with scores.
2. **Convert the 11 existing templates to `forms/*.json`.** Run both modes on forms with ground truth. Done when generic accuracy is within ~2 points of the old extractors per form.
3. **Tune confidence thresholds** against ground truth so `ok` ≥ 99% correct.
4. **Database loader** and review-queue query.
5. **Make generic the default**; keep the old mode for one release, then delete it.

## 11. Risks

- Multi-level headers (straight pass, rework sheet) are where generic grid reading is most likely to fail; `block_grid` hints are the backup.
- Token probability measures how sure the model is, not whether it's right. A confidently misread digit still scores high. Total checks and threshold tuning against ground truth cover this.
- Qwen3-VL-4B may be too weak for circled digits in crowded cells; Qwen3-VL-8B on an L4 is a config change.
- Stroke-only checksheets produce no defect counts by design (section 3).

## 12. Open decisions

1. Form specs in the repo (`forms/`, version-controlled; recommended) or on Drive?
2. PostgreSQL server, or SQLite file to start?
3. Ground-truth CSVs for the IOT Weld Shop forms. Without them, step 1 is checked by eye and thresholds can't be tuned.
