# Quick Integration Guide: Flow Diagram Visualization

## Add to Your teradata-explain-analyzer.md Skill

### Section to Add After "5. Generate Visualizations"

Add this new section:

---

### 5A. Create Interactive Flow Diagram (PRIMARY DELIVERABLE)

**ALWAYS create an interactive flow diagram** as the main visualization for EXPLAIN analysis. This should be presented BEFORE or INSTEAD OF tabular/text representations.

#### Flow Diagram Structure:

**Layout:** Vertical flow (top to bottom)
```
Lock Acquisition
    ↓
Parallel Retrieval (side-by-side if applicable)
    ↓
Sequential Operations (joins, sorts, etc.)
    ↓
Final Output
```

**Visual Encoding:**
- 🟢 Green border: Good performance (high confidence, fast)
- 🟡 Orange border: Warning (low confidence, moderate issues)
- 🔴 Red border: Critical (no confidence, product joins, >15% of time)
- 🔵 Blue border: Information (locks, output)

**Required Elements:**

1. **Header Section:**
   - Query text in code block
   - Execution summary (total time, steps, issues)

2. **Summary Metrics (4-6 cards):**
   - Total execution time
   - Result size
   - Critical issues count
   - Confidence distribution

3. **Legend:**
   - Color meanings
   - Badge explanations
   - Operation type icons

4. **Interactive Controls:**
   - "Highlight Critical Path" button
   - "Highlight Product Joins" button
   - "Reset" button

5. **Flow Diagram Container:**
   - Each step as a node with:
     - Step number badge
     - Operation type label ([ALL-AMP], [HASH-J], etc.)
     - Confidence badge (HIGH/LOW/NONE)
     - Human-readable title
     - Key metrics (rows, size, time, method)
     - Hover tooltip with detailed explanation
   - Arrows between steps showing:
     - Data flow direction
     - Spool names and sizes
     - Data volume changes

6. **Key Insights Panel:**
   - Critical path analysis (top 5 slowest steps)
   - Data growth through pipeline
   - Immediate optimization opportunities

#### Each Flow Node Structure:

```html
<div class="flow-node [critical/warning/good/info]" data-step="N">
    <div class="node-header">
        <div class="node-step">Step N</div>
        <div class="node-type">[OPERATION-TYPE]</div>
        <div class="confidence-badge [high/low/none]">Confidence</div>
    </div>
    <div class="node-title">Human-readable operation</div>
    <div class="node-metrics">
        <div class="node-metric">
            <span class="node-metric-label">Rows</span>
            <span class="node-metric-value">X</span>
        </div>
        <div class="node-metric">
            <span class="node-metric-label">Size</span>
            <span class="node-metric-value">X MB</span>
        </div>
        <div class="node-metric">
            <span class="node-metric-label">Time</span>
            <span class="node-metric-value">X.XXs (XX%)</span>
        </div>
        <div class="node-metric">
            <span class="node-metric-label">Method</span>
            <span class="node-metric-value">Description</span>
        </div>
    </div>
    <div class="tooltip-content">
        <strong>Purpose:</strong> What this step does<br>
        <strong>Technical:</strong> Join conditions, filters<br>
        <strong>Performance:</strong> Why fast/slow<br>
        <strong>Issue:</strong> Specific problems if any<br>
        <strong>Fix:</strong> Optimization suggestion
    </div>
</div>
```

#### Parallel Execution Visualization:

When EXPLAIN shows parallel steps (e.g., "We execute the following steps in parallel"):

```html
<div class="parallel-start">
    ◆ PARALLEL EXECUTION START (Steps X-Y)
</div>

<div class="flow-row parallel">
    <div class="flow-node">Step X.1</div>
    <div class="flow-node">Step X.2</div>
    <div class="flow-node">Step X.3</div>
</div>

<div class="parallel-end">
    ◆ PARALLEL EXECUTION COMPLETE
</div>
```

#### Step Classification Logic:

```python
def classify_step(step):
    # Critical (Red)
    if "product join" in step.text.lower():
        return "critical"
    if "no confidence" in step.text.lower():
        return "critical"
    if step.time > (total_time * 0.15):  # >15% of total
        return "critical"
    
    # Warning (Orange)
    if "low confidence" in step.text.lower():
        return "warning"
    if step.time > (total_time * 0.05):  # 5-15% of total
        return "warning"
    if step.spool_size > 500_000_000:  # >500 MB
        return "warning"
    
    # Good (Green)
    if "high confidence" in step.text.lower():
        if step.time < 0.01:
            return "good"
    
    # Default
    return "info"
```

#### Operation Type Labels:

Map EXPLAIN text to standardized labels:
- "all-AMPs RETRIEVE" → `[ALL-AMP]`
- "single-AMP RETRIEVE" → `[1-AMP]`
- "hash join" → `[HASH-J]`
- "merge join" → `[MERGE-J]`
- "product join" → `[PROD-J]` ⚠️
- "nested join" → `[NEST-J]`
- "SORT" → `[SORT]`
- "aggregate" → `[AGG]`

#### Interactive Features:

