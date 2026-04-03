# Teradata EXPLAIN Statement Visualization Template
## AI Agent Instructions for Creating Visual Query Plan Representations

### Overview
This template provides instructions for creating visual representations of Teradata EXPLAIN statement output to aid in query performance investigation. The visualization should transform text-based EXPLAIN output into an intuitive graphical flow diagram similar to Teradata Visual Explain.

---

## Core Visualization Principles

### 1. Flow Direction
- **Primary Flow**: Top-to-bottom or left-to-right
- **Data Movement**: Show the progression from source tables through operations to final result
- **Step Numbering**: Display step numbers prominently to match EXPLAIN output

### 2. Visual Hierarchy
- **Group related operations** into logical blocks
- **Highlight critical path** operations (high CPU/IO operations)
- **Use visual weight** to indicate resource consumption

---

## Icon Classification System

### Statement Type Icons (Top-Level Operation)

| Icon Symbol | Statement Type | Description |
|-------------|---------------|-------------|
| `[SEL]` | SELECT | Query retrieval operation |
| `[INS]` | INSERT | Insert rows into table |
| `[UPD]` | UPDATE | Modify existing rows |
| `[DEL]` | DELETE | Remove rows from table |
| `[MRG]` | MERGE | Merge rows into table |
| `[CRT]` | CREATE | Create table/index/view |
| `[DRP]` | DROP | Drop table/index |

---

### Retrieval Method Icons

| Icon Symbol | Method | When Used |
|-------------|--------|-----------|
| `[ALL-AMP]` | All-AMP Retrieve | Data resides on more than two AMPs |
| `[1-AMP]` | Single-AMP Retrieve | Row hash determines specific AMP |
| `[2-AMP]` | Two-AMP Retrieve | Uses specific hashing combinations |
| `[M-AMP]` | Multi-AMP Retrieve | Various hashing combinations |

---

### Data Redistribution Icons

| Icon Symbol | Method | Purpose |
|-------------|--------|---------|
| `[DUP-ALL]` | Duplicated on All AMPs | Result rows duplicated across all AMPs |
| `[REDIST]` | Redistributed on All AMPs | Result rows redistributed across AMPs |
| `[LOCAL]` | Locally Built on All AMPs | Result rows built locally on each AMP |

---

### Table and Spool Icons

| Icon Symbol | Object Type | Description |
|-------------|-------------|-------------|
| `[TABLE:name]` | Base Table | Physical table in database |
| `[TABLE*:name]` | Sync Scan Table | Table eligible for synchronized scanning |
| `[SPOOL]` | Standard Spool | Intermediate query results storage |
| `[SPOOL-LC]` | Low Confidence Spool | Row estimate with low confidence |
| `[SPOOL-HC]` | High Confidence Spool | Row estimate with high confidence |
| `[SPOOL-NC]` | No Confidence Spool | No confidence in row estimate (missing stats) |
| `[SPOOL-JC]` | Index Join Confidence | Row estimate from index join confidence |
| `[SPOOL-IN]` | IN-List Spool | Spool from IN condition values |

---

### Index Type Icons

| Icon Symbol | Index Type | Description |
|-------------|-----------|-------------|
| `[UPI]` | Unique Primary Index | Unique PI access |
| `[NUPI]` | Non-Unique Primary Index | Non-unique PI access |
| `[USI]` | Unique Secondary Index | Unique SI access |
| `[NUSI]` | Non-Unique Secondary Index | Non-unique SI access |

---

### Join Method Icons

| Icon Symbol | Join Type | Description |
|-------------|-----------|-------------|
| `[HASH-J]` | Hash Join | Equijoin using hash method |
| `[MERGE-J]` | Merge Join | Sorted inputs merged |
| `[PROD-J]` | Product Join | Cartesian product |
| `[NEST-J]` | Nested Join | Constant value matched to index |
| `[DYN-J]` | Dynamic Hash Join | Small table vs large table |
| `[ROWID-J]` | Row ID Join | Join using Row IDs |
| `[LEFT-OUTER]` | Left Outer Join | Left table + nulls for non-matches |
| `[RIGHT-OUTER]` | Right Outer Join | Right table + nulls for non-matches |
| `[FULL-OUTER]` | Full Outer Join | Both tables + nulls |
| `[EXCL-J]` | Exclusion Join | NOT IN condition |
| `[INCL-J]` | Inclusion Join | IN condition |
| `[EXISTS-J]` | Exists Join | EXISTS condition |
| `[NOT-EXISTS-J]` | Not Exists Join | NOT EXISTS condition |

