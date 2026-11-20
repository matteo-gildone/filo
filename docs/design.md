# filo: v0.1 design

filo (Italian for "thread") is a Go CLI that loads DTCG token files in a defined order, resolves aliases, reports collisions and cycles, and explains where an effective value came from.

```
filo explain color.brand.primary
```

The command follows the thread from the effective token back through its definitions, overrides and aliases.

This is the reference document. Day-to-day work is tracked in `CARDS.md`; do not re-read this file on every session start.

---

## 1. Product boundary

### Stage A: v0.1.0, diagnostic CLI

Read-only. No Sass output. filo answers:

- Which definition currently wins?
- Was a value overridden, and is that override allowed by policy?
- Does an alias resolve, and where does it lead?
- Is there a missing reference or a cycle?
- Why does this token have its current value?

Style Dictionary remains the build system. filo runs alongside it.

### Stage B: possible Sass emitter

Consider only after v0.1.0 has run against real project tokens for a few weeks. Proceed only if the diagnostic CLI caught a real issue or another developer finds it useful. Otherwise ship the finished diagnostic CLI and stop; a useful, completed tool is a valid outcome. Keep the emitter a separate design decision that never expands v0.1.

---

## 2. Scope

### In scope for v0.1

- DTCG 2025.10 input, implementing the subset used by the project.
- Explicit, ordered token layers.
- Deterministic file discovery and processing.
- Token and group parsing; group `$type` inheritance (per file, see 4.3); `$root` tokens.
- Alias extraction and resolution, including aliases nested in composite values.
- Missing-alias and cycle diagnostics.
- Layer collision and override diagnostics; provenance and override history.
- `filo check` and `filo explain <token.path>`.

### Out of scope for v0.1

- Sass or any other output format; replacing Style Dictionary.
- Other platforms or transforms; watch mode; LSP.
- `$ref` and `$extends`, unless real project tokens require them (see 3).
- The full DTCG Resolver module; features or modifiers not used by the current token model.
- Validating `$value` against `$type` (filo does not check that a colour value is a valid colour).
- Automatic fixes or suggestions; JSON output unless a real integration requires it.

---

## 3. Honesty rules

- Do not claim full DTCG 2025.10 conformance. The README states the supported subset.
- **Unsupported features are errors, not warnings.** An unsupported `$extends` makes inherited tokens vanish, which then surfaces as false "missing alias" errors; failing early with the real cause is better. `$ref`, `$extends` and unknown `$`-prefixed keys produce `FILO012` (error), naming the feature and position.
- Do not infer a token's type from its value.
- Do not silently misinterpret anything. If in doubt, diagnose.

## 3.1 Limits

- Parser nesting depth is capped (suggested: 256). Exceeding it is an error (`FILO004`). A Go stack overflow is fatal and cannot be recovered, so this cap protects the process. Real token files nest roughly 5 to 15 levels deep.
- Graph algorithms (Tarjan) may be recursive: Go stacks grow dynamically, and token graphs are small. Keep a 1,000-token chain test and go iterative only if that test or real data shows a problem.

---

## 4. Semantics

### 4.1 Pipeline

```
config -> discover files -> parse (ordered) -> build (dtcg model, per file)
       -> merge layers (tokens + groups, provenance)
       -> extract and resolve aliases -> detect cycles
```

Alias resolution and cycle detection operate on **effective** definitions only (the winner per path). Overridden definitions are kept in history for `explain` but their references are not resolved or checked.

### 4.2 Layers and precedence

- `filo.config.json` declares an ordered array of layers; later layers have higher priority.
- This is filo's own simple ordered-layer configuration, informed by DTCG Resolver concepts. It is not an implementation of the Resolver module, and `override` is a filo policy, not a Resolver property.
- Files within a layer are processed in lexicographic order of their slash-separated relative path.

### 4.3 Group `$type` inheritance is per file

DTCG describes a single document. Layers merge several documents, which the format does not define. filo computes `$type` inheritance **within each file** and never across files or layers. A token added in a later layer under a group whose `$type` was declared in an earlier layer does not inherit that type. Record this in `decisions.md` and test it. (If it proves surprising in real use, revisit it; do not guess now.)

### 4.4 Merge rules

| Situation | Result |
|-----------|--------|
| Same token path in two files of the **same layer** | Error `FILO030` |
| Token already defined; later layer has `override: true` | Allowed; history recorded |
| Token already defined; later layer has `override` unset or false | Error `FILO031` |
| A path is a token in one place and a group in another (any layers) | Error `FILO032`, always |
| An allowed override changes the effective `$type` | Warning `FILO033` |
| Token defined only in a later layer | Valid |

The merged set must index **groups as well as tokens**. A token-only index cannot detect that `color.primary` (a token in layer 1) must be a group because layer 2 defines `color.primary.light`.

