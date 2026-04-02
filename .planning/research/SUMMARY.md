# Project Research Summary

**Project:** ZMK Config — Keymap Diagram CI
**Domain:** GitHub Actions CI integration for keyboard firmware config
**Researched:** 2026-04-02
**Confidence:** MEDIUM

## Executive Summary

This project adds automated keymap diagram generation to an existing ZMK firmware config repo. The TOTEM keyboard's 38-key layout is parsed from `config/totem.keymap` using `keymap-drawer`, which has first-class ZMK support and a built-in TOTEM physical layout definition. The output SVG must be committed back to the repository so it renders in README.md on GitHub — a well-understood CI pattern with a standard community toolchain (`pip install keymap-drawer` + `stefanzweifel/git-auto-commit-action`).

The recommended approach is a parallel `draw` job added to the existing `build.yml`. This is architecturally forced: GitHub Actions prohibits mixing `uses:` and `steps:` in the same job, so the new steps cannot live alongside the existing `build` job's `uses: zmkfirmware/zmk/...` line. The `draw` job runs independently of firmware build — it reads source files, not compiled artifacts — so no `needs: build` dependency is required or desired.

The primary risks are: (1) infinite workflow loops caused by the commit-back push, mitigated by `[skip ci]` in the commit message; (2) the GITHUB_TOKEN permission not being write-enabled by default, fixed with `contents: write` on the job; and (3) GitHub stripping raw inline `<svg>` tags from Markdown, meaning the SVG must be committed as a file and referenced, not pasted inline. All three are straightforward to prevent if handled up front.

---

## Key Findings

### Recommended Stack

keymap-drawer is the only tool with first-class ZMK parse support and a built-in TOTEM layout. It runs as a CLI tool installed via `pip` on the standard `ubuntu-latest` runner — no extra setup required. The commit-back is handled by `stefanzweifel/git-auto-commit-action@v5`, which is the community-standard action for this pattern and gracefully skips the commit when nothing has changed.

**Core technologies:**
- `keymap-drawer >=0.14`: Parse ZMK keymap and render SVG — only tool with ZMK + TOTEM support
- `Python 3.11`: Runtime for keymap-drawer; already on `ubuntu-latest`; required >=3.10
- `actions/checkout@v4`: Repo checkout — current stable
- `actions/setup-python@v5`: Python install with pip cache support
- `stefanzweifel/git-auto-commit-action@v5`: Commit-back — community standard, built-in no-op guard

**Permissions:** `contents: write` on the `draw` job only. No PAT needed.

### Expected Features

**Must have (table stakes):**
- ZMK parse via `keymap parse -z config/totem.keymap` — direct parse, no intermediate config file
- TOTEM physical layout via `-l totem` flag — staggered columnar shape, angled thumb cluster, rotated outer keys
- All-layers SVG showing all 4 layers: BASE, NAV, SYM, ADJ
- SVG committed as `keymap.svg` and referenced from README via `![Keymap](keymap.svg)`
- HTML comment markers (`<!-- keymap-drawer start/end -->`) for idempotent updates
- `contents: write` on the draw job for commit-back
- `[skip ci]` in auto-commit message to prevent workflow loops
- `if: github.event_name != 'pull_request'` to skip on fork PRs (no write access)
- Hold-tap labels on `&mt` keys (automatic when `&mt` is parsed correctly)
- Combo arc for `combo_esc` (positions 0+1, automatic with correct physical layout)

**Should have (differentiators — defer to v2):**
- Custom `draw_config` label overrides for German umlauts (`RA(S)` → `ß`)
- Version-pinned keymap-drawer install for reproducibility
- Intermediate `keymap.yaml` committed for debugging
- Bot committer attribution

**Defer (v2+):**
- PNG output
- Per-branch diagram previews
- Layer filtering via `--select-layers`
- Custom `keymap-drawer.yaml` config file

### Architecture Approach