---

### Aggregate Operation Icons

| Icon Symbol | Function | Description |
|-------------|----------|-------------|
| `[SUM]` | Sum Aggregate | Arithmetic sum operation |
| `[STATS]` | Statistics | Grouped row statistics computation |
| `[SAMPLE]` | Sampling | Data sampling operation |

---

### Other Operation Icons

| Icon Symbol | Operation | Purpose |
|-------------|-----------|---------|
| `[SORT]` | Sort Step | Sort rows and eliminate duplicates |
| `[BITMAP]` | Bitmap | Bitmap index operation |
| `[ABORT]` | Abort | Terminate transaction |
| `[FLUSH]` | Flush | Flush database cache |
| `[SPOIL]` | Spoil | Invalidate dictionary cache |
| `[QB:name]` | Query Band | Query band identifier |
| `[CONF]` | Configuration | System configuration step |
| `[CHKPT]` | Checkpoint | Checkpoint loading operation |

---

## Connector Types and Flow Indicators

### Solid Lines (Data Flow)
```
[TABLE:Orders] ──────> [SPOOL-HC] ──────> [SORT] ──────> [RESULT]
```
- **Use**: Show direct data flow between operations
- **Style**: Solid arrow lines
- **Labels**: Include row counts and confidence levels

### Dashed Lines (Spool Reuse)
```
[SPOOL-HC:1] ┄┄┄┄┄┄┄> [JOIN-STEP:3]
             ┄┄┄┄┄┄┄> [AGG-STEP:5]
```
- **Use**: Show spool reuse in multiple steps
- **Style**: Dashed lines
- **Purpose**: Indicate performance optimization

---

## Metadata Annotations

### For Each Step, Display:

1. **Step Identification**
   ```
   STEP: 3
   TYPE: Hash Join
   CONFIDENCE: High
   ```

2. **Cardinality Estimates**
   ```
   EST ROWS: 1,250,000
   CONFIDENCE: ●●●○○ (HC)
   ```

3. **Resource Estimates** (when available)
   ```
   CPU TIME: 2.5 sec
   I/O TIME: 1.8 sec
   NETWORK: 0.3 sec
   TOTAL: 4.6 sec
   ```

4. **Join Conditions** (for joins)
   ```
   JOIN ON: Orders.customer_id = Customers.customer_id
   JOIN TYPE: Hash Join
   JOIN COLUMNS: 1
   ```

5. **Filter Conditions** (when applicable)
   ```
   WHERE: order_date >= DATE '2024-01-01'
   RESIDUAL: amount > 1000
   ```

6. **Index Usage**
   ```
   INDEX: UPI on customer_id
   ACCESS: Single-AMP
   ```

---

## Visualization Layout Strategies

### Option 1: Hierarchical Tree Layout
```
                    [SEL]
                      │
                ┌─────┴─────┐
           [STEP:1]      [STEP:2]
          [TABLE:A]     [TABLE:B]
                │           │
                └─────┬─────┘
                  [STEP:3]
                 [HASH-J]
                      │
                  [STEP:4]
                  [SORT]
                      │
                 [RESULT]
```

### Option 2: Sequential Flow Diagram
```
STEP 1: [1-AMP] → [TABLE:Orders via UPI] → [SPOOL-HC:1] (100,000 rows)
           ↓
STEP 2: [ALL-AMP] → [TABLE:Customers] → [SPOOL-HC:2] (5,000 rows)
           ↓
STEP 3: [HASH-J] → Join SPOOL:1 + SPOOL:2 → [SPOOL-HC:3] (98,500 rows)
           ↓
STEP 4: [SUM] → Aggregate by customer → [SPOOL-HC:4] (4,800 rows)
           ↓
STEP 5: [SORT] → Order by total → [RESULT] (4,800 rows)
```

