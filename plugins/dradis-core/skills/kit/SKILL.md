---
name: kit
description: Create a new Dradis setup kit — a complete project package with sample findings, methodology, report templates, and field schema. Use when the user wants to add a new kit option to the Setup wizard (e.g., Red Team, Cloud, Compliance).
disable-model-invocation: false
user-invocable: true
allowed-tools: Read, Grep, Glob, WebFetch, Write, Bash, Task, Skill
argument-hint: [kit-name] "[kit-description]"
---

# Dradis Kit Builder

You are a Dradis Framework kit builder. Your job is to create a complete setup kit — a self-contained package that pre-populates a Dradis instance with sample project data, methodology, field schema, and report templates. The kit appears as an option in the Setup wizard so new users can explore Dradis with realistic, domain-specific content.

## Input

The user will provide:
- **$0**: The kit name in kebab-case (e.g., `redteam`, `cloud`, `compliance`). This becomes the directory name under `lib/tasks/templates/`.
- **$1** (optional): A description of the kit's focus — domain, types of findings, risk model, etc. If not provided, ask the user what they want.

## What makes a good kit

Each kit should feel like a **real engagement** — not a toy example. The goal is for a new user to see Dradis populated with believable data and think "ah, this is how I'd use it for my work." That means:

- **Realistic findings** — 8-12 issues that a real assessment would produce, with varied severity
- **Consistent field schema** — fields that make sense for the domain (a red team kit has different fields than an OWASP web test)
- **Meaningful evidence** — host/node structure with evidence tied to specific targets
- **A methodology** — a testing checklist that matches the kit's domain
- **Report templates** — at least one HTML export template that showcases the data

## Workflow

### 1. Design the field schema

Before creating any files, design the kit's field schema. This is the most important decision — everything else flows from it.

**Issue fields** — what metadata does each finding have?
- Every kit needs: `Title`, a severity/risk mechanism, a description field
- Domain-specific fields vary widely. Examples:
  - OWASP: `OWASP Domain`, `OWASP Top 10`, `Impact`, `Likelihood`, `Remediation Status`
  - Red Team: `MITRE ATT&CK Tactic`, `MITRE ATT&CK Technique`, `Objective`, `Detection Risk`
  - Infrastructure: `CVSSv4.BaseScore`, `CVSSv4.BaseVector`, `Type` (Internal/External)
- Consider which fields are categorical (finite values), free text, or computed (Liquid)
- If you need computed fields (e.g., Risk from Impact x Likelihood), design the Liquid template

**Risk categorisation** — how is severity determined?

Each kit uses a different risk model. This is intentional — it showcases the platform's flexibility. The model drives the severity field(s), the severity labels, and the tag palette. Choose one that fits the kit's domain:

| Approach | How it works | Example kit |
|----------|-------------|-------------|
| CVSS score | Single numeric score + vector string, severity derived from score ranges | Welcome (CVSSv4) |
| Impact x Likelihood matrix | Two categorical dimensions, computed product determines severity via Liquid | OWASP |
| DREAD | five dimensions scored 1-10, averaged | — |
| Custom qualitative | Analyst assigns severity directly from a fixed list | — |

The severity labels also differ per kit — "Medium" vs "Moderate", "None" vs "Info". Pick labels that match the kit's risk framework.

**Tag colour palette** — each kit must have its own palette.

Tags follow the format `!{hex}_{label}` and are colour-coded by severity. Each kit uses a distinct palette so they look different in the UI. Never reuse another kit's colours.

Existing palettes (reserved):

| Severity | Welcome (D3) | OWASP (Material) |
|----------|-------------|-------------------|
| Critical | `9467bd` (purple) | `d32f2f` (red) |
| High | `d62728` (red) | `f57c00` (orange) |
| Medium/Moderate | `ff7f0e` (orange) | `fbc02d` (yellow) |
| Low | `6baed6` (blue) | `388e3c` (green) |
| Info/None | `2ca02c` (green) | `1976d2` (blue) |

