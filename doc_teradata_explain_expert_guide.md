# Teradata `EXPLAIN`—An Expert-Focused Interpretation Guide (for Training AI Agents)

> **Goal:** Equip an AI agent to *analyze, reason about, and advise on* Teradata SQL `EXPLAIN` output at an expert level—covering semantics, plan structure, execution geography, join strategies, spools, statistics/confidence, static vs dynamic plans (IPE), parallel/pipelined steps, partitioned and column‑partitioned access, and common tuning heuristics. citeturn1search4

---

## 1) What `EXPLAIN` is (and isn’t)

- **Definition.** `EXPLAIN` is a request modifier that returns a **human‑readable summary** of the Optimizer’s step‑by‑step plan (the *AMP steps*) for processing a valid Teradata SQL request; the request itself is **not executed** (except where a **dynamic** plan is produced—see §6). citeturn1search4turn1search43
- **Why it’s useful.** The report shows access and join strategies, intermediate **spools**, chosen **indexes**, estimated **row counts** and **relative time**, and whether steps are dispatched **in parallel**—supporting query design, performance comparison, and regression checks across upgrades. citeturn1search4
- **Caveats.** The plan shown may **differ** from the plan used at runtime due to changing **statistics**, **data demographics**, or **dynamic planning** decisions; use **DBQL** (including XMLPlan) for the *actual* executed plan. citeturn1search4
- **Scope limits.** `EXPLAIN` applies to **SQL requests only** (not `USING` modifiers, another `EXPLAIN`, or standalone functions/procedures/methods). If the request involves parameter literals or built‑ins, **Request Cache peeking** can substitute values in the text. citeturn1search4
- **Time units.** Reported “seconds” are **relative cost units** for comparing formulations, not wall‑clock guarantees. citeturn1search4

---

## 2) Where `EXPLAIN` fits in Teradata’s architecture

- **Parsing Engine (PE).** The PE parses, optimizes, **decomposes SQL into steps**, and dispatches those steps to AMPs over the BYNET; `EXPLAIN` renders that plan in English. citeturn1search7
- **AMPs & BYNET.** AMPs execute steps **in parallel** against their local data (shared‑nothing), exchanging data via BYNET when redistribution/duplication is required; spools live on AMPs. citeturn1search12

---

## 3) Reading an `EXPLAIN` plan: structure & recurring phrases

### 3.1 Step preamble & locks
- Plans commonly start with **pseudo table** and table **locks** (ACCESS/READ/WRITE/EXCLUSIVE) and end with **END TRANSACTION** where applicable—these are visible in examples and help reason about concurrency. citeturn1search30turn1search8

### 3.2 AMP geography and maps
- **`single-AMP`**, **`all-AMPs`**, and **`few AMPs`** indicate degree of AMP engagement per step; **`in mapname`** denotes the *map* (AMP collection) used to execute the step. citeturn1search29
- **`(all_amps)`** means a spool is created on **all AMPs** of the map; **`(group_amps)`** means the spool is created on a **dynamic subset** of AMPs that actually receive rows. citeturn1search29

### 3.3 Spools & life cycle
- Each intermediate result is a **numbered spool** (`Spool n`), potentially flagged **`(Last Use)`** when it will be freed after the step; spools can be **built locally** on AMPs, **redistributed** across AMPs (by hash), or **duplicated** (broadcast) depending on the join geography. citeturn1search29turn1search38

### 3.4 Common phrase glossary (key items)
- **`redistributed by hash code`**: rows are **rehash‑moved** to co‑locate join keys on target AMPs. citeturn1search38  
- **`duplicated on all AMPs`**: small relation is **broadcast** to all AMPs to avoid redistribution of a large table. citeturn1search38  
- **`by way of an all-rows scan`**: **full scan** path (no single‑partition elimination possible). citeturn1search29  
- **`by way of the primary index / unique primary index`**: direct single‑AMP access via PI/UPI (UPI ⇒ at most one row). citeturn1search29  
- **`RowHash match scan`**: join/read driven by rowhash ordering for matching pairs. citeturn1search29  
- **`eliminating duplicate rows`**: DISTINCT/duplicate removal in spool. citeturn1search29  
- **`estimated size/time`**: Optimizer’s estimates—accuracy depends on stats quality (see §4). citeturn1search29

> **Tip for the AI agent:** Always tie a step’s geography (**single/few/all AMPs**) and data movement (**redistribute/duplicate**) to the **chosen join method** and the **access path**—this is core to anticipating bottlenecks (skew, spool blow‑ups, hot AMPs). citeturn1search38

---

## 4) Statistics & **EXPLAIN Confidence Levels**