`$root` is an explicit path segment: `color.$root` is a token and `color` is its group. Do not collapse the root token into the group path. Verify the alias syntax for referencing root tokens against the 2025.10 text before writing tests.

### 4.5 Aliases

An alias is a string whose **entire** value is `{a.b.c}`. Scan the complete `$value` tree, including nested objects and arrays; every reference carries its own source position. Partial braces and ordinary strings containing braces are not aliases.

Diagnose: missing target (`FILO040`), target is a group (`FILO041`), invalid reference path or malformed reference (`FILO042`).

Alias type inheritance (a token with no explicit or inherited `$type` taking the type of an aliased target): verify against the applicable spec text before implementing. Do not guess.

### 4.6 Cycles

Use strongly connected components (Tarjan), iterating tokens and successors in declaration order so results are deterministic.

- Report **one diagnostic per cyclic SCC**. A cyclic SCC is one with two or more members, or a single member that references itself.
- Include a deterministic representative cycle: start at the earliest-declared member and take the shortest path back to it (breadth-first, successors in declaration order).
- Do not enumerate every elementary cycle. If A references B, B references A and C, and C references B, that is one component {A, B, C}, one diagnostic.
- The diagnostic names the component size, the representative path, and a note with `file:line` for each hop.

---

## 5. Diagnostics

### 5.1 Format

```
file:line:col: severity: message [CODE]
  note: file:line:col: message
```

Notes are optional continuation lines (previous definition, each hop of a cycle). Sort by file, line, column, code. Codes are stable once released; never reuse a code for a different meaning.

```go
package diag

type Severity int // Warning, Error

type Note struct {
    File      string
    Line, Col int
    Message   string
}

type Diagnostic struct {
    Severity  Severity
    Code      string
    Message   string
    File      string
    Line, Col int
    Notes     []Note
}
```

### 5.2 Codes

| Code | Severity | Meaning |
|------|----------|---------|
| FILO001 | error | JSON syntax error |
| FILO002 | error | Invalid UTF-8 |
| FILO003 | error | Duplicate key in a JSON object |
| FILO004 | error | Nesting depth limit exceeded |
| FILO010 | error | Invalid token or group name |
| FILO011 | error | Invalid token or group structure (e.g. malformed `$value`) |
| FILO012 | error | Unsupported feature (`$ref`, `$extends`, unknown `$` key) |
| FILO020 | warning | Layer directory contains no token files |
| FILO030 | error | Duplicate token definition within one layer |
| FILO031 | error | Override not permitted for this layer |
| FILO032 | error | Group-versus-token collision |
| FILO033 | warning | Override changes `$type` |
| FILO040 | error | Alias target missing |
| FILO041 | error | Alias target is a group |
| FILO042 | error | Invalid or malformed reference |
| FILO043 | error | Alias cycle |

Operational problems (missing config, unreadable file, missing layer directory, unknown config field) are not diagnostics: they go to stderr and exit with code 2.

### 5.3 Streams and exit codes

- Diagnostics and the summary go to stdout.
- Operational and usage errors go to stderr.
- Warnings alone do not fail the command.

| Command | 0 | 1 | 2 |
|---------|---|---|---|
| `check` | no errors | one or more error diagnostics | usage, config or I/O error |
| `explain` | token found | token not found, or path is a group | usage, config or I/O error |

`explain` exits 1 for "no such token" following grep's convention. Record this.

---

## 6. Components

All signatures are sketches. Names are yours to change; record changes in `decisions.md`.

### 6.1 `ordered`: order-preserving JSON tree

