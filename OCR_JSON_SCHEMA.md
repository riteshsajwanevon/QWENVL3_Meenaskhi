# Expected OCR JSON: template PDFs

What the OCR step should return for each PDF in `Dataset/Template`. These values were transcribed by hand from the
scans and can be used both as the target output format and as ground truth for scoring the OCR.

## Rules for every response

- One JSON object per PDF page: `form_type`, `header`, the data block (`entries`, `sections` or `parts`), and `signatures`.
- Quantities are integers (`07` → `7`). Percentages are numbers without `%` (`82%` → `82`).
- An empty cell is `null`. A dash written in a quantity cell (MIS sheets) is `0`.
- Ditto marks (`"`, `11`, `,,`) are replaced with the value from the row above.
- Dates and months are returned as written (`02/06/26`, `July-26`). Converting them to ISO dates is the loader's job.
- Struck-out values are left out; the corrected value is returned.
- `signatures` holds `true` when the box has a signature and `false` when it is empty.
- Printed but empty rows on record forms are not returned. On matrix forms (MIS) every printed row is returned so the
  sheet can be rebuilt one-to-one in Excel.

## Index

| PDF | Response type | Data block | Rows to Excel / DB |
|---|---|---|---|
| `SUPPLIER LOT REJECTION SUMMARY.pdf` | `SUPPLIER_LOT_REJECTION_SUMMARY` | `entries[]` | 2 rows, one per entry |
| `SUPPLIER REJECTION REPORT.pdf` | `SUPPLIER_REJECTION` | `entries[]` | 2 rows, one per entry |
| `SUPPLIER REWORK REPORT.pdf` | `SUPPLIER_REWORK` | `entries[]` | 2 rows, one per entry |
| `SUPPLIER SEGREGATION REPORT.pdf` | `SUPPLIER_SEGREGATION` | `entries[]` | 2 rows, one per entry |
| `PRESS SHOP REJECTION REPORT.pdf` | `LINE_REJECTION` | `entries[]` | 7 rows, one per entry |
| `WELD SHOP REJECTION REPORT.pdf` | `LINE_REJECTION` | `entries[]` | 4 rows, one per entry |
| `SCRAP NOTE.pdf` | `SCRAP_NOTE` | `entries[]` | 5 rows, one per entry |
| `MIS PLATING.pdf` | `MIS_PLATING_DAILY` | `sections[].rows[]` | 42 defect rows × 14 columns |
| `DAILY MIS REPORT PCOATING.pdf` | `MIS_POWDER_COATING_DAILY` | `sections[].rows[]` | 38 defect rows × 17 columns |
| `ST. Pass weld data.pdf` | `ST_PASS_MONITORING` | `parts[].days{}` | 7 parts × days × shift A/B/TOTAL |
| `St . Pass data weld2.pdf` | `ST_PASS_MONITORING` | `parts[].days{}` | 7 parts × days × shift A/B/TOTAL |

### Matrix forms (MIS)

Each `sections[]` item has `rows` (one object per printed defect row, with one key per column in `columns`),
`section_total` and `section_pct` (printed total and % rows, in the same shape). `summary` holds the overall rows
at the bottom of the sheet. To build the Excel sheet, write `rows` in order, each followed by its section rows.
For the database, unpivot each row into `(defect, column, value)`.

### St. Pass sheets

`parts[].days["21"].A` holds the eight printed parameters for that part, day and shift:
`checked_qty`, `ok_qty`, `rej_qty`, `rew_qty`, `total_rework_ok_qty`, `inspection_chipping_pending`, `visual_defects_straight_pass_pct`, `non_conformity_count`. For the database, unpivot into `(part_no, day, shift, parameter, value)`.

---

## `SUPPLIER LOT REJECTION SUMMARY.pdf`

Response type: **`SUPPLIER_LOT_REJECTION_SUMMARY`**

```json
{
  "form_type": "SUPPLIER_LOT_REJECTION_SUMMARY",
  "header": {
    "company": "MEENAKSHI POLYMERS PRIVATE LIMITED (HARIDWAR)",
    "title": "SUPPLIER LOT REJECTION SUMMARY",
    "month": "JUN"
  },
  "entries": [
    {
      "date": "02/06/26",
      "part_name": "Pipe 12.70 X 1.5 X 442",
      "supplier_name": "UTTAM",
      "lot_qty": 2700,
      "rework_qty": 2700,
      "rejection_qty": 0,
      "ok_qty": 2700,
      "problem_description": "Cutting Chips",
      "action": "Used after Cleaning",
      "return_status": "Return",
      "capa_status": "NO",
      "remarks": null
    },
    {
      "date": "02/06/26",
      "part_name": "MIG Wire 1.0 MM",
      "supplier_name": "MW WIRE TECH",
      "lot_qty": 4000,
      "rework_qty": 0,
      "rejection_qty": 250,
      "ok_qty": 3750,
      "problem_description": "Rusty",
      "action": "Used after Seg.",
      "return_status": "Return",
      "capa_status": "NO",
      "remarks": null
    }
  ],
  "signatures": {
    "incoming_qa": false,
    "purchase": false,
    "hod_qa": true,
    "hod_plant": false
  }
}
```

Reading notes:

- Row 2 RETURN STATUS and CAPA STATUS are ditto marks (`11`), resolved to the values of row 1.
- Month is written as `JUN` (trailing letters unclear).

---

## `SUPPLIER REJECTION REPORT.pdf`

Response type: **`SUPPLIER_REJECTION`**

```json
{
  "form_type": "SUPPLIER_REJECTION",
  "header": {
    "company": "MEENAKSHI POLYMERS PRIVATE LIMITED (HARIDWAR)",
    "title": "SUPPLIER REJECTION REPORT",
    "month": "June-26"
  },
  "entries": [
    {
      "date": "03/06/26",
      "part_name": "Rubber Chang. Pedal Length",
      "supplier_name": "Ashirwad",
      "lot_qty": 3600,
      "rejection_qty": 3600,
      "ok_qty": 0,
      "problem_description": "Rusty",
      "remarks": null
    },
    {
      "date": "08/06/26",
      "part_name": "LD Stretch Film",
      "supplier_name": "Thirthankar Wes.",
      "lot_qty": 569,
      "rejection_qty": 569,
      "ok_qty": 0,
      "problem_description": "Length variation",
      "remarks": null
    }
  ],
  "signatures": {
    "incoming_qa": false,
    "purchase": false,
    "hod_qa": true,
    "hod_plant": false
  }
}
```

Reading notes:

- Row 1 part name and both supplier names are hard to read (`Rubber Chang. Pedal Length`, `Thirthankar Wes.`): confirm with the stores team.
- Row 2 LOT QTY was corrected on paper; `569` is the value written above the strike.

---

## `SUPPLIER REWORK REPORT.pdf`

Response type: **`SUPPLIER_REWORK`**

```json
{
  "form_type": "SUPPLIER_REWORK",
  "header": {
    "company": "MEENAKSHI POLYMERS PRIVATE LIMITED (HARIDWAR)",
    "title": "SUPPLIER REWORK REPORT",
    "month": "June-26"
  },
  "entries": [
    {
      "date": "02/06/26",
      "part_name": "Pipe 12.70 X 1.50 X 442",
      "supplier_name": "Uttam",
      "lot_qty": 2700,
      "rework_qty": 2700,
      "ok_qty": 2700,
      "problem_description": "Cutting Chips",
      "remarks": null
    },
    {
      "date": "06/06/26",
      "part_name": "Pipe 12.70 X 1.50 X 665",
      "supplier_name": "Parent",
      "lot_qty": 4220,
      "rework_qty": 4220,
      "ok_qty": 4220,
      "problem_description": "Rusty",
      "remarks": null
    }
  ],
  "signatures": {
    "incoming_qa": false,
    "purchase": false,
    "hod_qa": true,
    "hod_plant": false
  }
}
```

---

## `SUPPLIER SEGREGATION REPORT.pdf`

Response type: **`SUPPLIER_SEGREGATION`**

```json
{
  "form_type": "SUPPLIER_SEGREGATION",
  "header": {
    "company": "MEENAKSHI POLYMERS PRIVATE LIMITED (HARIDWAR)",
    "title": "SUPPLIER SEGREGATION REPORT",
    "month": null
  },
  "entries": [
    {
      "date": "02/06/26",
      "part_name": "MIG Wire 1.0 MM",
      "supplier_name": "Microwire Tech",
      "lot_qty": 4000,
      "segregation_qty": 250,
      "ok_qty": 3750,
      "problem_description": "Rusty",
      "remarks": null
    },
    {
      "date": "03/06/26",
      "part_name": "Pipe 12.70 X 1.20 X 498",
      "supplier_name": "Parent",
      "lot_qty": 12274,
      "segregation_qty": 570,
      "ok_qty": 11704,
      "problem_description": "Length variation 497 to 501, required 498±1",
      "remarks": null
    }
  ],
  "signatures": {
    "incoming_qa": false,
    "purchase": false,
    "hod_qa": true,
    "hod_plant": false
  }
}
```

Reading notes:

- MONTH is blank on the form, so `month` is `null`. REMARKS row 1 is a dash, so it is `null`.

---

## `PRESS SHOP REJECTION REPORT.pdf`

Response type: **`LINE_REJECTION`**

```json
{
  "form_type": "LINE_REJECTION",
  "header": {
    "company": "MEENAKSHI POLYMERS PRIVATE LIMITED HARIDWAR",
    "title": "LINE REJECTION REPORT",
    "doc_no": "MPPL/F/STR/07",
    "rev_no": "00",
    "rev_date": "00",
    "org_date": "01.04.2017",
    "shop": "PRESS SHOP",
    "month": "JUNE 2026"
  },
  "entries": [
    {
      "s_no": 1,
      "item_code": "SF500025",
      "material_particulars": "Pipe D and F (AAD)",
      "unit": "NOS",
      "rejection_qty": 13,
      "vendor_name": "INHOUSE",
      "reason_remarks": "Part damage and drill out"
    },
    {
      "s_no": 2,
      "item_code": "SF500079",
      "material_particulars": "Pipe F (AAD)",
      "unit": "NOS",
      "rejection_qty": 14,
      "vendor_name": "INHOUSE",
      "reason_remarks": "Bending out and mitering drop"
    },
    {
      "s_no": 3,
      "item_code": "SF500419",
      "material_particulars": "Pipe F (AAEL)",
      "unit": "NOS",
      "rejection_qty": 14,
      "vendor_name": "INHOUSE",
      "reason_remarks": "Bending out"
    },
    {
      "s_no": 4,
      "item_code": "SF500298",
      "material_particulars": "BP (ACKA)",
      "unit": "NOS",
      "rejection_qty": 28,
      "vendor_name": "INHOUSE",
      "reason_remarks": "Bending out"
    },
    {
      "s_no": 5,
      "item_code": "SF500262",
      "material_particulars": "Cross Pipe (ACKA)",
      "unit": "NOS",
      "rejection_qty": 2,
      "vendor_name": "INHOUSE",
      "reason_remarks": "Drill out"
    },
    {
      "s_no": 6,
      "item_code": "SF500023",
      "material_particulars": "Pipe B (AAD)",
      "unit": "NOS",
      "rejection_qty": 11,
      "vendor_name": "INHOUSE",
      "reason_remarks": "Mitering short"
    },
    {
      "s_no": 7,
      "item_code": "SF500024",
      "material_particulars": "Pipe C (AAD)",
      "unit": "NOS",
      "rejection_qty": 13,
      "vendor_name": "INHOUSE",
      "reason_remarks": "Mitering short"
    }
  ],
  "signatures": {
    "prepared_by": true,
    "prod_head": false,
    "qa_head": true,
    "vp_operations": false
  }
}
```

