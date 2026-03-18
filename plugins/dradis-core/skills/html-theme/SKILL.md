---
name: html-theme
description: Create an HTML export template (ERB) for a Dradis project kit. Generates a self-contained report with charts, risk dashboards, and detailed findings. Use when the user wants to build a new HTML report theme or redesign an existing one.
disable-model-invocation: false
user-invocable: true
allowed-tools: Read, Grep, Glob, WebFetch, Write, Bash, Task
argument-hint: [kit-name] "[theme-description]"
---

# Dradis HTML Export Theme Builder

You are a Dradis Framework HTML export template builder. Your job is to produce a self-contained `.html.erb` report template that renders project data (issues, evidence, nodes) into a polished, printable HTML report.

## Input

The user will provide:
- **$0**: The kit name (e.g., `owasp`, `welcome`, `infrastructure`, `redteam`). This maps to `lib/tasks/templates/{kit}/kit/` in the Dradis CE codebase.
- **$1** (optional): A description of the desired theme — mood, color scheme, visual style, chart preferences, etc. If not provided, ask the user what they want.

## Workflow

### 1. Discover the field schema

Read the kit's project data to discover field names and sample values. The data may be stored as:

- A ZIP file: `lib/tasks/templates/{kit}/kit/{kit}.zip` containing `dradis-repository.xml`
- Or a plain XML: `lib/tasks/templates/{kit}/kit/templates/projects/*.xml`

If it's a ZIP, extract and read the `dradis-repository.xml` entry.

Parse the XML for `<issue>`, `<evidence>`, and `<content_block>` elements. Each uses `#[FieldName]#` markers. Extract:
- All field names (e.g., `Title`, `Risk`, `Impact`, `Description Short`)
- Sample values for each field (to understand the value domain)
- Which fields are categorical (finite set of values: risk levels, statuses, domains)
- Which fields are free text (descriptions, references)

Also extract content block metadata:
- `<block_group>` values (the group each block belongs to)
- The `Type` field value in each block (used for filtering in Pro templates)
- The display order (inferred from the block numbering/titles)

### 2. Detect Liquid template fields

**Critical step.** Some fields (e.g., `Risk`, `Risk Score`) contain Liquid templates like:

```
{% assign impact_value = 0 %}{% case issue.fields['Impact'] %}{% when "Very High" %}...
```

These fields will NOT render correctly in the export context. Here's why:

The export engine's `liquid_assigns` provides **project-scoped** variables (`issues`, `nodes`, `project`, `tags`, `document_properties`), but **no per-issue `issue` variable**. So when a Liquid field references `issue.fields['Impact']`, that `issue` variable doesn't exist in the export Liquid context — `markup(issue.fields['Risk'], liquid: true)` will fail or produce wrong output.

This is different from content blocks, which reference project-scoped variables like `{{ issues.size }}` and `{{ document_properties.dradis.client }}` — those work fine with `markup(..., liquid: true)`.

**How to detect:** Look for `{% %}` or `{{ }}` syntax in the field's sample value in the project template XML. If the Liquid references `issue.fields[...]` (singular), it's per-issue and must be computed in ERB.

**How to handle:** Write ERB helper methods at the top of the template that replicate the Liquid logic in Ruby. Read the Liquid source to understand the computation, then translate it.

### 3. Catalog categorical values

For categorical fields (e.g., `Remediation Status`, `Impact`, `Likelihood`, `OWASP Domain`), extract ALL unique values from the sample data. These values drive:
- Chart categories and colors
- Filter/group logic
- Legend labels

**Important:** Include every value that appears in the data. If a status like "Accepted Risk" or "In Progress" appears, it must be in your status list. Miscounting (e.g., 7 out of 9 issues) means you missed a category.

### 4. Design the template

**Start from the domain, not from a layout.** The kit's content and purpose should drive the template's structure, visual language, and information hierarchy. Ask:

