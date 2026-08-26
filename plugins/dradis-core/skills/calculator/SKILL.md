---
name: calculator
description: Create a Dradis risk calculator add-on (a dradis-calculator_* gem) from a scoring model — a reference calculator, a published spec, a taxonomy, or a scoring table. Use when the user wants to add a new scoring system to Dradis alongside CVSS, DREAD and MITRE ATT&CK.
disable-model-invocation: false
user-invocable: true
allowed-tools: Read, Grep, Glob, WebFetch, Write, Edit, Bash, Task
argument-hint: [source-url-or-file] [calculator-name]
---

# Dradis Risk Calculator Builder

You are a Dradis Framework calculator builder. Your job is to produce a new
`dradis-calculator_*` add-on gem that brings an external scoring model into
Dradis, available both at the instance level and on each issue.

The defining constraint: **someone else owns the model.** Your output is
correct only if it agrees with that owner for every input. Two things follow,
and they drive the whole workflow — how faithfully you can reproduce the model
depends on what the source gives you, and how well you can *prove* you did
depends on what the source lets you check against.

## Input

The user will provide:
- **$0**: The source — a reference calculator, a spec document, a data feed, a
  scoring table, or a local file.
- **$1** (optional): The calculator name in kebab-case. If not provided, infer
  it from the source and confirm.

## Step 1 — Classify the source

Do this first. It decides how you build *and* how you verify, and getting it
wrong wastes the rest of the work.

| The source… | Strategy | Verification available |
|---|---|---|
| ships a usable implementation (JS library, calculator page with inline logic) | **Vendor it** | Differential vs. the reference |
| publishes a spec **with test vectors or worked examples** | Transcribe | The published vectors |
| is prose, a table, or a paper only | Transcribe | Hand-derived cases only |
| is a data feed (taxonomy, control catalogue) | Fetch and transform | Shape and referential checks |
| is the user's own model (an internal matrix, a spreadsheet, a description) | Transcribe from their description | **The user is the oracle** |

For a bespoke model the user owns, restate the whole model back to them — every
axis, every band, every label and the exact cell values — and get it confirmed
before building. That confirmation *is* the specification; without it there is
nothing to be correct against.

**Prefer vendoring.** If the model's owner publishes working code, copying it
verbatim removes an entire class of bug: there is no transcription to get
wrong, and upstream fixes are a re-copy rather than a re-read. This is what
the CVSS calculator does — it vendors FIRST's `cvsscalc31.js`, lookup tables
and help text (~214KB across two versions) and writes only a thin wrapper that
reads the form, calls upstream, and renders the result. Transcribe only when
there is no usable implementation, or when its licence forbids it.

Whichever applies, capture from the source: the input controls and their
values, the formulas in the source's own order of operations, any lookup or
decision table cell for cell, classification thresholds **and the order their
branches are tested in**, every user-visible string, and the state the
reference loads with.

Rounding and formatting are part of the model, not presentation. If the source
does `Math.round(x * 100) / 100` and prints the bare result, do exactly that.

## Step 2 — Classify the model's shape

This decides the UI, the `V1` model, and which sibling to clone.

| Shape | Example | UI | Clone |
|---|---|---|---|
| Discrete-option metrics | CVSS, AIVSS-SSVC | Button groups or selects per metric | CVSS / DREAD |
| Numeric scales | DREAD | Radio rows, or selects for longer scales | DREAD |
| Hierarchical taxonomy | MITRE ATT&CK | Dependent selects, backed by a data asset | MITRE |
| Lookup table / matrix | OWASP RR, 5×5 risk matrices | Axes in, cell out; show the cell hit | DREAD |
| Macrovector | CVSS v4 | Metrics in, vendored table out | CVSS |
| Decision tree | Triage trees | Dependent questions; record the path | none yet |
| Multi-version model | CVSS 3.0/3.1/4.0 | Version menu, one partial set per version | CVSS |

A model may combine these — AIVSS-SSVC is discrete-option inputs plus numeric
scales plus a decision table. Take the UI for each part from its own row.

Two shape details that are easy to miss and expensive to retrofit:

- **A decision tree is not a formula.** The outcome comes from the path, so
  later questions can depend on earlier answers. Whatever UI you choose, the
  path is part of the result: record it as an output field, because it is the
  justification a reader needs. No existing calculator implements this shape,
  so there is no house pattern to copy — take the UI from the reference and
  say what you based it on.