Two parallel jobs in the existing `build.yml`: the unchanged `build` job (reusable workflow call) and a new `draw` job with its own steps. The `draw` job checks out the repo, installs keymap-drawer, generates the SVG from `config/totem.keymap`, updates README.md using a Python `re.sub` with DOTALL, and commits back via `stefanzweifel/git-auto-commit-action`. The SVG is committed as `keymap.svg` (not inlined) and referenced from README with a Markdown image tag.

**Major components:**

1. **`build` job** — existing reusable workflow call to `zmkfirmware/zmk`; unchanged
2. **`draw` job** — new parallel job; runs keymap-drawer, updates README, commits back
3. **`config/totem.keymap`** — read-only source of truth; parsed by `draw` job
4. **`keymap.svg`** — committed artifact; referenced from README
5. **`README.md`** — updated in-place by Python regex between HTML comment markers

### Critical Pitfalls

1. **Infinite workflow loop** — auto-commit triggers another `on: push` run. Prevent with `[skip ci]` in commit message; `stefanzweifel` action also skips when no diff. Must be in place before first test push.

2. **Missing `contents: write` permission** — GITHUB_TOKEN is read-only by default; `git push` fails with 403. Set `permissions: contents: write` at the `draw` job level.

3. **`uses:` + `steps:` conflict** — adding steps to the existing `build` job invalidates the YAML entirely. The draw logic must be a separate job. Foundational constraint.

4. **Inline SVG stripped by GitHub** — raw `<svg>` XML in Markdown is invisible. Commit `keymap.svg` as a file and reference it with `![Keymap](keymap.svg)`.

5. **`-l totem` flag required** — without it, keymap-drawer may render a flat rectangular grid instead of TOTEM's actual staggered shape. Verify layout name with `keymap list-layouts` after install.

6. **`&mt` inline override may be silently ignored** — some keymap-drawer versions skip the inline `&mt { ... }` override block. Validate by inspecting generated YAML for `hold`/`tap` fields on BASE layer mod-tap keys.

7. **`sed` fails on multiline SVG** — use Python `re.sub(..., flags=re.DOTALL)` for README updates, not `sed`.

---

## Implications for Roadmap

Based on research, the implementation is small (one job, ~30 lines of YAML) but several pitfalls must be addressed in strict order. Suggested 3-phase structure:

### Phase 1: Workflow Scaffolding
**Rationale:** The `uses:` constraint is foundational — every other decision depends on correctly placing the `draw` job as a separate parallel job. Permissions and loop prevention must also be in place before the first push.
**Delivers:** A valid `build.yml` with a `draw` job skeleton that runs, has correct permissions, and won't loop.
**Addresses:** Table stakes — separate job, `contents: write`, `[skip ci]`, PR exclusion condition.
**Avoids:** Pitfalls 1 (loop), 2 (permissions), 3 (`uses:` conflict).

### Phase 2: Diagram Generation
**Rationale:** Once the job scaffolding is correct, add the actual keymap-drawer invocation. The `-l totem` flag and SVG embedding strategy (file reference, not inline) are the key decisions here.
**Delivers:** A generated `keymap.svg` committed to the repo and visible in README via `![Keymap](keymap.svg)`.
**Uses:** `keymap-drawer`, Python `re.sub`, `stefanzweifel/git-auto-commit-action@v5`.
**Implements:** `draw` job steps; README marker setup.
**Avoids:** Pitfalls 4 (layout flag), 6 (inline SVG stripped), 7 (sed multiline).

### Phase 3: Validation and Polish
**Rationale:** After the diagram is generating and committing, validate rendering quality (hold-tap labels, combo arc, layout shape) and address any remaining rough edges.
**Delivers:** Confirmed correct SVG with all 4 layers, hold-tap annotations, and combo arc; German umlauts noted for v2 if desired.
**Validates:** Pitfall 4 (layout shape), Pitfall 5 (`&mt` hold-tap fields in generated YAML).
**Defers:** Label overrides for `RA()` keys to a future polish pass.