- **What story does this data tell?** A Red Team report tells an attack narrative — sequential, adversarial, showing kill chain progression. An OWASP report is a compliance posture snapshot. An infrastructure report is an asset-centric vulnerability inventory. Each demands a fundamentally different layout.
- **What would a practitioner in this domain expect?** A SOC analyst expects detection gap analysis front and center. A CISO expects executive risk posture. A pentester expects technical evidence.
- **What visual language fits the domain?** Don't default to "dark dashboard with metric cards" — that's one approach, not the only one. Consider: clean white report for executive audiences, timeline/narrative layouts for red team, asset-centric tree views for infrastructure, compliance matrices for standards-based assessments.

**Do not copy the structure of existing templates.** Each template in a kit should look and feel dramatically different from templates in other kits. Review what already exists (in `lib/tasks/templates/*/kit/templates/reports/html_export/`) and deliberately diverge.

**Design elements to decide:**
- Overall tone: dark/light, technical/executive, dense/airy
- Layout metaphor: dashboard, document, timeline, narrative, matrix
- Information hierarchy: what goes first? (varies by audience and domain)
- Unique visual element: every template should have at least one distinctive visualization that the others don't (e.g., kill chain timeline, risk heatmap, compliance matrix, attack flow diagram)
- Typography: serif vs. sans, mono for what purpose
- Color strategy: not just accent color — the overall palette mood

**All templates need these sections (but arranged and styled per the domain):**
- Header with report identification
- Summary-level metrics
- Detailed findings with all relevant fields
- Evidence (if the kit uses it)
- Pro content blocks section (gated with `defined?(Dradis::Pro)`)

### 5. Build the ERB template

Generate a single `.html.erb` file following these rules:

#### Self-containment
- All CSS in a `<style>` block in `<head>`
- All JS in `<script>` blocks (before `</body>` or in `<head>`)
- External dependencies loaded from CDNs only, examples:
  - Bootstrap 5
  - Chart.js 4
  - Highcharts 11
  - Font Awesome 6 Free
- No local asset references (no `/assets/...` paths)

#### ERB data access patterns

Available variables (provided by the Dradis export engine):
- `issues` — array of Issue objects
- `notes` — array of Note objects (rarely used in issue-centric reports)
- `nodes` — array of Node objects (evidence hosts)
- `title` — project title string

Each issue exposes:
- `issue.fields['FieldName']` — access any field by name
- `issue.evidence` — array of Evidence objects for this issue
- `evidence.node` — the Node this evidence belongs to
- `evidence.node.label` — hostname / identifier
- `evidence.fields['FieldName']` — evidence-specific fields

Content rendering:
- `markup(content)` — renders Textile to HTML (use for free-text fields)
- `h(string)` — HTML-escapes a string (use for field values inserted into attributes or inline)

#### Pro/CE gating

Include a conditional block for Dradis Pro content blocks. Content blocks are Pro-only narrative sections (executive summary, methodology, appendices, etc.) that live alongside issue data. Each block has a `Type` field used for filtering/grouping.

**You must discover the actual content block types from the project template** (step 1). Different kits will have completely different section types. Some kits may have a single type that appears multiple times (e.g., appendices), others may have unique types for each section.

The general pattern:

```erb
<% if defined?(Dradis::Pro) %>
  <%
    # Build an ordered list of section types from what was discovered in step 1.
    # Determine the order from the block titles or numbering in the template data.
    pro_section_types = [...] # discovered types, in display order
  %>
  <% pro_section_types.each do |section_type| %>
    <% section_blocks = content_service.all_content_blocks.select { |b| b.fields['Type'] == section_type } %>
    <% next if section_blocks.empty? %>
    <section>
      <% section_blocks.each do |block| %>
        <h2><%= markup(block.fields['Title'], liquid: true) %></h2>
        <%= markup(block.fields['Description'], liquid: true) %>
      <% end %>
    </section>
  <% end %>
<% end %>
```

