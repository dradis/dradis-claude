# Dradis Risk Calculator Reference

Everything a `dradis-calculator_*` add-on needs: the file layout, the
conventions each file follows, the naming traps, the patterns for vendored
code and external datasets, and how to verify the result.

Throughout, `{name}` is the kebab-case calculator name; the examples below
use `aivss-ssvc` to make the substitutions concrete. So `{name}` is
`aivss-ssvc`, `{path}` its underscored form used in file paths and routes
(`aivss_ssvc`), `{Module}` the Ruby module (`AivssSsvc`), and `{PREFIX}` the
issue field prefix (`AIVSS-SSVC`).

## File layout

```
dradis-calculator_{name}/
├── dradis-calculator_{name}.gemspec
├── CHANGELOG.md                     # one entry for the new calculator
├── README.md
├── Gemfile  Rakefile  LICENSE  CONTRIBUTING.md  .gitignore  .github/
├── config/
│   └── routes.rb
├── lib/
│   ├── dradis-calculator_{name}.rb
│   └── dradis/plugins/calculators/{path}/
│       ├── engine.rb
│       ├── gem_version.rb
│       └── version.rb
└── app/
    ├── models/dradis/plugins/calculators/{path}/v1.rb
    ├── controllers/dradis/plugins/calculators/{path}/
    │   ├── base_controller.rb        # instance level
    │   └── issues_controller.rb      # issue level
    ├── views/
    │   ├── dradis/plugins/calculators/{path}/
    │   │   ├── _tools_menu.html.erb          # view hook: Tools menu
    │   │   ├── base/index.html.erb           # instance-level layout
    │   │   ├── base/_*.html.erb              # shared content partials
    │   │   └── issues/
    │   │       ├── _show-tabs.html.erb       # view hook: issue tab
    │   │       └── edit.html.erb             # issue-level layout
    │   └── layouts/dradis/plugins/calculators/{path}/base.html.erb
    └── assets/
        ├── javascripts/dradis/plugins/calculators/{path}/
        │   ├── {path}_calculator.js
        │   ├── base.js                       # standalone page manifest
        │   └── manifests/hera.js             # in-app manifest
        └── stylesheets/dradis/plugins/calculators/{path}/
            ├── _{path}.scss                  # the actual rules
            ├── base.css.scss                 # standalone page manifest
            └── manifests/hera.scss            # in-app manifest
```

## Naming

| Thing | Form | Example |
|---|---|---|
| Gem, folder | `dradis-calculator_{name}` | `dradis-calculator_aivss-ssvc` |
| Ruby module | `Dradis::Plugins::Calculators::{Module}` | `…::AivssSsvc` |
| File paths | `dradis/plugins/calculators/{path}/` | `…/aivss_ssvc/` |
| Route | `/calculators/{path}` | `/calculators/aivss_ssvc` |
| Mounted as | `:{path}_calculator` | `:aivss_ssvc_calculator` |
| Issue fields | `{PREFIX}.FieldName` | `AIVSS-SSVC.RiskScore` |

**Routes use underscores.** dradis-ce's own `config/routes.rb` has no
hyphenated path segments. Underscoring also lets Rails auto-generate the
route names, so no explicit `as:` is needed anywhere.

### The Zeitwerk trap for acronym names

The existing calculators use all-caps modules (`CVSS`, `DREAD`, `MITRE`) and
register `inflect.acronym` in the engine so Zeitwerk can map the directory
name back to the constant. **This does not extend to two acronyms joined by
an underscore.** With both acronyms registered:

```ruby
"aivss_ssvc".camelize   # => "AIVSSSSVC", not "AIVSS_SSVC"
```

so Zeitwerk would look for `AIVSSSSVC` and fail to load. Before committing to
a module name, check it round-trips:

```bash
ruby -e 'require "active_support/all"
  ActiveSupport::Inflector.inflections { |i| i.acronym("YOUR"); i.acronym("ACRONYM") }
  p "your_module_path".camelize
  p "YourModule".underscore'
```