```javascript
// Highlight critical path (top 5 slowest steps)
function toggleCriticalPath() {
    const criticalSteps = findTopStepsByTime(5);
    highlightSteps(criticalSteps, 'critical-path');
}

// Highlight product joins
function toggleProductJoins() {
    const productJoinSteps = findStepsByType('product-join');
    highlightSteps(productJoinSteps, 'product-join-highlight');
}

// Reset all highlights
function resetHighlight() {
    removeAllHighlights();
}
```

#### File Naming and Delivery:

```python
# Create file
filename = f"{query_identifier}_flow_diagram.html"
filepath = f"/home/claude/{filename}"

# Write HTML with all elements
write_flow_diagram_html(filepath, steps_data)

# Copy to outputs
shutil.copy(filepath, f"/mnt/user-data/outputs/{filename}")

# Present as PRIMARY deliverable
present_files([f"/mnt/user-data/outputs/{filename}"])
```

#### What to Include in Response Text:

After presenting the flow diagram, provide a BRIEF summary (3-4 paragraphs max):

1. **Critical Finding** (1-2 sentences)
2. **Top 3 Bottlenecks** (bullet list)
3. **Quick Win Optimization** (1 code example)
4. **Expected Impact** (quantified improvement)

DO NOT:
- Write extensive analysis in text (it's in the visual)
- Duplicate what's in the diagram
- Overwhelm with details

The flow diagram IS the analysis. Text should be minimal.

---

### Modified Section 5 (Original Visualizations)

**Keep tabular metrics** as SECONDARY deliverable:
- Step-by-step table
- Confidence dashboard
- Performance heat map (bar chart)

But make flow diagram the HERO visualization.

---

### Modified Section 11 (Report/Response)

Update the example response structure:

```markdown
# Teradata EXPLAIN Analysis

## 🚨 Critical Finding
[One sentence: biggest issue]

📊 **Interactive Flow Diagram** (click to explore)
[Present flow diagram file here]

## Top Bottlenecks
1. Step X: [Issue] - Y.YYs (ZZ%)
2. Step X: [Issue] - Y.YYs (ZZ%)
3. Step X: [Issue] - Y.YYs (ZZ%)

## Quick Win
```sql
[Optimization code]
```
**Impact:** [Quantified improvement]
```

---

## CSS Reference (Essential Styles)

Include these core styles in your flow diagram HTML:

```css
/* Flow Container */
.flow-diagram {
    background: #f8f9fa;
    padding: 30px;
    border-radius: 8px;
    overflow-x: auto;
}

.flow-container {
    display: flex;
    flex-direction: column;
    gap: 20px;
    min-width: 1200px;
}

.flow-row {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 20px;
}

.flow-row.parallel {
    justify-content: space-around;
}

/* Flow Nodes */
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

.flow-node:hover {
    transform: translateY(-5px) scale(1.02);
    box-shadow: 0 8px 20px rgba(0,0,0,0.2);
}

.flow-node.good { border-color: #27ae60; }
.flow-node.warning { border-color: #f39c12; }
.flow-node.critical { border-color: #e74c3c; }
.flow-node.info { border-color: #3498db; }

/* Arrows */
.flow-arrow {
    display: flex;
    flex-direction: column;
    align-items: center;
    margin: 10px 0;
}

.arrow-line {
    width: 3px;
    height: 30px;
    background: linear-gradient(180deg, #95a5a6, #7f8c8d);
}

.arrow-line::after {
    content: '';
    position: absolute;
    bottom: -8px;
    left: 50%;
    transform: translateX(-50%);
    border-left: 8px solid transparent;
    border-right: 8px solid transparent;
    border-top: 10px solid #7f8c8d;
}

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

.flow-node:hover .tooltip-content {
    display: block;
}

/* Badges */
.confidence-badge {
    padding: 2px 8px;
    border-radius: 10px;
    font-size: 10px;
    font-weight: bold;
    text-transform: uppercase;
}

.confidence-badge.high { background: #27ae60; color: white; }
.confidence-badge.low { background: #f39c12; color: white; }
.confidence-badge.none { background: #e74c3c; color: white; }
```

---

## Checklist Before Presenting

- [ ] All EXPLAIN steps visualized as nodes
- [ ] Color coding applied correctly (critical/warning/good/info)
- [ ] Parallel execution shown side-by-side
- [ ] Arrows show data flow + spool info
- [ ] Interactive buttons implemented and tested
- [ ] Tooltips provide detailed explanations
- [ ] Legend explains all visual elements
- [ ] Critical path is identifiable
- [ ] File saved to /mnt/user-data/outputs/
- [ ] File presented via present_files tool

---

## Example Output Flow

```
User: "Analyze this EXPLAIN plan"

Agent:
1. Parse EXPLAIN → 14 steps identified
2. Classify: 3 green, 2 orange, 8 red, 1 blue
3. Identify: 2 product joins (steps 10, 13)
4. Calculate: Critical path = steps 6,7,10,11,13 (91% of time)
5. CREATE: flow_diagram.html with all elements
6. PRESENT: "Here's your query execution flow..."

Result: User immediately sees bottlenecks and solutions
```