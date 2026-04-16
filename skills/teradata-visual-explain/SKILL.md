---
name: teradata-explain-analyzer
description: Analyze Teradata SQL EXPLAIN output to visualize query execution plans, identify performance bottlenecks, and provide actionable optimization recommendations. Outputs an interactive HTML report (default) or a Markdown report with Mermaid flowchart (for agents and text-based workflows). Use proactively when users ask about query performance, slow queries, EXPLAIN analysis, query plan interpretation, or SQL optimization on Teradata.
---

# Teradata EXPLAIN Analyzer

## Overview

Transform raw Teradata `EXPLAIN` output into visual query plans with prioritized, actionable optimization recommendations. Serve both beginners needing educational context and experts expecting deep optimizer reasoning.

**Output formats:**
- **HTML** (default) — Interactive flow diagram with hover tooltips, highlight controls, and Teradata branding. Best for human users in a browser.
- **Markdown** — Mermaid flowchart with structured tables. Best for AI agents, text-based workflows, or when the user requests `markdown`/`mermaid` output.

---

## Core Workflow

Execute this systematic 8-step workflow for every EXPLAIN analysis:

### Step 1: Input Processing

**If user provides SQL:**
- Assess query complexity to choose EXPLAIN type:
  - **STATIC EXPLAIN**: Simple queries, few joins, straightforward predicates
  - **DYNAMIC EXPLAIN**: Correlated subqueries, multiple nesting levels, partition elimination potential, complex predicates
- Execute via `teradata:base_readQuery`:
  ```sql
  EXPLAIN SELECT ...;
  -- or
  DYNAMIC EXPLAIN SELECT ...;
  ```
- On failure: explain the error clearly, offer to accept user-pasted EXPLAIN text

**If user provides EXPLAIN output text:**
- Validate it contains step patterns (look for "We do", "We lock", estimated times)
- Proceed directly to parsing

**Determine output format:**
- If the user explicitly requests `markdown`, `md`, `mermaid`, or `text` → **Markdown mode**
- If the user is an AI agent or tool consuming the output → **Markdown mode**
- Otherwise (including `html`, `visual`, `interactive`, `browser`, or no preference) → **HTML mode** (default)

---

### Step 2: Parse EXPLAIN Output

Extract every component systematically:

**Step structure to extract:**
- Step number and operation type (RETRIEVE, JOIN, SORT, AGG, LOCK)
- Lock types (READ/WRITE/EXCLUSIVE) and scope
- Table or spool names referenced
- Estimated row counts and time (relative cost units, not wall-clock)
- Confidence level: `no confidence` / `low confidence` / `high confidence` / `index join confidence`
- Access method: `all-rows scan` / `by way of the unique primary index` / `by way of index #n`
- AMP geography: `single-AMP` / `all-AMPs` / `few AMPs` / `(all_amps)` / `(group_amps)`
- Data movement: `redistributed by hash code` / `duplicated on all AMPs` / `locally built`
- Spool creation, numbering, and `(Last Use)` markers
- Parallel execution markers: `execute the following steps in parallel`
- Join methods: `merge join` / `hash join` / `product join` / `nested join` / `exclusion merge join` / `inclusion merge join`
- Partition phrases: `single partition of` / `n partitions of` / `all partitions of` / `m1 column partitions of`
- Conditional DML branches: `If no update in step X then INSERT`

**Key phrases glossary:** See [explain-glossary.md](references/explain-glossary.md) for the full EXPLAIN phrase → optimizer intent mapping table.

---

### Step 3: Collect Query Metadata

For each table referenced in the query, gather supporting context:

```
teradata:base_tableDDL       → Table structure, indexes, partitioning
teradata:base_columnDescription → Column types, constraints, nullable
teradata:dba_tableSpace      → Current table size (rows, bytes)
teradata:dba_tableUsageImpact → Usage patterns, which users/apps touch this table
teradata:base_tableAffinity  → Commonly joined tables (for index suggestions)
teradata:base_tablePreview   → Sample rows for data distribution context
```

**Statistics Assessment:**
- Identify which predicates, join keys, GROUP BY columns, and partition columns lack statistics
- Check for stale statistics where timestamps are available
- Correlate missing stats with Low/No confidence steps in the plan

**Index Inventory:**
- List UPI, NUPI, USI, NUSI, join indexes on referenced tables
- Note which indexes were used vs. available but ignored
- Identify index candidates for full-scan steps

---

### Step 4: Classify Issues by Severity

