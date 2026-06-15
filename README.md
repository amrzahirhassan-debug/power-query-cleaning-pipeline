# RAMP & DCR — Data Cleaning Methodologies
**Developed by Amr Zahir**

Two complementary methodologies for cleaning, standardizing, and documenting data — one for Power Query pipelines, one for direct data correction tasks.

---

## RAMP — Reference, Audit, Map, Pipeline

### What it is
A structured Power Query technique for cleaning categorical columns. RAMP replaces the standard Conditional Column approach with a maintainable, scalable, audit-safe pipeline.

### Why it exists
The naive approach (Column Profile → Conditional Column) has four critical weaknesses:
- **Manual** — every new variant requires opening Power Query and editing code
- **Case-sensitive** — "Full-time" and "full time" are treated as different values without pre-normalization
- **Silent nulls** — unmapped values fall through with no warning
- **Does not scale** — 50+ variants or evolving datasets make the conditional column unmaintainable

RAMP fixes all four.

### The Four Pillars

| Letter | Pillar | What it means |
|--------|--------|---------------|
| R | Reference | Raw data stays untouched; all transforms happen in a referenced query |
| A | Audit | Validate structure and data types before any transformation |
| M | Map | A living external mapping table drives standardization — no hardcoded logic |
| P | Pipeline | Steps run in a fixed, repeatable order; refresh reruns everything automatically |

### The 8-Step Pipeline

**Step 1 — Load data**
Open Excel → `Data → Get Data → From File`. The original file is never touched.

**Step 2a — Validate structure and data types**
Check data types (Text, Number, Date) and fix before any transformation.

**Step 2b — Reference the raw query**
Right-click the raw query → **Reference**. All cleaning happens in this referenced query only. The raw query stays permanently untouched.

> The raw query's Applied Steps should only ever contain: `Source`, `Promoted Headers`, `Changed Type`. Never add any other transforms to it.

**Step 3 — Normalize the column**
`Add Column → Custom Column`:
```
Text.Proper(Text.Clean(Text.Trim([your_column_name])))
```
- **Text.Trim** — removes leading/trailing spaces
- **Text.Clean** — removes invisible non-printable characters
- **Text.Proper** — capitalizes the first letter of each word

This dramatically reduces the number of distinct values to map.

> Always manually set the new column's data type to **Text** — Power Query defaults custom columns to "Any".

**Step 4 — Column Profile**
`View → Column Distribution / Quality / Profile`. Change from `top 1000 rows` → `entire dataset`. Extract distinct values as a separate Connection Only query.

**Step 5 — Build the mapping table**
Copy distinct values from the Power Query preview pane (static paste) into a new Excel sheet. Add a second column for the standardized value. Fill in one standard value per row.

> Never load a live query output directly into the mapping table's first column — a refresh that inserts new values mid-list will shift column A without shifting manually typed column B, causing misalignment.

**Step 6 — Load and merge the mapping table**
Load the mapping sheet as a Power Query table (Connection Only). In the cleaning query: `Home → Merge Queries → Left Outer Join` on the normalized column. Expand only the standard value column.

**Step 7 — Validate**
- Row count must match raw data
- Null check: any null in standard_value = unmapped variant → add to mapping table → refresh
- Distinct values scan: filter the standard_value column and confirm only expected categories appear

**Step 8 — Close and Load**

| Query | Load as |
|-------|---------|
| Cleaning query | Table → `clean_data` sheet |
| Raw query | Connection Only |
| Distinct values query | Connection Only |
| Mapping table query | Connection Only |

### Optional: Audit Log Query
A parallel query that reads row counts, null counts, and distinct value counts from the pipeline automatically on every refresh — giving a pass/fail summary without re-scanning data.

### Handling New Data

| Scenario | Action |
|----------|--------|
| New rows, same variants | Just refresh |
| New rows with unknown variant | Appears as null → add one row to mapping table → refresh |

### Query Architecture
```
Query 1 — Raw data (untouched, Connection Only)
      ↓ referenced by
Query 2 — Cleaning query (all transforms live here)
      ↓ referenced by
Query 3 — Distinct values (feeds mapping table build)
            ↓ feeds (via Mapping query) back into
Query 2 — via Merge step
```

---

## DCR — Data Change Report

### What it is
A documentation standard for any data cleaning task performed directly via code (e.g. pandas). DCR produces a markdown report alongside the fix so the work is fully traceable without re-deriving it.

### When to use RAMP vs DCR

| Scenario | Use |
|----------|-----|
| User cleans data themselves in Excel/Power Query | RAMP |
| Claude directly edits/fixes a file via code | DCR |
| User asks "what did you change and why" | DCR |
| User asks about mapping tables, Merge, Power Query steps | RAMP |

### Report Structure

```markdown
# Cleaning Report — [filename] — [date]

## Task
[One or two sentences: what was requested]

## Column(s) affected
[Column name(s), total row count, how many rows were examined]

## Changes made

| Row | Column | Before | After | Reason |
|-----|--------|--------|-------|--------|
| 14  | Full_Name | maikil | Michael | Typo correction |

## Summary
- X rows changed
- Y rows unchanged
- Z rows flagged for review

## Unresolved / flagged items
[Anything ambiguous — left as-is with a note on why]
```

### Core Rules

1. **One row per change** — only log rows where a value actually changed
2. **Always state the reason** — a before/after pair without a reason is not useful for an audit
3. **Flag uncertainty** — never silently guess on ambiguous cases; list them under "Unresolved / flagged items"
4. **Summarize before detailing** — the Summary section lets the reader understand scope in one glance
5. **Save as .md** — lightweight, cheap to generate, cheap to re-read

### Example

```markdown
# Cleaning Report — employee_names.csv — 2026-06-13

## Task
Fix obvious typos in the Full_Name column.

## Column(s) affected
Full_Name — 102 rows total, 5 rows changed

## Changes made

| Row | Column | Before | After | Reason |
|-----|--------|--------|-------|--------|
| 14  | Full_Name | maikil ross | Michael Ross | Typo correction |
| 27  | Full_Name | jhon smith | John Smith | Typo correction + casing |
| 45  | Full_Name | KATE brown | Kate Brown | Casing standardized |
| 60  | Full_Name | sara  ahmed | Sara Ahmed | Double space removed |
| 88  | Full_Name | Ron ald Vasquez | Ronald Vasquez | Space removed mid-word |

## Summary
- 5 rows changed
- 97 rows unchanged
- 1 row flagged for review

## Unresolved / flagged items
- Row 73: "Jon Davis" — could be a typo for "John Davis" or a legitimate name. Left unchanged. Confirm before correcting.
```

---

## How They Work Together

RAMP and DCR cover complementary scenarios — RAMP for self-serve Power Query pipelines, DCR for AI-assisted or script-based corrections. Together they ensure that any data cleaning task, regardless of who performs it or which tool is used, produces a consistent, auditable, and traceable outcome.

---

*Methodologies developed by Amr Zahir*