A small custom recursive-descent parser. It exists because the project needs member order, duplicate-key detection and source positions. Do not build a general-purpose JSON framework. (`encoding/json`'s `Decoder.InputOffset()` reports where a token ends, so recovering where one started means re-scanning whitespace yourself.)

```go
package ordered

type Kind int
const (Object Kind = iota; Array; String; Number; Bool; Null)

type Pos struct{ Offset, Line, Col int } // Line/Col added in 1.3; 1-based; Col in runes

type Node struct {
    Kind    Kind
    Pos     Pos
    Str     string
    Num     json.Number
    Bool    bool
    Members []Member
    Elems   []*Node
}

type Member struct {
    Key    string
    KeyPos Pos
    Value  *Node
}

// Syntax errors: nil tree, one FILO001/FILO002/FILO004 diagnostic.
// Duplicate keys: tree returned with both members kept, one FILO003 per duplicate.
func Parse(name string, src []byte) (*Node, []diag.Diagnostic)
func (n *Node) Get(key string) (*Node, bool) // first match
```

Rules:
- Keep numbers as raw spelling (`json.Number`); never convert to `float64`.
- Strict JSON: no comments, no trailing commas.
- Decode a string's contents by passing its raw byte slice to `json.Unmarshal` rather than writing an escape decoder.
- `encoding/json` silently replaces invalid UTF-8 with U+FFFD. Reject invalid UTF-8 explicitly in one upfront pass (`utf8.Valid`, then locate the first bad byte for the position).
- BOM policy (recommended: accept and skip a leading UTF-8 BOM, keeping offsets relative to the original bytes). Record the decision.
- Depth cap per section 3.1.

Required tests: member order (10+ keys); duplicate keys and positions; escapes and Unicode; number spellings; invalid JSON including comments and trailing commas; invalid UTF-8; depth cap.
Optional (do not let them stall the card): BOM, `FuzzParse` (must never panic), pathological inputs.

Positions (card 1.3): build line-start offsets once per file, map an offset with `sort.Search`, count columns in runes. Test LF, CRLF, multibyte characters before an error, errors at the start and end of a file.

### 6.2 `dtcg`: token model

```go
package dtcg

type Path []string
func (p Path) String() string
func ParsePath(s string) (Path, error)

type Token struct {
    Path        Path
    Type        string // effective, per file; "" if none
    Value       *ordered.Node
    Description string
    File        string
    Pos         ordered.Pos
}

type Group struct {
    Path Path
    File string
    Pos  ordered.Pos
}

type File struct {
    Name   string
    Tokens []Token // declaration order
    Groups []Group
}

func Build(name string, root *ordered.Node) (*File, []diag.Diagnostic)
```

Implement only what the project needs: group `$type` inheritance, token `$type`, `$root`, `$description`, `$deprecated`, `$extensions` (preserved as data). Preserve declaration order. Names must not start with `$` (except the reserved ones) or contain `.`, `{` or `}`.

Tests: group type inheritance; explicit type overriding inherited; invalid names; `$root` alone and alongside children; alias to `$root`; root value not an object; invalid `$value` structure; unknown `$` keys (error, FILO012); extension preservation; type inheritance does not cross files.

### 6.3 `config` and discovery

`filo.config.json`:

```json
{
  "layers": [
    { "name": "base",  "dir": "tokens/global" },
    { "name": "brand", "dir": "tokens/brand-a", "override": true }
  ]
}
```

Rules: discover `*.tokens` and `*.tokens.json`; use `fs.WalkDir`; ignore hidden files and symlinks; slash-separated `io/fs` paths internally; sort lexicographically by relative path; reject unknown config fields (`DisallowUnknownFields`); reject duplicate layer names; missing directory is an error (exit 2); empty directory is a warning (`FILO020`); `override` defaults to false; no glob syntax in v0.1.

```go
package config
type Layer struct{ Name, Dir string; Override bool }
type Config struct{ Layers []Layer }
func Load(fsys fs.FS, name string) (Config, error)

package load
type LayerFile struct{ Layer config.Layer; Path string }
func Discover(fsys fs.FS, cfg config.Config) ([]LayerFile, []diag.Diagnostic)
```

Tests: layer order; file sorting; supported extensions; hidden files; symlinks; missing and empty directories; duplicate layer names; unknown config fields; default override; slash paths. Use `fstest.MapFS`.

### 6.4 `load`: merge, collisions, provenance

```go
package load

type Definition struct {
    Layer string
    File  string
    Pos   ordered.Pos
    Type  string
    Value *ordered.Node
}

type Entry struct {
    Path    dtcg.Path
    Current Definition
    History []Definition // oldest first, includes Current
}

type ParsedFile struct {
    Layer config.Layer
    File  *dtcg.File
}

type Set struct { /* ordered entries, lookup indexes, group information */ }

func Merge(files []ParsedFile) (*Set, []diag.Diagnostic)
func (s *Set) Get(p dtcg.Path) (*Entry, bool)
func (s *Set) Entries() []*Entry // deterministic order
```

Parsing is outside `Merge`, so merge tests need no disk. Allowed overrides stay visible in provenance. Rules are in 4.4.

Tests: every row of the 4.4 table; three-layer override history; type change warning; token-to-group and group-to-token collisions; later-only token; deterministic entry order; provenance positions; `$root` collisions and overrides.

### 6.5 `resolve`: aliases and cycles

```go
package resolve

type Ref struct {
    From Path // token that contains the reference
    To   Path
    File string
    Pos  ordered.Pos
}

func Refs(t dtcg.Token) []Ref
func Resolve(s *load.Set) []diag.Diagnostic
func Cycles(s *load.Set) []diag.Diagnostic
```

Tests (aliases): plain alias; chain; multiple references in one composite value; references nested in arrays and objects; missing target; group target; malformed path; whitespace and partial-brace strings; inherited type behaviour (after verifying the spec).
Tests (cycles): self-reference; two-token; three-token; tail into a cycle; disjoint cycles; overlapping cycles in one SCC (one diagnostic); diamond (acyclic); chain of 1,000 tokens; repeated runs give identical output.

### 6.6 `cmd/filo`

`main` stays thin: `func run(args []string, stdout, stderr io.Writer) int`. One `flag.FlagSet` per subcommand; no Cobra.

`filo check [--config path]` (default `filo.config.json`): config, discover, parse, build, merge, resolve, cycles; diagnostics sorted and printed to stdout with a summary; exit codes per 5.3.

`filo explain <token.path>`: shows effective path, type and value; original definition; override history in order; alias references and where each resolves (every nested alias in a composite, not just the first); file and position for each. Use a compact deterministic renderer for ordered nodes; no formatting dependency.

Decisions to settle in the card and record:
- Group path: either list the children or return a clear error (suggested default: error that names the children, exit 1).
- Behaviour on an invalid token set: suggested default is best-effort. Skip unparsable files (report them on stderr), run the pipeline on what remains, and print any diagnostic that mentions the requested token as part of its output. `explain` is most useful when something is wrong, so it should not refuse to run on a set that `check` would reject.

Tests (`check`): clean project; syntax error; invalid token; missing alias; alias cycle; collision; warning only; missing config; invalid config; layer order; exit codes; stdout/stderr separation.
Tests (`explain`): token without overrides; two or more overrides; alias chain; multiple aliases in a composite; nested composite aliases; value rendering; unknown token; group path; `$root`; provenance; golden output.

---

## 7. Fixtures

Create small synthetic fixtures based on DTCG 2025.10 examples and the real project structure. For spec-derived fixtures record the source URL, version and licence. Spec examples are inputs; expectations (token paths, layer order, types, diagnostics, provenance) are written by hand.

Cover: global and brand layers; a normal override; a duplicate in one layer; a path that is a group in another layer; group `$type` inheritance and its non-inheritance across files; explicit token `$type`; `$root`; alias chains of three or more; missing aliases; cycles (including overlapping ones in one SCC); multiple aliases in a composite value; nested composites; aliases to `$root` tokens.

Do not commit company tokens without permission. A converter for real project files may be useful later and is not part of the initial implementation; if written it must report what it inferred, rewrote, dropped or could not convert, and never claim a conversion is lossless.

---

## 8. Working conventions

- Standard library only.
- Test first. Prefer table-driven tests with `t.Run`; `t.Helper` in helpers; `t.TempDir` for filesystem tests.
- Filesystem code accepts `fs.FS` where practical; use `fstest.MapFS` for in-memory tests.
- Golden files under `testdata/<case>/`, refreshed only with `UPDATE_GOLDEN=1`. Do not use a global `-update` flag: `go test ./... -update` fails in every package that does not declare it.
- Wrap errors with `%w`; typed errors where callers need `errors.As`. User-facing findings are `diag.Diagnostic`, not bare errors.
- Commit messages: `package: description`.
- End of session: run `just check`, leave the repository clean, update the `NEXT.md` header with the next concrete task. A `t.Skip("next: ...")` marker is an optional aid, not a ritual; never leave CI red.

## 9. Decisions to record in `docs/decisions.md`

Config format; file extensions and discovery; layer order and precedence; override policy; group-versus-token structure; `$root` path representation; per-file `$type` inheritance; only-effective-definitions-are-resolved; unsupported-feature handling; UTF-8 and BOM policy; nesting depth cap; duplicate-key handling; diagnostic codes, ordering and output streams; alias type inheritance; SCC and cycle reporting; `explain` group-path and invalid-set behaviour; exit codes for `explain`. For Stage B: Sass naming and flattening, variables versus maps, `@use` and `!default`, exact Style Dictionary version and compatibility expectations.

Do not leave important behaviour implicit in implementation details.

## 10. Stage B gate

Before any emitter work: run filo against the real project tokens (if permitted; this needs them in DTCG form, so write a throwaway converter then), use it in non-blocking CI for a few weeks, record the real issues it catches, ask whether another developer finds `explain` useful, then decide whether the evidence justifies an emitter. If there is demonstrated value, investigate separately: exact Style Dictionary version and output behaviour, supported dimensions and composites, Sass naming, `@use`, `!default`, output ordering, and how to compare filo output with Style Dictionary.

If the diagnostic CLI does not demonstrate value, finish it, consider finishing tokensnap or an upstream Style Dictionary contribution, and stop.

## 11. Definition of done for v0.1.0

- `filo check` reliably validates the supported token subset and the layer policy.
- `filo explain` makes the effective value and its provenance understandable.
- Results are deterministic and diagnostics carry useful source locations.
- The test suite covers the documented behaviour.
- The README explains scope and limitations honestly.
- The tool installs and runs from a clean checkout (`go install`); only the standard library is used.
- No company tokens or confidential material are committed.
- No Sass emitter or unrelated feature has crept into the release.
