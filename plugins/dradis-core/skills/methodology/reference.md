# Dradis Methodology XML Format Reference

This document defines the exact XML schema that Dradis Framework expects for methodology board files.

## Complete XML Schema

```xml
<?xml version="1.0" encoding="UTF-8"?>
<board version="3">
  <id>{board_id}</id>
  <name>{Methodology Name}</name>
  <node_id/>
  <list>
    <id>1</id>
    <name>To Do</name>
    <previous_id/>
    <card>
      <id>1</id>
      <name>{Card Title}</name>
      <description>
<![CDATA[#[Results]#
Tester comments

#[FieldName]#
Field content in Textile markup

]]></description>
      <due_date/>
      <previous_id/>
    </card>
    <card>
      <id>2</id>
      <name>{Second Card Title}</name>
      <description>
<![CDATA[#[Results]#
Tester comments

#[FieldName]#
More content here

]]></description>
      <due_date/>
      <previous_id>1</previous_id>
    </card>
  </list>
  <list><id>2</id><name>In Progress</name><previous_id>1</previous_id></list>
  <list><id>3</id><name>Done</name><previous_id>2</previous_id></list>
</board>
```

## Element Details

### `<board>`
- Root element. Use `version="3"`.
- Contains: `<id>`, `<name>`, `<node_id/>`, and exactly 3 `<list>` elements.

### `<list>`
- Represents a Kanban column.
- Always create 3 lists: **To Do** (id=1), **In Progress** (id=2), **Done** (id=3).
- `<previous_id>` defines ordering: To Do has none, In Progress points to 1, Done points to 2.
- Only the "To Do" list contains `<card>` elements.

### `<card>`
- Represents one methodology item (e.g., one OWASP Top 10 entry, one NIST control).
- `<id>`: Sequential integer starting at 1.
- `<name>`: Card title. XML-escape special characters (`&` → `&amp;`, `<` → `&lt;`).
- `<description>`: Contains CDATA-wrapped content with Dradis fields.
- `<due_date/>`: Always empty, include as self-closing tag.
- `<previous_id>`: Empty for first card, otherwise the `<id>` of the preceding card.
- Optional tags that can be omitted: `<assignees/>`, `<activities/>`, `<comments/>`.

### `<description>` — Dradis Fields

Inside the CDATA section, define fields using the `#[FieldName]#` syntax:

```
#[FieldName]#
Content for this field in Textile markup

#[AnotherField]#
More content here

```

**Rules:**
- Each field starts with `#[FieldName]#` on its own line
- Field content follows on subsequent lines
- Leave a blank line after field content before the next `#[...]#` marker
- The CDATA section should end with a blank line before `]]>`

### Standard Field Set (OWASP-style)

| Field | Purpose |
|-------|---------|
| `Results` | Always first. Default value: `Tester comments`. Where testers record findings. |
| `Overview` | Brief description of the item, statistics, notable CWEs. |
| `Details` | In-depth technical explanation. |
| `HowToPrevent` | Mitigation and prevention guidance. |
| `ExampleAttackScenarios` | Concrete attack examples. |
| `References` | External links to further reading. |
| `CWEs` | Related Common Weakness Enumerations with links. |

You are not limited to these fields. Adapt to match the source material's structure.

## Textile Markup Quick Reference

Dradis uses Textile for rich text inside fields. Key syntax:

### Inline Formatting
- **Bold**: `*text*`
- **Italic**: `_text_`
- **Bold italic**: `*_text_*`

### Links
```textile
"Link text":https://example.com/page
```

### Lists
```textile
* Bullet item 1
* Bullet item 2
** Nested bullet

# Numbered item 1
# Numbered item 2
```

### Code Blocks

Single-line:
```textile
bc. single line of code
```

Multi-line (note the double dots — everything after is code until a `p.` paragraph tag):
```textile
bc.. line one
line two
line three

p. Back to normal paragraph text.
```

### Headings
```textile
h1. Main Heading
h2. Sub Heading
h3. Sub-sub Heading
```

### Special Characters Inside CDATA
- Inside `<![CDATA[...]]>`, you do NOT need to XML-escape `&`, `<`, `>`.
- Use them literally: `&`, `<`, `>`.
- The ONLY sequence you cannot use inside CDATA is `]]>` (the CDATA closing delimiter).

### Special Characters in `<name>` Tags
- Outside CDATA, you MUST XML-escape: `&` → `&amp;`, `<` → `&lt;`, `>` → `&gt;`.

## Example: Minimal Card

```xml
<card>
  <id>1</id>
  <name>A01:2025 – Broken Access Control</name>
  <description>
<![CDATA[#[Results]#
Tester comments

#[Overview]#
Access control enforces policy such that users cannot act outside of their intended permissions.

#[Details]#
Common vulnerabilities include:

* Violation of the principle of least privilege
* Bypassing access control checks by modifying the URL
* Permitting viewing or editing someone else's account

#[HowToPrevent]#
* Except for public resources, deny by default.
* Implement access control mechanisms once and re-use them.
* Log access control failures, alert admins when appropriate.

#[References]#
* "OWASP Cheat Sheet: Authorization":https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html

]]></description>
  <due_date/>
  <previous_id/>
</card>
```
