# Features: keymap-drawer CI Integration

**Domain:** ZMK config repo — automated keymap diagram generation
**Researched:** 2026-04-02

---

## Table Stakes (must have)

These are expected in any functional keymap-drawer CI integration:

- **ZMK parse**: `keymap parse -z config/totem.keymap` — direct ZMK keymap parsing without intermediate YAML
- **All-layers SVG**: Generate a single SVG showing all 4 layers (BASE, NAV, SYM, ADJ)
- **TOTEM physical layout**: Use the `totem` built-in layout for correct key geometry (staggered, angled thumb cluster, rotated outer keys)
- **README embedding**: Inline SVG between HTML comment markers; idempotent replace on every build
- **`contents: write` permissions**: Diagram job must commit back to main
- **Separate job from firmware build**: GitHub Actions hard constraint — `uses:` and `steps:` cannot coexist in one job
- **Loop prevention**: `[skip ci]` in commit message + `git diff` guard to skip commit when nothing changed
- **PR exclusion**: Skip commit-back step on `pull_request` events (GITHUB_TOKEN has no write on fork PRs)

## TOTEM-Specific Table Stakes

- **`-l totem` layout flag**: Pass the `totem` layout identifier at parse time
- **Hold-tap rendering**: `&mt` keys should show both tap (key) and hold (modifier) labels
- **Combo rendering**: `combo_esc` (positions 0+1) should show an arc connecting the two keys
- **Macro label display**: The `gif` macro (SYM layer) should show its label, not raw binding

## Differentiators (nice to have, not in v1)

- **Layer filtering**: `--select-layers` to show only specific layers
- **Custom `draw_config` styling**: Per-key label overrides (e.g., `RA(S)` → `ß` for German umlauts)
- **Intermediate YAML commit**: Commit the parsed `keymap.yaml` for inspection/debugging
- **Bot commit attribution**: Custom git committer name/email for the auto-commit
- **Main-only condition**: `if: github.ref == 'refs/heads/main'` to skip diagram on feature branches
- **Version pinning**: Pin keymap-drawer to a specific version for reproducibility

## Anti-Features (explicitly out of scope for v1)

- **PNG output** — SVG inline only (user decision)
- **Per-branch diagram previews** — commit to main only (user decision)
- **Separate workflow file** — add to existing `build.yml` (user decision)
- **Custom `keymap-drawer.yaml` config** — use direct ZMK parse, no intermediate config file (user decision)
- **Diagrams from shield keymap** — `boards/shields/totem/totem.keymap` is the older reference file; only `config/totem.keymap` is the active source
- **`needs: build` dependency** — diagram job should run in parallel, not depend on firmware build

---

## Feature Complexity Notes

| Feature | Complexity | Notes |
|---------|------------|-------|
| ZMK parse + draw | Low | 2 CLI commands |
| TOTEM layout flag | Low | One flag; verify name with `keymap list-layouts` |
| README SVG injection | Low | ~10 lines of Python |
| Commit-back | Low | One action with correct flags |
| Loop prevention | Medium | Requires `[skip ci]` + `if:` guard on PR events |
| Hold-tap rendering | None | Automatic if `&mt` is parsed correctly |
| Combo rendering | None | Automatic if physical layout is correct |

---

## Feature Dependencies

- Correct physical layout → correct combo arc positions
- Hold-tap rendering → depends on keymap-drawer version supporting `&mt` inline override parsing
- README embedding → depends on Python available on runner (it is, on `ubuntu-latest`)