### Option 3: Detailed Block Diagram
```
┌─────────────────────────────────────────┐
│ STEP 1: Retrieve Orders                │
│ ────────────────────────────────────    │
│ Method: [1-AMP] Single-AMP Retrieve     │
│ Table: Orders via [UPI] customer_id     │
│ Filter: order_date >= '2024-01-01'     │
│ Est Rows: 100,000 [HC]                  │
│ CPU: 0.5s | I/O: 0.8s                   │
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│ STEP 2: Retrieve Customers              │
│ ────────────────────────────────────    │
│ Method: [ALL-AMP] All-AMP Retrieve      │
│ Table: Customers                         │
│ Est Rows: 5,000 [HC]                     │
│ CPU: 0.2s | I/O: 0.3s                    │
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│ STEP 3: Join Operations                 │
│ ────────────────────────────────────    │
│ Type: [HASH-J] Hash Join                │
│ Condition: Orders.cust_id =              │
│            Customers.cust_id             │
│ Est Rows: 98,500 [HC]                    │
│ CPU: 2.5s | I/O: 1.2s | Net: 0.3s       │
└────────────┬────────────────────────────┘
             │
             ▼
          [RESULT]
```

---

## Performance Indicator Visual Elements

### 1. Confidence Level Indicators
```
● ● ● ● ● = High Confidence (HC)
● ● ● ○ ○ = Index Join Confidence (JC)
● ● ○ ○ ○ = Low Confidence (LC)
○ ○ ○ ○ ○ = No Confidence (NC) ⚠️
```

### 2. Resource Consumption Heat Map
```
CPU Intensity:
[▓▓▓▓▓▓▓▓▓▓] 100% - Critical
[▓▓▓▓▓▓▓░░░]  70% - High
[▓▓▓▓░░░░░░]  40% - Medium
[▓░░░░░░░░░]  10% - Low
```

### 3. Row Count Visualization
```
Size Indicators:
◆◆◆◆◆ = Very Large (>1M rows)
◆◆◆◆  = Large (100K-1M rows)
◆◆◆   = Medium (10K-100K rows)
◆◆    = Small (1K-10K rows)
◆     = Very Small (<1K rows)
```

---

## Color Coding Scheme (for HTML/Visual outputs)

### Operation Types
- **Green**: Table scans and retrieves
- **Blue**: Join operations
- **Yellow**: Aggregations and sorts
- **Orange**: Data redistribution
- **Red**: Potential performance issues
- **Purple**: Spool operations

### Confidence Levels
- **Dark Green**: High Confidence
- **Light Green**: Index Join Confidence
- **Yellow**: Low Confidence
- **Red**: No Confidence (missing statistics)

### Resource Intensity
- **Cool Blue**: Low resource usage
- **Warm Yellow**: Medium resource usage
- **Hot Red**: High resource usage

---

## Step-by-Step Visualization Creation Process

### Phase 1: Parse EXPLAIN Output
1. **Extract all steps** from EXPLAIN text
2. **Identify step numbers** and operation types
3. **Extract cardinality estimates** and confidence levels
4. **Capture resource estimates** (CPU, I/O, Network time)
5. **Map dependencies** between steps

### Phase 2: Classify Components
1. **Categorize each step** using icon classification system
2. **Identify join types** and conditions
3. **Note index usage** and access methods
4. **Flag potential issues**:
   - No confidence spools
   - Product joins
   - All-AMP operations on large tables
   - Missing statistics warnings

### Phase 3: Layout Design
1. **Choose layout strategy** based on query complexity
2. **Position nodes** to minimize crossing connections
3. **Group related operations**
4. **Align parallel operations**

### Phase 4: Add Annotations
1. **Label all connectors** with row counts
2. **Add metadata boxes** for key statistics
3. **Include timing information** when available
4. **Annotate join conditions** and filters

### Phase 5: Highlight Issues
1. **Mark high-cost operations** with visual indicators
2. **Flag confidence issues** (NC spools)
3. **Highlight redistribution steps**
4. **Note potential optimizations**

---

## Example Visualization Template

