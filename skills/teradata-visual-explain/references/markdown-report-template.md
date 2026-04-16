# Markdown / Mermaid Report Template

Use this template when generating the **Markdown** output format. The report is rendered inline in the conversation (no external file needed).

## Mermaid Flowchart Conventions

Build a `flowchart TD` (top-down) graph. Every EXPLAIN step becomes a node; every spool transfer becomes an edge.

### Node IDs

Use `S` + step number, replacing dots with underscores for parallel sub-steps:

```
S1      → Step 1
S2_1    → Step 2, sub-step 1 (parallel)
S2_2    → Step 2, sub-step 2 (parallel)
S3      → Step 3
```

### Node Labels

Include operation badge, description, and key metrics. Use `<br>` for line breaks inside node labels:

```
S3["[HASH-J] Step 3: Spool 3 ⟕ DBC.tvm<br>3,200 rows | 27.8 MB | 0.02s (11.8%)<br>⚠️ NO CONFIDENCE"]
```

Format: `[BADGE] Step N: short description<br>rows | size | time (%)<br>confidence indicator`

Confidence indicators:
- `✅ HIGH` for high confidence
- `⚠️ LOW` for low confidence
- `🔴 NONE` for no confidence
- `💎 IDX JOIN` for index join confidence

### Severity Style Classes

Define these classDef entries at the top of every graph:

```mermaid
flowchart TD
    classDef critical stroke:#C0392B,stroke-width:3px,fill:#fdf0ef
    classDef warning stroke:#FF5F02,stroke-width:3px,fill:#fff5ef
    classDef good stroke:#27AE60,stroke-width:3px,fill:#edfbf3
    classDef info stroke:#4A90E2,stroke-width:3px,fill:#edf4fd
```

Apply to nodes with `:::severity`:

```
S3["..."]:::warning
S4["..."]:::good
S9["..."]:::critical
```

### Edges (Spool Transfers)

Label edges with spool name and size:

```
S2_1 -->|"Spool 3: 4 rows, 164B"| S3
S3 -->|"Spool 5: 3,200 rows, 27.8 MB"| S5
```

For steps that feed multiple downstream steps (spool reuse), draw separate edges:

```
S4 -->|"Spool 8"| S6
S4 -->|"Spool 8"| S7
```

### Parallel Execution

Use a `subgraph` block:

```
subgraph par1 ["⚡ Parallel Execution"]
    direction LR
    S2_1["..."]:::warning
    S2_2["..."]:::good
end
```

### Fan-in After Parallel

Connect each parallel branch to its downstream consumer:

```
S2_1 -->|"Spool 3"| S3
S2_2 -->|"Spool 4"| S5
```

### Final Output Node

Always end with an output node:

```
S13(["📤 OUTPUT: 10 rows → user"]):::info
```

Use `([...])` for the stadium/rounded shape to visually distinguish the final step.

---

## Full Markdown Report Structure

Render the entire report inline as a single markdown response. Structure:

````markdown
## EXPLAIN Analysis: `{SQL query}`

### Summary

| Metric | Value |
|--------|-------|
| Total Est. Time | X.XXs |
| Steps | N |
| Critical Issues | N (product joins, no confidence on large tables) |
| Warnings | N |
| High Confidence Steps | N |
| Result Rows | N |

### Query Plan

```mermaid
flowchart TD
    classDef critical stroke:#C0392B,stroke-width:3px,fill:#fdf0ef
    classDef warning stroke:#FF5F02,stroke-width:3px,fill:#fff5ef
    classDef good stroke:#27AE60,stroke-width:3px,fill:#edfbf3
    classDef info stroke:#4A90E2,stroke-width:3px,fill:#edf4fd

    S1["🔒 Step 1: LOCK ..."]:::info
    ...nodes...
    ...edges...
```

### Step Details

| Step | Operation | Table/Spool | Rows | Size | Time (%) | Confidence | Severity |
|------|-----------|-------------|------|------|----------|------------|----------|
| 1 | LOCK | tvm, DB2, ... | — | — | — | — | info |
| 2.1 | ALL-AMP RETRIEVE | DBC.OU → Spool 3 | 4 | 164 B | 0.00s | none | warning |
| ... | ... | ... | ... | ... | ... | ... | ... |

### Top Bottlenecks

1. **Step N** — [issue] — X.XXs (ZZ% of total)
2. **Step N** — [issue] — X.XXs (ZZ% of total)
3. **Step N** — [issue] — X.XXs (ZZ% of total)

### Optimization Playbook

#### 🔴 Critical
- **Issue:** [exact EXPLAIN phrase]
  **Root cause:** ...
  **Action:**
  ```sql
  COLLECT STATISTICS ON ...;
  ```
  **Expected impact:** ...

#### 🟡 Warning
- ...

#### 🟢 Good Practices Confirmed
- ...

### Key Insights
- [Data flow summary, spool management observations, parallel execution notes]
````

---

## When to Use Markdown vs HTML

| Signal | Format |
|--------|--------|
| User says "markdown", "md", "mermaid", "text" | **Markdown** |
| User is an AI agent or tool consuming the output | **Markdown** |
| User says "html", "visual", "interactive", "browser" | **HTML** |
| No format specified | **HTML** (default) |
