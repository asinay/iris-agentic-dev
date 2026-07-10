---
author: asinay
description: Use when writing, debugging, or optimizing MDX queries against InterSystems
  IRIS BI cubes — engine choice (MDX vs SQL), hierarchy path syntax, NON EMPTY,
  %FILTER, %OR, calculated members, time series, and common traps. Load explicitly
  for MDX/BI work; do NOT load globally for general ObjectScript tasks.
iris_version: '>=2024.1'
name: asinay/iris-mdx
state: draft
tags:
- iris
- mdx
- deepsee
- analytics
- bi
- quirks
trigger: Use for asinay/iris-mdx
---

# IRIS MDX — Quirks and Patterns for IRIS BI Cube Queries

## HARD GATE

Before writing any IRIS MDX, check these. Every one has caused silent wrong results.

- [ ] **Discover cube structure first** — call `iris_info` with `what=sa_schema` to get real dimension spec paths for the IRIS BI cube before writing a single line of MDX
- [ ] **Use exact spec paths** — wrong hierarchy name returns null, no error: `[Outlet].[H1].[Region]` not `[Region].[Region]`
- [ ] **NON EMPTY on every axis** — without it, all empty members are returned regardless of filter
- [ ] **Side-by-side comparison** — put both members in a set `{.&[2023], .&[2024]}` on the axis; never two `%FILTER` on the same level
- [ ] **Side-by-side comparison multi-filter** — two `%FILTER` on the same level AND together → null data (not an error); use a set on the axis instead
- [ ] **Member keys are not always captions** — integer-keyed dimensions need `&[2]` not `&[Online]`; discover with `CURRENTMEMBER.PROPERTIES("KEY")`
- [ ] **`%MDX()` inside WITH MEMBER only** — placing it directly on an axis returns empty, no error
- [ ] **`%COUNT` is the correct measure name** — never invent names like `Patient Count` or `Transaction Count`

---

## 1. MDX vs SQL — When to Use Each

| Use MDX when… | Use SQL when… |
|---|---|
| You need aggregated totals, averages, counts | You need individual rows / raw records |
| The data is modelled in a cube | The data is only in source tables |
| You need time-series trends | You need JOINs not represented in a cube |
| You need cross-dimensional slicing | You need to write data (INSERT/UPDATE) |
| Performance matters — MDX is 3–15× faster than SQL for aggregations | |

**Always check `iris_info` with `what=sa_schema` first** — if an IRIS BI cube exists for the data, prefer MDX for aggregation questions.

---

## 2. Hierarchy Path Syntax — The #1 Source of Wrong Results

MDX dimension references must use the **exact spec path** from the cube definition. A wrong path fails silently with nulls — no error.

```mdx
-- CORRECT: full spec path
SELECT {MEASURES.[Amount Sold]} ON 0,
       NON EMPTY [Outlet].[H1].[Region].MEMBERS ON 1
FROM HoleFoods

-- WRONG: invented path — returns null, no error
SELECT {MEASURES.[Amount Sold]} ON 0,
       NON EMPTY [Region].[Region].MEMBERS ON 1
FROM HoleFoods
```

**How to get spec paths:**
```
iris_info(what=sa_schema, name=HoleFoods)
→ returns all dimension specs for the IRIS BI cube:
  [Outlet].[H1].[Region], [DateOfSale].[Actual].[YearSold], …
```

**Shorthand is allowed** (but use full paths for clarity):
```mdx
[GenD].[H1].[Gender].Female   -- full
[GenD].[H1].Female            -- omit level name
[GenD].Female                 -- omit hierarchy and level
GenD.Female                   -- omit brackets when name is alphanumeric
```

---

## 3. NON EMPTY — Always Suppress Empty Members

Without `NON EMPTY`, every member in the level is returned — including those with no data in the current filter context.

```mdx
-- WRONG: returns all 12 months even when filtered to a single year with sparse data
SELECT {MEASURES.[Amount Sold]} ON 0,
       [DateOfSale].[Actual].[MonthSold].MEMBERS ON 1
FROM HoleFoods

-- CORRECT: only months that have data
SELECT {MEASURES.[Amount Sold]} ON 0,
       NON EMPTY [DateOfSale].[Actual].[MonthSold].MEMBERS ON 1
FROM HoleFoods
```

**Rule:** put `NON EMPTY` before every `.MEMBERS` axis expression.

---

## 4. %FILTER vs WHERE — Parent-Level Filter with Child-Level Axis

`WHERE` and `%FILTER` produce identical MDXText internally in IRIS — the engine rewrites both to the same `WHERE` clause. Both work correctly when filtering a parent level while showing child members on an axis.

