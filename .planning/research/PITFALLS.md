# Pitfalls: keymap-drawer CI Integration for ZMK

**Domain:** ZMK keymap-drawer CI — GitHub Actions commit-back, ZMK parse, SVG rendering
**Researched:** 2026-04-02
**Confidence:** HIGH (derived from direct inspection of repo files + known GitHub Actions behavior)

---

## Critical Pitfalls

### Pitfall 1: Infinite Workflow Loop on Commit-Back

**What goes wrong:** The diagram job commits updated README.md, which triggers another `on: push` run of `build.yml`. This can loop or waste build minutes.

**Prevention:**
1. `[skip ci]` in commit message — GitHub Actions skips all workflows on this commit
2. `stefanzweifel/git-auto-commit-action` only commits when diff exists (built-in no-op guard)

**Detection:** Two back-to-back runs appearing after a single push; commit author is `github-actions[bot]`.

**Phase:** Must be in place before first test push.

---

### Pitfall 2: GITHUB_TOKEN Write Permission Not Granted by Default

**What goes wrong:** Job fails with `remote: Permission denied` or 403 on `git push`.

**Prevention:**
```yaml
jobs:
  draw:
    permissions:
      contents: write  # set at job level, not workflow level
```

Do NOT use a PAT — `GITHUB_TOKEN` with `contents: write` is sufficient.

**Detection:** `error: remote: Permission to <repo> denied to github-actions[bot]`

**Phase:** Must be set before first test run.

---

### Pitfall 3: Reusable Workflow Cannot Have Additional Steps

**What goes wrong:** Adding `steps:` to the existing `build` job (which uses `uses:`) causes a YAML parse error — the workflow file is invalid entirely.

**Prevention:** Add diagram as a **separate job** in the same `build.yml`. These fields are mutually exclusive on the same job.

**Detection:** GitHub workflow parser error: "Unexpected value 'steps'"

**Phase:** Foundational — shapes entire workflow design.

---

### Pitfall 4: keymap-drawer Physical Layout Mismatch for TOTEM

**What goes wrong:** SVG renders as a flat rectangular grid instead of the TOTEM's staggered columnar shape with angled outer keys and thumb cluster.

**Why:** `keymap-drawer` infers layout from `.keymap` alone but may not auto-detect `totem`. Must pass `-l totem` flag explicitly.

**Prevention:**
```bash
keymap parse -z config/totem.keymap -l totem > keymap.yaml
```

Verify layout name with `keymap list-layouts`. Cross-check `totem-layouts.dtsi` key rotations (centidegree values) against keymap-drawer's built-in totem definition.

**Detection:** SVG shows 38 uniformly spaced keys in a rectangle with no thumb cluster.

**Phase:** Verify on first functional test run.

---

### Pitfall 5: `&mt` Inline Override Silently Ignored by ZMK Parser

**What goes wrong:** The `&mt { quick-tap-ms = <100>; }` inline override block may be ignored by some keymap-drawer versions. Mod-tap keys render without hold annotation.

**Affected keys:** BASE layer positions 0, 9, 10, 19, 30, 37 and thumb mod-taps (LGUI/LALT on TAB/BACKSPACE).

**Prevention:** After first parse, check generated YAML:
```bash
grep -A5 "hold\|tap\|type: mt" /tmp/totem.yaml
```
Both `tap` and `hold` fields should be present for affected keys.

**Detection:** BASE layer in SVG shows `Q` with no modifier annotation on key 0.

**Phase:** Validate on first parse.

---

### Pitfall 6: Inline SVG Stripped by GitHub Markdown

**What goes wrong:** Raw `<svg>...</svg>` XML pasted into README.md is invisible — GitHub strips it for security.

**Prevention:** Commit the SVG as a file (`keymap.svg`) and reference it:
```markdown
![Keymap](keymap.svg)
```
OR embed between markers and reference the committed file. Do not paste raw SVG XML inline.

**Detection:** README shows comment markers with nothing visible between them.

**Phase:** Design decision — resolve before any implementation.

---

### Pitfall 7: `sed` Fails on Multiline SVG or `&` Characters

**What goes wrong:** `sed` replacement for SVG markers breaks on multiline content or HTML entities (`&amp;`, `&lt;`), corrupting README.

**Prevention:** Use Python:
```python
import re, pathlib
svg = pathlib.Path("keymap.svg").read_text()
readme = pathlib.Path("README.md").read_text()
result = re.sub(
    r"<!-- keymap-drawer start -->.*?<!-- keymap-drawer end -->",
    f"<!-- keymap-drawer start -->\n{svg}\n<!-- keymap-drawer end -->",
    readme, flags=re.DOTALL
)
pathlib.Path("README.md").write_text(result)
```

**Detection:** Old SVG persists after a successful run; or README shows `&amp;` artefacts.

**Phase:** Implement Python replacement from the start.

---

### Pitfall 8: `RA()` Keys Render as Modifier Combos, Not Characters

**What goes wrong:** `&kp RA(S)` (German ß) renders as `RA(S)` in the SVG rather than `ß`.

**Why:** keymap-drawer doesn't know the OS locale mapping.

**Consequence (low severity):** Technically accurate, not human-friendly for German umlauts.

**Prevention (deferred):** Use `draw_config` key overrides in a future polish pass.

**Phase:** Deferred to polish pass after basic diagram is working.

---

## Phase-Specific Warnings Summary

| Phase Topic | Pitfall | Mitigation |
|-------------|---------|------------|
| Workflow restructuring | Pitfall 3 | Separate job, not added steps to existing job |
| First test push | Pitfall 1 | `[skip ci]` + stefanzweifel no-op guard |
| Permissions | Pitfall 2 | `contents: write` on draw job |
| Physical layout | Pitfall 4 | `-l totem` flag; verify against totem-layouts.dtsi |
| Hold-tap rendering | Pitfall 5 | Inspect generated YAML for hold/tap fields |
| SVG embedding strategy | Pitfall 6 | Commit SVG as file, reference via `![](keymap.svg)` |
| README update script | Pitfall 7 | Use Python re.sub with DOTALL, not sed |