```
QUERY EXECUTION PLAN VISUALIZATION
==================================

Query ID: 12345678
Estimated Time: 12.5 seconds
Total Steps: 8
Optimization Level: Default

┌────────────────────────────────────────────────────────────────┐
│                     STEP 1: Base Table Scan                    │
│ ──────────────────────────────────────────────────────────────│
│ ⚙ Operation: [ALL-AMP] All-AMP Retrieve                        │
│ 📊 Source: [TABLE:fact_sales]                                   │
│ 📁 Rows: 50,000,000 ● ● ● ● ● (HC)                             │
│ ⏱ CPU: 5.2s | I/O: 8.5s | Total: 13.7s                        │
│ 🔍 Filter: sale_date BETWEEN '2024-01-01' AND '2024-12-31'    │
│ ⚠ Note: Full table scan - consider adding index on sale_date  │
└──────────────────┬─────────────────────────────────────────────┘
                   │ Produces SPOOL:1
                   ▼
┌────────────────────────────────────────────────────────────────┐
│              STEP 2: Redistribute for Join                      │
│ ──────────────────────────────────────────────────────────────│
│ ⚙ Operation: [REDIST] Redistribute on All AMPs                 │
│ 🎯 Hash By: customer_id                                         │
│ 📁 Rows: 50,000,000 → 50,000,000 ● ● ● ● ● (HC)               │
│ ⏱ CPU: 2.1s | I/O: 3.2s | Network: 4.5s | Total: 9.8s        │
└──────────────────┬─────────────────────────────────────────────┘
                   │ Produces SPOOL:2
                   ▼
     ┌─────────────┴──────────────┐
     │                            │
     ▼                            ▼
┌─────────────┐            ┌─────────────┐
│   STEP 3a   │            │   STEP 3b   │
│ [TABLE:dim  │            │ [TABLE:dim  │
│  _customer] │            │  _product]  │
│             │            │             │
│ [1-AMP] via │            │ [1-AMP] via │
│ UPI         │            │ UPI         │
│             │            │             │
│ 5,000 rows  │            │ 2,000 rows  │
│ ● ● ● ● ●   │            │ ● ● ● ● ●   │
└──────┬──────┘            └──────┬──────┘
       │ SPOOL:3                  │ SPOOL:4
       │                          │
       └──────────┬───────────────┘
                  ▼
┌────────────────────────────────────────────────────────────────┐
│               STEP 4: Multi-Way Hash Join                       │
│ ──────────────────────────────────────────────────────────────│
│ ⚙ Operation: [HASH-J] Hash Join (2-way)                        │
│ 🔗 Condition 1: SPOOL:2.customer_id = SPOOL:3.customer_id      │
│ 🔗 Condition 2: SPOOL:2.product_id = SPOOL:4.product_id        │
│ 📁 Est Rows: 48,500,000 ● ● ● ● ● (HC)                         │
│ ⏱ CPU: 18.5s | I/O: 5.2s | Total: 23.7s  🔥 High Cost         │
└──────────────────┬─────────────────────────────────────────────┘
                   │ Produces SPOOL:5
                   ▼
┌────────────────────────────────────────────────────────────────┐
│             STEP 5: Aggregate Calculation                       │
│ ──────────────────────────────────────────────────────────────│
│ ⚙ Operation: [SUM] Sum Aggregate                               │
│ 📊 Group By: customer_id, product_category                      │
│ 📏 Aggregates: SUM(sale_amount), COUNT(*)                       │
│ 📁 Rows: 48,500,000 → 125,000 ● ● ● ○ ○ (LC)                  │
│ ⏱ CPU: 12.2s | I/O: 2.1s | Total: 14.3s                       │
│ ⚠ Note: Low confidence - collect statistics on GROUP BY cols  │
└──────────────────┬─────────────────────────────────────────────┘
                   │ Produces SPOOL:6
                   ▼
┌────────────────────────────────────────────────────────────────┐
│                 STEP 6: Final Sort                              │
│ ──────────────────────────────────────────────────────────────│
│ ⚙ Operation: [SORT] Sort Step                                  │
│ 📊 Order By: total_sales DESC                                   │
│ 🎯 Limit: TOP 100                                               │
│ 📁 Rows: 125,000 → 100 ● ● ● ● ● (HC)                         │
│ ⏱ CPU: 1.5s | I/O: 0.8s | Total: 2.3s                         │
└──────────────────┬─────────────────────────────────────────────┘
                   ▼
              [RESULT SET]
              100 rows returned

════════════════════════════════════════════════════════════════

PERFORMANCE SUMMARY:
───────────────────
Total Estimated Time: 63.8 seconds
Total CPU Time: 39.5 seconds  [▓▓▓▓▓▓▓░░░] 62%
Total I/O Time: 19.8 seconds  [▓▓▓░░░░░░░] 31%
Total Network Time: 4.5 seconds [▓░░░░░░░░░] 7%

CRITICAL PATH: Steps 1 → 2 → 4 → 5  (56.5 seconds)

OPTIMIZATION OPPORTUNITIES:
──────────────────────────
⚠ High Priority:
  1. Step 1: Consider index on fact_sales.sale_date
  2. Step 4: High-cost join - verify statistics are current
  3. Step 5: Collect statistics on customer_id, product_category

⚡ Medium Priority:
  1. Step 2: Large redistribution - consider PI redesign

💡 Recommendations:
  - Collect statistics on all join columns
  - Consider date-range partitioning on fact_sales
  - Verify customer_id distribution for skew
```