```mdx
-- Both of these produce identical results in IRIS:
SELECT {MEASURES.[Amount Sold]} ON 0,
       NON EMPTY [DateOfSale].[Actual].[MonthSold].MEMBERS ON 1
FROM HoleFoods
WHERE [DateOfSale].[Actual].[YearSold].&[2024]

SELECT {MEASURES.[Amount Sold]} ON 0,
       NON EMPTY [DateOfSale].[Actual].[MonthSold].MEMBERS ON 1
FROM HoleFoods
%FILTER [DateOfSale].[Actual].[YearSold].&[2024]
```

**Prefer `%FILTER`** for programmatic query building — easier to append conditions one at a time, and composes cleanly with `%OR`. Use `WHERE` for simple one-off filters in ad-hoc queries.

Multiple `%FILTER` clauses chain as AND:
```mdx
-- Revenue for Asia region, Snack category only
SELECT MEASURES.[Amount Sold] ON 0,
       NON EMPTY [Product].[P1].[Product Category].MEMBERS ON 1
FROM HoleFoods
%FILTER [Outlet].[H1].[Region].&[Asia]
%FILTER [Product].[P1].[Product Category].&[Snack]
```

---

## 5. Side-by-Side Comparison — Never Double %FILTER on Same Level

Two `%FILTER` clauses on the same dimension AND together. Year=2023 AND Year=2024 = always empty.

```mdx
-- WRONG: AND logic → returns a row with empty member and null value
SELECT {MEASURES.[Amount Sold]} ON 0,
       NON EMPTY [Product].[P1].[Product Category].MEMBERS ON 1
FROM HoleFoods
%FILTER [DateOfSale].[Actual].[YearSold].&[2023]
%FILTER [DateOfSale].[Actual].[YearSold].&[2024]

-- CORRECT: both years as a set on the axis
SELECT {MEASURES.[Amount Sold]} ON 0,
       NON EMPTY {[DateOfSale].[Actual].[YearSold].&[2023],
                  [DateOfSale].[Actual].[YearSold].&[2024]} ON 1
FROM HoleFoods
```

**Rule:** for any "compare A vs B" question, put both members in a set `{member1, member2}` on an axis — never filter to each one separately.

---

## 6. %OR — OR Filter Without Double-Counting

`WHERE {a, b}` can double-count in the MDX spec. IRIS silently rewrites it to `%OR` internally, but use `%OR` explicitly to make intent clear and portable.

```mdx
-- OR two members of the same dimension (e.g. two diagnoses)
SELECT MEASURES.[%COUNT] ON 0
FROM Patients
%FILTER %OR({[DiagD].[H1].[Diagnoses].&[asthma],
             [DiagD].[H1].[Diagnoses].&[diabetes]})
-- Returns 119 (78 asthma + 46 diabetes - 5 with both) — correct union

-- OR across different dimensions
SELECT MEASURES.[%COUNT] ON 0
FROM Patients
WHERE %OR({[GenD].[H1].[Gender].&[Female],
           [ColorD].[H1].[Favorite Color].&[Orange]})

-- AND of ORs — chain %FILTER with %OR inside each
SELECT MEASURES.[%COUNT] ON 0
FROM Patients
%FILTER %OR({[ColorD].[H1].[Favorite Color].&[Orange],
             [ColorD].[H1].[Favorite Color].&[Purple]})
%FILTER [GenD].[H1].[Gender].&[Female]
-- Result: Female AND (Orange OR Purple)
```

---

## 7. Member Key Syntax — Captions vs Integer Keys

Member key syntax: `[Dim].[Hier].[Level].&[key]`

**The key is not always the display name.** String-keyed dimensions use the caption as the key. Integer-keyed dimensions (Channel, Discount Band) use a numeric ID.

```mdx
-- String-keyed: caption = key (works)
[Outlet].[H1].[Region].&[Asia]
[DateOfSale].[Actual].[YearSold].&[2024]

-- Integer-keyed: must use numeric ID (not display name)
[Channel].[H1].[Channel Name].&[2]       -- CORRECT for "Online"
[Channel].[H1].[Channel Name].&[Online]  -- WRONG — returns null, no error
```

**Discover actual keys:**
```mdx
SELECT [Channel].[H1].CURRENTMEMBER.PROPERTIES("KEY") ON 0,
       [Channel].[H1].[Channel Name].MEMBERS ON 1
FROM HoleFoods
-- Reveals: Wholesale=1, Online=2, etc.
```

---

## 8. % Prefix — Always Prefer IRIS Extensions Over Standard Equivalents

Any MDX keyword starting with `%` is an InterSystems extension. **`%`-prefixed features are faster** — implemented at engine level with optimised index access.

