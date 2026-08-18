# Dradis Risk Calculator Reference

Everything a `dradis-calculator_*` add-on needs: the file layout, the
conventions each file follows, the naming traps, and the differential
harness that proves the port is faithful.

Throughout, `{name}` is the kebab-case calculator name (`aivss-ssvc`),
`{path}` its underscored form used in file paths and routes (`aivss_ssvc`),
`{Module}` the Ruby module (`AivssSsvc`), and `{PREFIX}` the issue field
prefix (`AIVSS-SSVC`).

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

One class holding every definition taken from the source: input controls and
their options, derived groupings, defaults, and the issue field list. Views
iterate over it and the JS never restates any of it.

```ruby
module Dradis::Plugins::Calculators::{Module}
  class V1
    # ... option lists taken verbatim from the source ...

    # Ties each control to its options, its labels, and its issue fields, so
    # the view is pure markup and state restoration can be data-driven.
    INPUTS = [
      { id: 'threat', field: '{PREFIX}.Threat', value_field: '{PREFIX}.Threat.Value',
        label: '...', help: '...', options: THREAT_LEVELS }
    ].freeze

    # The state the reference implementation loads with.
    DEFAULTS = { ... }.freeze

    FIELD_NAMES = %i[ ... ].freeze
    FIELDS = FIELD_NAMES.map { |name| "{PREFIX}.#{name}".freeze }.freeze
  end
end
```

`%i[]` handles dotted names: `%i[Threat.Value]` gives `:"Threat.Value"`.

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

## JavaScript

Vanilla ES6 in a `turbo:load` listener, wired by `data-behavior` attributes.
Port the source's logic verbatim — same branch order, same comparisons, same
rounding — and mark it as ported so nobody "improves" it later.

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

## Differential testing

The only evidence that a port is faithful. Run the reference implementation
and your implementation side by side over the same inputs and compare every
output.

For a single-page reference calculator, jsdom can host both:

```js
const { JSDOM } = require('jsdom');

// Reference: the real page, its own scripts running.
const webDom = new JSDOM(fs.readFileSync('reference.html', 'utf8'), { runScripts: 'dangerously' });

// Under test: your rendered markup + your real calculator.js.
const dom = new JSDOM(fs.readFileSync('fixture.html', 'utf8'), { runScripts: 'outside-only' });
dom.window.eval(fs.readFileSync('.../{path}_calculator.js', 'utf8'));
dom.window.document.dispatchEvent(new dom.window.Event('turbo:load'));
```

Render the fixture from the **real ERB** rather than hand-writing HTML, so
the test covers the views too. Plain ERB does not auto-escape like Rails, so
emulate it or attributes containing quotes will differ from what Dradis
serves.

Coverage that actually finds bugs:

1. Every combination of the discrete inputs, with input vectors chosen to hit
   each classification branch
2. Every distinct value any averaged or derived quantity can take — enumerate
   them rather than sampling, so no threshold boundary is missed
3. A few thousand random inputs
4. Every user-visible string, and every element that echoes a value

Two cheaper checks worth adding alongside it:

- **Constants check**: parse the source page and assert your `V1` constants
  match it — option values, labels, thresholds, defaults. Catches a typo in a
  lookup table that a formula test would not.
- **Round-trip check**: drive the calculator, take its saved field output,
  feed it back through `selection_from_fields`, and assert the form state is
  identical. Also assert malformed input falls back rather than raising.

Set the fixture up so one process can be pointed at either layout; the
instance and issue views must produce identical numbers.

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
- **No calculator has tests.** Neither the three add-ons nor dradis-ce's suite
  covers them. Build the differential harness for your own confidence during
  the port; shipping it as specs is a departure worth raising with the user
  rather than assuming.