- The Optimizer annotates cardinality estimates with **No / Low / High / Index Join** confidence to indicate the **quality of its estimates**, heavily influenced by **presence/freshness** of stats and **dynamic AMP sampling**. Lower confidence yields **more conservative** join planning. citeturn1search6
- **Single‑table** heuristics: High confidence typically requires stats on predicate columns or PI (or sufficient dynamic sampling without skew); **Low** can arise with OR/AND combinations or partial stats; **No** means no usable stats/complex nondeterministic expressions. citeturn1search6
- **Joins:** confidence never exceeds the lower of the inputs; **Index Join** confidence indicates uniqueness (e.g., PK/FK or unique index). **Cumulative (multiplicative) errors** across joins are why fresh stats matter. citeturn1search6

---

## 5) Join planning: methods, geographies, and prep

### 5.1 Join methods commonly seen in `EXPLAIN`
- **Merge Join**: requires sorted inputs (often via “SORT by rowhash”); efficient for large, well‑distributed join keys. citeturn1search38
- **Hash Join** (and variants such as in‑memory hash join): build a hash table from one input and probe with the other. (Appears in phrase terminology and “hash table is built from Spool n…”.) citeturn1search29
- **Product Join**: Cartesian followed by qualification; expensive but sometimes optimal for tiny inner inputs or special cases highlighted in examples. citeturn1search38
- **Nested Joins** (local/remote): single‑row access enabling AMP‑local lookups via UPI/USI (covered in Optimizer topics referenced from the EXPLAIN chapter). citeturn1search2

### 5.2 Join geography (co‑location of rows)
- **Redistribution**: both or one side **rehash‑moved** to target AMPs on the join key. **SORT by rowhash** often follows redistribution prior to merge/hash joins. citeturn1search38
- **Duplication**: small input **broadcast to all AMPs** to avoid moving a large table. citeturn1search38
- **RPPI/Rowkey‑based** phrasing**:** joins may be optimized by **row‑partition** knowledge to limit partitions joined. citeturn1search29

### 5.3 Skew and PRPD (partial redistribution/duplication)
- Severe data skew on join keys can create **Hot AMPs** and spool exhaustion; Teradata introduced **Partial Redistribution and Partial Duplication (PRPD)** to keep skewed values local and only redistribute non‑skewed values, while **duplicating** corresponding “small side” rows for skewed values—reducing hot spots when stats reveal biased values. citeturn1search21

---

## 6) **STATIC** vs **DYNAMIC** `EXPLAIN` (Incremental Planning & Execution, IPE)

- **STATIC EXPLAIN** (default): produces a **static** plan report—no step is executed. citeturn1search5
- **DYNAMIC EXPLAIN**: requests an **IPE‑based dynamic plan** report (`DYNAMIC EXPLAIN …`), where the system may execute small **plan fragments** to feed **intermediate results** back into planning (e.g., partition elimination, transitive closure, sparse join index qualification). The report can show *actual* fragment sizes and special markers like `":*"` when **masked** for security. citeturn1search3
- **Behavioral nuances.**  
  - If dynamic plan display is **disabled** or IPE **thresholds** aren’t met, you’ll see the **static** plan with a preface explaining why. citeturn1search3  
  - `USING` variables can **disqualify** IPE; `DYNAMIC EXPLAIN` then returns the static plan. citeturn1search3  
  - You may request **XML** format (`IN XML`) for machine ingestion. citeturn1search3

---

## 7) Parallel and **pipelined** execution in `EXPLAIN`

- **Parallel steps.** `EXPLAIN` explicitly annotates when the system will **“execute the following steps in parallel”** (with indented numbered sub‑steps). The engine can run **up to 20** parallel steps when channels aren’t required (PI‑constrained), and uses **channel limits** (e.g., two or four) otherwise; the report shows **logical** parallelism, not the instantaneous concurrency cap. citeturn1search40
- **Pipelined steps.** Plans may also indicate **pipelining**, where producers/consumers overlap without materializing to disk for each boundary—improving latency; this is documented alongside EXPLAIN guidance. citeturn1search25turn1search44

---

## 8) Partitioned and **column‑partitioned** objects in `EXPLAIN`

- **Row partitioning (PPI/RPPI).** Expect phrases like **“single partition scans”**, **“dynamic partition elimination”**, and **“n partitions of … (all_amps)”** indicating the Optimizer’s ability to prune or target partitions. citeturn1search29
- **Column partitioning.** `EXPLAIN` reports **how many column partitions** are accessed and whether access uses **rowID spool** or **CP merge spool**—including that the **delete column partition** often counts in the total (e.g., “5 column partitions” when selecting 4 user columns + delete partition). citeturn1search46

---

## 9) Conditional DML: `MERGE` and **UPSERT** in `EXPLAIN`