- **"Not defined" is a real value.** Most published models have a skip value
  with defined semantics — CVSS's `X` appears throughout its metrics and means
  "use the default weight", not "zero". Model it explicitly and make sure it
  round-trips; do not collapse it to blank or to the lowest option.

If the model needs an external dataset, follow MITRE: a script under
`scripts/` fetches and reduces upstream data to a JSON asset that ships with
the gem, and the JS fetches that asset via `asset_path`. Do not fetch from
upstream at runtime.

## Step 3 — Ask the open questions

Ask these together, then build without further interruption:

1. **Naming** — gem/folder, Ruby module, issue field prefix. See "Naming" in
   [reference.md](reference.md) and check the module round-trips through
   `camelize`/`underscore` *before* committing to one.
2. **State restoration** — vector string (CVSS, DREAD) or individual issue
   fields (MITRE). If the model defines a vector, use it.
3. **Visual fidelity** — Bootstrap/Hera styling like the existing calculators,
   or a port of the reference's own visuals. Default to the former.
4. **Extras** — which affordances from the reference to carry over. Judge each
   against the existing calculators rather than porting reflexively.

## Step 4 — Design the output fields

The fields the calculator writes are a **public interface**, not internal
bookkeeping. Kits, report templates and issue tables read them by name: the
`welcome` kit's HTML export does `issue.fields['CVSSv4.BaseScore'].to_f` and
colour-codes findings by it, and its issue note template lists the field so
every new issue has the slot.

So design for the consumer:

- **One numeric field** carrying the headline score, written bare so `.to_f`
  parses it — no units, no suffix, no thousands separator
- **One human-readable field** carrying the verdict or band, for grouping and
  for prose
- **The vector or path**, if the model has one, so a score can be audited and
  re-imported
- **The individual metrics**, so a report can explain how the score was reached

Choose the names once and treat them as frozen. Renaming a field silently
breaks every template that reads it; see "Output fields as an interface" in
[reference.md](reference.md) for how CVSS carries a legacy name to avoid
exactly that.

How many fields is a judgement call. CVSS and DREAD write a handful and write
them unconditionally. If your model produces enough that writing all of them
would bury the issue, consider letting the user pick — see "The field picker"
in [reference.md](reference.md) for the pattern. Raise it with the user rather
than deciding silently either way.

If the calculator is meant to feed a specific kit or report theme, say so —
`/dradis-core:kit` and `/dradis-core:html-theme` build the consuming side.

## Step 5 — Build the gem

Clone the sibling chosen in Step 2 and strip it: remove its `.git`, `lib/`,
`app/`, `config/` and `CHANGELOG.md`; keep `.github/`, `.gitignore`,
`CONTRIBUTING.md`, `Gemfile`, `LICENSE`, `Rakefile`, `CHANGELOG.template`.

Follow [reference.md](reference.md) for the layout and the per-file
conventions. Vendored code keeps its own rules: verbatim under a `vendor/`
directory, unmodified, so it can be re-copied when upstream changes.

Three rules carry most of the weight, and retrofitting any of them is painful:

- **`V1` is the single source of truth, browser included.** Every value you
  transcribed goes there — thresholds, labels, multipliers, lookup cells, and
  the ones that feel like UI (badge classes, verdict copy). Views iterate over
  `V1`; the JS receives one serialized config blob as a data attribute and
  restates no value of its own. Where the model is *rules* rather than a table,
  the branch order can stay in the JS — the numbers and strings it tests
  against still come from `V1`.
- **The server renders the field output.** `V1.field_output` builds the
  `#[Field]#` block and a `POST` endpoint serves it; the JS posts its computed
  values and swaps in the response. Do not re-implement dradis-ce's
  `FieldParser` regex on the client — that duplication drifts from the Ruby
  that has to parse it back.
- **Register inflections once**, in `lib/dradis-calculator_{name}.rb`, before
  `engine.rb` is required. An engine initializer is too late.

Also strip what the clone left behind: the sibling gemspec's obsolete
dependency comments, any leading blank line in `.gitignore`, and every string
naming the calculator you copied.

## Step 6 — Wire up both levels

An add-on is not done until it works in both places:

- **Instance level**: `base#index` at `/calculators/{name}`, plus `_tools_menu.html.erb`
- **Issue level**: `issues#edit` / `issues#update`, plus `issues/_show-tabs.html.erb`
- **Shared**: `base#fields`, the `POST` endpoint both levels use to render the
  field output

Both partials are discovered automatically by `render_view_hooks` — dradis-ce
needs no change. Render the same content partials from both entry points so
the two levels cannot drift.