| Extension | Use instead of | Why |
|---|---|---|
| `%OR({a, b})` | `{a, b}` in WHERE | Explicit union semantics, no double-counting, engine-optimised |
| `%NOT` | `EXCEPT` | Single-member exclusion, no intermediate set |
| `%FILTER` | `WHERE` | Composable, chains as AND, applies after axis evaluation |
| `%COUNT` | `COUNT(*)` in SQL | Native fact count measure, always present in every cube |
| `%TIMERANGE(s, e)` | `{start:end}` | Open-ended ranges, INCLUSIVE/EXCLUSIVE control |
| `%MDX("SELECT FROM cube")` | No equivalent | Scalar subquery immune to cell context — use for percent-of-total |
| `%LAST(set, measure)` | Manual iteration | Last non-null across time — correct for snapshot/balance measures |
| `%CELL(col, row)` | No equivalent | Positional cell reference for running totals |
| `%LABEL(member, caption)` | No equivalent | Override auto-generated column header |

---

## 9. Calculated Members — WITH MEMBER Patterns

### Percent-of-total with %MDX()

`%MDX()` returns a scalar from a separate query, immune to the current cell context. **Must be inside `WITH MEMBER`** — not directly on an axis.

```mdx
-- CORRECT: %MDX inside WITH MEMBER
WITH MEMBER MEASURES.[PctOfTotal] AS
    '100 * MEASURES.[Amount Sold] / %MDX("SELECT MEASURES.[Amount Sold] ON 0 FROM HoleFoods")'
SELECT MEASURES.[PctOfTotal] ON 0,
       NON EMPTY [Outlet].[H1].[Region].MEMBERS ON 1
FROM HoleFoods
-- Denominator is always total revenue regardless of which row is evaluated

-- WRONG: %MDX directly on axis → empty result, no error
SELECT %MDX("SELECT MEASURES.[Amount Sold] ON 0 FROM HoleFoods") ON 0
FROM HoleFoods
```

### Period-over-period with PrevMember

```mdx
WITH MEMBER MEASURES.[PrevUnits] AS
    '([DateOfSale].[Actual].CurrentMember.PrevMember, MEASURES.[Units Sold])'
SELECT {MEASURES.[Units Sold], MEASURES.[PrevUnits]} ON 0,
       NON EMPTY [DateOfSale].[Actual].[MonthSold].MEMBERS ON 1
FROM HoleFoods
```

**Known bug:** the auto-generated column header for a PrevMember measure shows the dimension name (`DateOfSale`) instead of the measure name (`PrevUnits`). Fix with `%LABEL`:
```mdx
SELECT {MEASURES.[Units Sold],
        %LABEL(MEASURES.[PrevUnits], "Units (Prev Month)", "")} ON 0, ...
```

### YTD / rolling window

```mdx
-- Year-to-date through today
WITH MEMBER CalcD.[YTD] AS
    '%OR(PERIODSTODATE([DateOfSale].[Actual].[YearSold],
                       [DateOfSale].[Actual].[DaySold].[NOW]))'
SELECT MEASURES.[Amount Sold] ON 0,
       CalcD.[YTD] ON 1
FROM HoleFoods

-- Last 90 days
WITH MEMBER CalcD.[Last90] AS
    '%OR([DateOfSale].[Actual].[DaySold].[NOW-90]:[NOW])'
```

### Distinct member count

```mdx
WITH MEMBER MEASURES.[ActiveDoctors] AS
    'COUNT([DocD].[H1].[Doctor].MEMBERS, EXCLUDEEMPTY)'
SELECT MEASURES.[ActiveDoctors] ON 0,
       NON EMPTY [GenD].[H1].[Gender].MEMBERS ON 1
FROM Patients
```

---

## 10. FILTER, ORDER, and Aggregation Functions

### FILTER — aggregate HAVING, not row-level WHERE

`FILTER(set, condition)` evaluates the condition against each member's **aggregated** value — it is equivalent to SQL `HAVING`, not `WHERE`:

```mdx
-- Returns only regions where total revenue > 2000 (aggregate test)
NON EMPTY FILTER([Outlet].[H1].[Region].MEMBERS, MEASURES.[Amount Sold] > 2000) ON 1
```

### ORDER — preserve vs break hierarchy

```mdx
-- ASC/DESC: sort within parent groups (hierarchy preserved)
ORDER([HomeD].[H1].MEMBERS, MEASURES.[Avg Age], DESC)

-- BASC/BDESC: sort globally across all members (hierarchy broken — flat ranked list)
ORDER([HomeD].[H1].MEMBERS, MEASURES.[Avg Age], BDESC)
```

Use `BDESC`/`BASC` for ranked lists. Use `DESC`/`ASC` when parent-child grouping must be preserved.

### Summary functions append a summary row/column

```mdx
SELECT MEASURES.[%COUNT] ON 0,
       {[DiagD].[H1].[Diagnoses].MEMBERS,
        MAX([DiagD].[H1].[Diagnoses].MEMBERS, MEASURES.[%COUNT])} ON 1
FROM Patients
-- All diagnosis rows + a MAX row at the bottom
```

