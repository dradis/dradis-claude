---
name: calculator
description: Create a Dradis risk calculator add-on (a dradis-calculator_* gem) by porting an external scoring model — a reference calculator page, a published spec, or a scoring table. Use when the user wants to add a new scoring system to Dradis alongside CVSS, DREAD and MITRE ATT&CK.
disable-model-invocation: false
user-invocable: true
allowed-tools: Read, Grep, Glob, WebFetch, Write, Edit, Bash, Task
argument-hint: [source-url-or-file] [calculator-name]
---

# Dradis Risk Calculator Builder

You are a Dradis Framework calculator builder. Your job is to produce a new
`dradis-calculator_*` add-on gem that ports an external scoring model into
Dradis, available both at the instance level and on each issue.

The defining constraint of this skill: **a calculator is a port, not a
design.** Someone else owns the model. Your output is correct only if it
produces the same numbers and the same verdicts as the reference for every
input. Everything below serves that.

## Input

The user will provide:
- **$0**: The source — a reference calculator URL (e.g. `https://aivss.owasp.org/ssvc.html`),
  a spec document, or a local file describing the scoring model.
- **$1** (optional): The calculator name in kebab-case (e.g. `aivss-ssvc`, `owasp-rr`).
  If not provided, infer it from the source and confirm with the user.

## Before you start: ask

Four decisions change the shape of the output and are cheap to ask about
up front. Ask them together, then build without further interruption:

1. **Naming** — the gem/folder name, the Ruby module, and the issue field
   prefix. See "Naming" in [reference.md](reference.md); there is a Zeitwerk
   trap for multi-acronym names that you must check before committing to one.
2. **State restoration** — how the form reloads a saved score: a vector
   string (CVSS, DREAD) or the individual issue fields (MITRE). See
   "Restoring saved state".
3. **Visual fidelity** — Bootstrap/Hera styling like the existing calculators,
   or a port of the reference page's own visuals. Default to the former.
4. **Extras** — which affordances from the reference page to carry over.
   Judge each against the three existing calculators rather than porting
   reflexively: a live score badge has precedent, a "reset to example"
   button does not.

## Workflow

### 1. Extract the model from the source

Fetch the source and pull out the scoring logic **verbatim**. For a
single-page calculator, download the HTML and read its `<script>` — the
decision tables, thresholds, formulas, labels and help text are usually all
there in literal form.

Capture every one of these that the source has:

- The input controls, their options, and each option's numeric value
- The formulas, in the source's own order of operations
- Any lookup or decision table, copied cell for cell
- Classification thresholds and the **order** their branches are tested in
- Every user-visible string: option labels, help text, rationale sentences,
  verdict names, timeline text
- The state the reference loads with

Rounding and formatting are part of the model, not presentation. If the
source does `Math.round(x * 100) / 100` and prints the bare result, do
exactly that — do not substitute `toFixed(2)`.

### 2. Choose a skeleton and clone it

Copy the closest existing calculator wholesale, then strip it:

- **DREAD** — simplest. Numeric inputs → derived score → field output. Best
  default for a scoring model.
- **MITRE** — data-driven selects, vanilla-JS class, in-place field patching.
  Best when the model is a taxonomy or the newest patterns are wanted.
- **CVSS** — multi-version, vendored upstream JS, per-field toggles. Only when
  the model genuinely needs that.

Remove the source calculator's `.git`, `lib/`, `app/`, `config/` and
`CHANGELOG.md`, keep `.github/`, `.gitignore`, `CONTRIBUTING.md`, `Gemfile`,
`LICENSE`, `Rakefile`, `CHANGELOG.template`.

### 3. Build the gem

Follow [reference.md](reference.md) for the file-by-file layout, the engine,
routes, controllers, model and view conventions, and the asset manifests.

Put every definition from the model — inputs, options, values, labels, help
text, factors, defaults, field names — in the `V1` model class. It is the
single source of truth. Views iterate over it; the JS never restates it.

### 4. Wire up both levels

An add-on is not done until it works in both places:

- **Instance level**: `base#index` at `/calculators/{name}`, plus a
  `_tools_menu.html.erb` partial
- **Issue level**: `issues#edit` / `issues#update`, plus an
  `issues/_show-tabs.html.erb` partial

Both partials are discovered automatically by `render_view_hooks` in
dradis-ce — no change to dradis-ce's views is needed. Render the same content
partials from both entry points so the two levels cannot drift.

### 5. Verify against the reference

**This is the step that matters, and it is not optional.** Reading the
ported code back to yourself proves nothing. Build a differential test that
drives the real reference implementation and your implementation over the
same inputs and compares every output.

See "Differential testing" in [reference.md](reference.md) for the harness.
Cover, at minimum:

- Every combination of the discrete inputs
- Every distinct value reachable by any derived/averaged quantity, so no
  classification branch or threshold boundary is missed
- A few thousand random inputs across the whole space
- Every user-visible string, not just the final number

Report the assertion count and the failure count. If you cannot build a
harness for a given source, say so plainly rather than claiming parity you
have not measured.

### 6. Wire into dradis-ce

Add the gem to the Calculators block in `dradis-ce/Gemfile`. For local
testing before the gem is published, use a path reference:

```ruby
gem 'dradis-calculator_{name}', path: '../dradis-calculator_{name}'
```

Confirm with the user before editing their `dradis-ce` checkout.

## Output Rules

- Output is a **new sibling directory** `dradis-calculator_{name}/`, never a
  change inside an existing calculator
- Mirror the existing calculators' structure and idiom wherever the model
  does not force a difference; where it does, say why
- `git init` the new folder — the gemspec computes `spec.files` from
  `git ls-files`, so without a repo it builds an empty gem
- Keep `CHANGELOG.md` to a single entry: `- Calculator: Add {name} calculator`
- Set `spec.authors = ['Dradis Team']`
- Version to match the sibling calculators (they ship in lockstep)
- Never commit or push unless the user asks

## Quality Checks

Before reporting done, verify:

- [ ] The differential test passes, and you have stated its assertion count
- [ ] Constants in `V1` match the source (check them programmatically, not by eye)
- [ ] A saved score reopens the form in the exact state it was saved in
- [ ] Malformed or partial field values fall back cleanly instead of raising
- [ ] Both `/calculators/{name}` and the issue tab render the same partials
- [ ] Route helpers resolve (`calculators_{name}_path`, `{name}_project_issue_path`)
- [ ] Every `data-behavior` hook in the views is read by the JS, and vice versa
- [ ] Ruby parses (`ruby -c`), JS parses (`node --check`)
- [ ] No leftover strings from the calculator you cloned
- [ ] The model class is the only place any definition appears
