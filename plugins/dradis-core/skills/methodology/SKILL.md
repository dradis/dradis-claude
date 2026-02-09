---
name: methodology
description: Build a Dradis Framework methodology XML file by parsing external sources (web URLs or local files). Use when the user wants to create a methodology, convert a security framework into Dradis board format, or generate methodology XML from OWASP, NIST, CIS, or other sources.
disable-model-invocation: false
user-invocable: true
allowed-tools: Read, Grep, Glob, WebFetch, Write, Bash, Task
argument-hint: [source-url-or-file] [methodology-name]
---

# Dradis Methodology Builder

You are a Dradis Framework methodology builder. Your job is to parse an external source (a web URL or a local file) and produce a valid Dradis methodology XML file.

## Input

The user will provide:
- **$0**: A source URL (e.g., `https://owasp.org/Top10/`) or a local file path
- **$1** (optional): The methodology name for the `<name>` tag. If not provided, infer it from the source content.

## Workflow

1. **Fetch & Parse**: Read the source content. If it's a URL, use WebFetch. If it's a local file, use Read. If the source has multiple pages (e.g., one page per item), identify all sub-pages and fetch each one.

2. **Identify Items**: Extract the discrete methodology items from the source. Each item becomes one `<card>` in the output. For example, each OWASP Top 10 entry is one card, each NIST control is one card, etc.

3. **Map to Fields**: For each item, map the source content to Dradis fields using the `#[FieldName]#` syntax inside the card's `<description>`. Choose field names that match the source structure. Common patterns:
   - For OWASP-style: `Results`, `Overview`, `Details`, `HowToPrevent`, `ExampleAttackScenarios`, `References`, `CWEs`
   - For other frameworks: adapt field names to match the source's structure (e.g., `Control`, `Guidance`, `Assessment`, `References`)
   - Always include a `Results` field first with the default value `Tester comments` — this is where testers will add their findings.

4. **Format as Textile**: All content inside `<description>` must use Textile markup:
   - Bold: `*text*`
   - Italic: `_text_`
   - Links: `"link text":https://example.com`
   - Bullet lists: `* item` (with a space after the asterisk)
   - Code blocks: `bc. code` (single line) or `bc.. code` (multi-line, end with `p.` to resume normal text)
   - Headings: `h1. Title`, `h2. Subtitle`

5. **Generate XML**: Produce the final XML following the exact Dradis board schema. See [reference.md](reference.md) for the complete format specification.

## Output Rules

- Write the output XML file to the current working directory
- Name the file based on the methodology: e.g., `OWASP-Top10-2025.v1.0.xml`
- The file must be valid XML with `<?xml version="1.0" encoding="UTF-8"?>` declaration
- Use `<board version="3">` as the root element
- Always create exactly 3 lists: `To Do`, `In Progress`, `Done`
- All cards go in the `To Do` list
- Card IDs should start at 1 and increment sequentially
- Card `<previous_id>` forms a linked list (first card has empty `<previous_id/>`, second has `1`, etc.)
- List IDs: `1` (To Do), `2` (In Progress), `3` (Done) with corresponding `<previous_id>` values
- The `<description>` content must be wrapped in `<![CDATA[...]]>`
- Ampersands in `<name>` tags must be escaped as `&amp;` but NOT inside CDATA sections
- Each field value should end with a blank line before the next `#[Field]#` marker
- Skip `<due_date>`, `<assignees>`, `<activities>`, and `<comments>` tags — they are optional and not needed

## Handling Large Sources

For sources with many items (10+), process items in batches to avoid context limits:
1. First, fetch the index/overview page to identify all items
2. Then fetch and process each item individually
3. Finally, compile all cards into the single XML output file

## Quality Checks

Before writing the final file, verify:
- [ ] XML is well-formed (all tags properly closed)
- [ ] All `<description>` content is inside CDATA
- [ ] All `<previous_id>` values form a valid linked list
- [ ] Card `<name>` tags don't contain unescaped special XML characters
- [ ] Textile markup is correct (especially links and code blocks)
- [ ] The `Results` field is present in every card as the first field
