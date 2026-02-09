# Dradis Skills for Claude Code

A Claude Code plugin providing skills for [Dradis Framework](https://dradis.com) — the cybersecurity collaboration and reporting platform.


## Installation

```
/plugin install dradis-skills
```


## Available Skills

### `/dradis-skills:methodology`

Build a Dradis methodology XML file by parsing external sources.

**Usage:**

```
/dradis-skills:methodology https://owasp.org/Top10/2025/ "OWASP Top 10 - 2025"
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


## Samples

The `samples/` directory contains reference files:

- `minimum.xml` — Minimal valid methodology with a few cards
- `OWASP-Top10-2021.v1.0.xml` — Full OWASP Top 10 (2021) methodology
- `OWASP-Top10-2025.v1.0.xml` — Full OWASP Top 10 (2025) methodology
- `prompt.txt` — The original prompt used to generate the 2025 file


## License

GPL-2.0 — See [LICENSE](LICENSE) for details.
