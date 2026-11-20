# filo v0.1: cards

Working checklist. Design detail lives in [docs/design.md](docs/design.md) (section numbers below). Read this file at the start of a session, not the design doc.

One card = one session (1 to 3 hours). Each card ends with `just check` green, a clean tree, and an updated `NEXT.md` header. Commit messages: `package: description`.

Heavy cards (do not start cold after a long gap): **1.2**, **2.2**. After a break of a month or more, start with 1.1, 1.3, 2.1 or a docs card.

Total: about 27 h (plan for 25 to 32).

- [ ] **0.1 Inventory** (1 h). Write `docs/inventory.md`: SD version, platforms/transforms/formats, file layout and loading order, how global and brand tokens combine, current override behaviour, warnings emitted and ignored, anything implicit. At least 3 concrete surprising examples: token path, expected source, actual source, why surprising, what `explain` should show.
  *Done:* inventory committed with 3+ examples. (No Go.)

- [ ] **0.2 Fixtures** (1.5 h). Synthetic fixtures from DTCG 2025.10 examples plus hand-authored layering cases (design 7). Record spec source URL, version and licence. Write expectations by hand. No company tokens.
  *Done:* `testdata/spec/` and `testdata/layers/` with written expectations.

- [ ] **1.1 Skeleton and diag** (2 h). Layout, `go.mod`, thin `main`, `run(args, stdout, stderr) int`, version stub, `justfile`, CI, `AGENTS.md`, `NEXT.md`, `docs/decisions.md`. Add `internal/diag` (`Diagnostic`, `Note`, formatting, stable sort; design 5.1).
  *Tests:* no args, help, version; diag formatting and sort order.
  *Done:* CI green; `go run ./cmd/filo` prints usage.

- [ ] **1.2 Ordered parser** (3 h) HEAVY. Recursive-descent parser with member order, duplicate-key diagnostics, byte offsets, depth cap, upfront UTF-8 check (design 6.1, 3.1).
  *Tests:* member order (10+ keys); duplicate keys; escapes/Unicode; number spellings; invalid JSON incl. comments and trailing commas; invalid UTF-8; depth cap. Optional: BOM, fuzz.
  *Done:* required tests pass; syntax errors return a diagnostic (offset only is fine until 1.3).

- [ ] **1.3 Positions** (1.5 h). Line index, `sort.Search` mapping, rune-based columns; syntax errors and duplicate keys report `line:col`.
  *Tests:* LF, CRLF, multibyte before an error, start and end of file.
  *Done:* `file:14:5: error: expected ':' [FILO001]`.

- [ ] **1.4 DTCG model** (2.5 h). `Path`, `Token`, `Group`, `File`, `Build`. Per-file `$type` inheritance, `$root`, `$description`, `$deprecated`, `$extensions`; unsupported features are errors `FILO012` (design 6.2, 4.3, 3).
  *Tests:* inheritance and override; invalid names; `$root`; invalid structure; unknown `$` keys; extensions preserved; type does not cross files.
  *Done:* `Build` on `testdata/spec/` gives the hand-written expectations.

- [ ] **2.1 Config and discovery** (2 h). `filo.config.json`, `fs.WalkDir`, sorting, hidden/symlink rules, unknown fields rejected, duplicate layer names, missing dir (error, exit 2), empty dir (warning FILO020), `override` default false (design 6.3). Decide and record config format.
  *Tests (`fstest.MapFS`):* layer order; sorting; extensions; hidden; symlinks; missing/empty dirs; duplicates; unknown fields; default override.
  *Done:* discovery matches a golden listing for `testdata/layers/`.

- [ ] **2.2 Merge and provenance** (3 h) HEAVY. `Set` indexing tokens **and groups**; `Merge` with the 4.4 rules; history oldest-first; deterministic entry order (design 6.4).
  *Tests:* every row of the 4.4 table; three-layer history; type-change warning; token/group collisions both ways; later-only token; `$root`.
  *Done:* `testdata/layers/` yields the expected FILO030 to FILO033.

- [ ] **2.3 Aliases** (2 h). `Refs` over the whole `$value` tree with per-reference positions; resolve against effective definitions only; FILO040 to FILO042. **Verify in the spec first:** `$root` alias syntax, alias type inheritance (design 4.5).
  *Tests:* plain; chain; multiple in a composite; nested in arrays/objects; missing; group target; malformed; partial braces and whitespace.
  *Done:* diagnostics name the referencing file and line.

- [ ] **2.4 Cycles** (2 h). Tarjan SCC in declaration order; one FILO043 per cyclic SCC with a shortest representative cycle and per-hop notes (design 4.6).
  *Tests:* self; 2-token; 3-token; tail into cycle; disjoint; overlapping in one SCC (one diagnostic); diamond; 1,000-token chain; identical output across repeated runs.
  *Done:* deterministic output.

- [ ] **3.1 `filo check`** (2 h). Wire the pipeline; sorted diagnostics and summary to stdout; operational errors to stderr; exit 0/1/2 (design 6.6, 5.3).
  *Tests:* clean; syntax error; invalid token; missing alias; cycle; collision; warning only; missing/invalid config; layer order; exit codes; stream separation.
  *Done:* `filo check` on `testdata/layers` behaves as documented. **Earliest useful stop point.**

- [ ] **3.2 `filo explain`** (2.5 h). Effective value and type, original definition, override history, every alias (including nested ones), source positions. Compact deterministic node renderer. Decide and record group-path and invalid-set behaviour; exit 1 for unknown token (design 6.6).
  *Tests:* no overrides; two+ overrides; alias chain; multiple/nested aliases in composites; rendering; unknown token; group path; `$root`; goldens (`UPDATE_GOLDEN=1`).
  *Done:* answers at least one real question from `docs/inventory.md`.

- [ ] **3.3 Release v0.1.0** (1.5 h). README (scope, limitations, config, `check` and `explain` examples, install), changelog, `runtime/debug.ReadBuildInfo` version with dev fallback, licence and IP review, name availability check. Before tagging: tests, `go vet`, `just check`, clean tree, `go install` from a clean clone, stdlib only, no company material.
  *Done:* tag `v0.1.0`; the Stage B gate (design 10) starts.

## Session rules

1. Start: read `NEXT.md`, run `just check`, then do the card.
2. End: `just check` green, tree clean, `NEXT.md` header updated (last session date, card in progress, next concrete task, any open question).
3. Record each decision in `docs/decisions.md`: what, why, what else was considered.
4. Never leave CI red. A `t.Skip("next: ...")` marker is optional.