---

## 11. Time Navigation

### NOW member — relative offsets

```mdx
[DateOfSale].[Actual].[DaySold].[NOW]       -- today
[DateOfSale].[Actual].[DaySold].[NOW-30]    -- 30 days ago
[DateOfSale].[Actual].[YearSold].[NOW-1]    -- last year (timeline-based level)
[DateOfSale].[Actual].[DaySold].[NOW-4y3m2d] -- compound offset: 4y 3m 2d ago
```

**Only works on timeline-based levels** (e.g. `YearSold`, `MonthSold`, `DaySold`). Does not work on date-part levels (e.g. `Quarter`, `Month`).

### Timeline-based vs date-part levels

| Type | Example members | PREVMEMBER crosses boundary? |
|---|---|---|
| Timeline-based | `Q1 2024`, `Jan 2024` | Yes — Q1 2024 PREVMEMBER → Q4 2023 |
| Date-part-based | `Q1`, `January` | No — Q1 PREVMEMBER → null |

Use **timeline-based** levels for period-over-period comparisons (`PREVMEMBER`, `%TIMERANGE`, `NOW±n`).
Use **date-part** levels for grouping across all years (e.g., "all January months combined").

---

## 12. Common Silent Failures

| Situation | Result | How to catch |
|---|---|---|
| Wrong hierarchy path | Row with empty member and null value, no error | Verify with `iris_info what=sa_schema` |
| Typo in dimension name | Dimension silently ignored | Check known totals against a control query |
| Nonexistent member caption | Null (`*`), no error | Use `CURRENTMEMBER.PROPERTIES("KEY")` to discover real keys |
| `%MDX()` directly on axis | Empty result, no error | Always wrap in `WITH MEMBER` |
| Two `%FILTER` on same level | Empty member row with null value, no error | Use set `{m1, m2}` on axis for OR/comparison |
| Integer-keyed dimension with caption key | Null, no error | Use numeric key `&[2]` not `&[Online]` |
| Nonexistent measure | `ERROR #5001: Measure not found` | Check cube definition for exact name |
| Nonexistent cube | `ERROR #5001: Cannot find Subject Area` | Check `iris_info what=sa_schema` |

---

## 13. IRIS BI Cube Discovery Workflow

Always run discovery before writing MDX against an unfamiliar cube:

```
Step 1 — List available IRIS BI cubes:
    iris_info(what=sa_schema, name=<namespace>)

Step 2 — Get cube structure (measures + dimension spec paths):
    iris_info(what=sa_schema, name=<CubeName>)
    → copy exact dimension spec paths (e.g. [Outlet].[H1].[Region])
    → note measure names (e.g. Amount Sold, Units Sold, %COUNT)

Step 3 — Discover member keys for integer-keyed dimensions:
    SELECT [Dim].[Hier].CURRENTMEMBER.PROPERTIES("KEY") ON 0,
           [Dim].[Hier].[Level].MEMBERS ON 1
    FROM Cube

Step 4 — Write the MDX using exact spec paths and verified keys
```

---

## EXAMPLE: Monthly Revenue for a Specific Year

```mdx
-- Full correct pattern: %FILTER for year, NON EMPTY for months, exact spec paths
SELECT {MEASURES.[Amount Sold]} ON 0,
       NON EMPTY [DateOfSale].[Actual].[MonthSold].MEMBERS ON 1
FROM HoleFoods
%FILTER [DateOfSale].[Actual].[YearSold].&[2024]
```

## EXAMPLE: Year-over-Year Comparison

```mdx
-- Both years as a set on the axis — not two %FILTER clauses
SELECT {MEASURES.[Amount Sold]} ON 0,
       NON EMPTY {[DateOfSale].[Actual].[YearSold].&[2023],
                  [DateOfSale].[Actual].[YearSold].&[2024]} ON 1
FROM HoleFoods
```

## EXAMPLE: Revenue by Region with Percentage of Total

```mdx
WITH MEMBER MEASURES.[Pct] AS
    '100 * MEASURES.[Amount Sold] / %MDX("SELECT MEASURES.[Amount Sold] ON 0 FROM HoleFoods")'
SELECT {MEASURES.[Amount Sold], MEASURES.[Pct]} ON 0,
       NON EMPTY [Outlet].[H1].[Region].MEMBERS ON 1
FROM HoleFoods
```

## EXAMPLE: Top 5 Products by Revenue

```mdx
SELECT {MEASURES.[Amount Sold]} ON 0,
       TOPCOUNT([Product].[P1].[Product Name].MEMBERS, 5, MEASURES.[Amount Sold]) ON 1
FROM HoleFoods
```
