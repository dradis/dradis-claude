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

```ruby
# Gate behind Pro check
if defined?(Dradis::Pro)
  content_service.all_content_blocks.each do |cb|
    cb.fields['Title']       # => String
    cb.fields['Description'] # => String (Textile)
  end
end
```

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

Fields whose sample values contain `{% %}` or `{{ }}` are **Liquid templates**. They are processed by the Dradis web UI but arrive as raw Liquid source in the ERB export context.

**You must compute these values in ERB instead.**

Common Liquid-computed fields:
- `Risk` — typically computed from `Impact` x `Likelihood`
- `Risk Score` — numeric version of the above

#### Detection method

```ruby
# In the project template XML, if a field value contains:
{% assign ... %}
{% case ... %}
{{ ... }}
# Then it's a Liquid field — do NOT use issue.fields['FieldName'] for it.
```

#### ERB computation pattern

```erb
<%
  impact_map     = { 'Very High' => 5, 'High' => 4, 'Moderate' => 3, 'Low' => 2, 'Very Low' => 1 }
  likelihood_map = { 'Very High' => 5, 'High' => 4, 'Moderate' => 3, 'Low' => 2, 'Very Low' => 1 }

  def compute_risk(impact, likelihood, impact_map, likelihood_map)
    imp = impact_map[impact.to_s.strip] || 0
    lik = likelihood_map[likelihood.to_s.strip] || 0
    score = imp * lik
    case
    when score >= 20 then 'Critical'
    when score >= 15 then 'High'
    when score >= 8  then 'Moderate'
    when score >= 3  then 'Low'
    else                  'Info'
    end
  end

  def compute_risk_score(impact, likelihood, impact_map, likelihood_map)
    imp = impact_map[impact.to_s.strip] || 0
    lik = likelihood_map[likelihood.to_s.strip] || 0
    imp * lik
  end
%>
```

## Categorical Field Patterns

### Extracting unique values

Always extract ALL unique values for categorical fields. Missing a value means charts and counts won't add up.

```erb
<%
  # Collect all unique statuses from actual data
  all_statuses = issues.map { |i| i.fields['Remediation Status'].to_s.strip }.uniq.reject(&:empty?)

  # Or define the known set explicitly (from the project template):
  statuses = ['Open', 'In Progress', 'Partially Remediated', 'Accepted Risk', 'Remediated']

  # Count per category
  status_counts = Hash.new(0)
  issues.each { |i| status_counts[i.fields['Remediation Status'].to_s.strip] += 1 }
%>
```

### Color maps

Define a color for every categorical value:

```erb
<%
  risk_color = {
    'Critical' => '#dc3545',
    'High'     => '#fd7e14',
    'Moderate' => '#ffc107',
    'Low'      => '#0dcaf0',
    'Info'     => '#6c757d'
  }

  status_colors = {
    'Open'                  => '#dc3545',
    'In Progress'           => '#fd7e14',
    'Partially Remediated'  => '#ffc107',
    'Accepted Risk'         => '#6c757d',
    'Remediated'            => '#198754'
  }
%>
```

## Sorting Issues

Sort by computed risk (highest first):

```erb
<%
  risk_order = { 'Critical' => 0, 'High' => 1, 'Moderate' => 2, 'Low' => 3, 'Info' => 4 }

  sorted_issues = issues.sort_by { |i|
    computed = compute_risk(i.fields['Impact'], i.fields['Likelihood'], impact_map, likelihood_map)
    risk_order[computed] || 99
  }
%>
```

## Risk Heatmap Data

For Impact x Likelihood heatmaps:

```erb
<%
  labels = ['Very Low', 'Low', 'Moderate', 'High', 'Very High']
  heatmap = Array.new(5) { Array.new(5, 0) }
  heatmap_issues = Array.new(5) { Array.new(5) { [] } }

  issues.each do |issue|
    imp = impact_map[issue.fields['Impact'].to_s.strip]
    lik = likelihood_map[issue.fields['Likelihood'].to_s.strip]
    next unless imp && lik
    heatmap[imp - 1][lik - 1] += 1
    heatmap_issues[imp - 1][lik - 1] << issue.fields['Title']
  end
%>
```

## Heatmap Cell Colors

The color of each heatmap cell is derived from the product of impact and likelihood indices:

```erb
<%
  heatmap_colors = [
    ['#6c757d', '#6c757d', '#0dcaf0', '#0dcaf0', '#ffc107'],  # Impact: Very Low
    ['#6c757d', '#0dcaf0', '#0dcaf0', '#ffc107', '#ffc107'],  # Impact: Low
    ['#0dcaf0', '#0dcaf0', '#ffc107', '#fd7e14', '#fd7e14'],  # Impact: Moderate
    ['#0dcaf0', '#ffc107', '#fd7e14', '#fd7e14', '#dc3545'],  # Impact: High
    ['#ffc107', '#fd7e14', '#fd7e14', '#dc3545', '#dc3545']   # Impact: Very High
  ]
%>
```

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

## OWASP Kit Field Reference

For the OWASP kit specifically, the issue fields are:

| Field | Type | Liquid? | Notes |
|-------|------|---------|-------|
| `Title` | Free text | No | Issue title |
| `OWASP Domain` | Categorical | No | Values: `Web`, `API`, `Mobile` |
| `OWASP Top 10` | Categorical | No | e.g., `A01:2021 - Broken Access Control` |
| `Description Short` | Free text | No | Brief description |
| `Description Long` | Free text (Textile) | No | Detailed description |
| `Remediation Status` | Categorical | No | Values: `Open`, `In Progress`, `Partially Remediated`, `Accepted Risk`, `Remediated` |
| `References` | Free text | No | External links |
| `Impact` | Categorical | No | Values: `Very Low`, `Low`, `Moderate`, `High`, `Very High` |
| `Likelihood` | Categorical | No | Values: `Very Low`, `Low`, `Moderate`, `High`, `Very High` |
| `Risk` | Computed | **Yes** | Liquid template — compute from Impact x Likelihood |
| `Risk Score` | Computed | **Yes** | Liquid template — compute as numeric value |
