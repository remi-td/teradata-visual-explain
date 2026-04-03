---
name: teradata-explain-analyzer
description: Analyze Teradata SQL EXPLAIN output to visualize query execution plans, identify performance bottlenecks, and provide actionable optimization recommendations. Use proactively when users ask about query performance, slow queries, EXPLAIN analysis, query plan interpretation, or SQL optimization on Teradata.
---

# Teradata EXPLAIN Analyzer

## Overview

Transform raw Teradata `EXPLAIN` output into interactive visual query plans with prioritized, actionable optimization recommendations. Serve both beginners needing educational context and experts expecting deep optimizer reasoning.

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

**Key phrases glossary (map these to optimizer intent):**

| Phrase | Optimizer Intent |
|--------|-----------------|
| `single-AMP RETRIEVE by way of the unique primary index` | Best-case tactical access, no spool needed |
| `by way of an all-rows scan` | Full table scan — examine predicates, stats, partitioning |
| `redistributed by hash code` | Row movement to co-locate join keys — check skew risk |
| `duplicated on all AMPs` | Broadcast small input — verify it's truly small |
| `(group_amps)` | Spool built on subset of AMPs — potential skew signal |
| `(all_amps)` | Spool built on every AMP — expected for large tables |
| `SORT to order Spool n by row hash` | Preparing for merge/hash join — sorted by rowhash |
| `estimated with no confidence` | Missing statistics — unreliable cardinality estimate |
| `estimated with low confidence` | Partial/stale statistics — conservative planning |
| `estimated with high confidence` | Good statistics — optimizer has reliable estimates |
| `execute the following steps in parallel` | Independent sub-steps dispatched concurrently |
| `RowHash match scan` | Join/read driven by rowhash ordering |
| `eliminating duplicate rows` | DISTINCT or duplicate removal in spool |

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

### Step 5: Generate Interactive Flow Diagram (PRIMARY DELIVERABLE)

**ALWAYS generate an HTML flow diagram as the primary output.** This is not optional.

#### Layout: Vertical Flow (Top → Bottom)
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

#### Node Structure (each EXPLAIN step = one node)
```html
<div class="flow-node [critical|warning|good|info]" data-step="N">
    <div class="node-header">
        <div class="node-step">Step N</div>
        <div class="node-type">[ALL-AMP]</div>
        <div class="confidence-badge [high|low|none]">HIGH</div>
    </div>
    <div class="node-title">Human-readable description of operation</div>
    <div class="node-metrics">
        <span class="metric-label">Rows</span><span class="metric-value">1,250,000</span>
        <span class="metric-label">Size</span><span class="metric-value">573 MB</span>
        <span class="metric-label">Time</span><span class="metric-value">2.5s (18%)</span>
        <span class="metric-label">Method</span><span class="metric-value">Hash join</span>
    </div>
    <div class="tooltip-content">
        <strong>Purpose:</strong> ...<br>
        <strong>Technical:</strong> Join condition: A.id = B.id<br>
        <strong>Performance:</strong> Why fast or slow<br>
        <strong>Issue:</strong> Specific problem if any<br>
        <strong>Fix:</strong> Immediate optimization action
    </div>
</div>
```

#### Operation Type Labels (standardized icons)

| EXPLAIN Text | Label | Notes |
|---|---|---|
| all-AMPs RETRIEVE | `[ALL-AMP]` | |
| single-AMP RETRIEVE | `[1-AMP]` | Best access |
| hash join | `[HASH-J]` | |
| merge join | `[MERGE-J]` | |
| product join | `[PROD-J]` | ⚠️ Always flag critical |
| nested join | `[NEST-J]` | |
| SORT | `[SORT]` | |
| aggregate/SUM | `[AGG]` | |
| redistribute | `[REDIST]` | |
| duplicate on all AMPs | `[DUP-ALL]` | |
| lock | `[LOCK]` | |
| output to user | `[OUTPUT]` | |

#### Arrows Between Steps
```html
<div class="flow-arrow">
    <div class="arrow-line"></div>
    <div class="arrow-label">Spool 3: 573 MB, 46,005 rows</div>
</div>
```

#### Parallel Execution Pattern
```html
<div class="parallel-start">◆ PARALLEL EXECUTION START (Steps 3.1–3.3)</div>
<div class="flow-row parallel">
    <!-- Nodes side-by-side using flex-direction: row -->
</div>
<div class="parallel-end">◆ PARALLEL EXECUTION COMPLETE</div>
```

