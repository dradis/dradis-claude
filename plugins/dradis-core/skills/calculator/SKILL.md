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
| Lookup table / macrovector | CVSS v4 | Metrics in, vendored table out | CVSS |
| Multi-version model | CVSS 3.0/3.1/4.0 | Version menu, one partial set per version | CVSS |

A model may combine these — AIVSS-SSVC is discrete-option inputs plus numeric
scales plus a decision table. Take the UI for each part from its own row.

If the model needs an external dataset, follow MITRE: a script under
`scripts/` fetches and reduces upstream data to a JSON asset that ships with
the gem, and the JS fetches that asset via `asset_path`. Do not fetch from
upstream at runtime.

## Step 3 — Ask the open questions

Ask these together, then build without further interruption:

1. **Naming** — gem/folder, Ruby module, issue field prefix. See "Naming" in
   [reference.md](reference.md); there is a Zeitwerk trap for multi-acronym
   names to check *before* committing to one.
2. **State restoration** — vector string (CVSS, DREAD) or individual issue
   fields (MITRE). If the model defines a vector, use it.
3. **Visual fidelity** — Bootstrap/Hera styling like the existing calculators,
   or a port of the reference's own visuals. Default to the former.
4. **Extras** — which affordances from the reference to carry over. Judge each
   against the existing calculators rather than porting reflexively.

## Step 4 — Build the gem

Clone the sibling chosen in Step 2 and strip it: remove its `.git`, `lib/`,
`app/`, `config/` and `CHANGELOG.md`; keep `.github/`, `.gitignore`,
`CONTRIBUTING.md`, `Gemfile`, `LICENSE`, `Rakefile`, `CHANGELOG.template`.

Follow [reference.md](reference.md) for the layout and the per-file
conventions. Put every definition you transcribed in the `V1` model class —
it is the single source of truth. Views iterate over it; the JS never restates
it. Vendored code is the exception: keep it verbatim under a `vendor/`
directory, unmodified, so it can be re-copied when upstream changes.

## Step 5 — Wire up both levels

An add-on is not done until it works in both places:

- **Instance level**: `base#index` at `/calculators/{name}`, plus `_tools_menu.html.erb`
- **Issue level**: `issues#edit` / `issues#update`, plus `issues/_show-tabs.html.erb`

Both partials are discovered automatically by `render_view_hooks` — dradis-ce
needs no change. Render the same content partials from both entry points so
the two levels cannot drift.

## Step 6 — Verify, at the strongest tier the source allows

**Do not claim parity you have not measured.** Reading the code back to
yourself proves nothing. Use the best tier available from Step 1, and state in
your report which tier you used and what it covered.

**Tier 1 — differential against a runnable reference.** Drive the reference
and your build over the same inputs, compare every output. The strongest
evidence; use it whenever the source ships something runnable. See
"Verifying the port" in [reference.md](reference.md).

**Tier 2 — published test vectors.** Encode the spec's vectors or worked
examples as cases and assert your build reproduces each. Weaker than Tier 1
(vectors are samples, not coverage), so combine with the structural checks
below.

**Tier 3 — hand-derived cases.** When the source is prose only: work several
cases through the model by hand, including every branch and both sides of
every threshold, and assert against those. Say plainly in your report that no
oracle existed.

At every tier, also run these — they are cheap and catch different bugs:

- **Constants check** — assert programmatically that the values in `V1` match
  the source. Catches a mistyped lookup cell that a formula test never will.
- **Boundary enumeration** — for each derived quantity, enumerate every
  distinct value it can take rather than sampling, so no threshold is missed.
- **Round-trip** — save a score, reload it, assert the form returns to the
  same state; assert malformed input falls back instead of raising.
- **Both levels** — the instance page and the issue view must agree.

## Step 7 — Wire into dradis-ce

Add the gem to the Calculators block in `dradis-ce/Gemfile`. Before the gem is
published, use a path reference:

```ruby
gem 'dradis-calculator_{name}', path: '../dradis-calculator_{name}'
```

Confirm with the user before editing their `dradis-ce` checkout.

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
