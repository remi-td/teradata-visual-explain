# Teradata SQL EXPLAIN: Comprehensive Training Guide

## Document Overview
**Purpose**: This document provides comprehensive training on the Teradata EXPLAIN operator for analyzing SQL query execution plans and optimizing database performance.

**Target Audience**: AI agents, database administrators, SQL developers, and query optimization specialists.

**Source**: Based on Teradata Vantage™ SQL Request and Transaction Processing Documentation (Release 17.10, July 2021)

---

## Table of Contents
1. [Introduction to EXPLAIN](#introduction-to-explain)
2. [Core Concepts](#core-concepts)
3. [EXPLAIN Confidence Levels](#explain-confidence-levels)
4. [EXPLAIN Terminology](#explain-terminology)
5. [Using EXPLAIN Effectively](#using-explain-effectively)
6. [EXPLAIN Types](#explain-types)
7. [Practical Examples](#practical-examples)
8. [Best Practices](#best-practices)

---

## 1. Introduction to EXPLAIN

### What is EXPLAIN?

The EXPLAIN request modifier is a powerful diagnostic tool that returns a summary of the static, step-by-step Optimizer plan for processing an SQL request. When prepended to any valid Teradata SQL statement, EXPLAIN provides:

- **Execution steps** the Optimizer would take to process the request
- **Estimated time** required to complete the request
- **Index usage** information
- **Join types** and strategies
- **Spool generation** details
- **Parallel execution** information

### Key Characteristics

**Format**: Prepend EXPLAIN to any SQL statement
```sql
EXPLAIN SELECT * FROM employee WHERE dept_no = 100;
```

**Output**: Text-based report describing the query execution plan

**Important Notes**:
- The actual execution plan may differ from EXPLAIN output if system conditions change (statistics, demographics, IPE dynamic planning)
- Times are reported in arbitrary units for **relative comparison**, not actual clock time
- EXPLAIN can only be used with SQL requests, not with stored procedures, functions, or methods
- Keep EXPLAIN results for documentation and troubleshooting after system upgrades

### What EXPLAIN Can Reveal

1. **Indexes**: Which indexes will be used (primary, secondary, unique, non-unique)
2. **Access Methods**: Full table scans, index scans, hash joins, merge joins, nested joins
3. **Spool Files**: Intermediate storage and data movement
4. **Join Strategies**: Product joins, hash joins, merge joins, exclusion joins
5. **Parallelism**: Whether steps execute sequentially or in parallel
6. **Row Estimates**: Expected number of rows at each step
7. **Confidence Levels**: How confident the Optimizer is about cardinality estimates
8. **Partition Access**: Row and column partition elimination

---

## 2. Core Concepts

### 2.1 Query Processing Flow

Understanding how Teradata processes queries helps interpret EXPLAIN output:

```
SQL Request → Parser → Resolver → Optimizer → Generator → Dispatcher → AMPs
```

**Key Components**:
- **Parser**: Checks syntax and converts SQL to internal representation
- **Resolver**: Resolves object names and checks security
- **Optimizer**: Creates the most efficient execution plan based on:
  - Statistics on tables and indexes
  - System configuration
  - Resource availability
  - Query complexity
- **Generator**: Converts logical plan to executable steps
- **Dispatcher**: Coordinates execution across AMPs
- **AMPs**: Execute the actual data operations in parallel

### 2.2 Execution Plans

**Static Plans**: Traditional query plans generated at parse time
- Used for most queries
- Based on current statistics
- May not adapt to runtime conditions

**Dynamic Plans**: Plans generated through Incremental Planning and Execution (IPE)
- Used for complex queries eligible for IPE
- Can adapt based on intermediate results
- More flexible but require runtime planning overhead

### 2.3 Spool Files

Spool files are temporary storage areas used during query processing:

**Types**:
- **Spool 1**: Typically holds final results returned to user
- **Spool 2, 3, etc.**: Hold intermediate results
- **(all_amps)**: Spool built on all AMPs
- **(group_amps)**: Spool built on a subset of AMPs
- **(one_amp)**: Spool built on a single AMP

**Common Spool Operations**:
- Redistributing data by hash for joins
- Duplicating small tables across all AMPs
- Sorting data
- Eliminating duplicate rows
- Storing intermediate aggregations

---

## 3. EXPLAIN Confidence Levels

The Optimizer assigns confidence levels to cardinality estimates based on available statistics and query complexity.

### 3.1 Confidence Level Types

| Confidence Level | Meaning | When Assigned |
|-----------------|---------|---------------|
| **No Confidence** | Optimizer has neither low nor high confidence in estimates | - No statistics collected<br>- Nondeterministic expressions<br>- Complex conditions statistics can't handle<br>- Unindexed columns with no statistics |
| **Low Confidence** | Moderate certainty in estimates | - Index exists but no statistics collected<br>- Statistics with ANDed unindexed columns<br>- Statistics with ORed conditions<br>- Partial grouping column statistics<br>- Single-AMP dynamic samples |
| **High Confidence** | High certainty in estimates | - Statistics on predicate columns (no skew)<br>- Covering multicolumn statistics<br>- Primary index with 5+ AMPs sampled<br>- Full grouping column statistics<br>- Constants or equality predicates on grouping columns |
| **Index Join** | High certainty due to index constraints | - Unique index on join columns<br>- Foreign key relationship between tables<br>- Uniqueness constraints |

### 3.2 Impact of Confidence Levels

**Low confidence** → Optimizer uses conservative strategies
- May choose less optimal join methods
- Estimates may be significantly off

**High confidence** → Optimizer uses aggressive optimization
- Better join planning
- More accurate resource allocation

**Critical Point**: Even with high confidence, **stale statistics** can lead to poor plans. Fresh statistics are essential for optimal performance.

### 3.3 Confidence Levels in Joins

For join operations:
- Confidence level **never exceeds** the lower of the two input relations
- A join with "no confidence" on one side = overall "no confidence"
- Join cardinality estimates have lower confidence than single-table retrievals

**Example**:
```
Left table: High Confidence
Right table: Low Confidence
→ Join: Low Confidence
```

---

## 4. EXPLAIN Terminology

Understanding EXPLAIN output requires familiarity with its specialized terminology. Here are the most important terms:

### 4.1 Access Methods

#### **all-rows scan**
- All rows in table are scanned row by row
- No index is used for access
- Most resource-intensive method

#### **by way of the primary index**
- Conditions on primary index columns allow direct AMP access
- Most efficient single-row access method
- Example: `WHERE primary_key_col = value`

#### **by way of the unique primary index**
- Direct access to at most one row via unique primary index
- Single AMP access
- Highest performance for single-row retrievals

#### **by way of index # n**
- Secondary index (NUSI or USI) used for access
- NUSI: May access multiple rows across AMPs
- USI: Two-AMP retrieve (index subtable + base table)

### 4.2 Join Methods

#### **product join**
- Cartesian product or nested loop join
- Small table duplicated to all AMPs
- Good when one table is very small
- WARNING: Can be expensive if both tables are large

#### **merge join**
- Both tables sorted by join column hash
- Rows matched via sort-merge algorithm
- Efficient for large tables with equality joins
- Requires redistribution if not already distributed by join column

#### **hash join**
- One table hashed into memory
- Other table probed against hash table
- Good for large tables
- May partition data into multiple hash join partitions

#### **nested join**
- Row from first table used to directly access second table via index
- Efficient when accessing via unique index
- Often used with primary index or unique secondary index

#### **exclusion merge join**
- Used for NOT IN, NOT EXISTS predicates
- Identifies rows that don't match
- Requires both inputs sorted

#### **inclusion merge join**
- Used for IN, EXISTS predicates
- Identifies rows that match
- Requires both inputs sorted

#### **rowkey-based merge join**
- Join on both partitioning and primary index columns
- Much faster for partitioned tables
- Each partition joins with at most one other partition

### 4.3 Redistribution and Duplication

#### **redistributed by hash code to all AMPs**
- Rows sent to AMPs based on join column hash
- Ensures matching rows land on same AMP
- Prepares for hash or merge join

#### **duplicated on all AMPs**
- Small table copied to every AMP
- Enables local join processing
- Eliminates need for redistribution
- Used when duplication is cheaper than redistribution

#### **redistributed by hash code to few AMPs**
- Redistribution to subset of AMPs
- Used when source is single-AMP and targets few AMPs
- More efficient than full redistribution

### 4.4 Aggregation

#### **aggregate results are computed globally**
- Full ARSA (Aggregate, Redistribute, Sort, Aggregate) process:
  1. Aggregate locally on each AMP
  2. Redistribute to target AMPs
  3. Sort redistributed rows
  4. Final global aggregation

#### **aggregate results are computed locally**
- Only local aggregation performed
- No redistribution needed
- Used when GROUP BY includes primary index

#### **by skipping global aggregation**
- Avoids redistribution and final aggregation
- Cost-based optimization decision

#### **by using cache during local aggregation**
- Output rows cached in memory
- Duplicates coalesced in cache
- Avoids sorting large input sets

### 4.5 Partition Operations

#### **n partitions of ...**
- Only n row partitions accessed (n > 1)
- Indicates row partition elimination

#### **a single partition of ...**
- Only one row partition accessed
- Best case for partitioned table queries

#### **all partitions of ...**
- All row partitions accessed
- No partition elimination possible

#### **m1 column partitions of ...**
- Accessing m1 column partitions
- Indicates column partition elimination

#### **using CP merge Spool**
- Column-Partitioned merge spool used
- Merges column partitions when contexts insufficient
- May impact performance if many merges required

### 4.6 Locking

#### **lock ... for read**
- READ lock acquired
- Allows concurrent reads
- Prevents writes during query

#### **lock ... for write**
- WRITE lock acquired
- Prevents concurrent reads/writes
- Required for DML operations

#### **lock ... for exclusive use**
- EXCLUSIVE lock acquired
- No concurrent access allowed
- Used for DDL operations

#### **on a reserved RowHash to prevent global deadlock**
- Proxy lock for deadlock prevention
- Acquired before actual locks
- Part of Teradata's deadlock prevention strategy

### 4.7 Confidence and Estimation

#### **estimated with [confidence level] to be n rows**
- Cardinality estimate with confidence qualifier
- Critical for understanding plan quality
- Check if estimates match actual data

#### **no residual conditions**
- All WHERE conditions already applied
- Rows selected meet all criteria

#### **with a condition of ...**
- Conditional expression still to be evaluated
- Additional filtering after initial access

#### **post join condition of ...**
- Condition evaluated after join
- Used for:
  - CASE expressions
  - NULL-sensitive predicates on outer join inner table
  - Complex OR terms with outer joins
  - NOT IN / NOT EXISTS predicates

---

## 5. Using EXPLAIN Effectively

### 5.1 Basic EXPLAIN Workflow

1. **Prepare**
   - Collect current statistics on relevant tables
   - Understand table structures and indexes
   - Note row counts and data distribution

2. **Run EXPLAIN**
   ```sql
   EXPLAIN SELECT ...;
   ```

3. **Analyze Output**
   - Check confidence levels
   - Identify expensive operations (high estimated time)
   - Look for full table scans
   - Check spool sizes
   - Verify index usage

4. **Identify Issues**
   - Missing indexes
   - Stale statistics
   - Poor join orders
   - Excessive redistribution
   - Large intermediate results

5. **Optimize**
   - Add appropriate indexes
   - Collect/update statistics
   - Rewrite query
   - Consider table redesign

### 5.2 Reading EXPLAIN Output

EXPLAIN output follows a step-by-step format:

```
Explanation
-------------------------------------------------------
1) First, we lock [table] for [read/write]...
2) Next, we lock [table] for [read/write]...
3) We do an all-AMPs RETRIEVE step from [table]...
   ...
-> The contents of Spool 1 are sent back to the user...
   The total estimated time is X.XX seconds.
```

**Reading Steps**:
- Steps execute in order (unless marked as parallel)
- Each step builds on previous steps
- Final step returns results to user

**Parallel Execution**:
```
3) We execute the following steps in parallel.
   1) Step A
   2) Step B
```
Substeps execute concurrently.

### 5.3 Key Metrics to Monitor

#### **Confidence Levels**
- **Goal**: High or Index Join confidence
- **Action if Low/No**: Collect statistics

#### **Estimated Time**
- **Compare** different query formulations
- **Focus on** steps with highest time
- **Remember**: Relative units, not clock time

#### **Spool Sizes**
- **Watch for**: Unexpectedly large spools
- **Large spools** indicate:
  - Missing join conditions
  - Cartesian products
  - Poor selectivity

#### **Access Methods**
- **Best**: Unique index access
- **Good**: Index scans, primary index access
- **Review**: Full table scans on large tables

#### **Join Methods**
- **Nested joins**: Good for small result sets
- **Hash/Merge joins**: Good for large tables
- **Product joins**: Review carefully (potential Cartesian product)

---

## 6. EXPLAIN Types

### 6.1 STATIC EXPLAIN

**Default EXPLAIN behavior** - generates a static plan at parse time.

**Syntax**:
```sql
EXPLAIN SELECT ...;
-- OR explicitly
STATIC EXPLAIN SELECT ...;
```

**Characteristics**:
- Plan generated before execution
- Based on current statistics
- Does not adapt to runtime conditions
- Used for most queries

**When to Use**:
- Understanding query structure
- Comparing query alternatives
- Initial performance analysis
- Documentation

**Example Output Prefix**:
```
This request is eligible for incremental planning and execution (IPE).
The following is the static plan for the request.
```

### 6.2 DYNAMIC EXPLAIN

**For queries eligible for Incremental Planning and Execution (IPE)**

**Syntax**:
```sql
DYNAMIC EXPLAIN SELECT ...;
```

**Characteristics**:
- Shows plan that would be generated with IPE
- Plan can adapt based on intermediate results
- Used for complex queries (typically with subqueries)
- May show multiple plan fragments

**When to Use**:
- Complex queries with subqueries
- Queries where intermediate result sizes affect planning
- Understanding adaptive optimization

**IPE Eligibility**:
Queries eligible for IPE typically have:
- Correlated subqueries
- Multiple levels of nesting
- Complex predicates
- Significant uncertainty in cardinality estimates

**Example Output Prefix**:
```
The following is the dynamic explain for the request.
```

**Plan Fragments**:
Dynamic plans may include:
```
We send an END PLAN FRAGMENT step for plan fragment n.
```
Each fragment is a separately optimized portion of the query.

### 6.3 Comparing STATIC vs DYNAMIC

| Aspect | STATIC EXPLAIN | DYNAMIC EXPLAIN |
|--------|----------------|-----------------|
| **Planning Time** | Parse time | Runtime (incremental) |
| **Adaptability** | Fixed plan | Adapts to intermediate results |
| **Typical Queries** | Simple to moderate | Complex with subqueries |
| **Overhead** | Minimal | Additional planning cost |
| **Accuracy** | Depends on statistics | Can be more accurate |
| **Use Case** | Most queries | Complex analytical queries |

### 6.4 EXPLAIN with USING Modifier

**Parameterized Queries**:
```sql
EXPLAIN USING (a INTEGER) 
SELECT * FROM table WHERE col = :a;
```

**Parameter Peeking**:
If parameter values provided, Optimizer may "peek" at values:
- Better cardinality estimates
- More accurate partition elimination
- Plan may vary based on actual values

**Note**: Cannot EXPLAIN parameterized requests with certain CLI data parcel flavors (3 and 71).

---

## 7. Practical Examples

### 7.1 Simple SELECT with Primary Index

**Query**:
```sql
EXPLAIN 
SELECT name, dept_no 
FROM employee 
WHERE empno = 10009;
```

**EXPLAIN Output**:
```
1) First, we do a single-AMP RETRIEVE step from Personnel.Employee 
   by way of the unique primary index 
   "PERSONNEL.Employee.EmpNo = 10009" with no residual conditions. 
   The estimated time for this step is 0.04 seconds.

-> The row is sent directly back to the user as the result of 
   statement 1. The total estimated time is 0.04 seconds.
```

**Analysis**:
- ✅ **Best case**: Unique primary index access
- ✅ Single AMP operation
- ✅ No spool files needed
- ✅ Direct return to user
- **Estimated time**: 0.04 (relative units)

### 7.2 Full Table Scan

**Query**:
```sql
EXPLAIN 
SELECT COUNT(*) 
FROM employee 
WHERE dept_no = 500 
  AND salary > 25000 
  AND yrs_exp >= 3 
  AND ed_lev >= 12;
```

**Key EXPLAIN Phrases**:
```
3) We do an all-AMPs SUM step to aggregate from PERSONNEL.Employee 
   by way of an all-rows scan with a condition of...
```

**Analysis**:
- ⚠️ **Full table scan** on all AMPs
- Multiple conditions in WHERE clause
- SUM aggregation (COUNT)
- **Improvement**: Consider indexes on frequently-queried columns

### 7.3 Secondary Index Usage

**Query**:
```sql
EXPLAIN 
SELECT emp_no, dept_no 
FROM employee 
WHERE name = 'Smith T';
```

**Key EXPLAIN Phrases**:
```
3) We do an all-AMPs RETRIEVE step from PERSONNEL.employee 
   by way of index # 4 "PERSONNEL.employee.Name = 'Smith T'" 
   with no residual conditions into Spool 1 (group_amps)...
```

**Analysis**:
- ✅ NUSI used for access
- ⚠️ Builds spool file
- Results in Spool 1
- **Low confidence** on cardinality estimate
- **Action**: Collect statistics on name column

### 7.4 Join Examples

#### **Product Join**

**Query**:
```sql
EXPLAIN 
SELECT Hours, EmpNo, Description 
FROM Charges, Project 
WHERE Charges.Proj_Id = 'ENG-0003' 
  AND Project.Proj_Id = 'ENG-0003' 
  AND Charges.WkEnd > Project.DueDate;
```

**Key EXPLAIN Phrases**:
```
2) We do a single-AMP RETRIEVE step from PERSONNEL.Project by way of 
   the unique primary index... into Spool 2, which is duplicated 
   on all AMPs.

3) We do an all AMPs JOIN step from Spool 2 (Last Use)... which is 
   joined to PERSONNEL.Charges... using a product join...
```

**Analysis**:
- Small Project table (1 row) duplicated to all AMPs
- Product join used (acceptable here due to small size)
- Efficient despite being a product join

#### **Merge Join**

**Query**:
```sql
EXPLAIN 
SELECT Name, DeptName, Loc 
FROM Employee, Department 
WHERE Employee.DeptNo = Department.DeptNo;
```

**Key EXPLAIN Phrases**:
```
2) We do an all-AMPs RETRIEVE step from PERSONNEL.Employee... into 
   Spool 2, which is redistributed by hash code to all AMPs. Then we 
   do a SORT to order Spool 2 by row hash.

3) We do an all-AMPs JOIN step from PERSONNEL.Department by way of a 
   RowHash match scan... joined to Spool 2 (Last Use). 
   PERSONNEL.Department and Spool 2 are joined using a merge join...
```

**Analysis**:
- Employee redistributed by dept_no hash
- Sorted by row hash for merge join
- Merge join performed (efficient for large tables)
- **Good plan** for joining two large tables

#### **Hash Join**

**Query**:
```sql
EXPLAIN 
SELECT Employee.EmpNum, Department.DeptName, Employee.Salary 
FROM Employee, Department 
WHERE Employee.Location = Department.Location;
```

**Key EXPLAIN Phrases**:
```
4) We do an all-AMPs RETRIEVE step from PERSONNEL.Employee... into 
   Spool 2 fanned out into 22 hash join partitions, which is 
   redistributed by hash code to all AMPs.

5) We do an all-AMPs RETRIEVE step from PERSONNEL.Department... into 
   Spool 3 fanned out into 22 hash join partitions...

6) We do an all-AMPs JOIN step from Spool 2... joined to Spool 3. 
   Spool 2 and Spool 3 are joined using a hash join of 22 partitions...
```

**Analysis**:
- Both tables redistributed by Location
- Hash join with 22 partitions
- Large result set expected
- **Consideration**: Is join condition correct? (Location seems unusual)

### 7.5 Bit Mapping for Multiple Indexes

**Query**:
```sql
EXPLAIN 
SELECT COUNT(*) 
FROM main 
WHERE num_a = '101' 
  AND num_b = '02' 
  AND kind = 'B' 
  AND event = '001';
```

**Key EXPLAIN Phrases**:
```
2) We do a BMSMS (bit map set manipulation) step that intersects 
   the following row id bit maps:
   1) The bit map built for TESTING.Main by way of index # 12...
   2) The bit map built for TESTING.Main by way of index # 8...
   3) The bit map built for TESTING.Main by way of index # 16...
   The resulting bit map is placed in Spool 3.

3) We do a SUM step to aggregate from TESTING.Main by way of 
   index # 4... and the bit map in Spool 3 (Last Use)...
```

**Analysis**:
- ✅ Multiple NUSIs used efficiently
- Bit maps intersected (AND operation)
- Final access via bit map + another index
- **Good**: Multiple index usage optimized

### 7.6 Partitioned Table Access

**Query on Row-Partitioned Table**:
```sql
EXPLAIN 
SELECT * 
FROM t1 
WHERE b > 2;

-- Table: CREATE TABLE t1 (a INTEGER, b INTEGER)
--        PRIMARY INDEX(a)
--        PARTITION BY RANGE_N(b BETWEEN 1 AND 10 EACH 1);
```

**Key EXPLAIN Phrases**:
```
3) We do an all-AMPs RETRIEVE step from 8 partitions of mws.t1 
   with a condition of ("mws.t1.b >= 3") into Spool 1...
```

**Analysis**:
- ✅ **Row partition elimination**: Only 8 of 10 partitions accessed
- Condition b > 2 eliminates partitions 1 and 2
- **Efficient**: Skips unnecessary partitions

**Single Partition Access**:
```sql
EXPLAIN 
SELECT * 
FROM t1 
WHERE b = 1;
```

**Key EXPLAIN Phrases**:
```
3) We do an all-AMPs RETRIEVE step from a single partition of mws.t1 
   with a condition of ("mws.t1.b = 1") into Spool 1...
```

**Analysis**:
- ✅ **Optimal**: Only one partition accessed
- Best case for partitioned table queries

---

## 8. Best Practices

### 8.1 Statistics Management

**Critical for Accurate EXPLAIN**:

1. **Collect Statistics Regularly**
   ```sql
   COLLECT STATISTICS ON table_name COLUMN column_name;
   COLLECT STATISTICS ON table_name INDEX index_name;
   ```

2. **Focus on Key Columns**:
   - Primary index columns (especially NUPI)
   - Join columns
   - Frequently filtered columns
   - Columns with skewed data
   - Partitioning columns

3. **Update After Significant Changes**:
   - After bulk loads (>10% data change)
   - After major deletions
   - When data demographics change
   - After database upgrades

4. **Monitor Confidence Levels**:
   - Low/No confidence → Need statistics
   - High confidence → Statistics current and useful

### 8.2 Index Strategy

**Based on EXPLAIN Analysis**:

1. **Primary Indexes**:
   - Choose based on most common access patterns
   - Consider UPI for unique access
   - Consider NUPI for even distribution

2. **Secondary Indexes**:
   - Add USI for unique lookups
   - Add NUSI for frequently filtered non-PI columns
   - **Cost**: Indexes require maintenance overhead
   - **Verify**: Check EXPLAIN to confirm index usage

3. **Join Indexes**:
   - Consider for complex multi-table joins
   - Pre-join frequently joined tables
   - Store only needed columns

4. **Avoid**:
   - Indexes rarely used
   - Indexes on low-cardinality columns (unless strategic)
   - Too many indexes (maintenance overhead)

### 8.3 Query Optimization Patterns

**Common Issues and Solutions**:

#### **Problem: Full Table Scan on Large Table**
**Solution**:
- Add index on filter columns
- Collect statistics
- Verify WHERE conditions are sargable (optimizable)

#### **Problem: Cartesian Product**
**Solution**:
- Check join conditions
- Ensure all tables properly joined
- Verify no missing ON clauses

#### **Problem: Large Spool Files**
**Solution**:
- Add WHERE filters earlier in query
- Break complex query into steps
- Consider temporary tables for intermediate results

#### **Problem: Low Confidence Estimates**
**Solution**:
- Collect statistics on relevant columns
- Update stale statistics
- Collect multicolumn statistics for correlated columns

#### **Problem: Excessive Redistribution**
**Solution**:
- Consider table redesign with better primary index
- Use appropriate join strategy
- Consider data duplication for small tables

### 8.4 EXPLAIN Checklist

**Before Optimization**:
- [ ] Statistics current on all accessed tables
- [ ] Indexes defined on join and filter columns
- [ ] Table structures appropriate for access patterns
- [ ] Baseline EXPLAIN captured

**During Analysis**:
- [ ] Check all confidence levels (aim for High)
- [ ] Identify most expensive steps (highest estimated time)
- [ ] Verify appropriate access methods used
- [ ] Check for unexpected full table scans
- [ ] Review spool sizes (watch for unexpectedly large)
- [ ] Confirm efficient join methods
- [ ] Look for partition elimination
- [ ] Check for parallelism opportunities

**After Changes**:
- [ ] Re-run EXPLAIN
- [ ] Compare estimated times
- [ ] Verify confidence levels improved
- [ ] Confirm expected indexes used
- [ ] Test with actual execution
- [ ] Monitor production performance

### 8.5 Common Pitfalls to Avoid

1. **Ignoring Confidence Levels**
   - Low/No confidence = unreliable estimates
   - Can lead to poor plan choices

2. **Over-Relying on Estimated Times**
   - Times are relative, not absolute
   - Use for comparison, not prediction

3. **Not Collecting Statistics**
   - Most common cause of poor plans
   - Statistics are essential for optimization

4. **Ignoring Actual Execution**
   - EXPLAIN shows planned behavior
   - Actual execution may differ
   - Use DBQL to capture actual query plans

5. **Not Considering Data Distribution**
   - Skewed data requires special handling
   - Statistics help identify skew
   - May need different optimization strategies

6. **Focusing Only on Query**
   - Table design matters
   - Primary index choice is critical
   - Sometimes redesign is better than rewriting

### 8.6 Advanced Tips

**1. Compare Alternative Formulations**
```sql
-- Option A
EXPLAIN SELECT ... WHERE col IN (SELECT ...);

-- Option B
EXPLAIN SELECT ... WHERE EXISTS (SELECT ...);

-- Option C
EXPLAIN SELECT ... JOIN ...;
```
Choose the one with best estimated time and confidence.

**2. Use Volatile Tables for Complex Queries**
```sql
CREATE VOLATILE TABLE temp_results AS (
    SELECT ... -- Complex subquery
) WITH DATA;

COLLECT STATISTICS ON temp_results ...;

EXPLAIN SELECT * FROM temp_results WHERE ...;
```

**3. Leverage EXPLAIN for Documentation**
- Save EXPLAIN output with query definitions
- Include in code reviews
- Track plan changes over time
- Help troubleshoot production issues

**4. Monitor for Plan Changes**
- Plans can change with:
  - New statistics
  - Database upgrades
  - System configuration changes
  - Data growth
- Periodically review critical queries

**5. Use EXPLAIN with Different Session Modes**
- ANSI vs Teradata mode affects transactions
- Compare EXPLAIN output in each mode
- Understand transaction boundaries

---

## Appendix A: EXPLAIN Phrase Quick Reference

### Access Methods
- `all-rows scan` - Full table scan
- `by way of the primary index` - Primary index access
- `by way of the unique primary index` - Unique PI access  
- `by way of index # n` - Secondary index access

### Join Types
- `product join` - Cartesian product/nested loop
- `merge join` - Sort-merge join
- `hash join` - Hash join
- `nested join` - Index-based nested loop
- `exclusion merge join` - Anti-join (NOT IN)
- `inclusion merge join` - Semi-join (IN/EXISTS)

### Data Movement
- `redistributed by hash code` - Hash redistribution
- `duplicated on all AMPs` - Broadcast
- `redistributed to few AMPs` - Partial redistribution

### Aggregation
- `aggregate globally` - Full ARSA process
- `aggregate locally` - Local aggregation only
- `by skipping global aggregation` - Optimization
- `by using cache during local aggregation` - Cache optimization

### Confidence
- `estimated with no confidence` - No statistics
- `estimated with low confidence` - Partial statistics
- `estimated with high confidence` - Good statistics
- `estimated with index join confidence` - Index constraints

### Partitioning
- `n partitions of` - Multiple partitions accessed
- `a single partition of` - One partition accessed
- `all partitions of` - All partitions accessed
- `m1 column partitions of` - Column partitions accessed

---

## Appendix B: Optimization Decision Tree

```
Query Performance Issue
│
├─ EXPLAIN shows "No Confidence"?
│  └─ YES → Collect statistics on accessed columns
│  
├─ EXPLAIN shows "all-rows scan" on large table?
│  └─ YES → Check if index exists on WHERE columns
│      ├─ No index → Consider adding index
│      └─ Index exists → Verify statistics current
│
├─ EXPLAIN shows "product join"?
│  └─ YES → Verify join conditions present
│      └─ Check if one table is small (duplication OK)
│
├─ EXPLAIN shows large spool files?
│  └─ YES → Check for:
│      ├─ Missing join conditions (Cartesian product)
│      ├─ Inefficient WHERE filters
│      └─ Need for query restructuring
│
├─ EXPLAIN shows excessive redistributions?
│  └─ YES → Consider:
│      ├─ Table redesign (primary index choice)
│      └─ Query restructuring
│
└─ EXPLAIN shows long estimated time?
   └─ Compare alternative query formulations
```

---

## Appendix C: Teradata-Specific Optimizations

### 1. Partitioned Primary Index (PPI)
- Enables row partition elimination
- Look for "n partitions of" in EXPLAIN
- Best for range queries on partition column

### 2. Column Partitioning
- Reduces I/O for column-subset queries  
- Look for "m1 column partitions of" in EXPLAIN
- Ideal for wide tables with selective column access

### 3. Join Indexes
- Pre-computed join results
- Can dramatically improve multi-table queries
- Verify usage in EXPLAIN output

### 4. Dynamic AMP Sampling
- Automatic statistics sampling
- Provides Low confidence estimates
- Better than no statistics
- Collect full statistics for better plans

### 5. Derived Statistics
- System can derive statistics from constraints
- Foreign key relationships
- Unique constraints
- Check constraints

---

## Conclusion

Mastering EXPLAIN is essential for:
- Understanding query execution
- Identifying performance bottlenecks
- Validating optimization strategies
- Troubleshooting production issues

**Key Takeaways**:
1. **Always collect statistics** on accessed columns and indexes
2. **Check confidence levels** - they indicate plan reliability
3. **Compare alternatives** - use EXPLAIN to evaluate options
4. **Monitor over time** - plans change with data and statistics
5. **Combine with execution** - EXPLAIN predicts, DBQL confirms

**Remember**: EXPLAIN is a predictive tool. Always validate optimization with actual execution metrics.

---

**Document Version**: 1.0  
**Based on**: Teradata Vantage™ SQL Request and Transaction Processing, Release 17.10 (July 2021)  
**Last Updated**: January 2026
