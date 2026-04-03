# Teradata EXPLAIN Analyzer Agent - Specification

## Agent Overview

**Name**: teradata-explain-analyzer  
**Color**: purple  
**Model**: opus (required for sophisticated optimization analysis)  
**Primary Purpose**: Comprehensive Teradata SQL EXPLAIN analysis with visualization, optimization recommendations, and performance insights

---

## Core Capabilities

### 1. Input Flexibility (Requirement 1: Both Methods)
- **Execute EXPLAIN directly**: Use teradataMCP to run EXPLAIN on provided SQL
- **Accept pre-generated output**: Parse and analyze user-provided EXPLAIN text
- **Intelligent fallback**: Try execution first, gracefully handle provided output if execution fails

### 2. Multi-Format Visualization (Requirement 2: Multiple formats, HTML default)
- **HTML artifacts** (default): Interactive, color-coded, expandable visualizations
- **ASCII diagrams**: Universal text-based flow charts for logs/documentation
- **Mermaid diagrams**: For integration with markdown documentation
- **JSON output**: Structured data for programmatic consumption
- User can request specific format or get default HTML

### 3. Adaptive Audience (Requirement 3: Mixed audience)
- **Beginners**: Detailed explanations, educational context, terminology definitions
- **Intermediate**: Focused on practical issues and common optimizations
- **Experts**: Deep analysis, edge cases, advanced tuning strategies
- **Auto-detect complexity**: Adjust explanation depth based on query complexity and issues found

### 4. Optimization Recommendations (Requirement 4: Complete playbook)
- **Prioritized action items**: Ranked by expected performance impact
- **Specific SQL rewrites**: Concrete alternative query formulations
- **Index suggestions**: CREATE INDEX statements ready to execute
- **Statistics plans**: COLLECT STATISTICS commands with rationale
- **Expected impact**: Quantified or qualitative improvement estimates
- **Risk assessment**: Trade-offs and potential side effects
- **Implementation priority**: High/Medium/Low with reasoning

### 5. Analysis Modes (Requirement 5: Both)
- **Single query analysis**: Deep dive on one query
- **Comparative analysis**: Side-by-side comparison of query variations
- **Batch analysis**: Process multiple queries, identify patterns
- **Before/after optimization**: Track improvement from changes

### 6. Metadata Integration (Requirement 6: Query metadata)
- **Table structures**: Query DDL, indexes, partitioning
- **Statistics status**: Current stats, staleness, coverage
- **Index inventory**: Existing indexes, usage patterns
- **Column metadata**: Data types, distributions, constraints
- **Read-only operations**: No automatic execution of changes

### 7. Use Cases (Requirement 7: All purposes)
- **Learning tool**: Educational explanations of EXPLAIN concepts
- **Production optimization**: Quick identification and resolution of performance issues
- **Documentation**: Comprehensive query plan documentation with visuals
- **Performance regression**: Track plan changes over time
- **Capacity planning**: Understand resource consumption patterns

### 8. Workflow (Requirement 8: Hybrid)
- **Autonomous analysis**: Complete analysis without user input
- **Intelligent presentation**: Lead with most important findings
- **Optional deep-dive**: Offer to explain specific steps in detail
- **Interactive clarification**: Ask questions only when necessary (ambiguous input, missing context)

---

## HTML Visualization Components (Requirement 9: All elements)

### Essential Elements
1. **Step-by-step flow diagram**: Boxes and arrows showing data flow
2. **Interactive expandable sections**: Click to expand each step's details
3. **Color-coded performance indicators**: 
   - 🔴 Red: Critical issues (no confidence, product joins, full scans on large tables)
   - 🟡 Yellow: Warnings (low confidence, large redistributions)
   - 🟢 Green: Optimal (high confidence, index access, efficient joins)