Reading notes:

- `prepared_by` signature has the name `Abhay Kaushik` written under it.

---

## `WELD SHOP REJECTION REPORT.pdf`

Response type: **`LINE_REJECTION`**

```json
{
  "form_type": "LINE_REJECTION",
  "header": {
    "company": "MEENAKSHI POLYMERS PRIVATE LIMITED HARIDWAR",
    "title": "LINE REJECTION REPORT",
    "doc_no": "MPPL/F/STR/07",
    "rev_no": "00",
    "rev_date": "00",
    "org_date": "01.04.2017",
    "shop": "WELDSHOP",
    "month": "JULY-26"
  },
  "entries": [
    {
      "s_no": 1,
      "item_code": "SF500257",
      "material_particulars": "Main stand plate step",
      "unit": "NOS",
      "rejection_qty": 54,
      "vendor_name": "MPPL P-10",
      "reason_remarks": "Bending out"
    },
    {
      "s_no": 2,
      "item_code": "RM500003",
      "material_particulars": "Pedal A 2.6 MM",
      "unit": "NOS",
      "rejection_qty": 55,
      "vendor_name": "MPPL P-10",
      "reason_remarks": "Pedal out due to robo torch not ok"
    },
    {
      "s_no": 3,
      "item_code": "SF500255",
      "material_particulars": "Bar guide holder pedal ACK",
      "unit": "NOS",
      "rejection_qty": 181,
      "vendor_name": "MPPL P-10",
      "reason_remarks": "Bar guide profile not ok"
    },
    {
      "s_no": 4,
      "item_code": "SF500252",
      "material_particulars": "Main stand forked tube L",
      "unit": "NOS",
      "rejection_qty": 290,
      "vendor_name": "P-10",
      "reason_remarks": "Hole out"
    }
  ],
  "signatures": {
    "prepared_by": true,
    "prod_head": false,
    "qa_head": true,
    "vp_operations": false
  }
}
```

Reading notes:

- S.No. values are circled on paper. Row 2 VENDOR NAME is a ditto mark, resolved to `MPPL P-10`.
- Row 4 item code and material are partly illegible (`SF500252`, `Main stand forked tube L`).

---

## `SCRAP NOTE.pdf`

Response type: **`SCRAP_NOTE`**

```json
{
  "form_type": "SCRAP_NOTE",
  "header": {
    "company": "MEENAKSHI POLYMERS (P) LTD HARIDWAR",
    "title": "Scrap Note",
    "doc_no": "MPPL/F/STR/07",
    "rev_no": "00",
    "rev_date": "00",
    "org_date": "01-04-2017",
    "month": "JULY-26",
    "period": "1-04/26 - 10/7/26"
  },
  "entries": [
    {
      "s_no": 1,
      "item_code": "WLD602503",
      "item_name": "Carrier AAD",
      "unit": "NOS",
      "rejection_qty": 7,
      "vendor_name": "IN HOUSE",
      "reason_remarks": "Penetration cut part"
    },
    {
      "s_no": 2,
      "item_code": "WLD602500",
      "item_name": "Engine Guard KST",
      "unit": "NOS",
      "rejection_qty": 6,
      "vendor_name": "IN HOUSE",
      "reason_remarks": "Penetration cut part"
    },
    {
      "s_no": 3,
      "item_code": "WLD602509",
      "item_name": "Saree Guard AAH",
      "unit": "NOS",
      "rejection_qty": 4,
      "vendor_name": "IN HOUSE",
      "reason_remarks": "Penetration cut part"
    },
    {
      "s_no": 4,
      "item_code": "WLD602514",
      "item_name": "PGC ACK",
      "unit": "NOS",
      "rejection_qty": 5,
      "vendor_name": "IN HOUSE",
      "reason_remarks": "Penetration cut part"
    },
    {
      "s_no": 5,
      "item_code": "WLD602539",
      "item_name": "Pedal Brake AAE",
      "unit": "NOS",
      "rejection_qty": 4,
      "vendor_name": "IN HOUSE",
      "reason_remarks": "Penetration cut part"
    }
  ],
  "signatures": {
    "prepared_by": true,
    "qc_head": true,
    "plating_head": false,
    "vp_operation": false
  }
}
```

Reading notes:

- UNIT, VENDOR NAME and REASON in rows 2, 3 and 5 are ditto marks (`11`), resolved from the row above.
- Rows 6–14 are printed but empty, so they are not returned.

---

## `MIS PLATING.pdf`

Response type: **`MIS_PLATING_DAILY`**