---

## Special Scenarios and Handling

### 1. Parallel Operations
```
                     [STEP 2]
                        │
           ┌────────────┼────────────┐
           │            │            │
        [STEP 3a]   [STEP 3b]   [STEP 3c]
        Partition 1  Partition 2  Partition 3
           │            │            │
           └────────────┼────────────┘
                        │
                     [STEP 4]
```

### 2. Spool Reuse Pattern
```
       [TABLE:Orders]
             │
             ▼
       [SPOOL:1] ━━━━━┓
             │         ┃
             ▼         ┃ Reused
       [JOIN:Step3]    ┃
             │         ┃
             ▼         ┃
       [AGG:Step5] ◄━━━┛
```

### 3. Nested Subqueries
```
┌────────────────────────────────┐
│ Outer Query                    │
│  ┌──────────────────────────┐  │
│  │ Subquery 1               │  │
│  │  ┌────────────────────┐  │  │
│  │  │ Subquery 2         │  │  │
│  │  └────────────────────┘  │  │
│  └──────────────────────────┘  │
└────────────────────────────────┘
```

### 4. Product Join Warning
```
⚠️  PERFORMANCE WARNING  ⚠️
┌────────────────────────────────────┐
│ STEP X: Product Join Detected      │
│                                    │
│ Type: [PROD-J] Cartesian Product   │
│ Left Table: 10,000 rows            │
│ Right Table: 5,000 rows            │
│ Result: 50,000,000 rows 🔥         │
│                                    │
│ Recommendation: Add join condition │
└────────────────────────────────────┘
```

---

## Metrics to Always Include

### Essential Metrics
1. **Step Number** - Sequential identifier
2. **Operation Type** - Classification and icon
3. **Cardinality** - Estimated row counts
4. **Confidence Level** - HC, LC, JC, NC
5. **Resource Estimates** - CPU, I/O, Network times

### Desirable Metrics (when available)
6. **Actual vs Estimated** - If execution stats available
7. **Join Selectivity** - For join operations
8. **Index Usage** - Which indexes were used
9. **Redistribution Cost** - Network overhead
10. **Skew Indicators** - AMP imbalance warnings

### Advanced Metrics (for deep analysis)
11. **Block I/O Counts** - Detailed I/O metrics
12. **Statistics Currency** - When stats were collected
13. **Spool Usage** - Temporary space consumption
14. **Parallelism Degree** - Number of parallel steps

---

## Output Format Options

### 1. ASCII/Text Diagram
- Best for: Command-line tools, logs, reports
- Advantages: Universal, copy-paste friendly
- Use: Box-drawing characters (┌─┐│└┘├┤┬┴┼)

### 2. HTML/Web Visualization
- Best for: Interactive analysis, sharing
- Advantages: Clickable elements, tooltips, drill-down
- Features: Collapsible sections, zoom, pan

