# filo: plan

Go CLI that loads W3C DTCG (2025.10) token files in a defined order, resolves aliases, reports collisions and cycles,
and explains where any value came from.

Name: `filo` (thread). `filo explain color.brand.primary` follows the thread back to its origin.

## Scope

**Stage A (v0.1.0): diagnostic only. Read-only. No Sass output.**
Style Dictionary keeps doing the build. filo answers "why is this value here?"

Commands:

- `filo check`: load, validate, resolve. Exit non-zero on collisions, cycles, missing targets.
- `filo explain <token.path>`: print the origin chain (defined in X, overridden in Y, alias-resolved to Z).

**Stage B (only after the gate): Sass emitter.** See the end of this file.

**Out of scope until proven needed:** other platforms, transforms beyond the work config, watcher, LSP, `$ref` /
`$extends` (unless the work tokens use them), draft-schema support.

## Fixed decisions

- Source format: W3C DTCG 2025.10 only.
- Own parser (no asimonim dependency; it is GPL-3.0 and DTCG-only).
- Layering modelled on the DTCG Resolver module (ordered sets). Read that spec before card 2.2.
- Parse with `json.Decoder.Token()` into an ordered tree. Never `map[string]any`; it loses declaration order.
- Every token carries provenance: `file:line`, layer, and every value it replaced.
- Overrides are loud: an error or warning unless the layer sets `override: true`. A group colliding with a leaf at the
  same path is always an error.
- Deterministic output: sorted, stable.
- Exit codes: 0 ok, 1 findings (collisions, cycles, missing targets), 2 usage or I/O error. Keep this contract stable.
- stdout for results, stderr for diagnostics.
- Config precedence: flags > env > file.

## Open decisions (record answers in `docs/decisions.md`)

- Config file format and name.
- Layer declaration syntax.
- Name flattening for paths (`-` join?) once the emitter exists.
- Alias handling in Sass output: preserved refs vs flat (Stage B).
- Vars vs maps, `@use` layout, `!default`, value formatting (Stage B).

## Cards

One card = one session of 1–3 hours. Each ends with a green `just check` and an updated `NEXT.md`.

### M0: Prep (no Go)

- **0.1 Inventory.** List the work SD platforms, transforms, formats, file layout, and the exact places loading order or
  silent overrides bite. Write `docs/inventory.md`. *Done: inventory committed, with at least 3 concrete pain examples.*
- **0.2 Convert tokens to DTCG.** Throwaway vanilla ESM script `scripts/sd2dtcg.js`: `value` to `$value`, `type` to
  `$type`, `comment` to `$description`, `{a.b.value}` to `{a.b}`, `"16px"` to `{ "value": 16, "unit": "px" }`, drop
  SD-only attributes. *Done: converted tokens exist locally. Commit only synthetic or sanitised fixtures, never real
  work tokens, until IP and licensing are confirmed with the employer.*

### M1: Loader

- **1.1 Skeleton.** `go mod`, `justfile` (`check` = vet, test, lint), CI, `AGENTS.md`, `NEXT.md`. *Done: CI green on
  main.*
- **1.2 Ordered parser.** Token stream into an ordered tree with `file:line`. *Done: tests for order preservation,
  duplicate keys, positions.*
- **1.3 Token model.** Groups, `$type` inheritance, `$root`, name validation (no `.`, `{`, `}`). *Done: fixture tests
  including invalid names.*

### M2: Resolve

- **2.1 Aliases.** `{group.token}` resolution, cycle detection, errors naming file and line. *Done: cycle and
  missing-target fixtures.*
- **2.2 Layers and provenance.** Config, ordered merge, collision rules, provenance chain. *Done: collision fixtures;
  `override: true` works; group-vs-leaf is an error.*

### M3: Commands

- **3.1 `check`.** Wire loader and resolver; exit-code contract. *Done: exit-code tests.*
- **3.2 `explain`.** Origin chain output. *Done: golden-output test.*

**Tag v0.1.0 here.**

## Gate

Run filo against the real work tokens for a few weeks (non-blocking CI job). Proceed to Stage B only if it caught
something real or someone other than you used it. If not: ship tokensnap, consider an upstream PR (e.g. an SD-format
adapter for asimonim), and stop here with a finished Go CLI.

## Stage B: Sass emitter (post-gate)

- **B1 Baseline.** Scratch project on SD 5.4+ (5.3.3 lacks dimension support for 2025.10 as far as I know; verify).
  Generate goldens from converted tokens.
- **B2 Sass emitter.** Variables with preserved references; colour goldens pass.
- **B3 Dimensions and numbers.** `{value, unit}` formatting and precision. Log every deliberate diff in
  `docs/decisions.md`.
- **B4 `build` command.** Flags, deterministic sorted output.
- **B5 tokensnap hook.** Snapshot after each build, plus CI step.
- **B6 Release.** GoReleaser, pinned binary download in CI, completions, Homebrew tap.
- **B7 Parallel run.** Diff SD output against filo until the diff stays empty. Keep SD as the fallback.

## Rules for the gaps between sessions

1. End every session with one failing test (or `t.Skip("next: ...")`) for the next step.
2. Update the `NEXT.md` header before closing the laptop.
3. Record each decision in `docs/decisions.md`, one paragraph: what, why, what else was considered.
4. After a long break, start with a small mechanical card, not 2.2 or B3.
5. Keep a backlog of session-sized issues. When cold, take the top one.
6. Never leave main red.

## Pre-publish checklist

- [ ] `pkg.go.dev`, Codeberg, GitHub: no `filo` clash
- [ ] `which filo` and `brew search filo`: no binary clash
- [ ] Employer IP and licence position confirmed before use at work
- [ ] Own licence chosen
