# Kit Builder — Technical Reference

This document defines the file structure, XML schema, and registration points for Dradis setup kits.

## Kit Directory Structure

```
lib/tasks/templates/{kit}/kit/
├── {kit}.zip                              # Project data package
├── mappings_seed.rb                       # Pro only: tool field mappings
├── rules_seed.rb                          # Pro only: rules engine auto-tagging
└── templates/
    ├── methodologies/
    │   └── {framework}.v{X.Y}.xml         # Methodology board(s)
    ├── notes/
    │   ├── issue.txt                      # Issue field schema
    │   └── evidence.txt                   # Evidence field schema
    ├── projects/
    │   └── {kit}-project-template.xml     # Project template (Pro: field definitions for UI)
    └── reports/
        ├── html_export/
        │   ├── *.html.erb                 # HTML report templates
        │   └── *.html.rb                  # Pro: RTP properties for HTML templates
        ├── word/
        │   ├── *.docx                     # Pro: Word report templates
        │   └── *.rb                       # Pro: RTP properties for Word templates
        └── excel/
            ├── *.xlsx or *.xlsm           # Pro: Excel report templates
            └── *.rb                       # Pro: RTP properties for Excel templates
```

## Project Data Package (ZIP)

The ZIP file contains `dradis-repository.xml` and any attachment files (screenshots, etc.).

### XML Schema

```xml
<?xml version="1.0" encoding="UTF-8"?>
<dradis-template version="4">
  <nodes>...</nodes>
  <issues>...</issues>
  <tags>...</tags>
  <methodologies>...</methodologies>
  <categories>...</categories>
</dradis-template>
```

### Node Types

| type-id | Purpose | Notes |
|---------|---------|-------|
| 0 | Generic node | Default for hosts/targets |
| 1 | Host node | IP address nodes |
| 4 | Report content | Document properties + content blocks |

The `Report content` node (type-id 4) is special:
- Its `properties` JSON holds document properties used by Liquid templates
- Content blocks are nested inside this node
- There should be exactly one Report content node per project

### Document Properties

Stored as JSON in the Report content node's `<properties>`:

```json
{
  "dradis.client": "Client Name",
  "dradis.project": "Project Title",
  "dradis.version": "v1.0",
  "dradis.author": "Author Name",
  "dradis.start_date": "1/02/2026",
  "dradis.end_date": "14/02/2026"
}
```

These are accessible in Liquid templates as `{{ document_properties.dradis.client }}`.

### Issue XML

```xml
<issue>
  <id>1000</id>
  <author>user@example.com</author>
  <text><![CDATA[#[Title]#
Finding Title

#[Severity]#
High

#[Description]#
Detailed description in Textile markup.
]]></text>
  <tags>!d62728_high</tags>
  <activities></activities>
  <comments></comments>
</issue>
```

### Evidence XML

Evidence lives inside the node it belongs to:

```xml
<evidence>
  <id>300</id>
  <author></author>
  <issue-id>1000</issue-id>
  <content><![CDATA[#[Output]#
bc.. Tool output here

#[Location]#
443/tcp
]]></content>
  <activities></activities>
  <comments></comments>
</evidence>
```

`issue-id` must reference a valid `<issue><id>`.

### Content Block XML

Content blocks live inside the Report content node:

```xml
<content_block>
  <id>100</id>
  <author>user@example.com</author>
  <block_group>Section Name</block_group>
  <content><![CDATA[#[Title]#
1. Section Title

#[Type]#
Section Name

#[Description]#
Narrative content. Can use Textile markup and Liquid templates like
{{ document_properties.dradis.client }} or {{ issues.size }}.
]]></content>
  <activities></activities>
  <comments></comments>
</content_block>
```

### Tags

Tags are colour-coded severity labels. Format: `!{hex_color}_{label}` (hex without `#`).

**Each kit must define its own colour palette.** This is a deliberate design choice — different palettes reinforce that each kit is a distinct experience.