A plain CamelCase module (`AivssSsvc`) round-trips with **no** inflection
initializer at all, and still underscores to the directory name that
`plugin_name` and the view-hook lookup need. Prefer it for multi-word names;
keep the all-caps form only for a single acronym, where it works.

Registering acronyms is also global — it changes `camelize`/`underscore`
everywhere in the host app. Not registering any is one less side effect.

## The engine

```ruby
module Dradis::Plugins::Calculators::{Module}
  class Engine < ::Rails::Engine
    isolate_namespace Dradis::Plugins::Calculators::{Module}

    include Dradis::Plugins::Base
    provides :addon
    description 'Risk Calculators: {NAME}'

    initializer 'calculator_{path}.asset_precompile_paths' do |app|
      app.config.assets.precompile += [
        'dradis/plugins/calculators/{path}/base.css',
        'dradis/plugins/calculators/{path}/base.js',
        'dradis/plugins/calculators/{path}/manifests/hera.css',
        'dradis/plugins/calculators/{path}/manifests/hera.js'
      ]
    end

    initializer 'calculator_{path}.mount_engine' do
      Rails.application.routes.append do
        # The enabled? check must be inside the block so the routes can be
        # re-enabled without a server restart.
        if Engine.enabled?
          mount Engine => '/', as: :{path}_calculator
        end
      end
    end
  end
end
```

`enabled?` comes from `Dradis::Plugins::Base` and defaults to true, so no
`addon_settings` block is needed unless the calculator has its own settings.

`description` matters beyond documentation: `render_view_hooks` sorts add-ons
by it, which fixes the order of the Tools menu entries.

## Routes

```ruby
Dradis::Plugins::Calculators::{Module}::Engine.routes.draw do
  get '/calculators/{path}' => 'base#index'

  resources :projects, only: [] do
    resources :issues, only: [] do
      member do
        get   '{path}' => 'issues#edit'
        patch '{path}' => 'issues#update'
      end
    end
  end
end
```

Yields `calculators_{path}_path` and `{path}_project_issue_path`. The latter
is what `simple_form_for [:{path}, current_project, @issue]` resolves to.

## The model (`V1`)

One class holding every definition taken from the source. Views iterate over
it and the JS never restates any of it. Its *shape* follows the model's shape
— there is no single schema to copy.

**Discrete-option metrics.** A list per metric, each option carrying its key,
label and numeric value:

```ruby
METRICS = [
  { key: 'AV', name: 'Attack Vector', options: [
    { key: 'N', label: 'Network', value: 0.85 },
    { key: 'A', label: 'Adjacent', value: 0.62 }
  ] }
].freeze
```

**Numeric scales.** The scale bounds and the text for each step; DREAD renders
these as radio rows with the guidance in the table cells.

**Hierarchical taxonomy.** Little or nothing in `V1` beyond the field list —
the data lives in a JSON asset (see "External datasets"). `V1` holds only the
field names the selects write to.

**Lookup table or matrix.** Keep the table verbatim, ideally vendored rather
than retyped; `V1` holds the axis definitions that index into it. Expose which
cell was hit, not just its value — a reader checking a 5×5 matrix wants to see
the row and column.

**Decision tree.** `V1` holds the nodes: each question, its options, and for
each option either the next question or a terminal outcome.

```ruby
NODES = {
  'exploitation' => {
    question: '...',
    options: [
      { key: 'active', label: 'Active', next: 'automatable' },
      { key: 'none',   label: 'None',   outcome: 'Defer' }
    ]
  }
}.freeze
```

The UI reveals questions as they unlock rather than showing all of them, and
the **path** is part of the result: record it as an output field so a reader
can audit how the outcome was reached. Restoring state means replaying the
path, so validate that a saved path is still walkable — a tree revision can
strand an old one, and it must fall back rather than raise.