```json
{
  "form_type": "MIS_PLATING_DAILY",
  "header": {
    "company": "MEENAKSHI POLYMERS PVT.LTD. HARIDWAR",
    "doc_no": "MPPL/F/QAD/35/PC",
    "rev_date": "02-01-2018",
    "org_date": "01/08/2016",
    "rev_no": "02",
    "report": "Plating SHIFT-A",
    "shift": "A",
    "date": "23-07-2026",
    "month": "July-26"
  },
  "columns": ["cr_aael", "cr_aad", "engine_guard", "pedal_brake", "pgc", "handle", "total", "renickel", "rechrome", "buff", "ok", "b_grade", "scrap", "grand_total"],
  "sections": [
    {
      "section": "PLATING",
      "rows": [
        {"sr_no": 1, "defect": "PEEL OFF", "reason": "PLATING SHOP", "cr_aael": 0, "cr_aad": 0, "engine_guard": 1, "pedal_brake": 0, "pgc": 0, "handle": 1, "total": 2, "renickel": 0, "rechrome": 0, "buff": 0, "ok": 0, "b_grade": 0, "scrap": 2, "grand_total": 2},
        {"sr_no": 2, "defect": "TOUCHING", "reason": "PLATING SHOP", "cr_aael": 0, "cr_aad": 0, "engine_guard": 0, "pedal_brake": 0, "pgc": 0, "handle": 0, "total": 0, "renickel": 0, "rechrome": 0, "buff": 0, "ok": 0, "b_grade": 0, "scrap": 0, "grand_total": 0},
        {"sr_no": 3, "defect": "DULL PLATING", "reason": "PLATING SHOP", "cr_aael": 25, "cr_aad": 20, "engine_guard": 31, "pedal_brake": 20, "pgc": 10, "handle": 0, "total": 106, "renickel": 106, "rechrome": 0, "buff": 0, "ok": 0, "b_grade": 0, "scrap": 0, "grand_total": 106},
        {"sr_no": 4, "defect": "CLEANING FAULT", "reason": "PLATING SHOP", "cr_aael": 0, "cr_aad": 2, "engine_guard": 7, "pedal_brake": 0, "pgc": 0, "handle": 0, "total": 9, "renickel": 9, "rechrome": 0, "buff": 0, "ok": 0, "b_grade": 0, "scrap": 0, "grand_total": 9},
        {"sr_no": 5, "defect": "PLATING PITTING", "reason": "PLATING SHOP", "cr_aael": 0, "cr_aad": 0, "engine_guard": 0, "pedal_brake": 0, "pgc": 0, "handle": 0, "total": 0, "renickel": 0, "rechrome": 0, "buff": 0, "ok": 0, "b_grade": 0, "scrap": 0, "grand_total": 0},
        {"sr_no": 6, "defect": "YELLOW", "reason": "PLATING SHOP", "cr_aael": 25, "cr_aad": 45, "engine_guard": 0, "pedal_brake": 60, "pgc": 15, "handle": 2, "total": 147, "renickel": 0, "rechrome": 147, "buff": 0, "ok": 0, "b_grade": 0, "scrap": 0, "grand_total": 147},
        {"sr_no": 7, "defect": "ROUGHNESS", "reason": "PLATING SHOP", "cr_aael": 16, "cr_aad": 14, "engine_guard": 20, "pedal_brake": 0, "pgc": 0, "handle": 0, "total": 50, "renickel": 0, "rechrome": 0, "buff": 0, "ok": 50, "b_grade": 0, "scrap": 0, "grand_total": 50},
        {"sr_no": 8, "defect": "PASSIVE BUFF", "reason": "PLATING SHOP", "cr_aael": 0, "cr_aad": 0, "engine_guard": 0, "pedal_brake": 0, "pgc": 0, "handle": 0, "total": 0, "renickel": 0, "rechrome": 0, "buff": 0, "ok": 0, "b_grade": 0, "scrap": 0, "grand_total": 0},
        {"sr_no": 9, "defect": "CHROME BUFF", "reason": "PLATING SHOP", "cr_aael": 16, "cr_aad": 12, "engine_guard": 22, "pedal_brake": 45, "pgc": 0, "handle": 0, "total": 95, "renickel": 0, "rechrome": 0, "buff": 95, "ok": 0, "b_grade": 0, "scrap": 0, "grand_total": 95},
        {"sr_no": 10, "defect": "HANDLE OD OVER SIZE", "reason": "PLATING SHOP", "cr_aael": 0, "cr_aad": 0, "engine_guard": 0, "pedal_brake": 0, "pgc": 0, "handle": 24, "total": 24, "renickel": 0, "rechrome": 0, "buff": 0, "ok": 24, "b_grade": 0, "scrap": 0, "grand_total": 24}
      ],
      "section_total": {"sr_no": null, "defect": "TOTAL PLATING DEFECT", "reason": null, "cr_aael": 82, "cr_aad": 93, "engine_guard": 81, "pedal_brake": 125, "pgc": 25, "handle": 27, "total": 433, "renickel": 115, "rechrome": 147, "buff": 95, "ok": 74, "b_grade": 0, "scrap": 2, "grand_total": 433},
      "section_pct": {"sr_no": null, "defect": "TOTAL PLATING DEFECT %", "reason": null, "cr_aael": null, "cr_aad": null, "engine_guard": null, "pedal_brake": null, "pgc": null, "handle": null, "total": null, "renickel": null, "rechrome": null, "buff": null, "ok": null, "b_grade": null, "scrap": null, "grand_total": null}
    },
    {
      "section": "WELDING",
      "rows": [
        {"sr_no": 11, "defect": "CLEANING FAULT", "reason": "WELD SHOP", "cr_aael": 0, "cr_aad": 0, "engine_guard": 0, "pedal_brake": 0, "pgc": 0, "handle": 0, "total": 0, "renickel": 0, "rechrome": 0, "buff": 0, "ok": 0, "b_grade": 0, "scrap": 0, "grand_total": 0},
        {"sr_no": 12, "defect": "SPATTER", "reason": "WELD SHOP", "cr_aael": 4, "cr_aad": 3, "engine_guard": 2, "pedal_brake": 0, "pgc": 0, "handle": 0, "total": 9, "renickel": 0, "rechrome": 0, "buff": 0, "ok": 9, "b_grade": 0, "scrap": 0, "grand_total": 9},
        {"sr_no": 13, "defect": "MIG WIRE", "reason": "WELD SHOP", "cr_aael": 2, "cr_aad": 1, "engine_guard": 0, "pedal_brake": 0, "pgc": 0, "handle": 0, "total": 3, "renickel": 0, "rechrome": 0, "buff": 0, "ok": 3, "b_grade": 0, "scrap": 0, "grand_total": 3},
        {"sr_no": 14, "defect": "PEDAL CUT", "reason": "WELD SHOP", "cr_aael": 0, "cr_aad": 0, "engine_guard": 0, "pedal_brake": 0, "pgc": 0, "handle": 0, "total": 0, "renickel": 0, "rechrome": 0, "buff": 0, "ok": 0, "b_grade": 0, "scrap": 0, "grand_total": 0},
        {"sr_no": 15, "defect": "TRACIBILITY MISS", "reason": "WELD SHOP", "cr_aael": 0, "cr_aad": 0, "engine_guard": 0, "pedal_brake": 0, "pgc": 0, "handle": 0, "total": 0, "renickel": 0, "rechrome": 0, "buff": 0, "ok": 0, "b_grade": 0, "scrap": 0, "grand_total": 0},
        {"sr_no": 16, "defect": "WELDING PUNCTURE", "reason": "WELD SHOP", "cr_aael": 6, "cr_aad": 4, "engine_guard": 15, "pedal_brake": 0, "pgc": 0, "handle": 0, "total": 25, "renickel": 0, "rechrome": 0, "buff": 0, "ok": 25, "b_grade": 0, "scrap": 0, "grand_total": 25},
        {"sr_no": 17, "defect": "WELDING PIN HOLE", "reason": "WELD SHOP", "cr_aael": 0, "cr_aad": 0, "engine_guard": 0, "pedal_brake": 0, "pgc": 0, "handle": 0, "total": 0, "renickel": 0, "rechrome": 0, "buff": 0, "ok": 0, "b_grade": 0, "scrap": 0, "grand_total": 0},
        {"sr_no": 18, "defect": "ID UNDER SIZE", "reason": "BOP", "cr_aael": 0, "cr_aad": 0, "engine_guard": 0, "pedal_brake": 0, "pgc": 0, "handle": 0, "total": 0, "renickel": 0, "rechrome": 0, "buff": 0, "ok": 0, "b_grade": 0, "scrap": 0, "grand_total": 0},
        {"sr_no": 19, "defect": "WIRE SHORT", "reason": "WELD SHOP", "cr_aael": 0, "cr_aad": 0, "engine_guard": 0, "pedal_brake": 3, "pgc": 0, "handle": 0, "total": 3, "renickel": 0, "rechrome": 0, "buff": 0, "ok": 3, "b_grade": 0, "scrap": 0, "grand_total": 3},
        {"sr_no": 20, "defect": "WELDING MISS", "reason": "WELD SHOP", "cr_aael": 0, "cr_aad": 0, "engine_guard": 0, "pedal_brake": 0, "pgc": 0, "handle": 0, "total": 0, "renickel": 0, "rechrome": 0, "buff": 0, "ok": 0, "b_grade": 0, "scrap": 0, "grand_total": 0},
        {"sr_no": 21, "defect": "WELDING SHIFT", "reason": "WELD SHOP", "cr_aael": 0, "cr_aad": 0, "engine_guard": 0, "pedal_brake": 0, "pgc": 0, "handle": 0, "total": 0, "renickel": 0, "rechrome": 0, "buff": 0, "ok": 0, "b_grade": 0, "scrap": 0, "grand_total": 0},
        {"sr_no": 22, "defect": "SPATTER SOUND", "reason": "WELD SHOP", "cr_aael": 0, "cr_aad": 0, "engine_guard": 3, "pedal_brake": 0, "pgc": 0, "handle": 0, "total": 3, "renickel": 0, "rechrome": 0, "buff": 0, "ok": 0, "b_grade": 3, "scrap": 0, "grand_total": 3},
        {"sr_no": 23, "defect": "WELDING IN COMPLETE", "reason": "WELD SHOP", "cr_aael": 1, "cr_aad": 1, "engine_guard": 0, "pedal_brake": 0, "pgc": 0, "handle": 0, "total": 2, "renickel": 0, "rechrome": 0, "buff": 0, "ok": 2, "b_grade": 0, "scrap": 0, "grand_total": 2},
        {"sr_no": 24, "defect": "EXCESS WELDING", "reason": "WELD SHOP", "cr_aael": 0, "cr_aad": 0, "engine_guard": 0, "pedal_brake": 0, "pgc": 0, "handle": 0, "total": 0, "renickel": 0, "rechrome": 0, "buff": 0, "ok": 0, "b_grade": 0, "scrap": 0, "grand_total": 0},
        {"sr_no": 25, "defect": "GRINDING MARK", "reason": "WELD SHOP", "cr_aael": 2, "cr_aad": 0, "engine_guard": 0, "pedal_brake": 0, "pgc": 0, "handle": 2, "total": 4, "renickel": 2, "rechrome": 0, "buff": 0, "ok": 0, "b_grade": 2, "scrap": 0, "grand_total": 4}
      ],
      "section_total": {"sr_no": null, "defect": "TOTAL WELDING DEFECT", "reason": null, "cr_aael": 15, "cr_aad": 9, "engine_guard": 20, "pedal_brake": 3, "pgc": 0, "handle": 2, "total": 49, "renickel": 2, "rechrome": 0, "buff": 0, "ok": 42, "b_grade": 5, "scrap": 0, "grand_total": 49},
      "section_pct": {"sr_no": null, "defect": "TOTAL WELDING DEFECT %", "reason": null, "cr_aael": null, "cr_aad": null, "engine_guard": null, "pedal_brake": null, "pgc": null, "handle": null, "total": null, "renickel": null, "rechrome": null, "buff": null, "ok": null, "b_grade": null, "scrap": null, "grand_total": null}
    },
    {
      "section": "BOP",
      "rows": [
        {"sr_no": 25, "defect": "RUSTY MARK", "reason": "BOP", "cr_aael": 0, "cr_aad": 0, "engine_guard": 0, "pedal_brake": 0, "pgc": 0, "handle": 0, "total": 0, "renickel": 0, "rechrome": 0, "buff": 0, "ok": 0, "b_grade": 0, "scrap": 0, "grand_total": 0},
        {"sr_no": 26, "defect": "GRAIN OPENING", "reason": "BOP", "cr_aael": 0, "cr_aad": 0, "engine_guard": 0, "pedal_brake": 0, "pgc": 0, "handle": 0, "total": 0, "renickel": 0, "rechrome": 0, "buff": 0, "ok": 0, "b_grade": 0, "scrap": 0, "grand_total": 0},
        {"sr_no": 27, "defect": "PGC THREAD MISS", "reason": "BOP", "cr_aael": 0, "cr_aad": 0, "engine_guard": 0, "pedal_brake": 0, "pgc": 0, "handle": 0, "total": 0, "renickel": 0, "rechrome": 0, "buff": 0, "ok": 0, "b_grade": 0, "scrap": 0, "grand_total": 0},
        {"sr_no": 28, "defect": "MATERIAL FAULT", "reason": "BOP", "cr_aael": 4, "cr_aad": 2, "engine_guard": 10, "pedal_brake": 0, "pgc": 0, "handle": 6, "total": 22, "renickel": 14, "rechrome": 0, "buff": 0, "ok": 0, "b_grade": 8, "scrap": 0, "grand_total": 22},
        {"sr_no": 29, "defect": "MATERIAL PITTING", "reason": "BOP", "cr_aael": 2, "cr_aad": 0, "engine_guard": 0, "pedal_brake": 0, "pgc": 0, "handle": 0, "total": 2, "renickel": 2, "rechrome": 0, "buff": 0, "ok": 0, "b_grade": 0, "scrap": 0, "grand_total": 2},
        {"sr_no": 30, "defect": "PGC THREAD OPPOSITE", "reason": "BOP", "cr_aael": 0, "cr_aad": 0, "engine_guard": 0, "pedal_brake": 0, "pgc": 0, "handle": 0, "total": 0, "renickel": 0, "rechrome": 0, "buff": 0, "ok": 0, "b_grade": 0, "scrap": 0, "grand_total": 0},
        {"sr_no": 31, "defect": "LINE MARK", "reason": "BOP", "cr_aael": 2, "cr_aad": 0, "engine_guard": 0, "pedal_brake": 0, "pgc": 0, "handle": 2, "total": 4, "renickel": 2, "rechrome": 0, "buff": 0, "ok": 0, "b_grade": 2, "scrap": 0, "grand_total": 4},
        {"sr_no": 32, "defect": "CD BEND/STAY", "reason": "WELD SHOP", "cr_aael": 0, "cr_aad": 0, "engine_guard": 3, "pedal_brake": 0, "pgc": 0, "handle": 0, "total": 3, "renickel": 0, "rechrome": 0, "buff": 0, "ok": 3, "b_grade": 0, "scrap": 0, "grand_total": 3},
        {"sr_no": 33, "defect": "BAR LINE", "reason": "PRESS SHOP", "cr_aael": 0, "cr_aad": 0, "engine_guard": 0, "pedal_brake": 0, "pgc": 0, "handle": 0, "total": 0, "renickel": 0, "rechrome": 0, "buff": 0, "ok": 0, "b_grade": 0, "scrap": 0, "grand_total": 0},
        {"sr_no": 34, "defect": "WRINKLE", "reason": "MACHINE SHOP", "cr_aael": 0, "cr_aad": 0, "engine_guard": 0, "pedal_brake": 0, "pgc": 0, "handle": 0, "total": 0, "renickel": 0, "rechrome": 0, "buff": 0, "ok": 0, "b_grade": 0, "scrap": 0, "grand_total": 0},
        {"sr_no": 35, "defect": "DIE MARK ON PIPE", "reason": "PRESS SHOP", "cr_aael": 0, "cr_aad": 0, "engine_guard": 0, "pedal_brake": 0, "pgc": 0, "handle": 0, "total": 0, "renickel": 0, "rechrome": 0, "buff": 0, "ok": 0, "b_grade": 0, "scrap": 0, "grand_total": 0}
      ],
      "section_total": {"sr_no": null, "defect": "TOTAL BOP DEFECT", "reason": null, "cr_aael": 8, "cr_aad": 2, "engine_guard": 13, "pedal_brake": 0, "pgc": 0, "handle": 8, "total": 31, "renickel": 18, "rechrome": 0, "buff": 0, "ok": 3, "b_grade": 10, "scrap": 0, "grand_total": 31},
      "section_pct": {"sr_no": null, "defect": "TOTAL BOP DEFECT%", "reason": null, "cr_aael": null, "cr_aad": null, "engine_guard": null, "pedal_brake": null, "pgc": null, "handle": null, "total": null, "renickel": null, "rechrome": null, "buff": null, "ok": null, "b_grade": null, "scrap": null, "grand_total": null}
    },
    {
      "section": "OTHER",
      "rows": [
        {"sr_no": 36, "defect": "DENT & SCRATCH", "reason": "HANDLING", "cr_aael": 7, "cr_aad": 4, "engine_guard": 10, "pedal_brake": 0, "pgc": 0, "handle": 14, "total": 35, "renickel": 17, "rechrome": 0, "buff": 0, "ok": 0, "b_grade": 18, "scrap": 0, "grand_total": 35},
        {"sr_no": 37, "defect": "BUFFING NG", "reason": "BUFFING SHOP", "cr_aael": 0, "cr_aad": 0, "engine_guard": 0, "pedal_brake": 0, "pgc": 0, "handle": 0, "total": 0, "renickel": 0, "rechrome": 0, "buff": 0, "ok": 0, "b_grade": 0, "scrap": 0, "grand_total": 0},
        {"sr_no": 38, "defect": "WRONG STAMP", "reason": "FEBRACTION", "cr_aael": 0, "cr_aad": 0, "engine_guard": 0, "pedal_brake": 0, "pgc": 0, "handle": 0, "total": 0, "renickel": 0, "rechrome": 0, "buff": 0, "ok": 0, "b_grade": 0, "scrap": 0, "grand_total": 0},
        {"sr_no": 39, "defect": "HANDLE PUNCH MISS", "reason": "FEBRACTION", "cr_aael": 0, "cr_aad": 0, "engine_guard": 0, "pedal_brake": 0, "pgc": 0, "handle": 0, "total": 0, "renickel": 0, "rechrome": 0, "buff": 0, "ok": 0, "b_grade": 0, "scrap": 0, "grand_total": 0},
        {"sr_no": 40, "defect": "HANDLE STAMP MISS", "reason": "FEBRACTION", "cr_aael": 0, "cr_aad": 0, "engine_guard": 0, "pedal_brake": 0, "pgc": 0, "handle": 0, "total": 0, "renickel": 0, "rechrome": 0, "buff": 0, "ok": 0, "b_grade": 0, "scrap": 0, "grand_total": 0}
      ],
      "section_total": {"sr_no": null, "defect": "TOTAL OTHER PLATING DEFECT", "reason": null, "cr_aael": null, "cr_aad": null, "engine_guard": null, "pedal_brake": null, "pgc": null, "handle": null, "total": null, "renickel": null, "rechrome": null, "buff": null, "ok": null, "b_grade": null, "scrap": null, "grand_total": null},
      "section_pct": {"sr_no": null, "defect": "OTHER DEFECT %", "reason": null, "cr_aael": null, "cr_aad": null, "engine_guard": null, "pedal_brake": null, "pgc": null, "handle": null, "total": null, "renickel": null, "rechrome": null, "buff": null, "ok": null, "b_grade": null, "scrap": null, "grand_total": null}
    },
    {
      "section": "CENTER LESS",
      "rows": [
        {"sr_no": 41, "defect": "CENTERLESS MARK", "reason": "CENTER LESS SHOP", "cr_aael": 0, "cr_aad": 0, "engine_guard": 0, "pedal_brake": 0, "pgc": 0, "handle": 0, "total": 0, "renickel": 0, "rechrome": 0, "buff": 0, "ok": 0, "b_grade": 0, "scrap": 0, "grand_total": 0}
      ],
      "section_total": {"sr_no": null, "defect": "TOTAL CENTER LESS DEFECT", "reason": null, "cr_aael": null, "cr_aad": null, "engine_guard": null, "pedal_brake": null, "pgc": null, "handle": null, "total": null, "renickel": null, "rechrome": null, "buff": null, "ok": null, "b_grade": null, "scrap": null, "grand_total": null},
      "section_pct": {"sr_no": null, "defect": "CENTER LESS DEFECT %", "reason": null, "cr_aael": null, "cr_aad": null, "engine_guard": null, "pedal_brake": null, "pgc": null, "handle": null, "total": null, "renickel": null, "rechrome": null, "buff": null, "ok": null, "b_grade": null, "scrap": null, "grand_total": null}
    }
  ],
  "summary": [
    {"sr_no": null, "defect": "OVER ALL TOTAL DEFECT", "reason": null, "cr_aael": 112, "cr_aad": 108, "engine_guard": 124, "pedal_brake": 128, "pgc": 25, "handle": 51, "total": 548, "renickel": 152, "rechrome": 147, "buff": 95, "ok": 119, "b_grade": 33, "scrap": 2, "grand_total": 548},
    {"sr_no": null, "defect": "TOTAL DEFECT %", "reason": null, "cr_aael": null, "cr_aad": null, "engine_guard": null, "pedal_brake": null, "pgc": null, "handle": null, "total": null, "renickel": null, "rechrome": null, "buff": null, "ok": null, "b_grade": null, "scrap": null, "grand_total": null},
    {"sr_no": null, "defect": "TOTAL STRAIGHT PASS", "reason": null, "cr_aael": null, "cr_aad": null, "engine_guard": null, "pedal_brake": null, "pgc": null, "handle": null, "total": null, "renickel": null, "rechrome": null, "buff": null, "ok": null, "b_grade": null, "scrap": null, "grand_total": null},
    {"sr_no": null, "defect": "PRODUCTION", "reason": null, "cr_aael": null, "cr_aad": null, "engine_guard": null, "pedal_brake": null, "pgc": null, "handle": null, "total": null, "renickel": null, "rechrome": null, "buff": null, "ok": null, "b_grade": null, "scrap": null, "grand_total": null},
    {"sr_no": null, "defect": "REWORK PLANT PRODUCTION", "reason": null, "cr_aael": null, "cr_aad": null, "engine_guard": null, "pedal_brake": null, "pgc": null, "handle": null, "total": null, "renickel": null, "rechrome": null, "buff": null, "ok": null, "b_grade": null, "scrap": null, "grand_total": null},
    {"sr_no": null, "defect": "REWORK PLANT DEFECT QTY", "reason": null, "cr_aael": null, "cr_aad": null, "engine_guard": null, "pedal_brake": null, "pgc": null, "handle": null, "total": null, "renickel": null, "rechrome": null, "buff": null, "ok": null, "b_grade": null, "scrap": null, "grand_total": null},
    {"sr_no": null, "defect": "CUSTOMER RETURN DEFECT QTY", "reason": null, "cr_aael": null, "cr_aad": null, "engine_guard": null, "pedal_brake": null, "pgc": null, "handle": null, "total": null, "renickel": null, "rechrome": null, "buff": null, "ok": null, "b_grade": null, "scrap": null, "grand_total": null},
    {"sr_no": null, "defect": "TOTAL DEFECT QTY", "reason": null, "cr_aael": null, "cr_aad": null, "engine_guard": null, "pedal_brake": null, "pgc": null, "handle": null, "total": null, "renickel": null, "rechrome": null, "buff": null, "ok": null, "b_grade": null, "scrap": null, "grand_total": null}
  ],
  "b_grade_note": {
    "carrier_aad": null,
    "engine_guard": null,
    "pedal_brake": null,
    "handle": null,
    "pgc": null
  },
  "signatures": {
    "prepared_by": false,
    "reviewed_by": false,
    "approved_by": true
  }
}
```

