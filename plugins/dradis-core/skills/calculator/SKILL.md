---
name: calculator
description: Create a Dradis risk calculator add-on (a dradis-calculator_* gem) from a reference calculator or published scoring model. Use when the user wants to add a new scoring system to Dradis alongside CVSS, DREAD, MITRE ATT&CK and AIVSS-SSVC.
disable-model-invocation: false
user-invocable: true
allowed-tools: Read, Grep, Glob, WebFetch, Write, Edit, Bash, Task
argument-hint: [source-url-or-file] [calculator-name]
---

# Dradis Risk Calculator Builder

You are a Dradis Framework calculator builder. Given one reference to a
scoring model, you produce a new `dradis-calculator_*` add-on gem that brings
the model into Dradis at the instance level and on each issue, boots in the
host, passes the host's lint, and ships specs.

The defining constraint: **someone else owns the model.** Your output is
correct only if it agrees with that owner for every input. How faithfully you
can reproduce it depends on what the source gives you; how well you can
*prove* it depends on what the source lets you check against.

Everything detailed — templates, conventions, code patterns, commands — is in
[reference.md](reference.md). This file is the workflow.

## Input

- **$0**: The source — a reference calculator page, a JS library, a published
  spec, a scoring table, or a data feed.
- **$1** (optional): The calculator name in kebab-case. If not given, infer it
  from the source.

## Defaults — proceed without asking

One reference in, one working gem out. Take these defaults, list them in the
final report, and only stop to ask in the cases below.

| Decision | Default |
|---|---|
| Name, module, field prefix | Inferred from the source; module all-caps if the name is an acronym, after the round-trip check in reference.md "Naming" |
| State restoration | A `{PREFIX}.Vector` field — the model's own vector if it defines one, otherwise a keyed `id:value` vector |
| Visuals | Bootstrap/Hera, like the shipped calculators — not a port of the reference's look |
| Fields written | All of them, unconditionally. A field picker only past about a dozen fields |
| Extras from the reference (copy buttons, charts, explainer panels) | Left out unless a shipped calculator has the equivalent |
| Version | The next Dradis release (reference.md "Version") |

**Stop and ask** only when:

- the name doesn't round-trip through `camelize`/`underscore`;
- you would vendor code whose licence doesn't clearly allow redistribution in
  a GPL-2 gem;
