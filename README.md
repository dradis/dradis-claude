# Dradis Skills for Claude Code

A Claude Code plugin providing skills for [Dradis Framework](https://dradis.com) — the cybersecurity collaboration and reporting platform.


## Installation

```
/plugin install dradis-skills
```


## Available Skills

### `/dradis-core:methodology`

Build a Dradis methodology XML file by parsing external sources.

**Usage:**

```
/dradis-core:methodology https://owasp.org/Top10/2025/ "OWASP Top 10 - 2025"
```

The skill will:
1. Fetch and parse the source content (URL or local file)
2. Extract methodology items
3. Generate a valid Dradis board XML with proper Textile markup and field structure
4. Write the output file to the current directory

**Supported sources:**
- Web URLs (e.g., OWASP, NIST, CIS Benchmarks)
- Local files (Markdown, HTML, CSV, YAML)

See `samples/` for example output files.


### `/dradis-core:html-theme`

Create an HTML export template (ERB) for a Dradis project kit — self-contained reports with charts, risk dashboards, and detailed findings.

**Usage:**

```
/dradis-core:html-theme owasp "dark cyberpunk dashboard with radar charts"
```

The skill will:
1. Read the kit's project template to discover the field schema and sample values
2. Detect Liquid-computed fields (e.g., `Risk`, `Risk Score`) and generate ERB equivalents
3. Build a self-contained `.html.erb` template with CSS, charts, and print styles
4. Deploy to both the kit directory and `storage/templates/` for immediate testing

**Supports:** Bootstrap 5, Chart.js, Highcharts, Font Awesome via CDN. Vanilla JS preferred.


## Samples

The `samples/` directory contains reference files:

- `minimum.xml` — Minimal valid methodology with a few cards
- `OWASP-Top10-2021.v1.0.xml` — Full OWASP Top 10 (2021) methodology
- `OWASP-Top10-2025.v1.0.xml` — Full OWASP Top 10 (2025) methodology
- `prompt.txt` — The original prompt used to generate the 2025 file


## License

GPL-2.0 — See [LICENSE](LICENSE) for details.
