# HTML Export Template — Technical Reference

This document defines the ERB template API, available variables, and patterns for Dradis HTML export templates.

## Template Location

```
lib/tasks/templates/{kit}/kit/templates/reports/html_export/dradis_template-{kit}-{slug}.v{X.Y}.html.erb
```

Runtime (for immediate testing):
```
storage/templates/reports/html_export/
```

## Available ERB Variables

These variables are provided by the Dradis export engine and are available in every template:

| Variable | Type | Description |
|----------|------|-------------|
| `issues` | `Array<Issue>` | All issues in the project |
| `notes` | `Array<Note>` | All notes in the project |
| `nodes` | `Array<Node>` | All nodes (hosts) in the project |
| `title` | `String` | The project title |

### Issue Object

```ruby
issue.fields['FieldName']  # => String — the field's raw text value
issue.evidence             # => Array<Evidence> — all evidence for this issue
issue.tags                 # => Array<Tag> — tags assigned to this issue
```

### Evidence Object

```ruby
evidence.fields['FieldName']  # => String — evidence-specific field value
evidence.node                 # => Node — the host/node this evidence belongs to
evidence.node.label           # => String — the node's display name (hostname, IP, etc.)
```

### Node Object

```ruby
node.label       # => String — display name
node.evidence    # => Array<Evidence>
node.notes       # => Array<Note>
```

### Content Blocks (Dradis Pro only)

Content blocks are Pro-only narrative sections (executive summary, methodology, conclusions, appendices, etc.) that live alongside the issue data. Each block belongs to a `block_group` and has a `Type` field used for filtering.

```ruby
# Gate behind Pro check
if defined?(Dradis::Pro)
  content_service.all_content_blocks.each do |cb|
    cb.fields['Title']       # => String
    cb.fields['Type']        # => String — used to filter/group blocks
    cb.fields['Description'] # => String (Textile, may contain Liquid)
  end
end
```

**Important:** Content block field values may contain Liquid templates (e.g., `{{ document_properties.dradis.client }}`, `{{ issues.size }}`). These reference project-scoped variables that are available in the export Liquid context, so `markup(block.fields['Description'], liquid: true)` works correctly.

#### Discovering content block types

The `Type` field values come from the project template ZIP (`dradis-repository.xml`). Always discover the actual values — never assume. Extract them from the `<content_block>` elements in the XML.

#### Recommended iteration pattern

Discover the section types from the project template XML (step 1 of the SKILL workflow), then iterate in display order:

```erb
<% if defined?(Dradis::Pro) %>
  <%
    # Populate this list from the <content_block> elements in the project template.
    # Determine display order from the block titles or numbering.
    # Each kit will have different types — never assume.
    pro_section_types = [...] # e.g., discovered from the XML
  %>
  <% pro_section_types.each do |section_type| %>
    <% section_blocks = content_service.all_content_blocks.select { |b| b.fields['Type'] == section_type } %>
    <% next if section_blocks.empty? %>
    <section id="pro-<%= section_type.downcase.gsub(' ', '-') %>">
      <% section_blocks.each do |block| %>
        <h2><%= markup(block.fields['Title'], liquid: true) %></h2>
        <%= markup(block.fields['Description'], liquid: true) %>
      <% end %>
    </section>
  <% end %>
<% end %>
```

**Notes:**
- When multiple blocks share the same `Type` (common for appendices), each renders with its own title.
- Some blocks may have kit-specific fields beyond `Title`, `Type`, and `Description` — always check.

#### Common content block fields

| Field | Description |
|-------|-------------|
| `Title` | Section/block heading (may contain Liquid) |
| `Type` | Category — used for filtering (e.g., `Document Control`, `Recommendations`, `Appendix`) |
| `Description` | Main body content (Textile + Liquid) |

Some blocks may have additional fields depending on the kit (e.g., `Scope Type`, `Testing Approach`, `Environment` for engagement overview blocks).

## Helper Methods

### `markup(content)`

Renders Textile markup to HTML. Use for free-text fields that contain Textile formatting.

```erb
<%= markup(issue.fields['Description Long']) %>
```

### `h(string)`

HTML-escapes a string. Use for field values inserted into attributes, JS strings, or anywhere that needs escaping.