**🔴 CRITICAL — Must fix:**
- Steps with **No Confidence** on large tables
- **Product joins** on large tables (potential Cartesian explosion)
- **All-rows scans** on very large tables (>1M rows)
- Steps consuming **>15% of total estimated time**
- **Hot AMP signals**: `(group_amps)` materializing on very few AMPs
- Missing join conditions causing unintended Cartesian products
- Massive intermediate spool files (>1 GB estimated)

**🟡 WARNING — Should fix:**
- Steps with **Low Confidence**
- Steps consuming **5–15% of total estimated time**
- Large redistributions (>500 MB spool)
- Secondary index scans with large result sets
- Multiple sequential redistributions
- Missing partition elimination on a partitioned table
- Stale statistics identified from collection timestamps

**🟢 GOOD — Document and confirm:**
- **High Confidence** or **Index Join Confidence** estimates
- Unique primary index access (single-AMP, no spool)
- Single partition access on partitioned tables
- Efficient broadcast duplication (small table to all AMPs)
- Parallel execution of independent steps
- Spool reuse patterns (`Last Use` properly placed)
- Local aggregation (no redistribution needed)

**Severity classification logic:**
```
IF product join OR no confidence → CRITICAL (red)
IF step time > 15% of total    → CRITICAL (red)
IF low confidence               → WARNING (orange)
IF step time > 5% of total     → WARNING (orange)
IF spool > 500MB               → WARNING (orange)
IF high confidence AND fast     → GOOD (green)
ELSE                           → INFO (blue)
```

---

### Step 5: Generate Visual Report (PRIMARY DELIVERABLE)

**ALWAYS generate a visual flow diagram as the primary output.** This is not optional. Use the format determined in Step 1.

Both formats share the same vertical layout logic:

```
Lock Acquisition
      ↓
[Parallel Branch Start — if applicable]
  Step A  |  Step B  |  Step C   (side-by-side)
[Parallel Branch End]
      ↓
Sequential Operations (joins, sorts, aggregations)
      ↓
Final Output
```

**Operation type labels:** See [operation-labels.md](references/operation-labels.md) for the mapping from EXPLAIN text to standardized badge labels (e.g. `all-AMPs RETRIEVE` → `[ALL-AMP]`, `product join` → `[PROD-J]`).

---

#### Step 5A: HTML Mode (default)

Read these reference files when generating the HTML flow diagram:

- **Node, arrow & parallel templates:** [html-node-templates.md](references/html-node-templates.md) — HTML patterns for each flow node, arrows between steps, and parallel execution blocks. Includes data-attribute conventions and severity → CSS class mapping.
- **Interactive controls:** [interactive-controls.js](assets/interactive-controls.js) — JavaScript functions for highlight/reset buttons. Include verbatim in a `<script>` tag.

**Required interactive features:**

1. **Hover tooltips** — Explain each step: purpose, technical details, performance implications, fix suggestions
2. **"Highlight Critical Path"** button — Illuminate top 5 slowest steps
3. **"Highlight Product Joins"** button — Flash all `[PROD-J]` nodes
4. **"Highlight No Confidence"** button — Flash all No Confidence steps
5. **"Reset"** button — Return all nodes to normal state

**Required sections in HTML file:**

1. **Teradata-branded page header** (navy background, orange accent, base64 logo)
2. **Summary metrics bar** — 4–6 cards: total time, step count, critical issues, confidence distribution, result rows
3. **Legend** — Color meanings, badge types, operation icons
4. **Interactive control buttons**
5. **Flow diagram container** — All steps as nodes with arrows
6. **Key Insights panel** — Critical path breakdown, data growth table, top 3 optimization actions

**Teradata Brand Assets** (mandatory — include verbatim):

- **CSS:** Read [teradata-brand.css](assets/teradata-brand.css) and include the full content inside a `<style>` tag. You may extend it with additional styles for layout (summary cards, insights panels, etc.) but never modify the base brand rules.
- **Logo:** Read [teradata-logo.html](assets/teradata-logo.html) and include the `<img>` tag inside the `.td-page-header` div. Never recreate or modify the base64 logo.

**Delivery:** Save as `[query_name]_flow_diagram.html` in the current working directory and open in the browser.

---

#### Step 5B: Markdown Mode

Read the full template and Mermaid conventions from [markdown-report-template.md](references/markdown-report-template.md).

Generate the report **inline in the conversation response** (no external file). The report contains:

1. **Summary table** — Total time, step count, critical issues, confidence distribution, result rows
2. **Mermaid flowchart** — `flowchart TD` with severity-styled nodes, labeled edges (spool names/sizes), and `subgraph` blocks for parallel execution
3. **Step details table** — One row per step with operation, table/spool, rows, size, time (%), confidence, severity
4. **Top bottlenecks** — Ranked list of costliest steps
5. **Optimization playbook** — Same recommendations as HTML mode, formatted as markdown
6. **Key insights** — Data flow summary, spool management, parallel execution notes