**"Not defined" values.** Most published models have a skip value with defined
semantics (CVSS's `X` means "use the default weight", not zero). Give it a real
option key, keep it out of the arithmetic the way the source does, and make
sure it survives a save/reload.

Whatever the shape, a small amount of extra structure pays off: tie each
control to its options, its labels **and** its issue field(s) in one place, so
views become pure markup and state restoration can loop rather than repeat
field names:

```ruby
INPUTS = [
  { id: 'severity', field: '{PREFIX}.Severity', label: '...', options: SEVERITY_LEVELS }
].freeze
```

Also in `V1`: the state the reference loads with (`DEFAULTS`), and the issue
field list:

```ruby
FIELD_NAMES = %i[ ... ].freeze
FIELDS = FIELD_NAMES.map { |name| "{PREFIX}.#{name}".freeze }.freeze
```

`%i[]` handles dotted names: `%i[Base.Score]` gives `:"Base.Score"`.

## Output fields as an interface

The `{PREFIX}.*` fields are consumed by name outside the calculator — kits,
HTML export templates and issue tables all read them. The `welcome` kit's
export template does:

```erb
<%= markup(issue.fields['CVSSv4.BaseScore'], liquid: true) %>
```

and colour-codes findings from `issue.fields['CVSSv4.BaseScore'].to_f`, while
the kit's issue note template lists `#[CVSSv4.BaseScore]#` so every new issue
carries the slot. That makes the field names a contract:

- Write the headline score **bare** — `7.5`, not `7.5/10` or `High (7.5)` —
  so `.to_f` parses it
- Keep the human-readable verdict in its **own** field rather than decorating
  the number
- Order `FIELDS` the way a reader wants them: identity and score first,
  individual metrics after

### Renaming is a breaking change

A field name that has shipped is referenced by templates you cannot see. CVSS
still reads its own legacy name years later:

```ruby
field_value_v3 = @issue.fields['CVSSv3.Vector'] || @issue.fields['CVSSv3Vector']
```

If a name must change, read both and write the new one, exactly as above.

This is also why the model class is `V1`. A revision of the scoring model that
changes what a field *means* wants a `V2` alongside it, with its own field
namespace, rather than a redefinition that silently changes historical scores.

## Vendoring an upstream implementation

When the model's owner publishes working code, prefer copying it over
transcribing it — there is no transcription to get wrong, and upstream fixes
become a re-copy rather than a re-read.

CVSS is the worked example. It vendors FIRST's own files unmodified:

```
app/assets/javascripts/dradis/plugins/calculators/cvss/
├── v3/vendor/cvsscalc31.js            # upstream scoring
├── v3/vendor/cvsscalc31_helptext.js   # upstream tooltip text
├── v3/calculator.js.coffee            # thin wrapper
└── v4/vendor/{app,cvss_config,cvss_lookup,max_composed,…}.js
```

The wrapper does three things only: read the form, call upstream, render the
result. It contains no scoring logic of its own.

Rules for vendored code:

- Keep it **byte-identical** to upstream. Never reformat or "fix" it — that
  forfeits the whole benefit and makes the next re-copy a merge.
- Put it under a `vendor/` directory so it is obvious what is not yours.
- Record the upstream URL and version, so a future update is mechanical.
- List each file in the asset manifests explicitly, in dependency order.
- Check the licence permits redistribution before vendoring.

## External datasets

For taxonomy- or catalogue-shaped models, ship the data as an asset rather
than fetching upstream at runtime. MITRE is the worked example:

```
scripts/download_mitre_data.rb        # fetches upstream, reduces it, writes the asset
app/assets/data/…/mitre_data.json     # the reduced asset that ships (~148KB)
```

The script pulls the full upstream feeds, extracts only the fields the
calculator needs, and writes a compact JSON file. The JS loads it through the
asset pipeline:

```js
const response = await fetch("<%= asset_path('…/mitre_data.json') %>");
```

which requires the calculator file be named `*.js.erb`. Add the JSON to the
engine's `assets.precompile` list. Commit both the script and its output —
the script is how the data gets refreshed, the output is what ships.

## Multi-version models

When the model has versions in active use (CVSS 3.0/3.1/4.0), keep them side
by side rather than replacing:

- One partial set per version under `base/v3/`, `base/v4/`
- A `_version_menu.html.erb` select, and a `@version` ivar the controller sets
  by sniffing which version's fields the issue already carries
- Separate field namespaces per version (`CVSSv3.*`, `CVSSv4.*`) so an issue
  scored under one is not misread as the other
- On update, delete stale fields of the version being replaced

Default a new score to the newest version, but open an existing one on the
version it was scored with.

## Restoring saved state

The form must reopen on the score that was saved. Two established patterns:

**Vector string** (CVSS, DREAD) — when the model defines one. Store it in a
`{PREFIX}.Vector` field, validate against a `VECTOR_REGEXP`, and redirect
with an alert when it does not match:

```ruby
if field_value =~ V1::VECTOR_REGEXP
  field_value.split('/').each { |pair| @vector.store(*pair.split(':')) }
else
  redirect_to main_app.project_issue_path(current_project, @issue),
              alert: 'The format of the Vector field is invalid.'
end
```

**Individual fields** (MITRE) — when the model has no vector. Rebuild from
the separate issue fields, falling back per-field so a partially scored issue
still opens on a usable form. Accept both the stored label and the internal
key, and reject out-of-range values:

```ruby
def self.selection_from_fields(issue_fields = {})
  issue_fields ||= {}
  selection = {}
  INPUTS.each do |input|
    selection[input[:id]] =
      key_for(input[:options], issue_fields[input[:field]]) || DEFAULTS[input[:id]]
  end
  selection
end
```

Test this by feeding the calculator's own saved output back through it.

## Controllers

`BaseController < ActionController::Base` for the instance page.
`IssuesController < ::IssuesController` for the issue page, which needs:

```ruby
skip_before_action :remove_unused_state_param
```

Both actions build the `#[Field]#` skeleton the JS will patch:

```ruby
@issue_fields = V1::FIELDS.map do |field|
  value = @issue.fields[field]
  value = 'N/A' if value.blank?
  "#[#{field}]#\n#{value}"
end.join("\n\n")
```

`update` parses the textarea back with dradis-ce's own regex:

```ruby
fields = Hash[*params[:{path}_fields].to_s.scan(FieldParser::FIELDS_REGEX).flatten.map(&:strip)]
fields.each { |name, value| @issue.set_field(name, value) }
```

## Views

**Two entry views, shared content partials.** `base/index.html.erb` and
`issues/edit.html.erb` each lay themselves out and render the same
`base/_*.html.erb` partials. Do not add a wrapper partial that branches on a
layout flag — the existing calculators do not, and the entry views are where
a layout belongs.

Content partials read controller ivars directly (as DREAD's read
`@dread_vector`). No locals plumbing.

### The two levels have very different widths

The standalone page uses the add-on's own layout: a bare `.container`, no
sidebars. The issue page renders inside `layouts/hera`, between the main
sidebar (`14rem`) and the secondary issue sidebar (`14rem × 1.25`) — roughly
500px of chrome before padding.

**Bootstrap's `col-lg-*` keys off viewport width, not container width**, so a
side-by-side split that reads well standalone will fire on the issue page
with far less room than it assumes. This is why CVSS and DREAD use nav-pills
on the issue view and side-by-side on the instance page. Follow them:

```erb
<ul class="nav nav-pills w-100" id="{path}-tabs">
  <li class="nav-item"><a href="#{path}-edit-inputs" data-bs-toggle="pill" class="nav-link active">Inputs</a></li>
  <li class="nav-item pull-right">
    <a href="#{path}-edit-result" data-bs-toggle="pill" class="nav-link">
      Result: <span data-behavior="{path}-score">0</span>
    </a>
  </li>
</ul>
```

Keep the live score in the pill and put only the field output behind the tab
— that is what CVSS and DREAD do.

Note `pull-right` is `float: right`, which is inert inside Bootstrap 5's flex
`.nav`. The existing calculators' Result pills therefore do not actually
right-align. Match them anyway; a fix to `pull-right` should land on all the
calculators at once rather than one diverging.

### Selects

Add `data-combobox-config="no-combobox"` to every `<select>`, or Dradis's
combobox module will rewrite them on the issue page.

### View hooks

Two partials, both discovered automatically by `render_view_hooks` — nothing
in dradis-ce needs editing:

```erb
<%# _tools_menu.html.erb %>
<li>
  <%= link_to 'Risk Calculators - {NAME}', {path}_calculator.calculators_{path}_path,
      class: 'dropdown-item', data: { turbo: false } %>
</li>

<%# issues/_show-tabs.html.erb %>
<li class="nav-item">
  <%= link_to {path}_calculator.{path}_project_issue_path(current_project, @issue), class: 'nav-link' do %>
    <i class="fa-solid fa-calculator"></i> {NAME}
  <% end %>
</li>
```

### Other available hooks

`render_view_hooks` is used in more places than the two a calculator normally
fills. Any of these can be filled by adding a matching partial — no change to
dradis-ce:

| Hook | Rendered in | Use for a calculator |
|---|---|---|
| `tools_menu` | Main nav | The instance-level link (all three use this) |
| `issues/show-tabs` | Issue view | The calculator tab (all three use this) |
| `issues/widget` | Issue sidebar | Show the current score without opening the calculator |
| `issues/show-content` | Issue body | Render the score inline on the issue |
| `issues/edit-content` | Issue editor | Surface fields while editing |
| `export/index-tabs` | Export page | Only with `feature: :export` |

The sidebar widget is worth considering: it puts the score in front of a reader
who is not going to open the calculator, which is most readers.

## JavaScript

Vanilla ES6 in a `turbo:load` listener, wired by `data-behavior` attributes.
If you vendored an upstream implementation, this file is a thin wrapper: read
the form, call upstream, render the result, and hold no scoring logic of its
own. If you transcribed instead, port the source's logic verbatim — same
branch order, same comparisons, same rounding — and mark it as ported so
nobody "improves" it later.

```js
document.addEventListener('turbo:load', () => {
  const root = document.querySelector('[data-behavior~={path}-calc]');
  if (!root) return;
  // ... constants ported verbatim from the source ...
  new Calculator(root).init();
});
```

**Patch the field skeleton in place** rather than regenerating it, so the
field list and its order live only in Ruby:

```js
escapeRegex(string) {
  return string.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');
}

updateResult(field, value) {
  const regex = new RegExp(`(#\\[${this.escapeRegex(field)}\\]#\\n)(.*?)(\\n|$)`, 'g');
  this.result.value = this.result.value.replace(regex, (_m, before, _old, after) => `${before}${value}${after}`);
}
```

Escape the field name — field names contain dots. Use a replacement function
rather than `'$1' + value + '$3'` so a `$` in a value cannot corrupt output.

**Use `querySelectorAll` when updating by behavior**, not `querySelector`:
the issue view echoes the score in the Result pill as well as the results
panel, and a single-element update silently leaves one of them stale.

### Asset manifests

`base.js` (standalone page) requires jquery3/popper/bootstrap plus the
calculator; `manifests/hera.js` (in-app) requires only the calculator, since
Dradis already loads the rest. Same split for the stylesheets: put the rules
in `_{path}.scss` and have both manifests import it.

Style against Hera's theme custom properties — `--primary-bg`,
`--primary-bg-subtle`, `--border-color`, `--text-default`, `--text-muted` —
so the calculator follows light and dark themes.

## Verifying the port

Match the technique to what the source affords. State which you used.

### Tier 1 — differential against a runnable reference

The strongest evidence, available whenever the source ships something
runnable. Run the reference and your build over the same inputs and compare
every output. For a self-contained reference page, jsdom can host both:

```js
const { JSDOM } = require('jsdom');

// Reference: the real page, its own scripts running.
const ref = new JSDOM(fs.readFileSync('reference.html', 'utf8'), { runScripts: 'dangerously' });

// Under test: your rendered markup + your real calculator.js.
const dom = new JSDOM(fs.readFileSync('fixture.html', 'utf8'), { runScripts: 'outside-only' });
dom.window.eval(fs.readFileSync('.../{path}_calculator.js', 'utf8'));
dom.window.document.dispatchEvent(new dom.window.Event('turbo:load'));
```

If the reference is a library rather than a page, require it directly and skip
the DOM on that side. If it is a hosted service, capture its responses once to
a fixture rather than calling it per assertion.

Render your fixture from the **real ERB** rather than hand-written HTML, so
the views are covered too. Plain ERB does not auto-escape like Rails does, so
emulate that or attributes containing quotes will not match what Dradis
serves.

### Tier 2 — published test vectors

Many specs publish vectors or worked examples. Encode each as a case and
assert your build reproduces it exactly. Vectors are samples rather than
coverage, so pair them with the structural checks below.

### Tier 3 — hand-derived cases

When the source is prose or a table only, work cases through the model by
hand: every branch, and both sides of every threshold. Record the derivation
next to each expected value so a reviewer can check your arithmetic rather
than trusting it. Say plainly that no oracle existed.

### Tier 4 — user confirmation

For a model the user owns, there is no external oracle: their sign-off on the
model you restated back to them is the specification. Encode that restatement
as the test — every axis, band and cell as explicit cases — so the thing they
approved is the thing that is asserted, and a later change to the model shows
up as a failing case rather than a quiet drift.

### Structural checks, at every tier

These are cheap and catch bugs the tiers above miss:

- **Constants check** — parse the source and assert programmatically that the
  values in `V1` match: option values, labels, thresholds, defaults. Catches a
  mistyped lookup cell that no formula test will.
- **Boundary enumeration** — for each derived quantity, enumerate every
  distinct value it can reach rather than sampling. Averages and lookups have
  far fewer distinct outputs than inputs, so this is usually cheap and it is
  the only way to guarantee no threshold boundary is skipped.
- **Random fuzz** — a few thousand inputs across the whole space.
- **Round-trip** — drive the calculator, take its saved field output, feed it
  back through state restoration, assert the form state is identical. Assert
  malformed and partial input falls back rather than raising.
- **Both layouts** — point the harness at the instance page and the issue view
  and assert they agree; the layouts differ, the numbers must not.
- **Every user-visible string**, and every element that echoes a value, not
  just the headline number.

For a dataset-backed calculator there is no arithmetic to diff. Check instead
that every taxonomy node resolves, that IDs and names are consistent with
upstream, that the selects' dependent levels populate, and that the asset
parses and is the shape the JS expects.

## Gotchas

- **`git ls-files` in the gemspec** — the gemspec computes `spec.files` from
  it, so an un-`init`ed folder builds an empty gem. `git init` the add-on.
- **Rounding is part of the model.** Match the source's arithmetic exactly.
- **Don't copy sibling bugs.** MITRE defines `escapeRegex` but forgets to
  apply it in `updateResult`; CVSS carries a rhetorical comment about
  inheriting from a "no-frills controller" that is wrong on `IssuesController`.
  Port the patterns, not the defects.
- **`spec.authors`** — the clone leaves the original author's name on code
  they did not write. Set `['Dradis Team']`.
- **No calculator ships tests**, and dradis-ce's suite does not cover them
  either — not for want of something to test against (CVSS has FIRST's own
  implementation, MITRE has the ATT&CK feeds, DREAD's formulas are fixed).
  Verify during the port regardless; shipping the harness as specs is a
  departure worth raising with the user rather than assuming.
- **A model with no oracle is still portable**, just less provable. Say which
  verification tier you reached instead of implying Tier 1 coverage.
- **Licence-check anything you vendor.** Redistribution in a GPL-2 gem is not
  automatic.