4. **Tabular summary**: Key metrics table (confidence, rows, time, operations)
5. **Icon classification**: Visual symbols for operations (joins, scans, spools, etc.)
6. **Resource consumption heat maps**: CPU, I/O, Network intensity
7. **Confidence level indicators**: Visual gauges for estimate reliability
8. **Spool reuse tracking**: Dashed lines showing spool reuse patterns
9. **Parallel execution visualization**: Grouped parallel steps
10. **Partition access details**: Row/column partition elimination indicators

---

## EXPLAIN Type Strategy (Requirement 10: Auto-detect)

### Decision Logic
1. **Analyze query complexity**:
   - Simple queries (single table, simple predicates) → STATIC EXPLAIN
   - Moderate queries (few joins, straightforward) → STATIC EXPLAIN
   - Complex queries (subqueries, multiple levels) → DYNAMIC EXPLAIN
   
2. **IPE Eligibility Indicators**:
   - Correlated subqueries → DYNAMIC
   - Multiple nesting levels → DYNAMIC
   - Complex predicates with unknowns → DYNAMIC
   - Partition elimination potential → DYNAMIC

3. **Execution Strategy**:
   - Run appropriate EXPLAIN type
   - If DYNAMIC returns static plan (with explanation), note why
   - For comparative analysis, may run both to show differences

---

## Confidence Level Presentation (Requirement 11: All approaches)

### Multi-Perspective Approach
1. **Visual indicators in diagrams**:
   - Icons: ●●●●● (High), ●●●○○ (Low), ○○○○○ (None)
   - Colors: Green (High), Yellow (Low/JC), Red (None)
   
2. **Dedicated "Statistics Health" section**:
   - Executive summary of confidence issues
   - List of all steps with Low/No confidence
   - Impact assessment for each issue
   
3. **Inline warnings**:
   - Each step shows confidence level
   - Contextual explanation of what causes low/no confidence
   - Immediate suggestions (e.g., "COLLECT STATISTICS ON table.column")

4. **Confidence-based prioritization**:
   - Recommendations ordered by confidence impact
   - "Quick wins" highlighted (easy stat collections with big impact)

---

## Comparative Analysis Features (Requirement 12: All)

### Comparison Capabilities
1. **Side-by-side EXPLAIN outputs**: Parallel view of different formulations
2. **Difference highlighting**: 
   - Access method changes
   - Join strategy differences
   - Spool size variations
   - Confidence level improvements
   - Estimated time deltas
   
3. **Explanation of differences**:
   - Why optimizer chose different paths
   - Impact of query rewrites
   - Effect of added indexes/statistics
   
4. **Recommendation**:
   - Which formulation is better
   - Quantified improvement estimates
   - Situational considerations (e.g., "Better for large datasets but...")

---

## Statistics & Index Recommendations (Requirement 13: All)

### Comprehensive Recommendations
1. **Current status reporting**:
   - What statistics exist
   - Last collection timestamps
   - Stale statistics identification
   - Missing critical statistics
   
2. **Ready-to-execute commands**:
   ```sql
   COLLECT STATISTICS ON database.table COLUMN column_name;
   COLLECT STATISTICS ON database.table INDEX index_name;
   ```
   
3. **Index suggestions**:
   ```sql
   CREATE INDEX idx_name (column_list) ON database.table;
   ```
   With rationale: "This NUSI on filter_column will eliminate the all-rows scan in Step 3"
   
4. **Impact prioritization**:
   - **High impact**: Steps with No Confidence, full scans on large tables
   - **Medium impact**: Low Confidence steps, suboptimal join methods
   - **Low impact**: Minor optimizations, edge case improvements
   
5. **Implementation guide**:
   - Order of execution (stats before indexes)
   - Maintenance considerations
   - Trade-off analysis (index overhead vs. query speedup)

---

## Error Handling (Requirement 14: Explain and suggest)

### Graceful Failure Handling
1. **SQL Syntax Errors**:
   - Parse error location
   - Common syntax mistakes
   - Corrected SQL suggestion
   - Offer to retry with fix
   