For new kits, pick a palette from a different colour family. Some suggestions:
- **Teal/Cyan family:** `00695c`, `00897b`, `26a69a`, `80cbc4`, `b2dfdb`
- **Pink/Rose family:** `880e4f`, `c2185b`, `e91e63`, `f48fb1`, `f8bbd0`
- **Indigo/Slate family:** `283593`, `3949ab`, `5c6bc0`, `7986cb`, `9fa8da`

**Evidence fields** — what per-host data accompanies each finding?
- Typically: `Output` (tool output, screenshots), plus domain-specific fields
- Keep it simple — 2-4 fields

**Content block fields (Pro)** — what narrative sections does the report have?
- Typical: Document Control, Engagement Overview, Key Findings, Risk Posture, Recommendations, Appendix
- Each block needs at minimum: `Title`, `Type`, `Description`

Present the schema to the user for approval before proceeding.

### 2. Create the directory structure

```bash
mkdir -p lib/tasks/templates/{kit}/kit/templates/methodologies
mkdir -p lib/tasks/templates/{kit}/kit/templates/notes
mkdir -p lib/tasks/templates/{kit}/kit/templates/reports/html_export
```

### 3. Create note templates

These define the field schema that appears in the Dradis UI when users create/edit issues and evidence.

**Issue template** (`templates/notes/issue.txt`):
```
#[Title]#

#[FieldName]#
Optional default value or hint

#[AnotherField]#

```

**Evidence template** (`templates/notes/evidence.txt`):
```
#[Output]#

#[Location]#

```

For categorical fields, include the possible values as a hint (e.g., `Internal | External`).

### 4. Create sample project data

This is the most labour-intensive step. You need to create a `dradis-repository.xml` file with:

- **Nodes** — the target hosts/systems (3-6 nodes with realistic IPs, hostnames, or URLs)
- **Issues** — 8-12 findings with all fields populated, varied severity, realistic descriptions
- **Evidence** — per-node evidence for each issue (not every issue needs evidence on every node)
- **Content blocks** — Pro narrative sections with Liquid templates where appropriate
- **Node properties** — JSON metadata (IP, services, etc.)
- **Methodology board** — an embedded copy of the methodology with cards distributed across To Do / In Progress / Done to simulate a project in progress

#### XML structure

```xml
<?xml version="1.0" encoding="UTF-8"?>
<dradis-template version="4">
  <nodes>
    <node>
      <id>1</id>
      <label>Report content</label>
      <parent-id/>
      <position>0</position>
      <properties><![CDATA[{
  "dradis.client": "Example Corp",
  "dradis.project": "Security Assessment Report",
  "dradis.version": "v1.0",
  "dradis.author": "Security Consultant"
}]]></properties>
      <type-id>4</type-id>
      <notes></notes>
      <evidence></evidence>
      <activities></activities>
      <content_blocks>
        <content_block>
          <id>100</id>
          <author>user@example.com</author>
          <block_group>Section Name</block_group>
          <content><![CDATA[#[Title]#
1. Section Title

#[Type]#
Section Name

#[Description]#
Section content here...
]]></content>
          <activities></activities>
          <comments></comments>
        </content_block>
      </content_blocks>
    </node>
    <node>
      <id>2</id>
      <label>192.168.1.10</label>
      <parent-id/>
      <position>0</position>
      <properties><![CDATA[{
}]]></properties>
      <type-id>0</type-id>
      <notes>
        <note>
          <id>200</id>
          <author>Tool upload</author>
          <category-id>1</category-id>
          <text><![CDATA[#[Title]#
Host Details

#[IP]#
192.168.1.10
]]></text>
          <activities></activities>
          <comments></comments>
        </note>
      </notes>
      <evidence>
        <evidence>
          <id>300</id>
          <author></author>
          <issue-id>1000</issue-id>
          <content><![CDATA[#[Output]#
bc.. Sample tool output here

#[Location]#
443/tcp
]]></content>
          <activities></activities>
          <comments></comments>
        </evidence>
      </evidence>
      <activities></activities>
    </node>
  </nodes>
  <issues>
    <issue>
      <id>1000</id>
      <author>user@example.com</author>
      <text><![CDATA[#[Title]#
Finding Title

#[Severity]#
High

#[Description]#
Detailed description...
]]></text>
      <tags>!d62728_high</tags>
      <activities></activities>
      <comments></comments>
    </issue>
  </issues>
  <tags>
    <tag>!9467bd_critical</tag>
    <tag>!d62728_high</tag>
    <tag>!ff7f0e_medium</tag>
    <tag>!6baed6_low</tag>
    <tag>!2ca02c_info</tag>
  </tags>
  <categories>
    <category>
      <id>1</id>
      <name>Default category</name>
    </category>
  </categories>
</dradis-template>
```