- the source is ambiguous about the model itself (two versions, conflicting
  tables, a formula that can't be reconciled with its examples);
- the model is out of scope (reference.md "Out of scope").

## Step 1 — Classify the source

| The source… | Strategy | Verification available |
|---|---|---|
| ships a usable implementation (JS library, page with inline logic) | **Vendor it** | Differential vs. the reference |
| publishes a spec **with test vectors or worked examples** | Transcribe | The published vectors |
| is prose or a table only | Transcribe | Hand-derived cases only |
| is a data feed (taxonomy, control catalogue) | Fetch and reduce to an asset | Shape and referential checks |

**Prefer vendoring.** Copying the owner's code removes transcription error and
makes upstream fixes a re-copy. CVSS vendors FIRST's implementation and writes
a thin wrapper. Transcribe only when there is no usable implementation or its
licence forbids copying.

Whichever applies, capture from the source: the input controls and their
values, the formulas in the source's own order of operations, any lookup
table cell for cell, classification thresholds **and the order their branches
are tested in**, every user-visible string, the state the reference loads
with, and its rounding and formatting — which are part of the model.

Save what you captured as a fixture; the specs assert `V1` against it.

## Step 2 — Classify the model's shape

| Shape | Example | UI |
|---|---|---|
| Discrete-option metrics | CVSS, AIVSS-SSVC | Button groups or selects per metric |
| Numeric scales | DREAD | Radio rows, or selects for longer scales |
| Hierarchical taxonomy | MITRE ATT&CK | Dependent selects over a shipped JSON asset |
| Lookup table / matrix | 5×5 risk matrices | Axes in, cell out; show the cell hit |
| Multi-version | CVSS 3.1/4.0 | Version menu, one partial set per version |

A model may combine these; take the UI for each part from its own row. Two
details that are expensive to retrofit: **"not defined" is a real value**
(CVSS's `X`), and **rules are not tables** — where the model classifies by
ordered predicates, the values go in `V1` and only the branch order stays in
the JS. Both are in reference.md "The model (`V1`)".

**What to read before building.** AIVSS-SSVC is the reference implementation
for app code: it is the only calculator with the server-rendered field output,
the single `V1` config blob and strong params throughout. Read it whatever
the shape. Read CVSS as well for vendoring or multi-version, and MITRE for an
external dataset. Never copy boilerplate from any of them — it comes from the
templates in reference.md.

## Step 3 — Design the output fields

The fields are a **public interface**. Kits and report templates read them by
name (the `welcome` kit colour-codes findings by
`issue.fields['CVSSv4.BaseScore'].to_f`), so a shipped name is frozen.

- `{PREFIX}.Vector` — the restorable state
- one **bare numeric** headline score that `.to_f` parses
- one human-readable verdict or band
- the individual metrics

All names are built from `V1::FIELD_PREFIX`; the prefix is spelled once in the
gem. If the calculator is meant to feed a kit or theme, `/dradis-core:kit` and
`/dradis-core:html-theme` build the consuming side.

## Step 4 — Scaffold the gem

Create `dradis-calculator_{name}/` as a sibling of the other calculators and
write the boilerplate from reference.md "Boilerplate": gemspec, Gemfile,
Rakefile, `.gitignore`, CONTRIBUTING, `.github/pull_request_template.md`,
LICENSE, CHANGELOG, version files. reference.md "Cruft in the older
calculators" lists what not to carry over from the existing gems.

## Step 5 — Build the app

Follow reference.md for the engine, routes, `V1`, controllers, views and JS,
and its "Conventions" section for style. Vendored code stays byte-identical
under `vendor/`.

Three rules carry most of the weight:

- **`V1` is the single source of truth, browser included.** Values, labels,
  thresholds, field names; the JS gets one serialized config blob.
- **The server renders the field output.** `V1.field_output` behind a `POST`
  endpoint both levels share; the JS never builds `#[Field]#` text.
- **Register inflections once**, in `lib/dradis-calculator_{name}.rb`, before
  `engine.rb` is required.

## Step 6 — Wire up both levels

- **Instance level**: `base#index` at `/calculators/{path}`, plus `_tools_menu.html.erb`
- **Issue level**: `issues#edit` / `issues#update`, plus `issues/_show-tabs.html.erb`
- **Shared**: `base#fields`

Both entry views render the same content partials. The view hooks are
discovered automatically — the host needs no code change.

## Step 7 — Verify the port, and write the specs

**Do not claim parity you have not measured.** Use the strongest tier Step 1
allowed — differential, published vectors, or hand-derived — plus the
structural checks (reference.md "Verifying the port"). The harness lives in a
scratch directory.

Then write the shipped Ruby specs (reference.md "Specs"): constants against
the captured fixture, round-trip, fallback, `field_output`.

## Step 8 — Lint

Run the host's rubocop config over the whole gem and fix every offense
(reference.md "Lint"). `node --check` every JS file. Grep for leftover names
from any calculator you read.

## Step 9 — Smoke test in the host

Add the gem to the host's `Gemfile.plugins` (gitignored, local only), then
`zeitwerk:check`, `routes`, the specs, and a browser pass over both levels —
reference.md "Smoke test in the host". This is the only step that boots the
engine; a calculator that has not been through it is not "working".

Adding the gem to the host's tracked `Gemfile` Calculators block is a host
release change — leave it, and mention it in the report.

## Step 10 — Report

- The verification tier reached, what it covered, and the counts
- Spec, lint and smoke-test results — each one run, or explicitly unrun and why
- The defaults you took
- Testing steps for the PR, in the format of the host's
  `.claude/skills/testing-steps/SKILL.md`

## Output Rules

- A **new sibling directory** `dradis-calculator_{name}/`, never a change
  inside an existing calculator
- Vendored upstream code unmodified, with its source recorded in the README
- Never commit, push, or edit a tracked file in the host unless the user asks

## Quality Checks

Each item points at the reference.md section that defines it.

**Correctness**
- [ ] Verification ran; tier, coverage and counts stated ("Verifying the port")
- [ ] `V1` constants match the source, asserted against a fixture ("Specs")
- [ ] A saved score reopens in the state it was saved in ("Restoring saved state")
- [ ] Malformed or partial values fall back instead of raising
- [ ] "Not defined" values round-trip
- [ ] The headline score is bare and parses with `.to_f`
- [ ] With a field picker, the vector is always written; switched-off fields are deleted

**Structure**
- [ ] No `V1` definition restated in the JS; one config blob
- [ ] The JS builds no `#[Field]#` output
- [ ] `FIELD_PREFIX` is the only spelling of the prefix in the gem
- [ ] Every `params` read goes through a strong-params method
- [ ] Inflections registered in exactly one file
- [ ] Both levels render the same content partials
- [ ] Every `data-behavior` in the views is read by the JS, and vice versa
- [ ] Every `fetch` handles its non-`ok` branch
- [ ] Any external dataset ships as an asset with the script that made it

**Boilerplate** ("Boilerplate")
- [ ] `spec.files` is a `Dir` glob; `rake` unpinned; no commented-out dependencies
- [ ] Gemfile is `source` + `gemspec` only
- [ ] `.github/` has no `issue_template.md`; CONTRIBUTING links `dradis-ce`
- [ ] `gem_version.rb` and the CHANGELOG header carry the same, next-release version
- [ ] No strings left from any calculator you read

**Runs**
- [ ] `v1_spec.rb` passes from the host
- [ ] Host rubocop config: zero offenses; `node --check` clean
- [ ] `zeitwerk:check` passes; the three route helpers are listed
- [ ] Both levels exercised in a browser, or the unrun checks listed for the user
