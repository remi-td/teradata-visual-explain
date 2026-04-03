# Flow Diagram Visualization Instructions for Teradata EXPLAIN Analysis

## When to Create Flow Diagrams

**ALWAYS create an interactive flow diagram visualization for EXPLAIN plan analysis.** Flow diagrams should be the PRIMARY deliverable, not an optional add-on.

Create flow diagrams for:
- All EXPLAIN plan analyses (regardless of query complexity)
- Queries with 3+ steps
- Any query where visual representation adds clarity

## Flow Diagram Design Principles

### 1. Vertical Flow Layout (Top to Bottom)

**Structure:**
```
Lock Acquisition (Steps 1-2)
    ↓
Parallel Execution Branch (if applicable)
    Step 3.1 | Step 3.2 | Step 3.3 (side by side)
    ↓
Data flows downward through joins
    ↓
Final Output
```

**Why Vertical:**
- Natural reading order (top to bottom)
- Shows data flow progression clearly
- Easier to see data size growth
- Better for long execution plans (10+ steps)

### 2. Visual Encoding System

**Color Coding (Border Colors):**
- 🟢 **Green (#27ae60)**: Good performance, high confidence, fast operations
- 🟡 **Orange (#f39c12)**: Warnings, low confidence, moderate issues
- 🔴 **Red (#e74c3c)**: Critical issues, no confidence, product joins, bottlenecks
- 🔵 **Blue (#3498db)**: Informational, output steps, neutral operations
- 🟣 **Purple (#9b59b6)**: Special operations (locks, parallel markers)

**Node Size:**
- Keep all nodes consistent size (min-width: 280px)
- Don't vary size by data volume (use labels instead)
- Consistency > visual tricks

**Badges:**
- Confidence levels: HIGH (green), LOW (orange), NONE (red)
- Step numbers: Purple/blue background, white text
- Operation types: Gray background badges ([ALL-AMP], [HASH-J], [PROD-J])

### 3. Node Structure

Each flow node should contain:

```html
<div class="flow-node [critical/warning/good/info]" data-step="X">
    <div class="node-header">
        <div class="node-step">Step X</div>
        <div class="node-type">[OPERATION-TYPE]</div>
        <div class="confidence-badge [high/low/none]">Confidence</div>
    </div>
    <div class="node-title">Human-readable operation description</div>
    <div class="node-metrics">
        <div class="node-metric">
            <span class="node-metric-label">Rows</span>
            <span class="node-metric-value">1,000</span>
        </div>
        <div class="node-metric">
            <span class="node-metric-label">Size</span>
            <span class="node-metric-value">100 MB</span>
        </div>
        <div class="node-metric">
            <span class="node-metric-label">Time</span>
            <span class="node-metric-value">0.50s</span>
        </div>
        <div class="node-metric">
            <span class="node-metric-label">Method</span>
            <span class="node-metric-value">Hash join</span>
        </div>
    </div>
    <div class="tooltip-content">
        <!-- Detailed explanation on hover -->
    </div>
</div>
```

**Key Metrics to Display:**
- **Rows:** Estimated or actual row count
- **Size:** Spool size in KB/MB/GB
- **Time:** Step execution time + % of total
- **Method:** Access method, join type, or operation
- **Additional:** Join keys, filter conditions, confidence level

### 4. Arrow/Connector Design

**Between Sequential Steps:**
```html
<div class="flow-arrow">
    <div class="arrow-line"></div>
    <div class="arrow-spool">Spool 5: 573 MB, 46,005 rows</div>
</div>
```

**Arrow Styling:**
- Use CSS-drawn arrows (not images)
- Include spool information if applicable
- Show data volume growth between steps
- Use subtle colors (#7f8c8d, gray)

**For Parallel Execution:**
```html
<div class="parallel-start">
    ◆ PARALLEL EXECUTION START (Steps X-Y)
</div>

<div class="flow-row parallel">
    <!-- Multiple nodes side-by-side -->
</div>

<div class="parallel-end">
    ◆ PARALLEL EXECUTION COMPLETE
</div>
```

### 5. Operation Type Icons/Labels

**Standardized Labels:**
- `[ALL-AMP]` - All-AMP retrieve
- `[1-AMP]` - Single-AMP retrieve
- `[HASH-J]` - Hash join
- `[MERGE-J]` - Merge join
- `[PROD-J]` - Product join (⚠️ warning!)
- `[NEST-J]` - Nested join
- `[SORT]` - Sort operation
- `[AGG]` - Aggregation
- `[LOCK]` - Lock acquisition
- `[OUTPUT]` - Final output to client

### 6. Parallel Execution Visualization

**Layout Pattern:**
```
Regular Step
    ↓
[Parallel Start Marker]
    ↓
Step A | Step B | Step C  (displayed horizontally)
    ↓
[Parallel End Marker - merge outputs]
    ↓
Regular Step (continues)
```

**Implementation:**
- Use `flex-direction: row` for parallel steps
- Use `flex-direction: column` for sequential steps
- Add visual markers for parallel boundaries
- Show which spools are created in parallel

### 7. Critical Issues Highlighting

**Product Joins:**
```html
<div class="flow-node critical product-join">
    <!-- Node content -->
    <div class="product-join-warning">
        <span class="warning-icon">⚠️</span>
        PRODUCT JOIN - Major Bottleneck
    </div>
</div>
```

**Special styling:**
- Red border (3px solid)
- Warning banner below metrics
- Detailed explanation in tooltip
- Option to highlight via interactive button

**Other Critical Issues:**
- No confidence estimates
- All-rows scans on very large tables
- Excessive redistributions
- Missing statistics

### 8. Interactive Features

**Must Include:**

1. **Hover Tooltips:**
   - Detailed explanation of what the step does
   - Why it's performing this way
   - Performance implications
   - Optimization suggestions

2. **Highlight Buttons:**
   - "Highlight Critical Path" - Show top 5 slowest steps
   - "Highlight Product Joins" - Show problematic join types
   - "Highlight No Confidence" - Show statistics issues
   - "Reset" - Return to normal view

3. **Click Interactions:**
   - Click node to expand/collapse details (optional)
   - Click to copy step details (optional)

**Implementation Example:**
```javascript
function toggleCriticalPath() {
    const criticalSteps = ['6', '7', '10', '11', '13']; // Top 5 by time
    document.querySelectorAll('.flow-node').forEach(node => {
        const step = node.getAttribute('data-step');
        if (criticalSteps.includes(step)) {
            node.style.boxShadow = '0 0 20px 5px rgba(231, 76, 60, 0.6)';
            node.style.transform = 'scale(1.05)';
        } else {
            node.style.opacity = '0.4';
        }
    });
}
```

### 9. Legend and Context

**Always Include:**

1. **Legend** (at top of diagram):
   - Color meanings
   - Icon/badge explanations
   - Confidence level key
   - Join type symbols

2. **Summary Metrics** (before diagram):
   - Total execution time
   - Total steps
   - Number of critical issues
   - Data volume (input → output)

3. **Key Insights Panel** (after diagram):
   - Critical path analysis
   - Data growth through pipeline
   - Optimization impact estimates
   - Top bottlenecks summary

### 10. Mapping EXPLAIN Output to Visual Elements

**Step Identification:**
```python
# Parse EXPLAIN text for:
- Step number (1, 2, 3, etc.)
- Operation type (RETRIEVE, JOIN, SORT, etc.)
- Table/spool names
- Lock types
- Confidence level (high/low/no confidence)
- Estimated rows
- Estimated time
- Spool size
- Join method and conditions
- Access method (all-rows scan, index, etc.)
```

**Classification Logic:**
```python
def classify_step(step_data):
    if "product join" in step_data.lower():
        return "critical"  # Red
    elif "no confidence" in step_data.lower():
        return "critical"  # Red
    elif step_data.time > (total_time * 0.10):  # >10% of time
        return "critical"  # Red
    elif "low confidence" in step_data.lower():
        return "warning"   # Orange
    elif step_data.time > (total_time * 0.05):  # 5-10% of time
        return "warning"   # Orange
    elif "high confidence" in step_data.lower():
        return "good"      # Green
    else:
        return "info"      # Blue
```

**Time-Based Severity:**
- Steps consuming >15% of total time → Critical (red)
- Steps consuming 10-15% → Warning (orange)
- Steps consuming 5-10% → Warning (orange)
- Steps consuming <5% → Normal

### 11. HTML Structure Template

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Flow Diagram: [Query Summary]</title>
    <style>
        /* Include full CSS from reference implementation */
    </style>
</head>
<body>
    <div class="container">
        <!-- Header -->
        <div class="header">
            <h1>Query Execution Flow Diagram</h1>
            <div class="query-display">[SQL Query]</div>
        </div>
        
        <div class="content">
            <!-- Summary Metrics -->
            <div class="summary">
                <!-- 3-6 metric cards -->
            </div>
            
            <!-- Legend -->
            <div class="legend">
                <!-- Legend items -->
            </div>
            
            <!-- Interactive Controls -->
            <div class="controls">
                <button onclick="toggleCriticalPath()">Highlight Critical Path</button>
                <!-- More buttons -->
            </div>
            
            <!-- Main Flow Diagram -->
            <div class="flow-diagram">
                <div class="flow-container">
                    <!-- Lock Step -->
                    <!-- Parallel Marker (if needed) -->
                    <!-- Flow Nodes with Arrows -->
                    <!-- Output Step -->
                </div>
            </div>
            
            <!-- Key Insights -->
            <div class="key-insights">
                <!-- Critical path analysis -->
                <!-- Data growth stats -->
                <!-- Optimization recommendations -->
            </div>
        </div>
    </div>
    
    <script>
        // Interactive functions
    </script>
</body>
</html>
```

### 12. CSS Best Practices

**Core Styles:**
- Use flexbox for layout (easier than grid for vertical flow)
- Transitions on hover (0.3s ease)
- Box shadows for depth
- Border-left accent for step categories
- Consistent spacing (20px gaps)
- Responsive design (min-width on nodes)

**Color Palette:**
```css
:root {
    --color-critical: #e74c3c;
    --color-warning: #f39c12;
    --color-good: #27ae60;
    --color-info: #3498db;
    --color-special: #9b59b6;
    --color-text: #2c3e50;
    --color-text-light: #7f8c8d;
    --color-bg: #f5f7fa;
    --color-bg-card: #ffffff;
}
```

### 13. Responsive Design Considerations

**For Wide Diagrams:**
```css
.flow-diagram {
    overflow-x: auto;  /* Allow horizontal scroll if needed */
}

.flow-container {
    min-width: 1200px;  /* Ensure minimum width for complex flows */
}
```

**For Mobile (optional):**
- Stack parallel steps vertically on small screens
- Reduce font sizes
- Adjust node widths

### 14. Data Flow Annotation

**Show Data Growth:**
```html
<div class="flow-arrow">
    <div class="arrow-line"></div>
    <div class="arrow-label">Spool 5: 573 MB → 986 MB (72% growth)</div>
</div>
```

**Annotate Redistributions:**
```html
<div class="flow-node warning">
    <!-- Node content -->
    <div class="data-movement-badge">
        📊 Redistributed by hash(TableId)
    </div>
</div>
```

### 15. Tooltip Content Guidelines

**What to Include in Tooltips:**

1. **Purpose**: What is this step doing?
2. **Technical Details**: Join conditions, filter predicates, access methods
3. **Performance Analysis**: Why is it fast/slow?
4. **Issues**: Specific problems (no confidence, all-rows scan, etc.)
5. **Optimization Hints**: Quick suggestions for improvement

**Tooltip Structure:**
```html
<div class="tooltip-content">
    <strong>Purpose:</strong> Join customer data with orders<br>
    <strong>Join Method:</strong> Hash join on CustomerID<br>
    <strong>Performance:</strong> ⚠️ 0.85s - Large spool causing slowdown<br>
    <strong>Issue:</strong> No confidence estimate - missing statistics<br>
    <strong>Suggestion:</strong> COLLECT STATISTICS ON orders COLUMN CustomerID
</div>
```

### 16. Special Cases

**Subqueries:**
- Indent or nest visually
- Use dashed borders
- Label clearly as "Subquery" or "Nested Step"

**Recursive Queries:**
- Show iteration marker
- Use curved arrows for recursion
- Label "Recursive Step"

**Multiple Output Spools:**
- Show branching arrows
- Label each branch
- Converge at join point

### 17. Performance Optimization for Large Plans

**For queries with 20+ steps:**
- Consider collapsing parallel sections by default
- Add "Expand All / Collapse All" buttons
- Use virtual scrolling if needed
- Highlight only top 10 critical steps by default

**For queries with 50+ steps:**
- Create separate sections (e.g., "Parallel Phase", "Join Phase", "Aggregation Phase")
- Add navigation/jump-to buttons
- Consider minimap overview

### 18. Testing and Validation

**Before finalizing flow diagram:**
1. Verify all steps from EXPLAIN are represented
2. Check that confidence levels are correctly mapped
3. Ensure critical path is accurate (top 5-7 by time)
4. Validate that data flow arrows show correct spool names
5. Test interactive features (highlight buttons work)
6. Check tooltip content is helpful and accurate

### 19. Common Patterns to Recognize

**Pattern 1: Star Schema Join**
- Central fact table
- Multiple dimension lookups
- Often parallel dimension retrieves

**Pattern 2: Self-Join**
- Same table appears multiple times
- Use labels to differentiate (e.g., "Orders (current)" vs "Orders (prior)")

**Pattern 3: Aggregation Pipeline**
- Retrieve → Filter → Group → Sort → Output
- Show transformation at each stage

**Pattern 4: View Expansion**
- View query expands into multiple underlying table accesses
- Show view name prominently
- Indent underlying table accesses

### 20. Final Deliverable Checklist

Before presenting flow diagram to user, verify:

- [ ] All EXPLAIN steps are represented
- [ ] Color coding is consistent and meaningful
- [ ] Critical issues are clearly highlighted
- [ ] Interactive features work (test all buttons)
- [ ] Tooltips provide value-add information
- [ ] Legend explains all visual elements
- [ ] Summary metrics are accurate
- [ ] Key insights are actionable
- [ ] File is saved to /mnt/user-data/outputs/
- [ ] File is presented to user via present_files tool

## Integration with Existing Workflow

**Add to Step 5 in your EXPLAIN analysis workflow:**

After "Generate Visualizations":
1. Parse EXPLAIN output into structured steps
2. Classify each step (good/warning/critical)
3. Identify parallel execution blocks
4. Calculate critical path (top steps by time)
5. **CREATE flow diagram HTML file** with:
   - Header with query
   - Summary metrics
   - Legend
   - Interactive controls
   - Main flow diagram (vertical layout)
   - Key insights panel
6. Save to outputs directory
7. Present to user as PRIMARY deliverable

**File Naming:**
- Use descriptive names: `query_flow_diagram.html`
- Include query identifier if applicable: `columnsv_flow_diagram.html`

## Example Workflow

```markdown
User: "Analyze this EXPLAIN plan"

Agent Steps:
1. Execute/Parse EXPLAIN → Extract 14 steps
2. Classify each step:
   - Steps 3.1, 8, 12: Green (high confidence, fast)
   - Steps 3.2, 9: Orange (low confidence)
   - Steps 3.3, 4-7, 10, 11, 13: Red (no confidence/slow)
3. Identify patterns:
   - Parallel execution: Steps 3.1-3.3, Steps 8-9
   - Product joins: Steps 10, 13 (CRITICAL)
   - Critical path: Steps 6, 7, 10, 11, 13
4. CREATE flow_diagram.html:
   - Use vertical layout
   - Show parallel branches
   - Highlight product joins in red
   - Add interactive controls
   - Include detailed tooltips
5. Save to /mnt/user-data/outputs/
6. Present to user

User sees:
- Interactive visual flow
- Clear bottleneck identification
- Actionable optimization paths
```

## Quick Reference: Decision Tree

**Should this step be RED (critical)?**
- YES if: Product join, OR no confidence, OR >15% of total time, OR all-rows scan on huge table
- Otherwise, continue...

**Should this step be ORANGE (warning)?**
- YES if: Low confidence, OR 5-15% of total time, OR large spool (>500 MB)
- Otherwise, continue...

**Should this step be GREEN (good)?**
- YES if: High confidence, AND fast (<0.01s), AND efficient access method
- Otherwise, use BLUE (info)

## Common Mistakes to Avoid

❌ **Don't:**
- Use horizontal layout (harder to read for long plans)
- Make nodes different sizes (visual inconsistency)
- Omit confidence badges (critical information)
- Skip tooltips (loss of detail)
- Forget interactive features (user engagement)
- Use too many colors (visual noise)
- Hide the critical path (defeats purpose)
- Show EXPLAIN text instead of visualization (user requested visual)

✅ **Do:**
- Use consistent vertical flow
- Keep nodes uniform size
- Show confidence on every applicable step
- Provide rich tooltips with explanations
- Implement highlight buttons
- Stick to 4-5 core colors
- Make critical path obvious
- Prioritize visual over text

## Summary: The "Flow Diagram Formula"

For any EXPLAIN analysis, create:

**1. Header** (Query + Context)
**2. Metrics** (4-6 summary cards)
**3. Legend** (Decode visual language)
**4. Controls** (Interactive buttons)
**5. Flow Diagram** (Main deliverable):
   - Vertical layout
   - Color-coded nodes
   - Clear arrows + spool labels
   - Parallel execution markers
   - Hover tooltips
**6. Insights** (Critical path + optimization)

**File:** Save as `[query_name]_flow_diagram.html`
**Present:** Use present_files tool as PRIMARY deliverable

This creates a **complete, interactive, professional visualization** that immediately shows:
- What the query is doing
- Where the bottlenecks are
- How to fix them

---

## Advanced: Complexity-Based Adaptations

### Simple Query (3-5 steps)
- Larger nodes with more metrics
- Simpler tooltips
- Fewer controls
- Emphasize clarity over interactivity

### Medium Query (6-15 steps)
- Standard implementation (as described above)
- All features included
- Balance detail vs. overview

### Complex Query (16-30 steps)
- Collapsible sections
- "Jump to Step" navigation
- Minimap (optional)
- Summary view + detailed view toggle

### Very Complex Query (31+ steps)
- Group by operation type (e.g., "Parallel Retrieval Phase", "Join Phase", "Aggregation Phase")
- Multi-level expansion (expand phase → expand step)
- Search/filter capability
- Separate "Overview" and "Detailed" HTML files

---

## Final Note on Adoption

**Start simple:** Begin with basic vertical flow, color coding, and tooltips
**Add features:** Gradually add interactive controls, parallel visualization, etc.
**Iterate:** Refine based on user feedback and query patterns
**Maintain consistency:** Once you establish a visual language, stick with it

The goal is **instant comprehension** of query performance. If a user can look at the flow diagram and immediately identify:
1. Where the query is slow
2. Why it's slow  
3. How to fix it

...then your visualization is successful.