```xml
<tags>
  <tag>!{hex}_critical</tag>
  <tag>!{hex}_high</tag>
  <tag>!{hex}_moderate</tag>
  <tag>!{hex}_low</tag>
  <tag>!{hex}_info</tag>
</tags>
```

The severity label in the tag name should match the kit's risk model labels (e.g., `medium` not `moderate` if the kit uses CVSS terminology).

#### Existing palettes (do not reuse)

| Severity | Welcome (D3 palette) | OWASP (Material palette) |
|----------|---------------------|--------------------------|
| Critical | `9467bd` purple | `d32f2f` red |
| High | `d62728` red | `f57c00` orange |
| Medium/Moderate | `ff7f0e` orange | `fbc02d` yellow |
| Low | `6baed6` blue | `388e3c` green |
| Info/None | `2ca02c` green | `1976d2` blue |

#### Taggings

Tags link to issues via `<tagging>` elements inside the `<tag>` block. The `taggable-id` is the issue's `<id>` and `taggable-type` is `Note` (issues are stored as notes internally):

```xml
<tag>
  <id>1</id>
  <name>!d32f2f_critical</name>
  <taggings>
    <tagging>
      <taggable-id>1000</taggable-id>
      <taggable-type>Note</taggable-type>
    </tagging>
  </taggings>
</tag>
```

### Categories

```xml
<categories>
  <category>
    <id>1</id>
    <name>Default category</name>
  </category>
</categories>
```

Category ID 1 is the standard note category.

### Embedded Methodology Board

The project ZIP should include a `<methodologies>` block containing a methodology board with cards distributed across the three lists to simulate a project in progress. This goes after `<tags>` and before `<categories>`.

```xml
<methodologies>
<board version="4">
  <id>2000</id>
  <name>Methodology Name</name>
  <node_id/>
  <list>
    <id>2001</id><name>To Do</name><previous_id/>
    <card>
      <id>2009</id>
      <name>Upcoming Phase</name>
      <description><![CDATA[#[Results]#
Tester comments

#[Objective]#
...
]]></description>
      <due_date/>
      <previous_id/>
    </card>
    <!-- more To Do cards, each pointing to the previous card in this list -->
  </list>
  <list>
    <id>2002</id><name>In Progress</name><previous_id>2001</previous_id>
    <card>
      <id>2006</id>
      <name>Active Phase</name>
      <description><![CDATA[#[Results]#
Brief progress notes referencing what has been done so far.

#[Objective]#
...
]]></description>
      <due_date/>
      <previous_id/>
    </card>
    <!-- more In Progress cards -->
  </list>
  <list>
    <id>2003</id><name>Done</name><previous_id>2002</previous_id>
    <card>
      <id>2001</id>
      <name>Completed Phase</name>
      <description><![CDATA[#[Results]#
Realistic findings summary referencing the kit's sample issues and evidence.

#[Objective]#
...
]]></description>
      <due_date/>
      <previous_id/>
    </card>
    <!-- more Done cards -->
  </list>
</board>
</methodologies>
```

**Key rules:**
- Each list has its own `<previous_id>` chain — the first card in each list has `<previous_id/>`, subsequent cards point to the previous card in the same list (not across lists).
- "Done" cards should have `#[Results]#` populated with realistic summaries that reference the kit's sample findings.
- "In Progress" cards should have brief progress notes.
- "To Do" cards keep the default `#[Results]#\nTester comments`.
- Card content (all fields except `#[Results]#`) is identical to the methodology template file.

**Distinction from the template file:** The methodology template (`templates/methodologies/`) has all cards in "To Do" with default results — it's the clean reusable template. The embedded copy in the ZIP shows a project mid-flight.

## Note Templates

### Issue template (`templates/notes/issue.txt`)

Defines the field schema for issues in the Dradis UI:

```
#[Title]#

#[Severity]#
Critical | High | Medium | Low | Info

#[Description]#

#[Solution]#

#[References]#

```

Categorical fields include their allowed values as a pipe-separated hint after the field marker.

### Evidence template (`templates/notes/evidence.txt`)

```
#[Output]#

#[Location]#

```

## Controller Registration

