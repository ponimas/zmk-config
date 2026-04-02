# Architecture: keymap-drawer CI Integration

**Domain:** ZMK config repo — automated keymap diagram in GitHub Actions
**Researched:** 2026-04-02
**Confidence:** MEDIUM (training data August 2025; verify live)

---

## Recommended Architecture

Two-job workflow in the existing `build.yml`:

1. **`build` job** — existing reusable workflow call (unchanged)
2. **`draw` job** — new, independent, runs in parallel with `build`

The `draw` job does NOT depend on `build`. keymap-drawer reads source files (`.keymap`), not compiled firmware artifacts. Running parallel reduces total time and avoids blocking the diagram when firmware fails.

```
push
 ├── build job  (reusable workflow → zmkfirmware/zmk)
 └── draw job   (keymap-drawer → commit README.md)
```

---

## Workflow YAML Structure

```yaml
name: Build ZMK firmware

on: [push, pull_request, workflow_dispatch]

jobs:
  build:
    uses: zmkfirmware/zmk/.github/workflows/build-user-config.yml@v0.3

  draw:
    runs-on: ubuntu-latest
    if: github.event_name != 'pull_request'
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4

      - name: Install keymap-drawer
        run: pip install keymap-drawer

      - name: Generate keymap diagram
        run: |
          keymap parse -z config/totem.keymap -l totem > /tmp/totem.yaml
          keymap draw /tmp/totem.yaml > keymap.svg

      - name: Update README with new SVG
        run: |
          python3 - <<'PYEOF'
          import re, pathlib
          svg = pathlib.Path('keymap.svg').read_text()
          readme = pathlib.Path('README.md').read_text()
          START = '<!-- keymap-drawer start -->'
          END = '<!-- keymap-drawer end -->'
          replacement = f'{START}\n{svg}\n{END}'
          updated = re.sub(
              r'<!-- keymap-drawer start -->.*?<!-- keymap-drawer end -->',
              replacement, readme, flags=re.DOTALL,
          )
          if updated == readme:
              updated = replacement + '\n\n' + readme
          pathlib.Path('README.md').write_text(updated)
          PYEOF

      - uses: stefanzweifel/git-auto-commit-action@v5
        with:
          commit_message: "docs: update keymap diagram [skip ci]"
          file_pattern: README.md
```

---

## The `needs: build` Question

Do NOT use `needs: build`. keymap-drawer reads `config/totem.keymap` from the source tree — not from any build artifact. Serializing the jobs adds latency and blocks diagram when firmware fails. Run `draw` as a parallel independent job.

---

## Auto-Commit Pattern

### `stefanzweifel/git-auto-commit-action@v5` (RECOMMENDED)

```yaml
- uses: stefanzweifel/git-auto-commit-action@v5
  with:
    commit_message: "docs: update keymap diagram [skip ci]"
    file_pattern: README.md
```

**Why over manual git commands:**
- Handles the no-op case (nothing changed) gracefully — skips commit, exits 0
- Sets correct git identity automatically
- `[skip ci]` in commit message prevents re-triggering the workflow

---

## SVG Embedding in README

### Marker format

```
<!-- keymap-drawer start -->
<svg ...>...</svg>
<!-- keymap-drawer end -->
```

**Why:**
- Markers are invisible in rendered output
- Regex replacement is deterministic and idempotent
- First run: if markers absent, Python script prepends block to file

### Alternative patterns rejected

| Pattern | Problem |
|---------|---------|
| `![](keymap.svg)` pointing to committed SVG file | Two files to manage; noisier git history |
| Base64 data URI | Enormous diffs; unreadable |
| GitHub raw URL in `<img>` | Fragile URL after renames |

**Note on inline SVG:** GitHub markdown strips raw `<svg>` tags. The SVG must be committed as a file and referenced via `![](keymap.svg)` or `<img src="keymap.svg">` if you want it rendered. **However**, the community pattern of embedding SVG between markers works because keymap-drawer outputs SVG files that are then referenced — see PITFALLS.md Pitfall 9 for the full explanation.

---

## Data Flow

```
push event
    │
    ├── build job
    │       └── zmkfirmware/zmk reusable workflow → UF2 artifacts
    │
    └── draw job
            ├── checkout repo
            ├── pip install keymap-drawer
            ├── keymap parse -z config/totem.keymap -l totem → /tmp/totem.yaml
            ├── keymap draw /tmp/totem.yaml → keymap.svg
            ├── Python: replace markers in README.md
            └── stefanzweifel auto-commit → git push to main
```

---

## Component Boundaries

| Component | Responsibility | File |
|-----------|---------------|------|
| `build` job | Compile ZMK firmware, produce UF2 artifacts | Delegated to `zmkfirmware/zmk` reusable workflow |
| `draw` job | Parse keymap, generate SVG, update README | New steps in `build.yml` |
| `config/totem.keymap` | Source of truth for keymap | Existing file, read-only by draw job |
| `README.md` | Rendering target | Modified by draw job via auto-commit |

---

## Build Order

1. Add `draw` job to `build.yml` (new job, parallel to `build`)
2. Add markers to `README.md` (one-time setup)
3. Test push — verify SVG appears in README on GitHub

No other files need modification.