- `EXPLAIN` reveals **conditional branches** such as *“If no update in step X, then do INSERT …”* for **UPSERT** (`UPDATE … ELSE INSERT`) and **MERGE** (with both `WHEN MATCHED` and `WHEN NOT MATCHED`). This helps verify atomic behavior and side effects on **join indexes** or triggers. citeturn1search50turn1search49turn1search54

---

## 10) Training the AI agent: deterministic reasoning patterns over `EXPLAIN`

The agent should apply the following **decision heuristics** consistently:

1. **Access path diagnosis**  
   - Prefer **single‑AMP** PI/UPI access over **all‑AMPs** scans; if a **full scan** appears, ask: *Are predicates sargable? Are stats present? Is partition elimination possible?* citeturn1search29turn1search6
2. **Join geography sanity**  
   - For **large‑to‑large** joins, redistribution on both sides may be acceptable with good distribution; otherwise, seek **duplicate‑small to all AMPs** or rekey to reduce movement. Watch for **SORT by rowhash** before Merge/Hash joins. citeturn1search38
3. **Spool risk**  
   - Large **redistributions**, **product joins**, or **low‑confidence** multi‑join chains increase **spool** and **skew** risk; consider **stats recollection**, *join re‑ordering*, or **PRPD**‑friendly rewrites. citeturn1search6turn1search21
4. **Confidence triage**  
   - Treat **No/Low** confidence steps as hotspots for statistics (on **predicates, join keys, grouping columns**, and relevant **multicolumn** sets). Recollect **stale** stats; dynamic single‑AMP samples are still only **Low** confidence. citeturn1search6
5. **Parallel/pipeline awareness**  
   - Validate that expensive, independent steps are **parallelized**; check for **pipelining** opportunities in scans → join → aggregation chains. citeturn1search40turn1search25
6. **Map/AMP coverage**  
   - Ensure steps run in the **appropriate map** and understand `(group_amps)` vs `(all_amps)` for estimating distribution/latency. citeturn1search29
7. **Static vs dynamic plan**  
   - If **DYNAMIC EXPLAIN** is used, incorporate **observed fragment sizes** and note security **masking** (`":*"`)—but remind that runtime may still vary if data changes. citeturn1search3
8. **Lock footprint**  
   - Confirm that lock severity and scope make sense (e.g., READ vs WRITE) and understand transaction boundaries (**END TRANSACTION**). citeturn1search30

---

## 11) Worked micro‑examples (reading the lines)

> These are *patterns* the agent should recognize and explain.

- **Single‑row PI access**  
  *“We do a single‑AMP RETRIEVE … by way of the unique primary index …”* → **Ideal tactical access**, expect negligible spool, high confidence (even without PI stats). citeturn1search29
- **Scan + redistribute + sort**  
  *“all‑rows scan … into Spool 2 (all_amps), which is redistributed by hash code … Then we do a SORT to order Spool 2 by row hash …”* → Classic **join prep** for Merge/Hash join; validate **join key distribution** and stats. citeturn1search38
- **Duplicate small table**  
  *“… Spool 2, which is duplicated on all AMPs …”* → **Broadcast** pattern; confirm that the duplicated side is **small** and stable. citeturn1search38
- **Group AMPs spool**  
  *“Spool 1 (group_amps), which is built locally on the AMPs …”* → Result materializes only on AMPs that produced rows; beware downstream **skew** if a tiny AMP subset holds most rows. citeturn1search29

---

## 12) Column‑partitioned nuance: interpreting counts

- When selecting N columns from a **column‑partitioned** table, the count in `EXPLAIN` is often **N + 1** (the extra is the **delete column partition**). The phrasing also indicates whether access uses **rowID spool** or **CP merge spool** and if **contexts** are sufficient to process multiple partitions simultaneously. citeturn1search46

---

## 13) Conditional DML example cues to watch

- **UPSERT (`UPDATE … ELSE INSERT`)**: `EXPLAIN` often shows a **single step** that attempts the UPDATE and **conditionally** INSERTs if no row was updated; in more complex cases (e.g., join index maintenance) you’ll see **parallel sub‑steps** and **conditional** branches spelled out. citeturn1search50turn1search49
- **MERGE**: Look for the sequence **MERGE (attempt update) → if no update then INSERT**, often in **parallel step groups** with **(Last Use)** markers on spools. citeturn1search54

---

## 14) Practical checklist for tuning based on `EXPLAIN`