Reading notes:

- The printed doc no. on this plating sheet really reads `QAD/35/PC` (the other plating sheets say `/PL`).
- Dashes are returned as `0`; empty cells as `null`.
- Many handwritten totals sit half a row lower than their defect row. They are assigned to the row whose values they add up to.
- Row 18 ID UNDER SIZE / row 19 WIRE SHORT: the single `03` (PEDAL BRAKE) sits on the line between them. It is given to WIRE SHORT.
- TOTAL OTHER PLATING DEFECT was not filled in (row 36 alone is 35), so it is `null`.
- Every row TOTAL and every section total in this JSON has been checked against its cells and matches the sheet.

---

## `DAILY MIS REPORT PCOATING.pdf`

Response type: **`MIS_POWDER_COATING_DAILY`**

```json
{
  "form_type": "MIS_POWDER_COATING_DAILY",
  "header": {
    "company": "MEENAKSHI POLYMERS PVT.LTD. HARIDWAR",
    "doc_no": "MPPL/F/QAD/35/PC",
    "rev_date": "02-01-2018",
    "org_date": "01/08/2016",
    "rev_no": "02",
    "report": null,
    "shift": null,
    "date": "24/7/26",
    "month": "July-26"
  },
  "columns": ["sg_aah", "sg_aae", "sw_aah", "sp_aae", "sg_aacy", "sw_aacy", "eg_kvh", "cross_pipe_aael", "handle_abza", "eg_abza", "pb_abz", "pgc_ack", "pb_ack", "handle_aack", "side_stand", "sg_aael", "total"],
  "sections": [
    {
      "section": "POWDER COATING",
      "rows": [
        {"sr_no": 1, "defect": "PAINT PEEL OFF", "reason": "POWDER COATING", "sg_aah": 2, "sg_aae": 4, "sw_aah": 0, "sp_aae": 0, "sg_aacy": 1, "sw_aacy": 0, "eg_kvh": 0, "cross_pipe_aael": 0, "handle_abza": 0, "eg_abza": 0, "pb_abz": 0, "pgc_ack": 0, "pb_ack": 0, "handle_aack": 0, "side_stand": 0, "sg_aael": 0, "total": 7},
        {"sr_no": 2, "defect": "TOUCHING MARK", "reason": "POWDER COATING", "sg_aah": 0, "sg_aae": 0, "sw_aah": 0, "sp_aae": 0, "sg_aacy": 0, "sw_aacy": 0, "eg_kvh": 0, "cross_pipe_aael": 0, "handle_abza": 0, "eg_abza": 0, "pb_abz": 0, "pgc_ack": 0, "pb_ack": 0, "handle_aack": 0, "side_stand": 0, "sg_aael": 0, "total": 0},
        {"sr_no": 3, "defect": "DULL PAINT", "reason": "POWDER COATING", "sg_aah": 0, "sg_aae": 0, "sw_aah": 0, "sp_aae": 0, "sg_aacy": 0, "sw_aacy": 0, "eg_kvh": 0, "cross_pipe_aael": 0, "handle_abza": 0, "eg_abza": 0, "pb_abz": 0, "pgc_ack": 0, "pb_ack": 0, "handle_aack": 0, "side_stand": 0, "sg_aael": 0, "total": 0},
        {"sr_no": 4, "defect": "PAINT DRY", "reason": "POWDER COATING", "sg_aah": 18, "sg_aae": 8, "sw_aah": 0, "sp_aae": 6, "sg_aacy": 26, "sw_aacy": 0, "eg_kvh": 18, "cross_pipe_aael": 6, "handle_abza": 0, "eg_abza": 0, "pb_abz": 0, "pgc_ack": 0, "pb_ack": 0, "handle_aack": 0, "side_stand": 0, "sg_aael": 0, "total": 82},
        {"sr_no": 5, "defect": "UNCOVERAGE", "reason": "POWDER COATING", "sg_aah": 0, "sg_aae": 0, "sw_aah": 0, "sp_aae": 0, "sg_aacy": 0, "sw_aacy": 0, "eg_kvh": 0, "cross_pipe_aael": 0, "handle_abza": 0, "eg_abza": 0, "pb_abz": 0, "pgc_ack": 0, "pb_ack": 0, "handle_aack": 0, "side_stand": 0, "sg_aael": 0, "total": 0},
        {"sr_no": 6, "defect": "DUST MARK/RUSTY", "reason": "POWDER COATING", "sg_aah": 2, "sg_aae": 0, "sw_aah": 0, "sp_aae": 0, "sg_aacy": 0, "sw_aacy": 0, "eg_kvh": 11, "cross_pipe_aael": 0, "handle_abza": 0, "eg_abza": 0, "pb_abz": 0, "pgc_ack": 0, "pb_ack": 0, "handle_aack": 0, "side_stand": 0, "sg_aael": 0, "total": 13},
        {"sr_no": 7, "defect": "PIN HOLE", "reason": "POWDER COATING", "sg_aah": 31, "sg_aae": 28, "sw_aah": 0, "sp_aae": 0, "sg_aacy": 0, "sw_aacy": 0, "eg_kvh": 131, "cross_pipe_aael": 67, "handle_abza": 0, "eg_abza": 0, "pb_abz": 0, "pgc_ack": 0, "pb_ack": 0, "handle_aack": 0, "side_stand": 0, "sg_aael": 0, "total": 257},
        {"sr_no": 8, "defect": "ORANGE PEEL", "reason": "POWDER COATING", "sg_aah": 0, "sg_aae": 21, "sw_aah": 0, "sp_aae": 0, "sg_aacy": 0, "sw_aacy": 0, "eg_kvh": 0, "cross_pipe_aael": 0, "handle_abza": 0, "eg_abza": 0, "pb_abz": 0, "pgc_ack": 0, "pb_ack": 0, "handle_aack": 0, "side_stand": 0, "sg_aael": 0, "total": 21},
        {"sr_no": 9, "defect": "OIL MARK", "reason": "POWDER COATING", "sg_aah": 2, "sg_aae": 8, "sw_aah": 0, "sp_aae": 0, "sg_aacy": 3, "sw_aacy": 0, "eg_kvh": 8, "cross_pipe_aael": 0, "handle_abza": 0, "eg_abza": 0, "pb_abz": 0, "pgc_ack": 0, "pb_ack": 0, "handle_aack": 0, "side_stand": 0, "sg_aael": 0, "total": 21},
        {"sr_no": 10, "defect": "WATER MARK", "reason": "POWDER COATING", "sg_aah": 0, "sg_aae": 0, "sw_aah": 8, "sp_aae": 34, "sg_aacy": 0, "sw_aacy": 14, "eg_kvh": 0, "cross_pipe_aael": 0, "handle_abza": 0, "eg_abza": 0, "pb_abz": 0, "pgc_ack": 0, "pb_ack": 0, "handle_aack": 0, "side_stand": 0, "sg_aael": 0, "total": 56},
        {"sr_no": 11, "defect": "SISSING", "reason": "POWDER COATING", "sg_aah": 0, "sg_aae": 0, "sw_aah": 0, "sp_aae": 0, "sg_aacy": 0, "sw_aacy": 0, "eg_kvh": 0, "cross_pipe_aael": 0, "handle_abza": 0, "eg_abza": 0, "pb_abz": 0, "pgc_ack": 0, "pb_ack": 0, "handle_aack": 0, "side_stand": 0, "sg_aael": 0, "total": 0},
        {"sr_no": 12, "defect": "OVER PAINT FLOW", "reason": "POWDER COATING", "sg_aah": 8, "sg_aae": 48, "sw_aah": 2, "sp_aae": 4, "sg_aacy": 2, "sw_aacy": 0, "eg_kvh": 14, "cross_pipe_aael": 2, "handle_abza": 0, "eg_abza": 0, "pb_abz": 0, "pgc_ack": 0, "pb_ack": 0, "handle_aack": 0, "side_stand": 0, "sg_aael": 0, "total": 80},
        {"sr_no": 13, "defect": "GRINDER MARK", "reason": "POWDER COATING", "sg_aah": 0, "sg_aae": 0, "sw_aah": 0, "sp_aae": 0, "sg_aacy": 0, "sw_aacy": 0, "eg_kvh": 0, "cross_pipe_aael": 0, "handle_abza": 0, "eg_abza": 0, "pb_abz": 0, "pgc_ack": 0, "pb_ack": 0, "handle_aack": 0, "side_stand": 0, "sg_aael": 0, "total": 0},
        {"sr_no": 14, "defect": "SCRATCH MARK", "reason": "POWDER COATING", "sg_aah": 5, "sg_aae": 9, "sw_aah": 0, "sp_aae": 0, "sg_aacy": 6, "sw_aacy": 0, "eg_kvh": 6, "cross_pipe_aael": 4, "handle_abza": 0, "eg_abza": 0, "pb_abz": 0, "pgc_ack": 0, "pb_ack": 0, "handle_aack": 0, "side_stand": 0, "sg_aael": 0, "total": 30}
      ],
      "section_total": {"sr_no": null, "defect": "TOTAL POWDER COATING DEFECT QTY", "reason": null, "sg_aah": 68, "sg_aae": 126, "sw_aah": 10, "sp_aae": 44, "sg_aacy": 38, "sw_aacy": 14, "eg_kvh": 188, "cross_pipe_aael": 79, "handle_abza": 0, "eg_abza": 0, "pb_abz": 0, "pgc_ack": 0, "pb_ack": 0, "handle_aack": 0, "side_stand": 0, "sg_aael": 0, "total": 567},
      "section_pct": {"sr_no": null, "defect": "STRAIGHT PASS", "reason": null, "sg_aah": null, "sg_aae": null, "sw_aah": null, "sp_aae": null, "sg_aacy": null, "sw_aacy": null, "eg_kvh": null, "cross_pipe_aael": null, "handle_abza": null, "eg_abza": null, "pb_abz": null, "pgc_ack": null, "pb_ack": null, "handle_aack": null, "side_stand": null, "sg_aael": null, "total": null}
    },
    {
      "section": "OTHER",
      "rows": [
        {"sr_no": 14, "defect": "SPATTER SOUND", "reason": "WELD SHOP", "sg_aah": null, "sg_aae": null, "sw_aah": null, "sp_aae": null, "sg_aacy": null, "sw_aacy": null, "eg_kvh": null, "cross_pipe_aael": null, "handle_abza": null, "eg_abza": null, "pb_abz": null, "pgc_ack": null, "pb_ack": null, "handle_aack": null, "side_stand": null, "sg_aael": null, "total": null},
        {"sr_no": 15, "defect": "WELDING SPATTER/MIG WIRE", "reason": "WELD SHOP", "sg_aah": null, "sg_aae": null, "sw_aah": null, "sp_aae": null, "sg_aacy": null, "sw_aacy": null, "eg_kvh": null, "cross_pipe_aael": null, "handle_abza": null, "eg_abza": null, "pb_abz": null, "pgc_ack": null, "pb_ack": null, "handle_aack": null, "side_stand": null, "sg_aael": null, "total": null},
        {"sr_no": 16, "defect": "PEDAL CUT", "reason": "WELD SHOP", "sg_aah": null, "sg_aae": null, "sw_aah": null, "sp_aae": null, "sg_aacy": null, "sw_aacy": null, "eg_kvh": null, "cross_pipe_aael": null, "handle_abza": null, "eg_abza": null, "pb_abz": null, "pgc_ack": null, "pb_ack": null, "handle_aack": null, "side_stand": null, "sg_aael": null, "total": null},
        {"sr_no": 17, "defect": "PIVOT SHIFT", "reason": "WELD SHOP", "sg_aah": null, "sg_aae": null, "sw_aah": null, "sp_aae": null, "sg_aacy": null, "sw_aacy": null, "eg_kvh": null, "cross_pipe_aael": null, "handle_abza": null, "eg_abza": null, "pb_abz": null, "pgc_ack": null, "pb_ack": null, "handle_aack": null, "side_stand": null, "sg_aael": null, "total": null},
        {"sr_no": 18, "defect": "PUNCH MISS/STAMP MISS", "reason": "WELD SHOP", "sg_aah": null, "sg_aae": null, "sw_aah": null, "sp_aae": null, "sg_aacy": null, "sw_aacy": null, "eg_kvh": null, "cross_pipe_aael": null, "handle_abza": null, "eg_abza": null, "pb_abz": null, "pgc_ack": null, "pb_ack": null, "handle_aack": null, "side_stand": null, "sg_aael": null, "total": null},
        {"sr_no": 19, "defect": "PART/DRAIN HOLE MISS", "reason": "WELD SHOP", "sg_aah": null, "sg_aae": null, "sw_aah": null, "sp_aae": null, "sg_aacy": null, "sw_aacy": null, "eg_kvh": null, "cross_pipe_aael": null, "handle_abza": null, "eg_abza": null, "pb_abz": null, "pgc_ack": null, "pb_ack": null, "handle_aack": null, "side_stand": null, "sg_aael": null, "total": null},
        {"sr_no": 20, "defect": "BOLT BROKEN", "reason": "WELD SHOP", "sg_aah": null, "sg_aae": null, "sw_aah": null, "sp_aae": null, "sg_aacy": null, "sw_aacy": null, "eg_kvh": null, "cross_pipe_aael": null, "handle_abza": null, "eg_abza": null, "pb_abz": null, "pgc_ack": null, "pb_ack": null, "handle_aack": null, "side_stand": null, "sg_aael": null, "total": null},
        {"sr_no": 21, "defect": "STAY BEND", "reason": "WELD SHOP", "sg_aah": null, "sg_aae": null, "sw_aah": null, "sp_aae": null, "sg_aacy": null, "sw_aacy": null, "eg_kvh": null, "cross_pipe_aael": null, "handle_abza": null, "eg_abza": null, "pb_abz": null, "pgc_ack": null, "pb_ack": null, "handle_aack": null, "side_stand": null, "sg_aael": null, "total": null},
        {"sr_no": 22, "defect": "U-BKT BEND", "reason": "WELD SHOP", "sg_aah": null, "sg_aae": null, "sw_aah": null, "sp_aae": null, "sg_aacy": null, "sw_aacy": null, "eg_kvh": null, "cross_pipe_aael": null, "handle_abza": null, "eg_abza": null, "pb_abz": null, "pgc_ack": null, "pb_ack": null, "handle_aack": null, "side_stand": null, "sg_aael": null, "total": null},
        {"sr_no": 23, "defect": "GAUGE OUT/OD NG", "reason": "WELD SHOP", "sg_aah": null, "sg_aae": null, "sw_aah": null, "sp_aae": null, "sg_aacy": null, "sw_aacy": null, "eg_kvh": null, "cross_pipe_aael": null, "handle_abza": null, "eg_abza": null, "pb_abz": null, "pgc_ack": null, "pb_ack": null, "handle_aack": null, "side_stand": null, "sg_aael": null, "total": null},
        {"sr_no": 24, "defect": "BLOW HOLE", "reason": "WELD SHOP", "sg_aah": null, "sg_aae": null, "sw_aah": null, "sp_aae": null, "sg_aacy": null, "sw_aacy": null, "eg_kvh": null, "cross_pipe_aael": null, "handle_abza": null, "eg_abza": null, "pb_abz": null, "pgc_ack": null, "pb_ack": null, "handle_aack": null, "side_stand": null, "sg_aael": null, "total": null},
        {"sr_no": 25, "defect": "WELDING SHIFT", "reason": "WELD SHOP", "sg_aah": null, "sg_aae": null, "sw_aah": null, "sp_aae": null, "sg_aacy": null, "sw_aacy": null, "eg_kvh": null, "cross_pipe_aael": null, "handle_abza": null, "eg_abza": null, "pb_abz": null, "pgc_ack": null, "pb_ack": null, "handle_aack": null, "side_stand": null, "sg_aael": null, "total": null},
        {"sr_no": 26, "defect": "GRINDER MARK", "reason": "WELD SHOP", "sg_aah": null, "sg_aae": null, "sw_aah": null, "sp_aae": null, "sg_aacy": null, "sw_aacy": null, "eg_kvh": null, "cross_pipe_aael": null, "handle_abza": null, "eg_abza": null, "pb_abz": null, "pgc_ack": null, "pb_ack": null, "handle_aack": null, "side_stand": null, "sg_aael": null, "total": null},
        {"sr_no": 27, "defect": "EXCESS/OVER WELD", "reason": "WELD SHOP", "sg_aah": null, "sg_aae": null, "sw_aah": null, "sp_aae": null, "sg_aacy": null, "sw_aacy": null, "eg_kvh": null, "cross_pipe_aael": null, "handle_abza": null, "eg_abza": null, "pb_abz": null, "pgc_ack": null, "pb_ack": null, "handle_aack": null, "side_stand": null, "sg_aael": null, "total": null},
        {"sr_no": 28, "defect": "WELDING IN COMPLETE/LEAKAGE", "reason": "WELD SHOP", "sg_aah": null, "sg_aae": null, "sw_aah": null, "sp_aae": null, "sg_aacy": null, "sw_aacy": null, "eg_kvh": null, "cross_pipe_aael": null, "handle_abza": null, "eg_abza": null, "pb_abz": null, "pgc_ack": null, "pb_ack": null, "handle_aack": null, "side_stand": null, "sg_aael": null, "total": null},
        {"sr_no": 29, "defect": "WIRE SHORT", "reason": "WELD SHOP", "sg_aah": null, "sg_aae": null, "sw_aah": null, "sp_aae": null, "sg_aacy": null, "sw_aacy": null, "eg_kvh": null, "cross_pipe_aael": null, "handle_abza": null, "eg_abza": null, "pb_abz": null, "pgc_ack": null, "pb_ack": null, "handle_aack": null, "side_stand": null, "sg_aael": null, "total": null},
        {"sr_no": 30, "defect": "WELDING BROKEN BAR-A", "reason": "WELD SHOP", "sg_aah": null, "sg_aae": null, "sw_aah": null, "sp_aae": null, "sg_aacy": null, "sw_aacy": null, "eg_kvh": null, "cross_pipe_aael": null, "handle_abza": null, "eg_abza": null, "pb_abz": null, "pgc_ack": null, "pb_ack": null, "handle_aack": null, "side_stand": null, "sg_aael": null, "total": null},
        {"sr_no": 31, "defect": "WELDING BROKEN BAR-B", "reason": "WELD SHOP", "sg_aah": null, "sg_aae": null, "sw_aah": null, "sp_aae": null, "sg_aacy": null, "sw_aacy": null, "eg_kvh": null, "cross_pipe_aael": null, "handle_abza": null, "eg_abza": null, "pb_abz": null, "pgc_ack": null, "pb_ack": null, "handle_aack": null, "side_stand": null, "sg_aael": null, "total": null},
        {"sr_no": 32, "defect": "WELDING BROKEN BAR-C", "reason": "WELD SHOP", "sg_aah": null, "sg_aae": null, "sw_aah": null, "sp_aae": null, "sg_aacy": null, "sw_aacy": null, "eg_kvh": null, "cross_pipe_aael": null, "handle_abza": null, "eg_abza": null, "pb_abz": null, "pgc_ack": null, "pb_ack": null, "handle_aack": null, "side_stand": null, "sg_aael": null, "total": null},
        {"sr_no": 33, "defect": "PIPE LINE/CRACK", "reason": "BOP", "sg_aah": null, "sg_aae": null, "sw_aah": null, "sp_aae": null, "sg_aacy": null, "sw_aacy": null, "eg_kvh": null, "cross_pipe_aael": null, "handle_abza": null, "eg_abza": null, "pb_abz": null, "pgc_ack": null, "pb_ack": null, "handle_aack": null, "side_stand": null, "sg_aael": null, "total": null},
        {"sr_no": 34, "defect": "MATERIAL FAULT", "reason": "BOP", "sg_aah": null, "sg_aae": null, "sw_aah": null, "sp_aae": null, "sg_aacy": null, "sw_aacy": null, "eg_kvh": null, "cross_pipe_aael": null, "handle_abza": null, "eg_abza": null, "pb_abz": null, "pgc_ack": null, "pb_ack": null, "handle_aack": null, "side_stand": null, "sg_aael": null, "total": null},
        {"sr_no": 35, "defect": "MATERIAL PITTING", "reason": "BOP", "sg_aah": null, "sg_aae": null, "sw_aah": null, "sp_aae": null, "sg_aacy": null, "sw_aacy": null, "eg_kvh": null, "cross_pipe_aael": null, "handle_abza": null, "eg_abza": null, "pb_abz": null, "pgc_ack": null, "pb_ack": null, "handle_aack": null, "side_stand": null, "sg_aael": null, "total": null},
        {"sr_no": 36, "defect": "THREAD WASH", "reason": "BOP", "sg_aah": null, "sg_aae": null, "sw_aah": null, "sp_aae": null, "sg_aacy": null, "sw_aacy": null, "eg_kvh": null, "cross_pipe_aael": null, "handle_abza": null, "eg_abza": null, "pb_abz": null, "pgc_ack": null, "pb_ack": null, "handle_aack": null, "side_stand": null, "sg_aael": null, "total": null},
        {"sr_no": 37, "defect": "DENT & SCRATCH", "reason": "PRESS SHOP", "sg_aah": null, "sg_aae": null, "sw_aah": null, "sp_aae": null, "sg_aacy": null, "sw_aacy": null, "eg_kvh": null, "cross_pipe_aael": null, "handle_abza": null, "eg_abza": null, "pb_abz": null, "pgc_ack": null, "pb_ack": null, "handle_aack": null, "side_stand": null, "sg_aael": null, "total": null}
      ],
      "section_total": {"sr_no": null, "defect": "TOTAL OTHER DEFECT QTY", "reason": null, "sg_aah": null, "sg_aae": null, "sw_aah": null, "sp_aae": null, "sg_aacy": null, "sw_aacy": null, "eg_kvh": null, "cross_pipe_aael": null, "handle_abza": null, "eg_abza": null, "pb_abz": null, "pgc_ack": null, "pb_ack": null, "handle_aack": null, "side_stand": null, "sg_aael": null, "total": null},
      "section_pct": {"sr_no": null, "defect": "OTHER DEFECT STRAIGHT PASS", "reason": null, "sg_aah": null, "sg_aae": null, "sw_aah": null, "sp_aae": null, "sg_aacy": null, "sw_aacy": null, "eg_kvh": null, "cross_pipe_aael": null, "handle_abza": null, "eg_abza": null, "pb_abz": null, "pgc_ack": null, "pb_ack": null, "handle_aack": null, "side_stand": null, "sg_aael": null, "total": null}
    }
  ],
  "summary": [
    {"sr_no": null, "defect": "OVER ALL TOTAL DEFECT", "reason": null, "sg_aah": null, "sg_aae": null, "sw_aah": null, "sp_aae": null, "sg_aacy": null, "sw_aacy": null, "eg_kvh": null, "cross_pipe_aael": null, "handle_abza": null, "eg_abza": null, "pb_abz": null, "pgc_ack": null, "pb_ack": null, "handle_aack": null, "side_stand": null, "sg_aael": null, "total": null},
    {"sr_no": null, "defect": "PRODUCTION", "reason": null, "sg_aah": null, "sg_aae": null, "sw_aah": null, "sp_aae": null, "sg_aacy": null, "sw_aacy": null, "eg_kvh": null, "cross_pipe_aael": null, "handle_abza": null, "eg_abza": null, "pb_abz": null, "pgc_ack": null, "pb_ack": null, "handle_aack": null, "side_stand": null, "sg_aael": null, "total": null},
    {"sr_no": null, "defect": "OVER ALL TOTAL STRAIGHT PASS", "reason": null, "sg_aah": null, "sg_aae": null, "sw_aah": null, "sp_aae": null, "sg_aacy": null, "sw_aacy": null, "eg_kvh": null, "cross_pipe_aael": null, "handle_abza": null, "eg_abza": null, "pb_abz": null, "pgc_ack": null, "pb_ack": null, "handle_aack": null, "side_stand": null, "sg_aael": null, "total": null},
    {"sr_no": null, "defect": "REWORK PLANT PRODUCTION", "reason": null, "sg_aah": null, "sg_aae": null, "sw_aah": null, "sp_aae": null, "sg_aacy": null, "sw_aacy": null, "eg_kvh": null, "cross_pipe_aael": null, "handle_abza": null, "eg_abza": null, "pb_abz": null, "pgc_ack": null, "pb_ack": null, "handle_aack": null, "side_stand": null, "sg_aael": null, "total": null},
    {"sr_no": null, "defect": "REWORK PLANT DEFECT QTY", "reason": null, "sg_aah": null, "sg_aae": null, "sw_aah": null, "sp_aae": null, "sg_aacy": null, "sw_aacy": null, "eg_kvh": null, "cross_pipe_aael": null, "handle_abza": null, "eg_abza": null, "pb_abz": null, "pgc_ack": null, "pb_ack": null, "handle_aack": null, "side_stand": null, "sg_aael": null, "total": null},
    {"sr_no": null, "defect": "REWORK PLANT OK QTY", "reason": null, "sg_aah": null, "sg_aae": null, "sw_aah": null, "sp_aae": null, "sg_aacy": null, "sw_aacy": null, "eg_kvh": null, "cross_pipe_aael": null, "handle_abza": null, "eg_abza": null, "pb_abz": null, "pgc_ack": null, "pb_ack": null, "handle_aack": null, "side_stand": null, "sg_aael": null, "total": null},
    {"sr_no": null, "defect": "CUSTOMER RETURN DEFECT QTY", "reason": null, "sg_aah": null, "sg_aae": null, "sw_aah": null, "sp_aae": null, "sg_aacy": null, "sw_aacy": null, "eg_kvh": null, "cross_pipe_aael": null, "handle_abza": null, "eg_abza": null, "pb_abz": null, "pgc_ack": null, "pb_ack": null, "handle_aack": null, "side_stand": null, "sg_aael": null, "total": null},
    {"sr_no": null, "defect": "TOTAL DEFECT QTY", "reason": null, "sg_aah": null, "sg_aae": null, "sw_aah": null, "sp_aae": null, "sg_aacy": null, "sw_aacy": null, "eg_kvh": null, "cross_pipe_aael": null, "handle_abza": null, "eg_abza": null, "pb_abz": null, "pgc_ack": null, "pb_ack": null, "handle_aack": null, "side_stand": null, "sg_aael": null, "total": null}
  ],
  "signatures": {
    "prepared_by": false,
    "checked_by": true
  }
}
```