**Mermaid essentials (see template for full conventions):**
- Node IDs: `S1`, `S2_1`, `S3`, etc.
- Node labels: `[BADGE] Step N: description<br>rows | size | time<br>confidence`
- Style classes: `critical`, `warning`, `good`, `info` with matching `classDef`
- Parallel blocks: `subgraph` with `direction LR`
- Final output: stadium-shaped node `(["📤 OUTPUT: N rows → user"])`

**Delivery:** Render the entire report inline as markdown in the conversation response. No file is created.

---

### Step 6: Build Optimization Playbook

Generate prioritized, specific, ready-to-execute recommendations:

**Format per recommendation:**
```
Priority: 🔴 HIGH / 🟡 MEDIUM / 🟢 LOW
Issue: [Exact EXPLAIN phrase that triggered this]
Root Cause: [Why the optimizer chose this path]
Action:
  COLLECT STATISTICS ON db.table COLUMN col;
  -- or --
  CREATE INDEX idx_name (col) ON db.table;
  -- or --
  [Rewritten SQL]
Expected Impact: [Quantified or qualitative improvement]
Effort: [Low/Medium/High]
Risk: [None/Low/Moderate/High with explanation]
```

**Standard recommendation types:**

1. **Statistics Collection** (most common, lowest risk)
   ```sql
   COLLECT STATISTICS ON {db}.{table} COLUMN {predicate_col};
   COLLECT STATISTICS ON {db}.{table} COLUMN ({col1}, {col2}); -- multicolumn for correlated
   COLLECT STATISTICS ON {db}.{table} INDEX {index_name};
   ```

2. **Index Creation**
   ```sql
   CREATE INDEX ({filter_col}) ON {db}.{table};  -- NUSI for frequent filter
   ```
   Include maintenance cost warning for DML-heavy tables.

3. **Query Rewrite** — Make predicates sargable:
   - Remove functions on indexed columns: `CAST(col AS DATE)` → store as DATE
   - De-OR predicates: `col=1 OR col=2` → `col IN (1,2)`
   - Push filters before joins
   - Replace NOT IN with NOT EXISTS for nullable columns

4. **Join Geography Optimization**
   - If both large tables redistributed: consider one as staging pre-hashed on join key
   - If small table redistributed: consider `duplicated on all AMPs` approach
   - PI redesign when join pattern dominates workload

5. **Partition Strategy**
   - Add RPPI on date/range columns for frequent range predicates
   - Adjust CP design for column-selective access patterns

---

### Step 7: Organize Output by Severity

**If critical issues found → lead with crisis:**
```
🚨 CRITICAL ISSUES SUMMARY
    ↓
Quick Metrics Bar
    ↓
Flow Diagram (present_files)
    ↓
Step-by-step table
    ↓
Optimization Playbook (HIGH priority first)
    ↓
Metadata Summary
```

**If warnings only → lead with overview:**
```
Performance Overview
    ↓
Flow Diagram (present_files)
    ↓
Optimization Opportunities
    ↓
Detailed Step Analysis
    ↓
Statistics & Index Status
```

**If query is optimal → validate and educate:**
```
✅ PERFORMANCE VALIDATION
    ↓
Flow Diagram (educational)
    ↓
Step Analysis (with explanations)
    ↓
Best Practices Confirmation
```

**If comparative analysis → lead with winner:**
```
Comparison Summary (A vs B, winner, % delta)
    ↓
Side-by-side Flow Diagrams
    ↓
Key Differences Explained
    ↓
Recommendation with reasoning
```

---

### Step 8: Adapt Explanation Depth

**Auto-detect expertise from:**
- Terminology used in the request
- Query complexity and structure
- Questions asked ("what is a merge join?" vs. "check PRPD eligibility")

**Beginner** — Define all terms, use analogies, explain optimizer reasoning, educational tone
**Intermediate** — Practical focus, assume basic concepts known, quick wins emphasis
**Expert** — Technical depth, edge cases, PRPD/IPE/dynamic partition elimination, performance math

---

## Advanced Analysis Patterns

### Skew & PRPD Detection
Watch for:
- `(group_amps)` with far fewer AMPs than total system AMPs
- Large spools produced by small redistributed inputs
- Join keys with known hot values (e.g., NULL, default codes)

**PRPD (Partial Redistribution/Partial Duplication) opportunity:**
- Applicable when join key has severe skew on a subset of values
- Requires statistics to reveal skewed values to the optimizer
- Recommend: Collect statistics on join column + verify with `EXPLAIN` comparing skewed vs. balanced values