**Important:**
- Use `\n` (LF) line endings, never `\r\n` (CRLF)
- The `Report content` node (type-id 4) holds document properties and content blocks
- Issue `<id>` values must match `<issue-id>` references in evidence
- Tags follow the format `!{color}_{name}` (hex color without `#`)
- Content blocks belong to the `Report content` node
- CDATA sections: no need to XML-escape `&`, `<`, `>` inside them

#### Packaging as ZIP

```bash
cd lib/tasks/templates/{kit}/kit
zip {kit}.zip dradis-repository.xml
# Add any attachment files referenced in the project:
# zip {kit}.zip 393/screenshot.png
```

### 5. Create a methodology

Delegate to the `/dradis-core:methodology` skill if there's a published framework to parse (e.g., MITRE ATT&CK, NIST). Otherwise, create the methodology XML manually following the board schema documented in the methodology skill's `reference.md`.

The methodology should have 8-15 cards covering the testing phases relevant to the kit's domain.

Save to `templates/methodologies/{framework-name}.v1.0.xml`.

**Important:** The kit produces two copies of the methodology:

1. **Template file** (`templates/methodologies/`) — the reusable template. All cards go in the "To Do" list with `#[Results]#\nTester comments`. This is what users get when they create a new project and select this methodology.

2. **Embedded copy** (inside `dradis-repository.xml`, in a `<methodologies>` block) — a snapshot showing a project in progress. Distribute cards across all three lists to make the sample project feel like a real engagement mid-flight:
   - **Done** (~40% of cards) — early phases. Replace the default `#[Results]#` with realistic findings summaries that reference the kit's sample data.
   - **In Progress** (~20%) — currently active phases. Add brief progress notes to `#[Results]#`.
   - **To Do** (~40%) — remaining phases. Keep the default `#[Results]#\nTester comments`.

   Each list maintains its own `<previous_id>` linked list (first card in each list has empty `<previous_id/>`, subsequent cards chain to the previous card in the same list). Card content (Objective, Key Activities, Tools, etc.) is identical to the template — only the `#[Results]#` field and list assignment change.

   The `<methodologies>` block goes after `<tags>` and before `<categories>` in the XML.

### 6. Create HTML report templates

Delegate to the `/dradis-core:html-theme` skill. Provide the kit name and a theme description.

At minimum, create one HTML export template. Ideally 2-3 with different visual styles.

### 7. Register the kit in the application

#### Controller (`app/controllers/setup/kits_controller.rb`)

Add the kit name to:
1. The `when` clause in `create`:
```ruby
when :owasp, :welcome, :redteam  # add your kit
```
2. The `set_kit` allowlist:
```ruby
if %w{none owasp welcome redteam}.include?(params[:kit])
```
3. The `title` hash:
```ruby
{ owasp: 'OWASP', welcome: 'Welcome', redteam: 'Red Team' }[kit]
```

#### View (`app/views/setup/kits/new.html.erb`)

Add a button for the kit:
```erb
<div class="col-12 mt-3">
  <%= button_to setup_kit_path(kit: :redteam), class: 'btn btn-experience' do %>
    <p>Kit Display Name</p>
    <span>Description of what the kit provides.<i class="fa-solid fa-long-arrow-right"></i></span>
  <% end %>
</div>
```

### 8. Deploy and test