### Phase Ordering Rationale

- Phase 1 before Phase 2: The workflow must be structurally valid before any draw logic is added. A YAML-invalid `build.yml` breaks firmware builds too.
- Phase 2 before Phase 3: Can't validate rendering quality without a working generation step.
- Permissions and loop prevention in Phase 1: These have no cost to add early and catastrophic cost if missed — a looping workflow burns minutes and blocks until manually stopped.

### Research Flags

Phases likely needing verification live during implementation:
- **Phase 2:** Exact `-l totem` flag syntax — verify with `keymap list-layouts` after install. May be `totem` or `TOTEM` or a variant. See Open Questions.
- **Phase 2:** Exact keymap-drawer version — `pip install keymap-drawer` without pin may pull a version with breaking changes to ZMK parse.
- **Phase 3:** `&mt` inline override behavior — depends on keymap-drawer version; inspect generated YAML to confirm hold/tap fields are present.

Phases with standard patterns (research-phase likely not needed):
- **Phase 1:** GitHub Actions job structure and permission patterns are well-documented; no research needed.
- **Phase 2:** `stefanzweifel/git-auto-commit-action` usage is standard community pattern.

---

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | MEDIUM | `pip install keymap-drawer` and `stefanzweifel@v5` are community consensus; exact keymap-drawer version unverified |
| Features | HIGH | Derived from direct inspection of `config/totem.keymap`; table stakes are concrete and exhaustive |
| Architecture | HIGH | `uses:` + `steps:` constraint is a hard GitHub Actions rule; parallel job pattern is correct by construction |
| Pitfalls | HIGH | GITHUB_TOKEN behavior and loop mechanics are well-known; `&mt` parsing caveat is keymap-drawer-version-specific |

**Overall confidence:** MEDIUM — the architecture and feature set are solid; primary uncertainty is in keymap-drawer version behavior and exact CLI flag syntax, both of which are quickly validated with a single `pip install` + `keymap --help`.

### Gaps to Address

- **Exact keymap-drawer version:** Run `pip index versions keymap-drawer` or check PyPI before pinning. Version `>=0.14` is the research estimate; verify the latest stable release.
- **`-l totem` flag syntax:** Run `keymap list-layouts` after install to confirm the exact layout identifier. The research assumes `totem` (lowercase) based on keymap-drawer convention.
- **`[skip ci]` scope:** Confirm that `[skip ci]` prevents the firmware `build` job from re-running on the auto-commit push (not just the `draw` job). This is expected behavior but worth verifying on the first real push.
- **Inline SVG vs. file reference:** ARCHITECTURE.md notes community patterns exist for both; the recommended approach here is file reference (`![Keymap](keymap.svg)`) to avoid GitHub's SVG stripping. This is a design decision to confirm before Phase 2.

---

## Sources

### Primary (HIGH confidence)
- Direct repo inspection (`config/totem.keymap`, `build.yml`, `boards/shields/totem/totem.dtsi`) — key matrix, layer structure, existing workflow constraints
- GitHub Actions documentation (training data) — `uses:`/`steps:` mutual exclusivity, GITHUB_TOKEN permission model

### Secondary (MEDIUM confidence)
- keymap-drawer PyPI / GitHub README (training data, Aug 2025) — ZMK parse flags, TOTEM layout support, CLI interface
- `stefanzweifel/git-auto-commit-action` README (training data) — no-op behavior, `[skip ci]` pattern

### Tertiary (LOW confidence)
- Community ZMK config repos (training data) — `pip install keymap-drawer` convention, README embedding pattern
- keymap-drawer version `>=0.14` — estimate based on feature availability; verify on PyPI before implementing

---
*Research completed: 2026-04-02*
*Ready for roadmap: yes*