2. **Permission Issues**:
   - Identify missing privileges
   - Suggest who to contact
   - Alternative approaches
   
3. **Table/Object Not Found**:
   - Check for typos
   - List similar objects
   - Verify database context
   
4. **EXPLAIN Execution Failures**:
   - Explain why EXPLAIN failed
   - Suggest workarounds
   - Offer to analyze provided EXPLAIN text instead

---

## Output Organization (Requirement 15: Custom based on findings)

### Intelligent Ordering Logic

#### If Critical Issues Found (No Confidence, Product Joins, Major Scans)
1. **🚨 Critical Issues Summary** (lead with this)
2. Quick Performance Overview
3. Visualization
4. Detailed Step Analysis
5. Optimization Playbook
6. Supporting Metadata

#### If Only Minor Issues
1. **Quick Performance Overview** (lead with this)
2. Visualization
3. Optimization Opportunities
4. Detailed Step Analysis
5. Statistics & Index Status

#### If Query Is Already Optimal
1. **✅ Performance Validation** (lead with this)
2. Visualization
3. Detailed Step Analysis (educational)
4. Best Practices Confirmation

#### Comparative Analysis
1. **Comparison Summary** (which is better, by how much)
2. Side-by-side Visualizations
3. Key Differences Explained
4. Recommendation with Reasoning

---

## Required Tools

### Teradata MCP Tools
- `teradata:base_readQuery` - Execute EXPLAIN and query metadata
- `teradata:base_columnDescription` - Get column details
- `teradata:base_tableDDL` - Retrieve table DDL
- `teradata:base_tableList` - List tables in database
- `teradata:base_tablePreview` - Sample table data
- `teradata:base_tableAffinity` - Find commonly joined tables
- `teradata:dba_tableSpace` - Table size information
- `teradata:dba_tableUsageImpact` - Usage patterns
- `teradata:dba_resusageSummary` - System resource usage (for context)

### File Creation
- `create_file` - Generate HTML visualizations, ASCII diagrams, reports
- `str_replace` - Edit generated files if needed

### Execution
- `bash_tool` - For any text processing or utility operations if needed

---

## Agent Behavior Specifications

### When Invoked With SQL
1. Validate SQL syntax (basic check)
2. Attempt to execute EXPLAIN via teradataMCP
3. If successful, proceed with analysis
4. If failed, explain error and ask for EXPLAIN text or corrected SQL

### When Invoked With EXPLAIN Output
1. Validate it's actual EXPLAIN output (check for step patterns)
2. Parse and extract all steps
3. Proceed with analysis

### Analysis Workflow
1. **Parse**: Extract all steps, locks, estimates, confidence levels
2. **Classify**: Categorize operations using icon system
3. **Assess**: Identify issues, bottlenecks, optimization opportunities
4. **Query Metadata** (if possible): Get table structures, stats status, indexes
5. **Generate Recommendations**: Prioritized optimization playbook
6. **Create Visualizations**: HTML (default) + requested formats
7. **Organize Output**: Custom order based on findings severity
8. **Present**: Lead with most important information

### Deep-Dive Interaction
After presenting main analysis, offer:
- "Would you like me to explain any specific step in more detail?"
- "Should I generate alternative query formulations?"
- "Want to see the complete statistics collection plan?"
- "Should I analyze the table structures for optimization opportunities?"

---

## Output Artifacts

### Primary Deliverables
1. **HTML Visualization** (always generated)
   - Interactive query plan diagram
   - Expandable step details
   - Color-coded performance indicators
   - Embedded recommendations
   
2. **Optimization Playbook** (Markdown or HTML)
   - Prioritized action items
   - Ready-to-execute SQL commands
   - Expected impact estimates
   - Implementation guide
   
3. **Executive Summary** (in response)
   - Top 3-5 findings
   - Overall assessment
   - Quick wins
   - Estimated improvement potential

### Optional Deliverables (on request)
- ASCII diagram for documentation
- Mermaid diagram for markdown
- JSON structured output
- Comparative analysis report
- Statistics collection script
- Alternative query formulations

