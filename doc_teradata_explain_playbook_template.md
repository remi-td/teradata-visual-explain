# Teradata `EXPLAIN` Analysis Playbook Template

> **Purpose:** A repeatable checklist + worksheet to go from raw SQL/plan to specific, defensible tuning actions. Use one copy per query/revision.

---

## 0) Metadata
- **Analyst:** `___`
- **Date/Time (UTC):** `___`
- **Environment:** `Prod / Pre‑Prod / Dev`  
  **System (release / nodes / AMPs):** `___`  
  **Session mode:** `ANSI / Teradata`  
  **Maps in use:** `___`
- **Workload/QoS class:** `___`  
- **Business owner / SLA:** `___`

---

## 1) Problem framing
- **Goal of the request:** `baseline / speed‑up / reduce spool / eliminate hot AMP / stabilize`  
- **Symptoms:** `long runtime / spool exhaustion / blocking locks / skew / CPU / I/O`  
- **Target KPIs:** `runtime ≤ ___ / peak spool ≤ ___ / CPU ≤ ___ / rows returned = ___`

---

## 2) Inputs
- **Raw SQL (final text):**
```sql
-- paste the exact SQL here
```
- **EXPLAIN text (verbatim):**
```
-- paste entire EXPLAIN output here (STATIC or DYNAMIC); keep step numbers
```
- **DBQL evidence (if available):** `elapsed, CPU, skew %, steps, XMLPlan`  
- **Table/Index DDL (relevant only):** `PIs, PPIs, CPs, USI/NUSI, JI`  
- **Statistics inventory:** `list all collected stats & timestamps`

---

## 3) Phrase spotting (copy/paste exact snippets)
- **Access:** `unique/primary index`, `all‑rows scan`, `primary AMP index`  
- **Movement:** `redistributed by hash code`, `duplicated on all AMPs`  
- **Ordering:** `SORT to order Spool n by row hash / sort key`  
- **Geography:** `(single‑AMP / few AMPs / all‑AMPs)`, `(group_amps / all_amps)`, `in mapname`  
- **Partitioning:** `single partition scans`, `dynamic partition elimination`, `n column partitions`  
- **Pipelining/Parallel:** `execute the following steps in parallel`, pipelined phrasing  
- **Confidence:** `No / Low / High / Index Join confidence`, dynamic AMP sampling notes  
- **Conditionals:** `If no update in step … then INSERT`, MERGE branches  
- **Estimates:** `estimated size`, `estimated time`

---

## 4) Access path assessment
- **Desired vs actual:** `tactical PI/UPI vs scan`  
- **Sargability issues:** `implicit casts / functions / ORs`  
- **Partition elimination:** `static / dynamic / none`  
- **Column‑partition access:** counts & method (`rowID spool / CP merge spool`)

---

## 5) Join plan assessment
- **Methods used:** `merge / hash / product / nested`  
- **Geography:** `redistribution sides`, `duplication`, `rowkey‑based`  
- **Sorts:** where & why  
- **Skew risks:** `key cardinality / hot values`  
- **PRPD applicability:** `yes/no; which values; stats present?`

---

## 6) Spool & AMP utilization
- **Spool creators:** `which steps / spools`  
- **(group_amps) presence:** `yes/no; uneven distribution?`  
- **Peak spool estimate vs limits:** `___`  
- **Hot AMP signals:** `few AMPs / large redistributed spool / DBQL skew`

---

## 7) Confidence & statistics plan
- **Low/No confidence steps:** list step numbers  
- **Stats to (re)collect:**
  - **Predicates:** `table.col [, multicolumn]`
  - **Join keys:** `left.cols ⨝ right.cols`
  - **Grouping columns:** `cols`
  - **Partition stats:** `row/column partitioning`
  - **Sampling vs stale stats:** decisions & cadence

---

## 8) Optimization actions (ranked)
1) `Rewrite` — e.g., make predicate sargable, push filters before joins, de‑OR, use EXISTS/IN, rewrite OUTER→INNER when safe.  
2) `Join geography` — duplicate small side, pre‑hash keys, enforce same PI for staging, reduce movement.  
3) `Indexing` — add/drop **USI/NUSI**, leverage **join index** (covering & maintenance cost).  
4) `Partitioning` — add RPPI; adjust CP design; enable elimination.  
5) `Statistics` — collect multicolumn; refresh stale.  
6) `PRPD` — ensure stats reveal skew; verify effect.  
7) `Workload` — ensure eligible steps run in parallel; consider map placement.

> Record the *expected* effect (runtime/spool deltas) per action.

---

## 9) Validation plan
- **A/B runs:** original vs optimized SQL  
- **`EXPLAIN` deltas:** key phrase changes & estimates  
- **DBQL deltas:** elapsed/CPU/spool/skew  
- **Edge cases tested:** data ranges, nulls, outer join semantics  
- **Rollback plan:** if regression occurs

---

## 10) Final recommendations (exec summary)
- **Chosen changes:** `___`  
- **Expected improvements:** `___`  
- **Risks/Trade‑offs:** `___`  
- **Next stats recollection:** `___`

---

## 11) Appendix
- **Screenshots/attachments:** `Visual Explain, Viewpoint`  
- **Actual plan (XMLPlan extract):** `___`  
- **DDL/snippets:** `___`  
- **References used:**
  - Teradata **Interpreting EXPLAIN Output** (Request Modifier, Confidence Levels, Dynamic EXPLAIN, Joins, Parallel/Pipelined, Partitioned & Column‑Partitioned Access).  
  - Optimizer background & statistics behavior.  

---

### Prompts (for an AI assistant)
Use these to drive deterministic analysis:

```
Given the EXPLAIN text below, list all phrases indicating data movement and access paths. For each, explain why the Optimizer likely chose it and what alternative plans might exist.
```

```
From this EXPLAIN, identify the lowest-confidence steps and produce a stats recollection plan (predicate, join, group, partition). Prioritize by expected plan impact.
```

```
Detect signs of skew (group_amps, few AMPs, big redistributed spools). Propose PRPD-friendly rewrites, including which stats must exist for the optimizer to apply PRPD.
```

```
Compare STATIC vs DYNAMIC EXPLAIN outputs for the same SQL. What optimizations did IPE enable (partition elimination, sparse JI qualification), and how do they affect spool and join geography?
```