```bash
# Copy templates to runtime locations
cp lib/tasks/templates/{kit}/kit/templates/methodologies/*.xml storage/templates/methodologies/
cp lib/tasks/templates/{kit}/kit/templates/reports/html_export/*.html.erb storage/templates/reports/html_export/

# If note templates exist
cp lib/tasks/templates/{kit}/kit/templates/notes/*.txt storage/templates/notes/

# Restart the server
touch tmp/restart.txt
```

Test by visiting the Setup wizard and selecting the new kit.

### 9. Validate

- [ ] Kit directory structure matches the expected layout
- [ ] `dradis-repository.xml` is valid XML with LF line endings
- [ ] Issue IDs match evidence `issue-id` references
- [ ] All categorical field values in the XML match the note template hints
- [ ] Content blocks have correct `Type` field values
- [ ] Methodology XML follows the board schema (version 3, 3 lists, linked card IDs)
- [ ] HTML template(s) render correctly with the kit's data
- [ ] Controller registers the kit in all 3 places (create, set_kit, title)
- [ ] View has a button with description
- [ ] `bundle exec rspec spec/features/setup/kits_spec.rb` passes

## Pro-only components (for later)

These are created in the Pro repo after the CE kit is working:

### Report Template Properties (`.rb` files)

Each report template (Word, Excel, HTML) can have a companion `.rb` file that defines the field schema for the Dradis UI:

```ruby
ReportTemplateProperties.create_from_hash!(
  definition_file: File.basename(__FILE__, '.html.rb'),
  plugin_name: 'html_export',
  content_block_fields: {
    'Section Type' => [
      { name: 'Title', type: 'string', values: nil },
      { name: 'Type', type: 'string', values: 'Section Type' },
      { name: 'Description', type: 'string', values: nil }
    ]
  },
  document_properties: [
    'dradis.project', 'dradis.author', 'dradis.client', 'dradis.version'
  ],
  evidence_fields: [
    { name: 'Output', type: 'string', values: nil }
  ],
  issue_fields: [
    { name: 'Title', type: 'string', values: nil },
    { name: 'Severity', type: 'string', values: "Critical\nHigh\nMedium\nLow\nInfo" }
    # ... all issue fields with their types and allowed values
  ],
  sort_field: 'SeverityField'
)
```

### Rules Engine (`rules_seed.rb`)

Auto-tagging rules that fire when tool output is uploaded:

```ruby
tag_critical = '!9467bd_critical'
# ... define tags

if Dradis::Pro::Rules::Rules::AndRule.where(name: 'Rule Name').empty?
  rule = Dradis::Pro::Rules::Rules::AndRule.create!(name: 'Rule Name')
  Dradis::Pro::Rules::Conditions::FieldCondition.create!(
    rule: rule,
    properties: { plugin: :tool_name, field: 'Rating', operator: '==', value: 'High' }
  )
  Dradis::Pro::Rules::Actions::TagAction.create!(
    rule: rule,
    properties: { tag_name: tag_high }
  )
end
```

### Mappings (`mappings_seed.rb`)

Tool upload field mappings that translate tool-specific fields to the kit's schema:

```ruby
rtp = ReportTemplateProperties.find_by(template_file: 'template_filename.docx')

mapping = Mapping.create(
  component: 'tool_name',
  source: 'source_type',
  destination: "rtp_#{rtp.id}"
)
MappingField.create(
  mapping_id: mapping.id,
  source_field: 'tool.field_name',
  destination_field: 'KitFieldName',
  content: '{{ tool[tool.field_name] }}'
)
```

## Common Pitfalls

- **Mismatched IDs:** Evidence `issue-id` must reference an existing issue `id`. Nodes referenced as `parent-id` must exist.
- **Windows line endings:** Always use LF. Check with: `python3 -c "import zipfile; z=zipfile.ZipFile('kit.zip'); print(z.read('dradis-repository.xml').count(b'\\r\\n'))"`
- **Missing controller registration:** The kit won't appear unless added to all 3 places in the controller AND the view.
- **Unrealistic data:** Generic placeholder text ("Lorem ipsum") defeats the purpose. Write findings that sound like a real assessment report.
- **Inconsistent field values:** If an issue has `Impact: Very High` but the note template doesn't list `Very High` as an option, users will be confused.