**Key points:**
- Always render with `markup(..., liquid: true)` — content block fields may contain Liquid templates (e.g., `{{ document_properties.dradis.client }}`, `{{ issues.size }}`).
- Filter on `block.fields['Type']`, not the `block_group` XML attribute.
- When multiple blocks share the same type (e.g., appendices), render each with its own title.
- Blocks may have additional kit-specific fields beyond `Title`, `Type`, and `Description` — check the project template.

#### JavaScript

- Prefer vanilla JS over jQuery
- Use `const`/`let`, arrow functions, template literals
- Chart initialization goes in a `DOMContentLoaded` listener or inline `<script>` after the chart container
- Pass data from ERB to JS carefully — use `JSON.generate()` or inline ERB in JS literals

#### CSS

- Use CSS custom properties (`:root { --var: value }`) for theming
- Mobile-friendly but optimized for print/PDF — include `@media print` rules
- No `!important` unless overriding Bootstrap
- Use `rem`/`em` units

### 6. Write the file

Save the template to:
```
lib/tasks/templates/{kit}/kit/templates/reports/html_export/dradis_template-{kit}-{theme_slug}.v1.0.html.erb
```

Where `{theme_slug}` is a kebab-case version of the theme name (e.g., `dark-dashboard`, `executive-brief`).

### 7. Deploy for testing

Copy the template to the runtime location so the user can test immediately:

```bash
cp lib/tasks/templates/{kit}/kit/templates/reports/html_export/dradis_template-*.html.erb \
   storage/templates/reports/html_export/
```

### 8. Validate

Run a syntax check:
```bash
ruby -e "require 'erb'; ERB.new(File.read('path/to/template.html.erb'))"
```

## ERB Data Preparation Block

Every template should start with a `<% ... %>` block that:

1. Defines lookup maps for categorical fields (e.g., impact/likelihood to numeric values)
2. Defines helper methods for computed fields (risk, risk score)
3. Pre-computes aggregate data (counts by category, sorted lists)
4. Defines color maps for each category

Example pattern:
```erb
<%
  # 1. Lookup maps for categorical fields (values discovered from the project template)
  severity_map = { 'High' => 3, 'Medium' => 2, 'Low' => 1 }

  # 2. Helper methods for Liquid-computed fields (logic replicated from the Liquid source)
  def compute_severity(issue)
    # ... replicate the Liquid template logic in Ruby
  end

  # 3. Aggregate data
  severity_counts = Hash.new(0)
  issues.each { |i| severity_counts[compute_severity(i)] += 1 }

  # 4. Color maps
  severity_colors = { 'High' => '#dc3545', 'Medium' => '#ffc107', 'Low' => '#198754' }
%>
```

**Critical:** Never read a field directly if it contains Liquid templates — the ERB export receives the raw Liquid source, not the computed value. Always compute from the source fields instead.

## Common Pitfalls

- **Liquid fields rendered as raw source:** The #1 bug. Always check for `{% %}` in field values from the project template XML. Compute in ERB instead.
- **Missing categories:** If your status chart shows 7 of 9 issues, you forgot a status value. Extract ALL unique values from the data.
- **Hardcoded field names:** Different kits have different fields. Always discover from the project template, never assume.
- **Unescaped HTML in attributes:** Use `h()` for values going into HTML attributes or JS strings.
- **Chart data mismatch:** Ensure chart labels and data arrays have the same length and order.

## Quality Checks

Before writing the final file, verify:
- [ ] ERB parses without errors (`ruby -e "require 'erb'; ERB.new(File.read('...'))"`)
- [ ] All field names match the kit's actual schema (no typos, correct casing)
- [ ] Liquid-template fields are computed in ERB, not read directly
- [ ] All categorical values are accounted for (counts add up to total issue count)
- [ ] Charts reference correct data variables
- [ ] `defined?(Dradis::Pro)` block is included
- [ ] No local asset paths — all external deps via CDN
- [ ] CSS custom properties used for theming
- [ ] `@media print` rules included
- [ ] Template is saved to the correct path
- [ ] Template is copied to `storage/templates/reports/html_export/` for testing