Reading notes:

- The handwritten TOTAL column is shifted one row down on paper. Row totals here are assigned to their own row and match the sum of the row.
- Only the powder-coating section (rows 1–14) is filled; rows 14–37 and all summary rows are blank, so they are `null`.
- Column `side_stand` is printed `SIDE STAND` here (ABZA in other copies).

---

## `ST. Pass weld data.pdf`

Response type: **`ST_PASS_MONITORING`**

```json
{
  "form_type": "ST_PASS_MONITORING",
  "header": {
    "company": "MEENAKSHI POLYMERS PVT. LTD. HARIDWAR",
    "title": "BS6 MODEL COMPONENT STRAIGHT PASS MONITORING SHEET",
    "month": null,
    "days_printed": [21, 22, 23, 24, 25, 26, 27, 28, 29, 30]
  },
  "parts": [
    {
      "sno": 1,
      "part_name": "ENGINE GUARD KST",
      "part_no": "50803-KST-9400",
      "model": "KST",
      "days": {
        "21": {
          "A": {"checked_qty": 2920, "ok_qty": 2419, "rej_qty": null, "rew_qty": 495, "total_rework_ok_qty": 495, "inspection_chipping_pending": 82, "visual_defects_straight_pass_pct": 82, "non_conformity_count": null},
          "B": {"checked_qty": 1768, "ok_qty": 1093, "rej_qty": null, "rew_qty": 172, "total_rework_ok_qty": 672, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 61, "non_conformity_count": null},
          "TOTAL": null
        },
        "22": {
          "A": {"checked_qty": 2900, "ok_qty": 2283, "rej_qty": null, "rew_qty": 581, "total_rework_ok_qty": 561, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 78, "non_conformity_count": null},
          "B": {"checked_qty": 2180, "ok_qty": 1445, "rej_qty": null, "rew_qty": 735, "total_rework_ok_qty": 735, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 66, "non_conformity_count": null},
          "TOTAL": null
        },
        "23": {
          "A": {"checked_qty": 2700, "ok_qty": 1983, "rej_qty": null, "rew_qty": 717, "total_rework_ok_qty": 717, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 73, "non_conformity_count": null},
          "B": {"checked_qty": 2550, "ok_qty": 1968, "rej_qty": null, "rew_qty": 582, "total_rework_ok_qty": 582, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 77, "non_conformity_count": null},
          "TOTAL": null
        }
      }
    },
    {
      "sno": 2,
      "part_name": "CARRIER LINE-01",
      "part_no": "81200-AAD-3000",
      "model": "AAD",
      "days": {
        "21": "NO PLAN",
        "22": "NO PLAN",
        "23": "NO PLAN"
      }
    },
    {
      "sno": 3,
      "part_name": "CARRIER LINE-R",
      "part_no": "81200-AAD-3000",
      "model": "AAD",
      "days": {
        "21": {
          "A": null,
          "B": {"checked_qty": 1512, "ok_qty": 680, "rej_qty": null, "rew_qty": 834, "total_rework_ok_qty": 834, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 44, "non_conformity_count": null},
          "TOTAL": null
        },
        "22": {
          "A": null,
          "B": {"checked_qty": 1464, "ok_qty": null, "rej_qty": null, "rew_qty": null, "total_rework_ok_qty": null, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": null, "non_conformity_count": null},
          "TOTAL": null
        },
        "23": {
          "A": {"checked_qty": 1467, "ok_qty": 769, "rej_qty": 0, "rew_qty": 698, "total_rework_ok_qty": 698, "inspection_chipping_pending": 0, "visual_defects_straight_pass_pct": 52, "non_conformity_count": 0},
          "B": null,
          "TOTAL": null
        }
      }
    },
    {
      "sno": 4,
      "part_name": "CARRIER LINE-R AAEL",
      "part_no": "81200A-AAE-3000",
      "model": "AAEL",
      "days": {
        "21": {
          "A": {"checked_qty": 1185, "ok_qty": 705, "rej_qty": null, "rew_qty": 480, "total_rework_ok_qty": 480, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 59, "non_conformity_count": null},
          "B": {"checked_qty": 1210, "ok_qty": 594, "rej_qty": null, "rew_qty": 616, "total_rework_ok_qty": 611, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 50, "non_conformity_count": null},
          "TOTAL": {"checked_qty": 1200, "ok_qty": 711, "rej_qty": null, "rew_qty": 489, "total_rework_ok_qty": 489, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 59, "non_conformity_count": null}
        },
        "22": {
          "A": null,
          "B": {"checked_qty": 1180, "ok_qty": 547, "rej_qty": null, "rew_qty": 633, "total_rework_ok_qty": 633, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 48, "non_conformity_count": null},
          "TOTAL": null
        },
        "23": {
          "A": {"checked_qty": 1215, "ok_qty": 685, "rej_qty": 0, "rew_qty": 530, "total_rework_ok_qty": 530, "inspection_chipping_pending": 0, "visual_defects_straight_pass_pct": 56, "non_conformity_count": 0},
          "B": {"checked_qty": 1200, "ok_qty": 701, "rej_qty": null, "rew_qty": 499, "total_rework_ok_qty": 499, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 58, "non_conformity_count": null},
          "TOTAL": null
        }
      }
    },
    {
      "sno": 5,
      "part_name": "MAIN STAND",
      "part_no": "50500-AAE-3000",
      "model": "AAEE",
      "days": {
        "21": {
          "A": {"checked_qty": 527, "ok_qty": 363, "rej_qty": null, "rew_qty": 164, "total_rework_ok_qty": 164, "inspection_chipping_pending": 68, "visual_defects_straight_pass_pct": null, "non_conformity_count": null},
          "B": {"checked_qty": 572, "ok_qty": 277, "rej_qty": null, "rew_qty": 315, "total_rework_ok_qty": 315, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 48, "non_conformity_count": null},
          "TOTAL": null
        },
        "22": {
          "A": {"checked_qty": 610, "ok_qty": 408, "rej_qty": null, "rew_qty": 202, "total_rework_ok_qty": 202, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 66, "non_conformity_count": null},
          "B": {"checked_qty": 689, "ok_qty": 339, "rej_qty": null, "rew_qty": 350, "total_rework_ok_qty": 350, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 48, "non_conformity_count": null},
          "TOTAL": null
        },
        "23": {
          "A": {"checked_qty": 573, "ok_qty": 339, "rej_qty": null, "rew_qty": 234, "total_rework_ok_qty": 234, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 59, "non_conformity_count": null},
          "B": {"checked_qty": 551, "ok_qty": 283, "rej_qty": null, "rew_qty": 268, "total_rework_ok_qty": 268, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 51, "non_conformity_count": null},
          "TOTAL": null
        }
      }
    },
    {
      "sno": 6,
      "part_name": "MAIN STAND",
      "part_no": "50500-AACK-3000",
      "model": "ACK",
      "days": {
        "21": {
          "A": {"checked_qty": 953, "ok_qty": 623, "rej_qty": null, "rew_qty": 330, "total_rework_ok_qty": 330, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 65, "non_conformity_count": null},
          "B": {"checked_qty": 1100, "ok_qty": 638, "rej_qty": null, "rew_qty": 462, "total_rework_ok_qty": 462, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 58, "non_conformity_count": null},
          "TOTAL": null
        },
        "22": {
          "A": {"checked_qty": 1115, "ok_qty": 749, "rej_qty": null, "rew_qty": 366, "total_rework_ok_qty": 366, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 67, "non_conformity_count": null},
          "B": {"checked_qty": 1019, "ok_qty": 493, "rej_qty": null, "rew_qty": 525, "total_rework_ok_qty": 525, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 48, "non_conformity_count": null},
          "TOTAL": null
        },
        "23": {
          "A": {"checked_qty": 1008, "ok_qty": 617, "rej_qty": null, "rew_qty": 391, "total_rework_ok_qty": 391, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 61, "non_conformity_count": null},
          "B": {"checked_qty": 1161, "ok_qty": 579, "rej_qty": null, "rew_qty": 582, "total_rework_ok_qty": 582, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 49, "non_conformity_count": null},
          "TOTAL": null
        }
      }
    },
    {
      "sno": 7,
      "part_name": "PEDAL BRAKE",
      "part_no": "50500-AAE-0000",
      "model": "AAEN",
      "days": {
        "21": {
          "A": {"checked_qty": 1530, "ok_qty": 1272, "rej_qty": null, "rew_qty": 258, "total_rework_ok_qty": 258, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 83, "non_conformity_count": null},
          "B": {"checked_qty": 1950, "ok_qty": 1532, "rej_qty": null, "rew_qty": 418, "total_rework_ok_qty": 418, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 78, "non_conformity_count": null},
          "TOTAL": null
        },
        "22": {
          "A": {"checked_qty": 1460, "ok_qty": 1221, "rej_qty": null, "rew_qty": 239, "total_rework_ok_qty": 239, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 83, "non_conformity_count": null},
          "B": {"checked_qty": 1920, "ok_qty": 1565, "rej_qty": null, "rew_qty": 355, "total_rework_ok_qty": 355, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 81, "non_conformity_count": null},
          "TOTAL": null
        },
        "23": {
          "A": {"checked_qty": 1700, "ok_qty": 1413, "rej_qty": null, "rew_qty": 287, "total_rework_ok_qty": 287, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 83, "non_conformity_count": null},
          "B": {"checked_qty": 2200, "ok_qty": 1638, "rej_qty": null, "rew_qty": 562, "total_rework_ok_qty": 562, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 74, "non_conformity_count": null},
          "TOTAL": null
        }
      }
    }
  ],
  "signatures": {

  }
}
```