```erb
<span title="<%= h(issue.fields['Title']) %>">...</span>
```

## Field Discovery

### Reading the project template XML

Each kit stores a project template ZIP containing XML with the field schema:

```
lib/tasks/templates/{kit}/kit/templates/projects/{kit}-project-template.xml
```

Inside the XML, issues look like:

```xml
<issue>
  <id>...</id>
  <tags>...</tags>
  <fields>
    <![CDATA[#[Title]#
SQL Injection in Login Form

#[Impact]#
High

#[Likelihood]#
Very High

#[Risk]#
{% assign impact_value = 0 %}{% case issue.fields['Impact'] %}...

]]>
  </fields>
</issue>
```

### Liquid Template Fields

Fields whose sample values contain `{% %}` or `{{ }}` are **Liquid templates**. Whether they work with `markup(..., liquid: true)` depends on what variables the Liquid template references.

The export engine's `liquid_assigns` (see `dradis-html_export/lib/dradis/plugins/html_export/exporter.rb`) provides **project-scoped** variables:
- `issues` (array of IssueDrop), `nodes`, `project`, `tags`
- Pro only: `content_blocks`, `document_properties`

There is **no per-issue `issue` variable** in the export Liquid context.

#### Two categories of Liquid fields

**Project-scoped Liquid** — references global variables like `{{ issues.size }}`, `{{ document_properties.dradis.client }}`. These work fine with `markup(..., liquid: true)`. Common in content blocks.

**Per-issue Liquid** — references `issue.fields['Impact']` (singular `issue`). This variable doesn't exist in the export Liquid context, so `markup(issue.fields['Risk'], liquid: true)` will fail or produce wrong output. **You must replicate these in ERB.**

#### Detection method

```ruby
# If a field's Liquid source references per-issue variables:
{% case issue.fields['Impact'] %}     # ← per-issue, must compute in ERB
{{ issue.fields['Likelihood'] }}      # ← per-issue, must compute in ERB

# If it references project-scoped variables:
{{ issues.size }}                     # ← project-scoped, markup(…, liquid: true) works
{{ document_properties.dradis.client }}  # ← project-scoped, works
```

#### ERB computation pattern

Read the Liquid template source in the project XML to understand the computation logic, then replicate it as an ERB helper method. For example, if a `Risk` field computes from `Impact` x `Likelihood`:

```erb
<%
  # Build lookup maps from the categorical values discovered in step 1
  impact_map = { 'Very High' => 5, 'High' => 4, ... }

  # Replicate the Liquid logic as a Ruby method
  def compute_risk(impact, likelihood, impact_map, likelihood_map)
    imp = impact_map[impact.to_s.strip] || 0
    lik = likelihood_map[likelihood.to_s.strip] || 0
    score = imp * lik
    # Thresholds should match whatever the Liquid template defines
    case
    when score >= 20 then 'Critical'
    when score >= 15 then 'High'
    when score >= 8  then 'Moderate'
    when score >= 3  then 'Low'
    else                  'Info'
    end
  end
%>
```

The key is to **read and understand** the Liquid source, then replicate its logic. Don't assume a particular formula — different kits may compute risk differently.

## Categorical Field Patterns

### Extracting unique values

Always extract ALL unique values for categorical fields. Missing a value means charts and counts won't add up.

```erb
<%
  # Approach 1: Collect from actual data (defensive — catches any value)
  all_values = issues.map { |i| i.fields['SomeField'].to_s.strip }.uniq.reject(&:empty?)

  # Approach 2: Define the known set explicitly (from the project template)
  # Use this when you need a specific display order
  known_values = ['Value A', 'Value B', 'Value C']

  # Count per category
  counts = Hash.new(0)
  issues.each { |i| counts[i.fields['SomeField'].to_s.strip] += 1 }
%>
```

### Color maps

Define a color for every categorical value. Choose colors that are semantically appropriate (e.g., red for critical/open, green for resolved/low):

```erb
<%
  colors = {
    'Value A' => '#dc3545',
    'Value B' => '#ffc107',
    'Value C' => '#198754'
  }
%>
```

## Sorting Issues

Sort issues by severity (highest first). The sort key depends on the kit's risk model:

```erb
<%
  # Define a display order for your computed/discovered severity levels
  severity_order = { 'Critical' => 0, 'High' => 1, 'Moderate' => 2, 'Low' => 3, 'Info' => 4 }

  sorted_issues = issues.sort_by { |i|
    severity_order[compute_severity(i)] || 99
  }
%>
```

## Risk Heatmap Data

If the kit has two risk dimensions (e.g., Impact x Likelihood), you can build a heatmap matrix. Map each dimension's categorical values to numeric indices, then populate a 2D array:

```erb
<%
  # Dimensions and labels come from the discovered categorical values
  row_map = { ... }  # e.g., impact values => numeric indices
  col_map = { ... }  # e.g., likelihood values => numeric indices
  size = row_map.size

  heatmap = Array.new(size) { Array.new(size, 0) }
  issues.each do |issue|
    row = row_map[issue.fields['RowField'].to_s.strip]
    col = col_map[issue.fields['ColField'].to_s.strip]
    next unless row && col
    heatmap[row][col] += 1
  end
%>
```

Cell colors should reflect the product of the two dimensions — low-low is cool/neutral, high-high is red/critical.

## Passing Data to JavaScript

Use `JSON.generate` or inline ERB in JS:

```erb
<script>
  const riskData = {
    labels: <%= JSON.generate(risk_levels) %>,
    values: <%= JSON.generate(risk_levels.map { |r| risk_counts[r] || 0 }) %>,
    colors: <%= JSON.generate(risk_levels.map { |r| risk_color[r] }) %>
  };
</script>
```

## CDN URLs

Use the latest versions of these libraries.

### Bootstrap 5.3
```html
<link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css" rel="stylesheet"
      integrity="sha384-QWTKZyjpPEjISv5WaRU9OFeRpok6YcnS/1p8TQ6QBlwCP/5QUUZ3mGOEisMGjJyT"
      crossorigin="anonymous" />
<script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.bundle.min.js"
        integrity="sha384-YvpcrYf0tY3lHB60NNkmXc5s9fDVZLESaAA55NDzOxhy9GkcIdslK1eN7N6jIeHz"
        crossorigin="anonymous"></script>
```

### Chart.js 4
```html
<script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.7/dist/chart.umd.min.js"></script>
```

### Highcharts 11
```html
<script src="https://code.highcharts.com/11/highcharts.js"></script>
<script src="https://code.highcharts.com/11/highcharts-more.js"></script>
<script src="https://code.highcharts.com/11/modules/solid-gauge.js"></script>
```

### Font Awesome 6
```html
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.1/css/all.min.css"
      integrity="sha512-DTOQO9RWCH3ppGqcWaEA1BIZOC6xxalwEsw9c2QQeAIftl+Vegovlnee1c9QX4TctnWMn13TZye+giMm8e2LwA=="
      crossorigin="anonymous" referrerpolicy="no-referrer" />
```

## Print Styles

Always include print-specific CSS:

```css
@media print {
  body { background: white; color: black; }
  .no-print { display: none !important; }
  .page-break { page-break-before: always; }
  canvas { max-width: 100%; height: auto !important; }
}
```

## Template File Naming

```
dradis_template-{kit}-{theme_slug}.v{major}.{minor}.html.erb
```

Examples:
- `dradis_template-owasp-galactica_cic.v1.0.html.erb`
- `dradis_template-owasp-executive_brief.v1.0.html.erb`
- `dradis_template-infrastructure-dark_dashboard.v1.0.html.erb`

## Kit-Specific Field Discovery

Every kit has a different field schema. **Never hardcode field names or values** — always discover them from the project template XML as described in step 1 of the SKILL workflow.

When documenting your findings during discovery, classify each field as:

| Classification | Description | Example |
|----------------|-------------|---------|
| Free text | Unstructured content, possibly Textile | Title, Description, References |
| Categorical | Finite set of known values | Impact, Status, Domain |
| Computed (Liquid) | Contains `{% %}` or `{{ }}` — must be computed in ERB | Risk, Risk Score |

Similarly, content blocks will vary by kit. Discover the `Type` field values, display order, and any kit-specific fields from the `<content_block>` elements in the XML.