1. **Access path**: Is there a **PI/UPI** predicate? If not, can one be introduced (e.g., staging, derived key) to avoid **all‑AMP scans**? citeturn1search29  
2. **Stats**: Are **confidence levels** acceptable on **filters, joins, groups**? If not, **collect/refresh** stats (consider **multicolumn** and **partition stats**). citeturn1search6  
3. **Join geography**: Are large tables both **redistributed**? Could we **duplicate** a small side instead or **pre‑hash** keys in staging to reduce movement? citeturn1search38  
4. **Skew/Spool**: Do you see `(group_amps)` with **lopsided** row estimates, or very large redistributed spools? Consider **PRPD**, rewrite join order, or add stats to reveal skew. citeturn1search21  
5. **Parallelism**: Are independent DML statements marked **parallel**? If not, can the request be restructured (multi‑statement transactions vs single statements) to exploit parallel dispatch? citeturn1search40  
6. **Dynamic planning**: For complex predicates/partitioning, try **`DYNAMIC EXPLAIN`** to see **feedback**‑driven optimizations and verify masking policies. citeturn1search3

---

## 15) Example commands the AI agent should propose or recognize

```sql
-- Basic:
EXPLAIN SELECT ... ;

-- Dynamic planning and XML output for machine parsing:
DYNAMIC EXPLAIN IN XML SELECT ... ;

-- Compare formulations:
EXPLAIN SELECT ... WHERE col = ? ;        -- cache peeking may substitute values
EXPLAIN SELECT ... WHERE col IN ( ... );

-- Verify partition elimination and CP access:
EXPLAIN SELECT ... FROM ppi_tbl WHERE part_col BETWEEN ... ;
EXPLAIN SELECT a,b,g,p FROM cp_tbl ;      -- note “N+1” partitions (delete CP)
```
citeturn1search4turn1search3turn1search46

---

## 16) Appendix: phrase decoding quick‑reference (non‑exhaustive)

| Phrase (as seen in EXPLAIN) | Interpreted meaning (what the AI should infer) |
|---|---|
| `single-AMP RETRIEVE by way of the unique primary index …` | **Tactical**, direct **single‑AMP** lookup; best‑case access. citeturn1search29 |
| `all-rows scan` | **Full table scan**; examine predicates/partitioning/stats. citeturn1search29 |
| `redistributed by hash code` | **Row movement** to join co‑location; check skew risk and sort overhead. citeturn1search38 |
| `duplicated on all AMPs` | **Broadcast small input** across AMPs to avoid moving large side. citeturn1search38 |
| `(all_amps)` vs `(group_amps)` | Spool created on **all** vs **subset** of AMPs; subset might signal **skew**. citeturn1search29 |
| `SORT to order Spool n by row hash` | Preparing for **rowhash‑ordered** merge/hash join. citeturn1search38 |
| `estimated size/time …` | Cost estimates (dependent on **stats quality**). citeturn1search29 |
| `execute the following steps in parallel …` | **Parallel dispatch** of independent sub‑steps; check concurrency caps. citeturn1search40 |
| `:*` | **Masked** dynamic EXPLAIN intermediate values (security). citeturn1search3 |

---

## 17) Further reading (what the AI should link internally)

- **Interpreting EXPLAIN Output** (chapter landing) and subtopics (Request Modifier, Confidence Levels, Examples, Joins, Parallel/Pipelined, Partitioned & Column‑Partitioned Access). citeturn1search2turn1search4turn1search6turn1search30turn1search38turn1search40turn1search46
- **DYNAMIC EXPLAIN (IPE)**—masked/unmasked examples and eligibility nuances. citeturn1search3
- **Optimizer background**—how stats, demographics, and join planning work (for deeper reasoning). citeturn1search19

--- 

### Closing guidance

When training the agent, emphasize **traceable reasoning**: for any plan critique or tuning suggestion, it should **quote the exact EXPLAIN phrase(s)** (e.g., *“redistributed by hash code … then SORT”*, *“(group_amps)”*, *“Low confidence”*) and tie them to **optimizer behavior** and **actionable remedies** (stats, predicates, join re‑formulation, PRPD, parallelism). This mirrors how senior Teradata engineers debug performance in practice. citeturn1search4turn1search6turn1search38turn1search21

--- 

*Primary sources used in this guide are the Teradata Vantage documentation pages under **Interpreting EXPLAIN Output** and related sections (Request Modifier, Confidence Levels, DYNAMIC EXPLAIN, Joins, Parallel/Pipelined Steps, Column‑Partition Access), plus Optimizer background; PRPD concepts are additionally summarized from a specialist performance article.* citeturn1search2turn1search4turn1search6turn1search3turn1search38turn1search40turn1search46turn1search19turn1search21

--- 

**Example doc entry to store with plans:**  
- Keep `EXPLAIN` text alongside SQL in your **system documentation** and tie rumblings to DBQL actual plans—this supports post‑upgrade regressions and knowledge transfer. citeturn1search4