Three places in `app/controllers/setup/kits_controller.rb`:

### 1. `create` action — `when` clause

```ruby
when :owasp, :welcome, :newkit
  kit_folder = Rails.root.join('lib', 'tasks', 'templates', @kit.to_s)
  logger = Log.new.info("Loading #{title(@kit)} kit...")
  User.create!(email: 'adama@dradis.com') unless defined?(Dradis::Pro)
  KitImportJob.perform_later(kit_folder.to_s, logger: logger)
```

### 2. `set_kit` — allowlist

```ruby
if %w{none owasp welcome newkit}.include?(params[:kit])
```

### 3. `title` — display name hash

```ruby
{ owasp: 'OWASP', welcome: 'Welcome', newkit: 'New Kit' }[kit]
```

## View Registration

In `app/views/setup/kits/new.html.erb`:

```erb
<div class="col-12 mt-3">
  <%= button_to setup_kit_path(kit: :newkit), class: 'btn btn-experience' do %>
    <p>Display Name</p>
    <span>Description of what the kit provides and who it's for.<i class="fa-solid fa-long-arrow-right"></i></span>
  <% end %>
</div>
```

## KitImportJob Processing Order

The import job (`app/jobs/kit_import_job.rb`) processes kit contents in this order:

1. Copy kit to working directory (or extract ZIP if file)
2. Import methodology templates → `storage/templates/methodologies/`
3. Import note templates → `storage/templates/notes/`
4. Import project package (the `.zip` inside `kit/`) → creates a Project with nodes, issues, evidence
5. Import project templates → `storage/templates/projects/`
6. Import report template files → `storage/templates/reports/{plugin}/`
7. Pro only: Import report template properties (`.rb` files)
8. Pro only: Import rules (`rules_seed.rb`)
9. Pro only: Import mappings (`mappings_seed.rb`)
10. Pro only: Assign Word RTP to the project

## Liquid Fields in Project Data

Issue fields can contain Liquid templates for computed values. These render correctly in the Dradis web UI (where the per-issue Liquid context exists) but NOT in the export context (see html-theme skill reference for details).

Example — a computed Risk field:

```
#[Risk]#
{% assign impact_value = 0 %}{% case issue.fields['Impact'] %}{% when "Very High" %}{% assign impact_value = 5 %}{% when "High" %}{% assign impact_value = 4 %}...{% endcase %}{% assign score = impact_value | times: likelihood_value %}{% if score >= 20 %}Critical{% elsif score >= 15 %}High{% elsif score >= 8 %}Moderate{% elsif score >= 3 %}Low{% else %}Info{% endif %}
```

**Important:** Liquid templates in issue fields must be single-line (no line breaks within the `{% %}` blocks) — the field parser splits on newlines.

## Existing Kits (for reference)

| Kit | Domain | Risk model | Severity labels | Tag palette |
|-----|--------|-----------|-----------------|-------------|
| `welcome` | Infrastructure pentest | CVSS v4 (single score + vector) | Critical, High, Medium, Low, None | D3 (purple → red → orange → blue → green) |
| `owasp` | Web/API/Mobile security | Impact x Likelihood matrix (computed via Liquid) | Critical, High, Moderate, Low, Info | Material Design (red → orange → yellow → green → blue) |

### Welcome kit fields

`Title`, `CVSSv4.BaseScore` (number), `CVSSv4.BaseVector`, `CVSSv4.BaseSeverity`, `Type` (Internal/External), `Description`, `Solution`, `References`, `plugin`, `plugin_id`

### OWASP kit fields

`Title`, `OWASP Domain` (Web/API/Mobile), `OWASP Top 10`, `Description Short`, `Description Long`, `Remediation Status` (Open/In Progress/Partially Remediated/Accepted Risk/Remediated), `Impact` (Very Low → Very High), `Likelihood` (Very Low → Very High), `Risk` (Liquid computed), `Risk Score` (Liquid computed), `References`

## Related Skills

- `/dradis-core:methodology` — creates methodology XML from external sources
- `/dradis-core:html-theme` — creates HTML export templates for a kit