## Step 7 — Verify, at the strongest tier the source allows

**Do not claim parity you have not measured.** Reading the code back to
yourself proves nothing.

Use the best tier Step 1 said was available:

1. **Differential** against a runnable reference — the strongest evidence
2. **Published test vectors** — samples, not coverage
3. **Hand-derived cases** — when no oracle exists, say so plainly
4. **User confirmation** — for a bespoke model, their sign-off on the restated
   model is what you are testing against

Then run the structural checks — a programmatic constants check, boundary
enumeration over each derived quantity, a round-trip through state
restoration, and agreement between the two levels.

"Verifying the port" in [reference.md](reference.md) has the technique for
each. **Report the tier, what it covered, and the counts.**

## Step 8 — Wire into dradis-ce

Add the gem to the Calculators block in `dradis-ce/Gemfile`. Before the gem is
published, use a path reference:

```ruby
gem 'dradis-calculator_{name}', path: '../dradis-calculator_{name}'
```

Confirm with the user before editing their `dradis-ce` checkout.

## House Style

These are the review conventions the dradis maintainers apply. Write to them
the first time.

**Ruby**

- **No alignment padding** — not in hashes, not in constants, not in routes.
  One space after the key; never pad values into columns.
- **Multiline hashes** once an entry carries more than a couple of keys: one
  key per line, and don't mix the two styles inside one constant.
- **Strong params for every `params[...]` read**, including plain-text ones.
  Actions call a private params method; `permit(*V1::FIELDS)` makes the field
  list the whitelist.
- **Name every inline collection.** A literal array inside a conditional gets a
  local or a constant, because the name is what says what it is.

**JavaScript**

- **Constants live inside the class**, read in the constructor off
  `FRONTEND_CONFIG` or off elements' `data-` attributes. Nothing at module
  scope.
- **`if`/`else` rather than a multi-line ternary.**
- **Handle the failure branch of every `fetch`** — a non-2xx response does not
  throw, so check `response.ok`. Silently leaving stale output is worse than an
  error in the console.
- **Guard against out-of-order responses** when a request fires per interaction.

**Views**

- The standalone layout links `hera`'s stylesheet *before* the calculator's own.
- ERB nests one level per block; check each `<% end %>` against its opener.

**Docs**

- The README says what the defaults are; it never invites the user to edit the
  model owner's hardcoded values in the gem source.

## Output Rules

- Output is a **new sibling directory** `dradis-calculator_{name}/`, never a
  change inside an existing calculator
- Mirror the existing calculators' structure and idiom wherever the model does
  not force a difference; where it does, say why
- Keep vendored upstream code unmodified, and record where it came from
- `git init` the new folder — the gemspec computes `spec.files` from
  `git ls-files`, so without a repo it builds an empty gem
- `CHANGELOG.md` gets one entry: `- Calculator: Add {name} calculator`
- `spec.authors = ['Dradis Team']`
- Version to match the sibling calculators (they ship in lockstep)
- Never commit or push unless the user asks

## Quality Checks

- [ ] Verification ran, and you have stated its tier, its coverage and its counts
- [ ] `V1` constants match the source, checked programmatically
- [ ] A saved score reopens the form in the state it was saved in
- [ ] Malformed or partial field values fall back cleanly instead of raising
- [ ] Both `/calculators/{name}` and the issue tab render the same partials
- [ ] Route helpers resolve (`calculators_{name}_path`, `{name}_project_issue_path`)
- [ ] Every `data-behavior` hook in the views is read by the JS, and vice versa
- [ ] Ruby parses (`ruby -c`), JS parses (`node --check`)
- [ ] No leftover strings from the calculator you cloned
- [ ] Any external dataset ships as an asset, with the script that produced it
- [ ] The headline score field parses with `.to_f` — bare number, no decoration
- [ ] Any "not defined" value round-trips instead of collapsing to blank
- [ ] No definition from `V1` is restated in the JS — the browser gets one
      serialized config blob
- [ ] The JS builds no `#[Field]#` output of its own; the server renders it
- [ ] Every `params[...]` read goes through a strong-params method
- [ ] No alignment padding survives anywhere in the gem
- [ ] Every `fetch` handles its non-`ok` branch
- [ ] Inflections are registered in exactly one file
- [ ] The standalone page renders styled (Hera's stylesheet is linked)
- [ ] If fields are user-selectable, deselected ones are deleted rather than
      left at stale values
- [ ] No boilerplate cruft from the clone — obsolete gemspec comments, a
      leading blank line in `.gitignore`
