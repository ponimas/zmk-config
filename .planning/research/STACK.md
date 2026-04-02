# Technology Stack

**Project:** ZMK Config — Keymap Diagram CI
**Researched:** 2026-04-02
**Confidence:** MEDIUM (training data, August 2025 cutoff; verify versions live before implementing)

---

## Recommended Stack

### Core Tool

| Technology | Version | Purpose | Why |
|------------|---------|---------|-----|
| keymap-drawer | `>=0.14` (verify on PyPI) | Parse ZMK `.keymap`, render SVG | Only tool with first-class ZMK parse support; TOTEM is a built-in layout |
| Python | `3.11` | Runtime for keymap-drawer | `ubuntu-latest` runner default; keymap-drawer requires >=3.10 |

### GitHub Actions

| Action | Version | Purpose | Why |
|--------|---------|---------|-----|
| `actions/checkout` | `v4` | Clone repo | v4 is current stable |
| `actions/setup-python` | `v5` | Install Python | v5 supports pip cache |
| `stefanzweifel/git-auto-commit-action` | `v5` | Commit-back README | Community standard; handles no-diff case gracefully |

### Installation Method

**Use `pip install keymap-drawer`** — not pipx, not uv.

- `pip` is on every runner with no extra setup
- keymap-drawer is invoked as subprocess, not imported — isolation adds nothing
- Community pattern in ZMK config repos universally uses bare pip

### Permissions Model

```yaml
jobs:
  draw:
    runs-on: ubuntu-latest
    permissions:
      contents: write  # required to push README.md back
```

Set at job level (not workflow level) for least privilege. No PAT required.

### Critical Structural Constraint

The existing `build.yml` uses a single `uses:` line. GitHub Actions prohibits mixing `uses:` and `steps:` in the same job. The diagram step **must** be a separate job in the same `build.yml`:

```yaml
jobs:
  build:
    uses: zmkfirmware/zmk/.github/workflows/build-user-config.yml@v0.3

  draw:  # separate job, same file
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - ...
```

### SVG Injection Pattern

```yaml
- name: Inject SVG into README.md
  run: |
    python3 - <<'EOF'
    import re, pathlib
    svg = pathlib.Path("keymap.svg").read_text()
    readme = pathlib.Path("README.md").read_text()
    START = "<!-- keymap-drawer start -->"
    END   = "<!-- keymap-drawer end -->"
    block = f"{START}\n{svg}\n{END}"
    if START in readme:
        updated = re.sub(
            rf"{re.escape(START)}.*?{re.escape(END)}",
            block, readme, flags=re.DOTALL
        )
    else:
        updated = block + "\n\n" + readme
    pathlib.Path("README.md").write_text(updated)
    EOF

- uses: stefanzweifel/git-auto-commit-action@v5
  with:
    commit_message: "chore: update keymap diagram [skip ci]"
    file_pattern: README.md
```

**Note:** `[skip ci]` in commit message prevents the auto-commit from re-triggering the workflow.

---

## Alternatives Considered

| Category | Recommended | Alternative | Why Not |
|----------|-------------|-------------|---------|
| Install | `pip install` | `pipx install` | No isolation benefit; extra setup |
| Install | `pip install` | `uv pip install` | Requires `astral-sh/setup-uv`; overkill |
| Commit-back | `stefanzweifel@v5` | Raw git commands | Verbose; no built-in idempotency guard |
| Job placement | New `draw:` job in `build.yml` | Separate `draw.yml` file | Out of scope per PROJECT.md |

---

## Open Questions

1. Exact keymap-drawer version — verify with `pip index versions keymap-drawer`
2. `--layout` flag syntax for TOTEM — verify with `keymap parse --help` after install
3. Whether `[skip ci]` fully prevents firmware build re-trigger on the auto-commit push
