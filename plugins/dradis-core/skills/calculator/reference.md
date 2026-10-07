# Dradis Risk Calculator Reference

Everything a `dradis-calculator_*` add-on needs: the file layout, the
boilerplate, the conventions each file follows, the naming traps, the patterns
for vendored code and external datasets, how to verify and lint, and the
Dradis CE instructions handed to the user
the result.

Throughout, `{name}` is the kebab-case calculator name; the examples below
use `aivss-ssvc` to make the substitutions concrete. So `{name}` is
`aivss-ssvc`, `{path}` its underscored form used in file paths and routes
(`aivss_ssvc`), `{Module}` the Ruby module (`AIVSSSSVC`), `{PREFIX}` the
issue field prefix (`AIVSS-SSVC`) and `{NAME}` the display name.

## File layout

```
dradis-calculator_{name}/
├── dradis-calculator_{name}.gemspec
├── CHANGELOG.md  CHANGELOG.template  README.md  LICENSE  CONTRIBUTING.md
├── Gemfile  Rakefile  .gitignore
├── .github/pull_request_template.md
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
    │   ├── base_controller.rb        # instance level + shared fields endpoint
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
            └── manifests/hera.scss           # in-app manifest
```

## Boilerplate

Copy the non-app files from **dradis-calculator_aivss-ssvc**, the one
calculator whose boilerplate has been through review: the gemspec, `Gemfile`,
`Rakefile`, `.gitignore`, `LICENSE`, `CONTRIBUTING.md`, `CHANGELOG.template`,
`.github/pull_request_template.md`, and `lib/…/gem_version.rb` / `version.rb`.
Substitute the names, and nothing else, so the new gem matches the ones beside
it. Then:

- **Version** — set `gem_version.rb` as below.
- **`CHANGELOG.md`** — replace the body with the new gem's single entry.
- **`README.md`** — written fresh (see "Conventions"), not substituted.

### Version

The calculators ship in lockstep with Dradis, so the new gem targets the **next
Dradis release**, not whatever the siblings currently say. Read the latest
version from CVSS's CHANGELOG and check whether it has been released:

```bash
cd ../dradis-calculator_cvss
git fetch --tags
head -1 CHANGELOG.md     # vX.Y.Z (Month YYYY)
git tag -l 'vX.Y.Z'      # listed => released => target vX.(Y+1).0
```

If that version is tagged, target the next minor; otherwise target that
version. If Dradis's `release-X.Y.Z` branches disagree, ask.

`MAJOR`/`MINOR`/`TINY` in `gem_version.rb` and the CHANGELOG header carry
**the same version**:

```
vX.Y.Z (Month YYYY)
  - Calculator: Add {NAME} calculator
```

## Naming

| Thing | Form | Example |
|---|---|---|
| Gem, folder | `dradis-calculator_{name}` | `dradis-calculator_aivss-ssvc` |
| Ruby module | `Dradis::Plugins::Calculators::{Module}` | `…::AIVSSSSVC` |
| File paths | `dradis/plugins/calculators/{path}/` | `…/aivss_ssvc/` |
| Route | `/calculators/{path}` | `/calculators/aivss_ssvc` |
| Mounted as | `:{path}_calculator` | `:aivss_ssvc_calculator` |
| Issue fields | `{PREFIX}.FieldName` | `AIVSS-SSVC.RiskScore` |

**Routes use underscores.** dradis-ce's own `config/routes.rb` has no
hyphenated path segments, and underscoring lets Rails generate the route names
without any `as:`.

### Acronym names and where the inflections go

The existing calculators use all-caps modules (`CVSS`, `DREAD`, `MITRE`) and
register `inflect.acronym` so Zeitwerk can map the directory back to the
constant. Two acronyms joined by an underscore round-trip too, as long as both
are registered (`"aivss_ssvc".camelize # => "AIVSSSSVC"` and back). Check
whatever you pick before committing to it:

```bash
ruby -e 'require "active_support/all"
  ActiveSupport::Inflector.inflections { |i| i.acronym("YOUR"); i.acronym("ACRONYM") }
  p "your_module_path".camelize
  p "YourModule".underscore'
```

**Register the inflections in exactly one place: `lib/dradis-calculator_{name}.rb`,
before `engine.rb` is required.** `isolate_namespace` underscores the module
name at *require* time, so an engine initializer is too late:

```ruby
require 'dradis-plugins'

# Must run before requiring engine.rb: isolate_namespace underscores the
# module name at require time, so both acronyms need to exist already.
ActiveSupport::Inflector.inflections do |inflect|
  inflect.acronym('AIVSS')
  inflect.acronym('SSVC')
end

module Dradis
  module Plugins
    module Calculators
      module AIVSSSSVC
      end
    end
  end
end

require 'dradis/plugins/calculators/aivss_ssvc/engine'
require 'dradis/plugins/calculators/aivss_ssvc/version'
```

Acronyms are global — they change `camelize`/`underscore` across all of Dradis. A
plain CamelCase module (`AivssSsvc`) needs no registration; use it when the
name is not genuinely an acronym. `bin/rails zeitwerk:check`, in the Dradis CE
instructions, is the proof either way.

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

`enabled?` comes from `Dradis::Plugins::Base` and defaults to true. Add an
`addon_settings` block only if there is a setting worth exposing in the
Configuration Manager (the field picker's defaults are the usual one):

```ruby
addon_settings :{path} do
  settings.default_fields = "#{V1::FIELD_PREFIX}.Likelihood,#{V1::FIELD_PREFIX}.RiskScore"
end
```

`settings.default_fields =` sets the default for the `fields` key, read back as
`Engine.settings.fields`.

`description` sorts the add-ons in `render_view_hooks`, which fixes the order
of the Tools menu entries.

## Routes

```ruby
Dradis::Plugins::Calculators::{Module}::Engine.routes.draw do
  get '/calculators/{path}' => 'base#index'
  post '/calculators/{path}/fields' => 'base#fields', as: :calculators_{path}_fields

  resources :projects, only: [] do
    resources :issues, only: [] do
      member do
        get '{path}' => 'issues#edit'
        patch '{path}' => 'issues#update'
      end
    end
  end
end
```

Yields `calculators_{path}_path`, `calculators_{path}_fields_path` and
`{path}_project_issue_path` (what `simple_form_for [:{path}, current_project, @issue]`
resolves to). The `fields` endpoint is not project-scoped, so both levels post
to the one route.

## The model (`V1`)

One class holding every definition taken from the source. Views iterate over
it and the JS never restates any of it. Its shape follows the model's shape.

**Discrete-option metrics.** A list per metric; each option carries its key,
label and numeric value:

```ruby
THREAT_LEVELS = [
  { key: 'none', label: 'None', value: 0.2 },
  { key: 'poc', label: 'Public PoC', value: 0.5 },
  { key: 'active', label: 'Active', value: 0.9 }
].freeze
```

**Numeric scales.** The scale bounds and the text for each step; DREAD renders
these as radio rows with the guidance in the table cells. Keep the bounds as
constants — validation code (`/\A[1-5]\z/`) restating them is a second copy.

**Hierarchical taxonomy.** Little beyond the field list — the data lives in a
JSON asset (see "External datasets").

**Lookup table or matrix.** Keep the table verbatim, ideally vendored rather
than retyped; `V1` holds the axis definitions that index into it. Expose which
cell was hit, not just its value.

**"Not defined" values.** Most published models have a skip value with defined
semantics (CVSS's `X` means "use the default weight", not zero). Give it a real
option key, keep it out of the arithmetic the way the source does, and make
sure it survives a save/reload.

Tie each control to its options, its label **and** its issue field(s) in one
place, so views are pure markup and state restoration can loop:

```ruby
INPUTS = [
  {
    id: 'threat',
    field: "#{FIELD_PREFIX}.Threat",
    label: 'P(Threat): exploitation state',
    options: THREAT_LEVELS
  }
].freeze
```

Also in `V1`: the state the reference loads with (`DEFAULTS`), and the field
names:

```ruby
FIELD_PREFIX = '{PREFIX}'.freeze
VECTOR_FIELD = "#{FIELD_PREFIX}.Vector".freeze

FIELD_NAMES = %i[Vector Score Verdict ...].freeze
FIELDS = FIELD_NAMES.map { |name| "#{FIELD_PREFIX}.#{name}".freeze }.freeze
```

`%i[]` handles dotted names: `%i[Base.Score]` gives `:"Base.Score"`.

**`FIELD_PREFIX` and `VECTOR_FIELD` are the only spelling of the prefix in the
gem.** Controllers, views, the engine's settings defaults and the JS read them
from `V1`.

### V1 is the only source of truth — including for the browser

The JS restates no definition from `V1`: not option values, thresholds or
defaults, and not the things that feel like UI — outcome matrices, badge
classes, verdict copy, help text, field names. Serialize what the browser needs
as **one** constant and pass it as a single data attribute:

```ruby
FRONTEND_CONFIG = {
  outcomeMatrix: OUTCOME_MATRIX,
  badgeClass: BADGE_CLASS,
  vectorField: VECTOR_FIELD
}.freeze
```

```erb
<div
  data-behavior="{path}-calc"
  data-{path}-config="<%= …::V1::FRONTEND_CONFIG.to_json %>"
  data-{path}-fields-url="<%= calculators_{path}_fields_path %>"
>
```

camelCase keys — it is a JS object once it lands. Render the attribute in
**both** entry views. Per-option data a view already renders (an option's
value, label or field) goes on that element as a `data-` attribute; reserve the
blob for whole tables and maps.

#### When the model is rules, not a table

Some models classify by ordered predicates (*any axis above X, else two or more
above Y, else …*). That does not serialize to JSON without inventing a rule
language, so split it: every **value** the predicates test against and every
label or string they return goes in `V1` and travels in the config blob; only
the **branch order** stays in the JS.

```ruby
AGENT_THRESHOLDS = {
  primemover: 4.0,
  specialist: 3.0,
  copilot: 2.5
}.freeze
```

```js
// Only the order, and it is the reference's order.
classifyAgent(...averages) {
  const thresholds = this.agentThresholds;

  if (averages.some((avg) => avg >= thresholds.primemover)) return this.agent('primemover');
  if (averages.filter((avg) => avg >= thresholds.specialist).length >= 2) return this.agent('specialist');
  // ...
}
```

The check: **grep the scoring path in the JS for numeric literals and
user-visible strings.** Neither should appear. When refactoring an existing
calculator into this shape, keep every comparison operator and the branch
order unchanged, and diff old against new classification over every reachable
input.

## Output fields as an interface

The `{PREFIX}.*` fields are read by name outside the calculator. The `welcome`
kit's export template colour-codes findings from
`issue.fields['CVSSv4.BaseScore'].to_f`, and its issue note template lists
`#[CVSSv4.BaseScore]#` so every new issue carries the slot. So:

- Write the headline score **bare** — `7.5`, not `7.5/10` or `High (7.5)`
- Keep the human-readable verdict in its **own** field
- Order `FIELDS` the way a reader wants them: vector, score and verdict first,
  individual metrics after

**Renaming is a breaking change.** CVSS still reads its legacy name:

```ruby
field_value_v3 = @issue.fields['CVSSv3.Vector'] || @issue.fields['CVSSv3Vector']
```

If a name must change, read both and write the new one. A model revision that
changes what a field *means* gets a `V2` with its own namespace.

## Vendoring an upstream implementation

When the model's owner publishes working code, copy it rather than transcribe
it. CVSS vendors FIRST's files unmodified:

```
app/assets/javascripts/dradis/plugins/calculators/cvss/
├── v3/vendor/cvsscalc31.js            # upstream scoring
├── v3/vendor/cvsscalc31_helptext.js   # upstream tooltip text
├── v3/calculator.js.coffee            # thin wrapper
└── v4/vendor/{app,cvss_config,cvss_lookup,max_composed,…}.js
```

The wrapper reads the form, calls upstream, renders the result — no scoring
logic of its own.

- **Byte-identical** to upstream. Never reformat or "fix" it.
- Under a `vendor/` directory (rubocop already excludes `**/vendor/**`).
- Upstream URL and version recorded in the README, so an update is mechanical.
- Each file listed in the asset manifests explicitly, in dependency order.
- Licence checked: redistribution in a GPL-2 gem is not automatic. If it is
  unclear, stop and ask.

## External datasets

For taxonomy- or catalogue-shaped models, ship the data as an asset rather
than fetching upstream at runtime. MITRE is the worked example:

```
scripts/download_mitre_data.rb        # fetches upstream, reduces it, writes the asset
app/assets/data/…/mitre_data.json     # the reduced asset that ships
```

The JS loads it through the asset pipeline, which makes the calculator file a
`*.js.erb`:

```js
const response = await fetch("<%= asset_path('…/mitre_data.json') %>");
```

Add the JSON to `assets.precompile`. Commit the script and its output.

## Multi-version models

When versions are in active use side by side (CVSS 3.1/4.0): one partial set
per version (`base/v3/`, `base/v4/`), a `_version_menu.html.erb`, a `@version`
the controller sets by sniffing which version's fields the issue carries,
separate field namespaces per version, and stale fields of the replaced version
deleted on update. New scores default to the newest version; existing ones
open on the version they were scored with.

## Restoring saved state

The form must reopen on the score that was saved.

**Vector string** — the default. If the model defines a vector (CVSS), use it.
If it does not, define a keyed one (`id:value` pairs joined by `/`), so parsing
does not depend on the order the pairs were written in:

```ruby
VECTOR_PAIR_SEPARATOR = '/'.freeze
VECTOR_KEY_SEPARATOR = ':'.freeze

def self.selection_from_vector(vector)
  return if vector.blank?

  pairs = vector.split(VECTOR_PAIR_SEPARATOR).map { |pair| pair.split(VECTOR_KEY_SEPARATOR, 2) }
  return unless pairs.all? { |pair| pair.size == 2 }

  pairs = pairs.to_h
  return unless INPUTS.all? { |input| option_key?(input[:options], pairs[input[:id]]) }

  INPUTS.to_h { |input| [input[:id], pairs[input[:id]]] }
end

def self.option_key?(options, key)
  options.any? { |option| option[:key] == key }
end
private_class_method :selection_from_vector, :option_key?
```

Anything invalid returns `nil` and the caller falls back. Check pair length
before `to_h`, because a pair without a separator makes `to_h` raise. The JS writes the
vector, so both separators also travel in `FRONTEND_CONFIG`.

**Individual fields** (MITRE) — rebuild from the separate issue fields,
falling back per field so a partially scored issue still opens on a usable
form. Accept both the stored label and the internal key:

```ruby
def self.selection_from_fields(issue_fields = {})
  issue_fields ||= {}

  selection_from_vector(issue_fields[VECTOR_FIELD]) ||
    INPUTS.to_h do |input|
      [input[:id], key_for(input[:options], issue_fields[input[:field]]) || DEFAULTS[input[:id]]]
    end
end
```

**A field picker requires the vector.** If users can deselect fields, the
individual fields stop being a complete record, and an issue saved with a
subset cannot be restored. So with a picker: `VECTOR_FIELD` is always written,
rendered disabled-and-checked in the picker, and preferred by
`selection_from_fields` (AIVSS-SSVC `b1cbbc0`).

## Field output is rendered server-side

`V1` builds the `#[Field]#` block; the browser asks the server for it. The JS
never builds field output — that duplicates dradis-ce's `FieldParser` regex on
the client, where it drifts from the Ruby that parses it back.

```ruby
def self.field_output(values = {}, fields: FIELDS)
  (FIELDS & (fields | [VECTOR_FIELD])).map do |field|
    value = values[field]
    value = 'N/A' if value.blank?
    "#[#{field}]#\n#{value}"
  end.join("\n\n")
end
```

`FIELDS & …` filters to the requested subset and forces `FIELDS` order, so a
client cannot reorder or inject field names. `| [VECTOR_FIELD]` keeps the
vector in the output whatever the picker sends.

## Controllers

`BaseController < ActionController::Base` for the instance page and the shared
`fields` endpoint. `IssuesController < ::IssuesController` for the issue page,
with `skip_before_action :remove_unused_state_param`.

### Strong params, always

Every parameter goes through a private strong-params method — including arrays
and plain text — and `V1::FIELDS` is the whitelist:

```ruby
class BaseController < ActionController::Base
  def index
    @{path}_selection = V1::DEFAULTS
    @issue_fields = V1.field_output
  end

  def fields
    render plain: V1.field_output(field_values, fields: requested_fields)
  end

  private

  def fields_params
    params.permit(fields: [], values: V1::FIELDS)
  end

  def field_values
    fields_params.fetch(:values, {}).to_h
  end

  def requested_fields
    fields_params.fetch(:fields, V1::FIELDS)
  end
end
```

The issue-level `update` parses the textarea with dradis-ce's own regex:

```ruby
def update
  {path}_fields = {path}_fields_param
    .scan(FieldParser::FIELDS_REGEX)
    .to_h { |name, value| [name.strip, value.strip] }

  {path}_fields.each { |name, value| @issue.set_field(name, value) }

  # Fields the user deselected are removed rather than left stale.
  stale_fields = (@issue.fields.keys & V1::FIELDS) - {path}_fields.keys
  stale_fields.each { |name| @issue.delete_field(name) }

  if @issue.save
    redirect_to main_app.project_issue_path(current_project, @issue), notice: '{NAME} fields updated.'
  else
    render :edit
  end
end

private

def {path}_fields_param
  params.permit(:{path}_fields).fetch(:{path}_fields, '').to_s
end
```

Both are verified against actionpack: `permit(fields: [], values: FIELDS)`
drops unknown `values` keys, and `.to_h` with a block replaces the older
`Hash[*pairs.flatten.map(&:strip)]` idiom.

## Views

**Two entry views, shared content partials.** `base/index.html.erb` and
`issues/edit.html.erb` each lay themselves out and render the same
`base/_*.html.erb` partials. No wrapper partial that branches on a layout flag.
Content partials read controller ivars directly. Both entry views carry the
`data-behavior`, `data-{path}-config` and `data-{path}-fields-url` attributes.

**The standalone layout links Hera's stylesheet first**, so the standalone page
gets Hera's theme properties and Bootstrap before the calculator's rules:

```erb
<%= stylesheet_link_tag 'hera', media: 'all', 'data-turbo-track': 'reload' %>
<%= stylesheet_link_tag 'dradis/plugins/calculators/{path}/base', media: 'all', 'data-turbo-track': 'reload' %>
```

**The two levels have very different widths.** The standalone page is a bare
`.container`; the issue page sits between the main sidebar (`14rem`) and the
issue sidebar (`14rem × 1.25`). Bootstrap's `col-lg-*` and any media query key
off the *viewport*, so a split that reads well standalone fires on the issue
page with far less room. Two consequences:

- On the issue view, use nav-pills (inputs / result) as CVSS and DREAD do, with
  the live score in the Result pill:

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

  (`pull-right` is inert in Bootstrap 5's flex `.nav`; match the siblings
  anyway, and fix it across all calculators at once.)
- Multi-column groups inside a pane use `display: flex; flex-wrap: wrap` with a
  `min-width` per item, not a fixed `grid-template-columns` — a fixed grid
  overflows the issue tab when both sidebars are open (AIVSS-SSVC `c33692a`).

**Selects** get `data-combobox-config="no-combobox"`, or Dradis's combobox
module rewrites them on the issue page.

### The field picker (optional)

CVSS, DREAD and MITRE write all their fields, and for a handful that is right.
A picker earns its place only when writing everything would bury the issue —
as a rule of thumb, more than about a dozen fields. If you add one:

- Switches for which `{PREFIX}.*` fields get written, grouped (calculated
  results, inputs, intermediate values), with select all / none past a dozen.
- `VECTOR_FIELD` always on and disabled (see "Restoring saved state").
- The switches feed the `fields:` list posted to `base#fields`.
- Initial state from the issue's existing `{PREFIX}.*` fields, else from
  `Engine.settings.fields`.
- On `update`, switched-off fields are `delete_field`ed.
- The textarea becomes `class: 'd-none'`.

```erb
<div class="form-check form-switch mb-2">
  <input class="form-check-input"
         type="checkbox"
         role="switch"
         id="{path}-field-<%= field.parameterize %>"
         data-behavior="{path}-field-switch"
         data-field-name="<%= field %>"
         <%= 'checked' if @enabled_fields.include?(field) %>>
  <label class="form-check-label" for="{path}-field-<%= field.parameterize %>"><%= field %></label>
</div>
```

### View hooks

Discovered automatically by `render_view_hooks` — nothing in Dradis changes:

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

Other hooks exist (`issues/widget` for the issue sidebar, `issues/show-content`,
`issues/edit-content`). None of the shipped calculators use them; mention the
sidebar widget to the user as an option rather than building it unasked.

## JavaScript

Vanilla ES6 in a `turbo:load` listener, wired by `data-behavior` attributes.
Vendored: a thin wrapper. Transcribed: the source's logic verbatim — same
branch order, same comparisons, same rounding — marked as ported.

Everything the calculator needs lives on the instance, read in the constructor
from `FRONTEND_CONFIG` or the elements' `data-` attributes. Nothing at module
scope:

```js
document.addEventListener('turbo:load', () => {
  const root = document.querySelector('[data-behavior~={path}-calc]');
  if (!root) return;

  class {Module}Calculator {
    constructor(root) {
      this.root = root;

      const config = JSON.parse(root.dataset.{path}Config);
      this.outcomeMatrix = config.outcomeMatrix;
      this.badgeClass = config.badgeClass;

      this.fieldsUrl = root.dataset.{path}FieldsUrl;
      this.fieldSwitches = Array.from(root.querySelectorAll('[data-behavior~={path}-field-switch]'));
      this.fieldRequestId = 0;
      this.values = {};
    }

    // ... logic ported verbatim from the source, marked as ported ...
  }

  new {Module}Calculator(root).init();
});
```

Ask the server for the field output:

```js
async writeResult() {
  if (!this.result || !this.fieldsUrl) return;

  let fields;
  if (this.fieldSwitches.length) {
    fields = this.fieldSwitches.filter((s) => s.checked).map((s) => s.dataset.fieldName);
  } else {
    fields = Object.keys(this.values);
  }

  // Responses can land out of order; only the newest one may write.
  const requestId = ++this.fieldRequestId;
  const csrfToken = document.querySelector('meta[name=csrf-token]')?.content;

  const response = await fetch(this.fieldsUrl, {
    method: 'POST',
    credentials: 'same-origin',
    headers: {
      'Accept': 'text/plain',
      'Content-Type': 'application/json',
      'X-CSRF-Token': csrfToken
    },
    body: JSON.stringify({ fields, values: this.values })
  });

  if (response.ok) {
    const output = await response.text();
    if (requestId === this.fieldRequestId) this.result.value = output;
  } else {
    console.error(`{NAME}: failed to fetch field output (${response.status})`);
  }
}
```

`credentials` and the CSRF token are required or Rails rejects the POST.

Update by behavior with `querySelectorAll`: the issue view echoes the score in
the Result pill as well as the results panel.

### Asset manifests and styles

`base.js` (standalone) requires jquery3/popper/bootstrap plus the calculator;
`manifests/hera.js` (in-app) requires only the calculator. Same split for the
stylesheets: rules in `_{path}.scss`, imported by both manifests.

Style against Hera's theme custom properties — `--primary-bg`,
`--primary-bg-subtle`, `--border-color`, `--text-default`, `--text-muted` — so
the calculator follows light and dark themes.

## Conventions

The review conventions the dradis maintainers apply. Rubocop (see "Lint")
enforces the ones marked **(rubocop)**; the rest are on you.

**Ruby**

- No alignment padding — hashes, constants, routes. **(rubocop:
  `Layout/ExtraSpacing` with `AllowForAlignment: false`, `Layout/HashAlignment`)**
- Single-quoted strings unless interpolating. **(rubocop)**
- `{ }` for single-line blocks, `do … end` for multi-line. **(rubocop)**
- Multiline hashes once an entry has more than about three keys, one key per
  line; don't mix styles within one constant.
- Strong params for every `params` read.
- Name every inline collection: a literal array in a conditional becomes a
  local or constant (`input_fields`), because the name says what it is.
- Field names and the prefix come from `V1`, never as literals elsewhere.
- Prefer `to_h { }`, `each_with_object` and `index_with` over building a hash
  by mutation in an `each`.

**JavaScript**

- Constants inside the class, read in the constructor.
- `if`/`else` rather than a multi-line ternary.
- Handle the non-`ok` branch of every `fetch`.
- Guard against out-of-order responses.

**Views**

- ERB nests one level per block; check each `<% end %>` against its opener,
  especially where an `<% … do %>` and a tag open on the same line.

**Docs**

- The README says what the defaults are and that they match the reference; it
  never invites users to edit the model owner's values in the gem source.
- The README records the source URL, and for vendored code its version.

## Specs (only if the user asks)

No calculator ships specs, so the new one doesn't by default. The checks below
run during the port, in the harness. In the report, offer to ship them as specs.

If the user says yes, add `spec/models/dradis/plugins/calculators/{path}/v1_spec.rb`.
It uses Dradis CE's `rails_helper`, as other add-ons run from CE do, and reads
fixtures captured from the harness:

```ruby
require 'rails_helper'

describe Dradis::Plugins::Calculators::{Module}::V1 do
  let(:fixtures) { Dradis::Plugins::Calculators::{Module}::Engine.root.join('spec/fixtures') }

  describe '.selection_from_fields' do
    it 'restores the selection the calculator saved' do
      saved_fields = fixtures.join('saved_output.txt').read
        .scan(FieldParser::FIELDS_REGEX)
        .to_h { |name, value| [name.strip, value.strip] }
      saved_selection = JSON.parse(fixtures.join('saved_selection.json').read)

      expect(described_class.selection_from_fields(saved_fields)).to eq(saved_selection)
    end

    it 'falls back to DEFAULTS on a malformed vector' do
      fields = { described_class::VECTOR_FIELD => 'not-a-vector' }

      expect(described_class.selection_from_fields(fields)).to eq(described_class::DEFAULTS)
    end
  end

  describe '.field_output' do
    it 'writes the requested fields in FIELDS order' do
      requested = described_class::FIELDS.last(2).reverse
      written = described_class.field_output({}, fields: requested).scan(/#\[(.+?)\]#/).flatten

      expect(written & requested).to eq(requested.reverse)
    end

    it 'drops field names outside FIELDS' do
      expect(described_class.field_output({}, fields: ['Unknown.Field'])).not_to include('Unknown.Field')
    end
  end
end
```

`saved_output.txt` is the harness's field output for every input at a
non-default value; `saved_selection.json` is that selection. The specs need
Dradis CE to run, so the spec command goes in the CE instructions:

```bash
cd dradis-ce
bundle exec rspec ../dradis-calculator_{name}/spec
```

rspec resolves `rails_helper` against the working directory, so it loads CE's.

## Verifying the port

Match the technique to what the source affords, and state which you used.

**Tier 1 — differential against a runnable reference.** Run the reference and
your build over the same inputs and compare every output. jsdom hosts both:

```js
const { JSDOM } = require('jsdom');

const ref = new JSDOM(fs.readFileSync('reference.html', 'utf8'), { runScripts: 'dangerously' });

const dom = new JSDOM(fs.readFileSync('fixture.html', 'utf8'), { runScripts: 'outside-only' });
dom.window.eval(fs.readFileSync('.../{path}_calculator.js', 'utf8'));
dom.window.document.dispatchEvent(new dom.window.Event('turbo:load'));
```

Render the fixture from the **real ERB** so the views are covered; plain ERB
does not auto-escape like Rails, so emulate that. Stub `fetch` for the `fields`
endpoint with `V1.field_output` output. If the reference is a library, require
it directly; if it is a hosted service, capture responses once to a fixture.

**Tier 2 — published test vectors.** Encode each as a case. Samples, not
coverage.

**Tier 3 — hand-derived cases.** Every branch and both sides of every
threshold, with the derivation recorded next to each expected value. Say
plainly that no oracle existed.

**Structural checks, at every tier:**

- **Constants** — parse the source and assert programmatically that `V1`'s
  values match it: option values, labels, thresholds, defaults. Catches a
  mistyped lookup cell that no formula test will. (Nothing to check for
  vendored tables.)
- **Round-trip** — every input at a non-default value, through
  `field_output`, through `FieldParser::FIELDS_REGEX`, through
  `selection_from_fields`, equal to the start. Blank, malformed and partial
  values fall back instead of raising; "not defined" values survive.

- **Boundary enumeration** — enumerate every distinct value each derived
  quantity can reach rather than sampling; it is the only way to guarantee no
  threshold is skipped.
- **Random fuzz** — a few thousand inputs across the whole space.
- **Both layouts** — the instance page and the issue view agree.
- **Every user-visible string** and every element that echoes a value.

For a dataset-backed calculator: every taxonomy node resolves, IDs and names
match upstream, dependent selects populate, the asset parses into the shape
the JS expects.

The harness lives outside the gem, in a scratch directory. The Ruby-side checks
need only `activesupport`: define the empty namespace modules, then `load` the
`v1.rb` file.

```ruby
require 'active_support/all'

module Dradis; module Plugins; module Calculators; module {Module}; end; end; end; end
load 'app/models/dradis/plugins/calculators/{path}/v1.rb'
```

## Lint

Lint with Dradis CE's own rubocop config, fetched from its public repo. CE's
config disables every cop it doesn't list, so the stock rubocop defaults
don't apply:

```bash
curl -fsSL https://raw.githubusercontent.com/dradis/dradis-ce/develop/.rubocop.yml -o /tmp/dradis-ce-rubocop.yml
cd dradis-calculator_{name}
rubocop -c /tmp/dradis-ce-rubocop.yml --force-exclusion
```

Zero offenses before you report. If `rubocop` isn't installed, don't install
it; put the lint command in the CE instructions and say lint didn't run.

Also: `node --check` on every JS file, and grep the gem for the names of
any calculator you read, and their field prefixes.

## Dradis CE instructions

The final report includes these, filled in for the calculator. They're for a
user with a Dradis CE development checkout ([dradis/dradis-ce](https://github.com/dradis/dradis-ce),
`develop`) beside the gem folder.

````markdown
### Install

1. In `dradis-ce`, add this line to `Gemfile.plugins` (copy
   `Gemfile.plugins.template` to `Gemfile.plugins` first if it doesn't exist):

   ```ruby
   gem 'dradis-calculator_{name}', path: '../dradis-calculator_{name}'
   ```

2. From `dradis-ce`:

   ```bash
   bundle install
   bin/rails zeitwerk:check      # expect "All is good!"
   bin/rails routes -g {path}    # expect calculators_{path}, calculators_{path}_fields, {path}_project_issue
   ```

3. Start Dradis with `bin/rails server`, or restart it if it was already running.

### Test

1. Open **Tools > Risk Calculators - {NAME}**. The page loads styled and the
   field output panel shows the default selection.
2. Set the inputs to: {a case from the verification, as label values}.
3. Confirm the score reads {expected score} and the verdict reads
   {expected verdict}, the same as the reference.
4. Change one input. Confirm the field output updates.
5. Open any issue and select the **{NAME}** tab. Set the same inputs as step 2
   and save.
6. Confirm the issue now shows `{PREFIX}.Vector`, `{PREFIX}.{Score}` = {expected score}
   and the other `{PREFIX}.*` fields.
7. Reopen the **{NAME}** tab. Confirm every input is in the state you saved.
8. Edit the issue, change `{PREFIX}.Vector` to `not-a-vector`, and save.
   Reopen the tab. Confirm it opens on the defaults instead of erroring.
9. Check the browser console on both pages: no errors.

### Remove

Delete the line from `Gemfile.plugins`, run `bundle install` and restart.
````

Add, where they apply:

- the lint command, run from the gem with `BUNDLE_GEMFILE=../dradis-ce/Gemfile
  bundle exec rubocop -c ../dradis-ce/.rubocop.yml --force-exclusion`, if lint
  didn't run during the build;
- the spec command, if the user asked for specs;
- the field picker (switch a field off, save, confirm it's removed from the
  issue and the vector is still written), if there is one.

The new gem is not in CE's tracked `Gemfile`, so `Gemfile.plugins` is the
right place for it. Adding it to the `Gemfile` Calculators block is a Dradis
release change, not part of this skill.

## Out of scope

Not supported by this skill without a design conversation first:

- **Decision trees** (SSVC proper, triage trees). No shipped calculator has
  this shape, so there is no house UI. If asked, propose a design and get it
  agreed before building; the path through the tree must be an output field.
- **A model with no source** — the user's internal matrix or a description. Ask
  them to restate it as a table, confirm every axis, band and cell back to
  them, and treat that confirmed table as a Tier 3 source.

## Gotchas

- **Rounding is part of the model.** Match the source's arithmetic exactly.
- **Don't copy sibling defects.** MITRE defines `escapeRegex` and never applies
  it; CVSS has a comment about a "no-frills controller" that is wrong on
  `IssuesController`. Port the patterns, not the bugs.
- **`spec.authors`** is `['Dradis Team']`.