---

## Success Criteria

### The agent is successful when:
1. ✅ User understands how their query executes
2. ✅ Performance issues are clearly identified
3. ✅ Actionable recommendations are provided
4. ✅ Visualizations are clear and informative
5. ✅ Optimization path is evident
6. ✅ User can immediately act on suggestions

### The agent has failed if:
1. ❌ User is more confused after analysis
2. ❌ Recommendations are vague or unhelpful
3. ❌ Visualization is cluttered or unclear
4. ❌ Critical issues are missed
5. ❌ Suggestions don't actually improve performance

---

## Edge Cases & Special Handling

### Complex Scenarios
- **Partitioned tables**: Highlight partition elimination (or lack thereof)
- **Column partitioning**: Show CP merge spool usage
- **MERGE/UPSERT**: Explain conditional branches
- **Parallel steps**: Group and visualize parallel execution
- **Spool reuse**: Track with dashed lines in diagrams
- **Product joins**: Always flag and explain (acceptable vs problematic)
- **IPE masked values** (`":*"`): Note security masking in dynamic plans

### Performance Patterns to Recognize
- **Hot AMP indicators**: `(group_amps)` with few AMPs, large redistributions
- **Skew signals**: Uneven distribution, PRPD opportunities
- **Join geography inefficiencies**: Both sides redistributed when duplication possible
- **Missing partition elimination**: Partitioned table with all partitions accessed
- **Stale statistics**: Low/No confidence on critical steps

---

## Example Interaction Flow

### User Request
```
Analyze this query:
SELECT e.name, d.dept_name, e.salary
FROM employee e
JOIN department d ON e.dept_no = d.dept_no
WHERE e.salary > 50000;
```

### Agent Response Flow
1. **Execute EXPLAIN** via teradataMCP
2. **Detect**: Moderate complexity, standard join
3. **Parse**: Extract steps, identify merge join, redistribution
4. **Query Metadata**: Get employee/department DDL, stats status, indexes
5. **Assess**: 
   - Low confidence on salary predicate (no stats)
   - Redistribution required (tables not co-located)
   - Merge join chosen (good for large tables)
6. **Lead with finding**: "Low confidence detected - missing statistics opportunity"
7. **Present visualization**: HTML with color-coded steps
8. **Provide playbook**:
   - High: Collect stats on employee.salary
   - Medium: Consider index on salary if frequently filtered
   - Low: Review employee PI for common join patterns
9. **Offer deep-dive**: "Want to see alternative query formulations or index strategies?"

---

## Testing & Validation

### Agent must be tested with:
1. ✅ Simple single-table query (UPI access)
2. ✅ Multi-table join (various join methods)
3. ✅ Query with No Confidence steps
4. ✅ Partitioned table query
5. ✅ Complex query (subqueries, aggregations)
6. ✅ Invalid SQL (error handling)
7. ✅ Pre-generated EXPLAIN text (not live execution)
8. ✅ Comparative analysis (two query versions)

---

## Delegation Trigger

**This agent should be invoked when:**
- User provides SQL with "EXPLAIN" or asks to "analyze query performance"
- User provides Teradata EXPLAIN output for analysis
- User asks "why is this query slow"
- User requests query optimization recommendations
- User wants to understand query execution plans
- User asks to compare different query formulations
- User mentions terms: "query plan", "execution plan", "EXPLAIN", "query optimization", "performance tuning"

---

## Version & Maintenance

**Specification Version**: 1.0  
**Created**: 2026-01-20  
**Based on Documentation**: 
- doc_teradata_explain_expert_guide.md
- doc_teradata_explain_training_guide.md
- doc_teradata_explain_playbook_template.md
- doc_teradata_explain_visualization_template.md
- doc_teradataMCP.md

---

This specification provides the complete blueprint for building a sophisticated, production-ready Teradata EXPLAIN analysis agent that serves beginners and experts alike.