#### Confidence Indicators
- `●●●●●` High Confidence
- `●●●○○` Low Confidence
- `○○○○○` No Confidence ⚠️
- `●●●●◆` Index Join Confidence

#### Required Interactive Features

1. **Hover tooltips** — Explain each step: purpose, technical details, performance implications, fix suggestions
2. **"Highlight Critical Path"** button — Illuminate top 5 slowest steps
3. **"Highlight Product Joins"** button — Flash all `[PROD-J]` nodes
4. **"Highlight No Confidence"** button — Flash all No Confidence steps
5. **"Reset"** button — Return all nodes to normal state

```javascript
function toggleCriticalPath() {
    const criticalSteps = findTopStepsByTime(5); // top 5 by time %
    document.querySelectorAll('.flow-node').forEach(node => {
        if (criticalSteps.includes(node.dataset.step)) {
            node.style.boxShadow = '0 0 20px 5px rgba(231,76,60,0.6)';
            node.style.transform = 'scale(1.05)';
        } else {
            node.style.opacity = '0.4';
        }
    });
}
```

#### Required Sections in HTML File

1. **Teradata-branded page header** (navy background, orange accent, base64 logo — see CSS section)
2. **Summary metrics bar** — 4–6 cards: total time, step count, critical issues, confidence distribution, result rows
3. **Legend** — Color meanings, badge types, operation icons
4. **Interactive control buttons**
5. **Flow diagram container** — All steps as nodes with arrows
6. **Key Insights panel** — Critical path breakdown, data growth table, top 3 optimization actions

#### Teradata Brand CSS (mandatory — include verbatim)

```css
@import url('https://fonts.googleapis.com/css2?family=Inter:wght@300;400;500;600;700&display=swap');

:root {
    --td-orange: #FF5F02;
    --td-navy: #00233C;
    --td-white: #FFFFFF;
    --color-critical: #C0392B;
    --color-warning: #FF5F02;
    --color-good: #27AE60;
    --color-info: #4A90E2;
    --color-bg: #F5F7FA;
    --color-bg-card: #FFFFFF;
    --color-text: #00233C;
    --font-family: 'Inter', system-ui, -apple-system, sans-serif;
}
body { font-family: var(--font-family); color: var(--color-text); }

.td-page-header {
    background: var(--td-navy);
    border-bottom: 3px solid var(--td-orange);
    padding: 16px 30px;
    display: flex;
    align-items: center;
    gap: 16px;
}
.td-page-header .page-title { color: var(--td-white); font-weight: 600; font-size: 18px; }
h1, h2, h3 { color: var(--td-navy); font-weight: 600; }

/* Flow nodes */
.flow-node {
    background: white;
    border-radius: 10px;
    padding: 15px 20px;
    min-width: 280px;
    border: 3px solid;
    box-shadow: 0 2px 8px rgba(0,0,0,0.1);
    transition: all 0.3s ease;
    cursor: pointer;
    position: relative;
}
.flow-node:hover { transform: translateY(-5px) scale(1.02); box-shadow: 0 8px 20px rgba(0,0,0,0.2); }
.flow-node.good     { border-color: #27AE60; }
.flow-node.warning  { border-color: #FF5F02; }
.flow-node.critical { border-color: #C0392B; }
.flow-node.info     { border-color: #4A90E2; }

/* Confidence badges */
.confidence-badge { padding: 2px 8px; border-radius: 10px; font-size: 10px; font-weight: bold; text-transform: uppercase; }
.confidence-badge.high { background: #27AE60; color: white; }
.confidence-badge.low  { background: #FF5F02; color: white; }
.confidence-badge.none { background: #C0392B; color: white; }

/* Tooltips */
.tooltip-content {
    display: none;
    position: absolute;
    background: #2c3e50;
    color: white;
    padding: 15px;
    border-radius: 8px;
    max-width: 400px;
    font-size: 12px;
    top: 100%;
    left: 50%;
    transform: translateX(-50%);
    margin-top: 10px;
    z-index: 1000;
}
.flow-node:hover .tooltip-content { display: block; }

/* Flow container */
.flow-container { display: flex; flex-direction: column; gap: 20px; min-width: 1200px; }
.flow-row { display: flex; align-items: center; justify-content: center; gap: 20px; }
.flow-row.parallel { justify-content: space-around; }
.flow-diagram { background: #f8f9fa; padding: 30px; border-radius: 8px; overflow-x: auto; }
```