### Static vs. Dynamic EXPLAIN
- **STATIC** (default): Plan generated at parse time from statistics
- **DYNAMIC**: IPE-based plan that may execute small fragments to feed better cardinality back to planning
- Use `DYNAMIC EXPLAIN` for: correlated subqueries, multi-level nesting, queries where intermediate sizes dramatically change the plan
- Note `:*` in Dynamic EXPLAIN output = security-masked intermediate values
- If Dynamic returns a static plan with explanation, note why IPE threshold wasn't met

### Column-Partitioned Table Nuance
- Step count of `N+1` column partitions means N user columns + the delete column partition (always present)
- Phrases `using CP merge Spool` vs. `using rowID spool` indicate different CP access strategies
- CP merge spool incurs additional overhead when contexts are insufficient

### Conditional DML (MERGE/UPSERT)
- `EXPLAIN` shows: `If no update in step X, then INSERT ...`
- Parallel sub-steps for join index maintenance
- `(Last Use)` markers indicate spool freed — verify correct placement

---

## Error Handling

**SQL Syntax Error:**
```
❌ SQL Syntax Error at line N, position P
Issue: [specific problem]
Suggested fix: [corrected SQL snippet]
Offer to retry with correction.
```

**Permission Denied:**
```
⚠️ Permission Denied — insufficient SELECT on [table]
Contact DBA to grant access.
Offer: paste EXPLAIN output text instead.
```

**Table/Object Not Found:**
```
❌ Object not found: [name]
Did you mean: [similar objects from base_tableList]
```

**EXPLAIN Execution Failure:**
```
⚠️ Could not execute EXPLAIN: [reason]
Options: 1) Paste EXPLAIN text directly  2) Verify SQL and permissions
```

---

## Teradata MCP Tools Reference

| Tool | Purpose in EXPLAIN Analysis |
|------|----------------------------|
| `teradata:base_readQuery` | Execute EXPLAIN statement |
| `teradata:base_tableDDL` | Get table structure, indexes, partitioning |
| `teradata:base_columnDescription` | Column types, constraints, nullable |
| `teradata:dba_tableSpace` | Table size for scan severity assessment |
| `teradata:dba_tableUsageImpact` | Usage patterns to contextualize recommendations |
| `teradata:base_tableAffinity` | Commonly joined tables for index design |
| `teradata:base_tablePreview` | Sample data for distribution context |
| `teradata:dba_resusageSummary` | System resource usage for broader context |

---

## Validation Checklist (Before Presenting)

Run through this before every delivery:

**Both formats:**
- [ ] All EXPLAIN steps represented in flow diagram
- [ ] Color coding consistent (critical=red, warning=orange, good=green, info=blue)
- [ ] Confidence levels correctly classified per step
- [ ] Parallel execution blocks shown correctly (side-by-side in HTML, subgraph in Mermaid)
- [ ] Edges/arrows include spool names and sizes
- [ ] Summary metrics accurate (total time, step count, critical count)
- [ ] Key insights reference actual step numbers from the plan
- [ ] Each recommendation is specific with ready-to-run SQL

**HTML only:**
- [ ] Interactive buttons work (highlight critical path, product joins, reset)
- [ ] Tooltips contain purpose + technical detail + fix suggestion
- [ ] Legend explains all visual elements
- [ ] File saved in working directory and opened in browser

**Markdown only:**
- [ ] Mermaid `classDef` entries present for all four severity levels
- [ ] Step details table includes all steps with correct metrics
- [ ] Mermaid graph renders correctly (valid node IDs, balanced quotes, proper edge syntax)

---

## Response Text Format

**HTML mode:** Keep accompanying text brief (3–4 paragraphs max) — the diagram is the analysis. Summarize the top bottlenecks and quick-win SQL after presenting the file.

**Markdown mode:** The report IS the response — render the full markdown report inline per [markdown-report-template.md](references/markdown-report-template.md). No separate summary is needed since the report itself contains the summary table, Mermaid graph, step details, and playbook.

---

## Special Cases

| Scenario | Handling |
|----------|----------|
| Very complex query (20+ steps) | Add collapse/expand sections; navigation buttons |
| Query already optimal | Lead with ✅ validation; use diagram educationally |
| Comparative analysis | Side-by-side diagrams; winner table; delta metrics |
| Pre-generated EXPLAIN text | Validate format, skip execution step, proceed to parse |
| Dynamic EXPLAIN with masking | Note `:*` values; document IPE fragments used |
| MERGE/UPSERT | Show conditional branches; verify parallel sub-steps |
| Column-partitioned table | Note N+1 partition count; flag CP merge spool if present |
| Skew indicators | Highlight `group_amps` with few AMPs; recommend PRPD stats |