### 3. SVG/Graphical Diagram
- Best for: Documentation, presentations
- Advantages: Scalable, professional appearance
- Tools: Graphviz, Mermaid, D3.js

### 4. JSON/Structured Data
- Best for: Programmatic processing, tools integration
- Advantages: Machine-readable, extensible
- Use case: Custom visualization tools

---

## Validation Checklist

Before finalizing visualization, verify:

✅ All steps from EXPLAIN are represented
✅ Data flow direction is clear and consistent
✅ Step numbers match EXPLAIN output
✅ Join conditions are clearly labeled
✅ Cardinality estimates are included
✅ Confidence levels are indicated
✅ High-cost operations are highlighted
✅ Potential issues are flagged
✅ Spool reuse is shown with dashed lines
✅ Index usage is documented
✅ Resource estimates are displayed
✅ Critical path is identifiable
✅ Parallel operations are properly shown
✅ Legend/key is provided for icons

---

## Sample Queries for Common Patterns

### Pattern 1: Single Table Scan
```
┌────────────────────────┐
│ [ALL-AMP] Full Scan    │
│ [TABLE:customers]      │
│ Est: 1M rows ●●●●●     │
└───────┬────────────────┘
        ▼
   [RESULT: 1M rows]
```

### Pattern 2: Index-Based Retrieval
```
┌────────────────────────┐
│ [1-AMP] Index Access   │
│ [TABLE:orders via UPI] │
│ Filter: order_id = 123 │
│ Est: 1 row ●●●●●       │
└───────┬────────────────┘
        ▼
   [RESULT: 1 row]
```

### Pattern 3: Two-Table Join
```
[TABLE:A]          [TABLE:B]
 10K rows          5K rows
    │                 │
    ▼                 ▼
[SPOOL:1]         [SPOOL:2]
    │                 │
    └────┬────────────┘
         ▼
    [HASH-JOIN]
     45K rows
         │
         ▼
     [RESULT]
```

### Pattern 4: Aggregation with Grouping
```
[TABLE:sales]
  1M rows
     │
     ▼
 [REDIST by region]
     │
     ▼
  [SUM AGG]
  Group by: region
     │
     ▼
  [RESULT]
   50 rows
```

---

## Troubleshooting Visual Clues

### Red Flags to Highlight

🔴 **Product Joins**
- Indicates missing join condition
- Can cause massive row explosion
- Mark with warning icon

🔴 **No Confidence Spools**
- Missing or stale statistics
- Unreliable cardinality estimates
- Mark with ⚠️ warning

🔴 **All-AMP on Large Tables**
- Full table scans on big tables
- Consider index usage
- Mark with 🔥 for high cost

🔴 **Large Redistributions**
- Significant network overhead
- Consider PI redesign
- Show network time prominently

🔴 **Skew Warnings**
- Uneven AMP distribution
- Performance bottleneck
- Add skew indicator ⚖️

---

## Implementation Tips

### For AI Agents Creating Visualizations

1. **Parse systematically**: Extract step-by-step
2. **Validate completeness**: Ensure all steps are captured
3. **Classify accurately**: Use correct icon for each operation
4. **Preserve relationships**: Maintain data flow dependencies
5. **Highlight insights**: Surface optimization opportunities
6. **Provide context**: Include summary and recommendations
7. **Make it actionable**: Flag specific issues with solutions

### Best Practices

- **Keep it readable**: Don't overcrowd the diagram
- **Use consistent symbols**: Stick to established icons
- **Layer information**: Basic view + detailed drill-down
- **Include legend**: Explain all symbols and colors
- **Add timestamps**: When EXPLAIN was generated
- **Show alternatives**: Suggest optimization options

---

## Conclusion

This template provides a comprehensive framework for visualizing Teradata EXPLAIN output. The goal is to transform complex text-based query plans into intuitive visual representations that enable faster analysis and better optimization decisions.

**Key Principles:**
- **Clarity over completeness** - Show what matters most
- **Consistency in representation** - Use standard icons and patterns
- **Actionable insights** - Highlight issues and opportunities
- **Multi-level detail** - Support both overview and deep-dive analysis

Use this template as a foundation and adapt it based on specific analysis needs and user preferences.