**Teradata logo (base64 — use this exact value, never recreate the logo mark):**
```html
<img src="data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAEAAAABVCAMAAADOrBLEAAAAIGNIUk0AAHomAACAhAAA+gAAAIDoAAB1MAAA6mAAADqYAAAXcJy6UTwAAAGbUExURf9fAgAAAP9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv9fAv///3jJeXQAAACHdFJOUwAA4/Lx9HV8gggMBOHDxL0090PNnJ6YKgGB7pTktAfU3SD8YJ3GGf4QSfOtNgYDHk50g39kIdnqrlFBPT48I4na+u+4Uqr79vXtQNKTFGegDyfRd4jeMDkd2JBL+M4ib+nwUwJK0CbimQXoLna8ax8KK1iGyutCwS0sOkRMTRXnjBwW5ciKNyCa7VMAAAABYktHRIhrZhZaAAAACXBIWXMAAC4jAAAuIwF4pT92AAAAB3RJTUUH6gIDDSMcsQ7WtwAAAeNJREFUWMPt1tlbElEYBvCXMEhGssVwynIBCxBEMXQKynBDMAoqSElt0bKgXFpt07LM+bcdRpiGc0jOuaini/Nezjfze76zAo5Ym2py1AYLT9CkErELQAAC+E+AY80OOhLYgRYnleOtJziAejl5SgACEMC/B063naHiaucA7JDrhAuw8EQAAhCAAAQggL8MnAWXgHMk0MEJnCeBC518QBcJdPfwAa0k4PZwjQG9JKBevMQF2Lwk4HX5OHqAv49qwR0I9v+ZQGhgMDx0ORJ06O9geESlo1wJX43Grhm57qt6wOiN+Fi5z/GJyWbtKfqn1LpJTCd/JzVTASD7bxqvJNO3oOW22jjVv3mA37xvvJmsBtxp4QBG79bO1j0ZCLmYAcg5opC/r7UwqzADcwWykoO2LA+YgfkFsrJYXoiHi2wA8IiqWKFP7WNGwE5VvHpBjjxhA8JUZfpgbFJEYQAsWEqQlUJlfZefPmMBVp6TlRcwtnhHqjEgkYdfiRqHDsVYptQIQJC4AdNF06lF8eWr1ZL7EECb7rWk+blzvfbYAxuv37wNvMu/71sYS5hi/VAdaqfNtGCbH0HfG5Cl7MCnz1++bpkyv23cB9Kgs7Idv33f0T7n+xnSu/zhCcSVn7u/9vTx7wNeLG/Yd2GbHAAAAABJRU5ErkJggg=="
     alt="Teradata" height="32" style="filter: brightness(0) invert(1);" />
```

#### File Naming & Delivery
```
1. Create: /home/claude/[query_name]_flow_diagram.html
2. Copy:   /mnt/user-data/outputs/[query_name]_flow_diagram.html
3. Present via present_files tool as PRIMARY deliverable
```

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

- [ ] All EXPLAIN steps represented in flow diagram
- [ ] Color coding consistent (critical=red, warning=orange, good=green, info=blue)
- [ ] Confidence levels correctly classified per step
- [ ] Parallel execution blocks shown side-by-side
- [ ] Arrows include spool names and sizes
- [ ] Interactive buttons tested (highlight critical path, product joins, reset)
- [ ] Tooltips contain purpose + technical detail + fix suggestion
- [ ] Legend explains all visual elements
- [ ] Summary metrics accurate (total time, step count, critical count)
- [ ] Key Insights panel references actual step numbers from the plan
- [ ] HTML file saved to `/mnt/user-data/outputs/`
- [ ] File presented via `present_files` as PRIMARY deliverable
- [ ] Response text is brief (3–4 paragraphs max — the diagram IS the analysis)
- [ ] Each recommendation is specific with ready-to-run SQL

---

## Response Text Format (After Presenting Diagram)

Keep text to minimum — the visual does the heavy lifting:

```markdown
## 🚨 [or ✅] [One-sentence critical finding or validation]

[Flow diagram presented via present_files]

## Top Bottlenecks
1. Step X: [Issue] — Y.YYs (ZZ% of total)
2. Step X: [Issue] — Y.YYs (ZZ% of total)
3. Step X: [Issue] — Y.YYs (ZZ% of total)

## Quick Win
```sql
COLLECT STATISTICS ON db.table COLUMN col_name;
```
**Expected impact:** [Quantified or qualitative improvement estimate]

## Next Steps
1. [Immediate — do today]
2. [Short-term — this week]
3. [Consider — ongoing]
```

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