Reading notes:

- Same scan as `St . Pass data weld1.pdf`, stored sideways. Rotate 90° before OCR.
- Days 24–30 are empty and are not returned. Month is not on the sheet, so it is `null`.
- Each day is `{A, B, TOTAL}`, or the string `"NO PLAN"` when that is written across the day; a shift is `"NO PLAN"` when it is written across that shift only.
- Percent values are returned as numbers without `%`.
- Part 1 day 21 A: `82%` is written in the INSPECTION/CHIPPING PENDING row as well as the straight-pass row. It is returned as written.
- Part 1 day 23 B CHECKED QTY reads `2550` (OK 1968 + REW 582 = 2550).
- Part 4 day 21 TOTAL column holds values (1200 / 711 / 489 / 489 / 59%) that look like day 22 A written one column to the left. They are returned as written.
- Part 4 day 22 B OK QTY is unclear (`537`/`547`); `547` matches CHECKED − REW.
- Part 5 day 21 A: `68%` is written in the INSPECTION/CHIPPING PENDING row, and the straight-pass cell is empty.
- Part 6 day 22 TOTAL column has day-23 values written and struck out; they are left out.

---

## `St . Pass data weld2.pdf`

Response type: **`ST_PASS_MONITORING`**

```json
{
  "form_type": "ST_PASS_MONITORING",
  "header": {
    "company": "MEENAKSHI POLYMERS PVT. LTD. HARIDWAR",
    "title": "BS6 MODEL COMPONENT STRAIGHT PASS MONITORING SHEET",
    "month": null,
    "days_printed": [21, 22, 23, 24, 25, 26, 27, 28, 29, 30]
  },
  "parts": [
    {
      "sno": 8,
      "part_name": "PEDAL GEAR CHANGE",
      "part_no": "2470A-ACK-0000",
      "model": "ACKA",
      "days": {
        "21": {
          "A": null,
          "B": "NO PLAN",
          "TOTAL": null
        },
        "23": {
          "A": {"checked_qty": 1027, "ok_qty": 990, "rej_qty": null, "rew_qty": 61, "total_rework_ok_qty": 37, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 96, "non_conformity_count": null},
          "B": "NO PLAN",
          "TOTAL": null
        }
      }
    },
    {
      "sno": 9,
      "part_name": "PEDAL GEAR CHANGE",
      "part_no": "24701-AAE-3000",
      "model": "AAE",
      "days": {
        "21": {
          "A": {"checked_qty": 4295, "ok_qty": 4152, "rej_qty": null, "rew_qty": 225, "total_rework_ok_qty": 143, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 96, "non_conformity_count": null},
          "B": {"checked_qty": 1950, "ok_qty": 1815, "rej_qty": null, "rew_qty": 174, "total_rework_ok_qty": 135, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 93, "non_conformity_count": null},
          "TOTAL": null
        },
        "22": {
          "A": {"checked_qty": 2855, "ok_qty": 2743, "rej_qty": null, "rew_qty": 160, "total_rework_ok_qty": 112, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 96, "non_conformity_count": null},
          "B": {"checked_qty": 2360, "ok_qty": 2225, "rej_qty": null, "rew_qty": 160, "total_rework_ok_qty": 135, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 94, "non_conformity_count": null},
          "TOTAL": null
        },
        "23": {
          "A": {"checked_qty": 3385, "ok_qty": 3275, "rej_qty": null, "rew_qty": 160, "total_rework_ok_qty": 110, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 96, "non_conformity_count": null},
          "B": {"checked_qty": 2350, "ok_qty": 2201, "rej_qty": null, "rew_qty": 165, "total_rework_ok_qty": 149, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 93, "non_conformity_count": null},
          "TOTAL": null
        }
      }
    },
    {
      "sno": 10,
      "part_name": "STEP WOMEN",
      "part_no": "51104-AAH-0300",
      "model": null,
      "days": {
        "21": {
          "A": {"checked_qty": 1230, "ok_qty": 1120, "rej_qty": null, "rew_qty": 110, "total_rework_ok_qty": 110, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 91, "non_conformity_count": null},
          "B": "NO PLAN",
          "TOTAL": null
        },
        "22": {
          "A": {"checked_qty": 1200, "ok_qty": 880, "rej_qty": null, "rew_qty": 320, "total_rework_ok_qty": 320, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 73, "non_conformity_count": null},
          "B": "NO PLAN",
          "TOTAL": null
        },
        "23": "NO PLAN"
      }
    },
    {
      "sno": 11,
      "part_name": "STEP PILLION",
      "part_no": "51104-AAE-H000",
      "model": null,
      "days": {
        "21": {
          "A": {"checked_qty": 2950, "ok_qty": 2555, "rej_qty": null, "rew_qty": 400, "total_rework_ok_qty": 400, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 86, "non_conformity_count": null},
          "B": {"checked_qty": 2912, "ok_qty": 2582, "rej_qty": null, "rew_qty": 330, "total_rework_ok_qty": 330, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 88, "non_conformity_count": null},
          "TOTAL": null
        },
        "22": {
          "A": {"checked_qty": 2740, "ok_qty": 1920, "rej_qty": null, "rew_qty": 820, "total_rework_ok_qty": 820, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 70, "non_conformity_count": null},
          "B": {"checked_qty": 3261, "ok_qty": 2911, "rej_qty": null, "rew_qty": 300, "total_rework_ok_qty": 300, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 89, "non_conformity_count": null},
          "TOTAL": null
        },
        "23": {
          "A": {"checked_qty": 1800, "ok_qty": 1572, "rej_qty": null, "rew_qty": 223, "total_rework_ok_qty": 233, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 87, "non_conformity_count": null},
          "B": {"checked_qty": 3614, "ok_qty": 3264, "rej_qty": null, "rew_qty": 350, "total_rework_ok_qty": 350, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 90, "non_conformity_count": null},
          "TOTAL": null
        }
      }
    },
    {
      "sno": 12,
      "part_name": "SARI GUARD AAER",
      "part_no": "51103-AAE-H000",
      "model": "AAER",
      "days": {
        "21": {
          "A": null,
          "B": {"checked_qty": 2620, "ok_qty": 2240, "rej_qty": null, "rew_qty": 371, "total_rework_ok_qty": 371, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 85, "non_conformity_count": null},
          "TOTAL": null
        },
        "22": {
          "A": null,
          "B": {"checked_qty": 2570, "ok_qty": 2176, "rej_qty": null, "rew_qty": 394, "total_rework_ok_qty": 394, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 84, "non_conformity_count": null},
          "TOTAL": null
        },
        "23": {
          "A": {"checked_qty": 2520, "ok_qty": 2126, "rej_qty": null, "rew_qty": 394, "total_rework_ok_qty": 394, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 84, "non_conformity_count": null},
          "B": null,
          "TOTAL": null
        }
      }
    },
    {
      "sno": 13,
      "part_name": "SARI GUARD AAH",
      "part_no": "51103-AAH-1100",
      "model": "AAH",
      "days": {
        "21": {
          "A": {"checked_qty": 966, "ok_qty": 726, "rej_qty": null, "rew_qty": 270, "total_rework_ok_qty": 270, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 75, "non_conformity_count": null},
          "B": "NO PLAN",
          "TOTAL": null
        },
        "22": {
          "A": {"checked_qty": 983, "ok_qty": 643, "rej_qty": null, "rew_qty": 340, "total_rework_ok_qty": 340, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 65, "non_conformity_count": null},
          "B": "NO PLAN",
          "TOTAL": null
        },
        "23": "NO PLAN"
      }
    },
    {
      "sno": 14,
      "part_name": "CROSS PIPE",
      "part_no": "77215-AAD-3000",
      "model": "AAD",
      "days": {
        "21": "NO PLAN",
        "22": {
          "A": {"checked_qty": 2250, "ok_qty": 1649, "rej_qty": null, "rew_qty": 601, "total_rework_ok_qty": 563, "inspection_chipping_pending": null, "visual_defects_straight_pass_pct": 73, "non_conformity_count": null},
          "B": null,
          "TOTAL": null
        },
        "23": "NO PLAN"
      }
    }
  ],
  "signatures": {

  }
}
```

Reading notes:

- Parts 10 and 11 have no MODEL printed, so it is `null`.
- Part 8 day 21 A CHECKED QTY `429` is struck out; nothing else is filled for that day except NO PLAN on B.
- Part 12 day 23 B is a struck-out copy of day 23 A, so it is `null`.
- Part 11 day 22 A REW QTY / TOTAL REWORK OK QTY read `820` (OK 1920 + 820 = CHECKED 2740).
