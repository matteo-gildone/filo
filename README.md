# # filo

Follow a design token back to where it came from.

filo loads [W3C Design Tokens (DTCG) 2025.10](https://www.designtokens.org) files in a defined order, resolves aliases, reports collisions and cycles, and tells you which file and layer produced any value.

> **Status: pre-alpha. Nothing below is implemented yet.** Commands shown under "Planned usage" describe the intended interface and will change.

## Why

When token files are merged from several sources, the result depends on glob and file order. Overrides are silent, so you find out about a collision when the output looks wrong, and then you have to work out which file won.

filo makes that visible:

- **Explicit, ordered layers.** You declare the order. Within a layer, files load in a documented, deterministic order.
- **Provenance on every token.** Each value records its `file:line`, its layer, and every value it replaced.
- **Loud overrides.** An override is a finding unless the layer is marked as allowed to override. A group colliding with a leaf token at the same path is always an error.
- **Deterministic output.** Sorted and stable, so diffs are meaningful.

## Planned usage

```sh
# Validate, resolve, and report collisions, cycles and missing targets
filo check

# Show where a value came from
filo explain color.brand.primary
```

Expected `explain` output shape:

```
color.brand.primary = #0a5cff
  defined   tokens/base/color.tokens.json:14      (layer: base)
  overridden tokens/brand-a/color.tokens.json:3   (layer: brand)
  alias     {color.blue.500}
```

### Exit codes

| Code | Meaning |
|------|---------|
| 0 | OK |
| 1 | Findings (collisions, cycles, missing alias targets) |
| 2 | Usage or I/O error |

Results go to stdout, diagnostics to stderr.

## Scope

filo targets the stable **DTCG 2025.10** format only. The draft format is not supported.

Layering is modelled on the DTCG Resolver module (ordered sets) rather than an invented config format.

### Non-goals (for now)

- Platforms other than the ones its author needs
- A transform plugin system
- Watch mode, language server
- Replacing a full build tool before the diagnostic part has proven useful

## Roadmap

Detail lives in [PLAN.md](PLAN.md). This is the summary.

- [ ] **v0.1.0: diagnostics.** Ordered parser, token model, alias resolution with cycle detection, layers with collision rules and provenance, `check` and `explain`.
- [ ] **Gate.** Run against real-world token sets for a few weeks. Continue only if it surfaces real problems.
- [ ] **v0.2: Sass output.** `build` command, Sass variables with preserved references, structured colour and dimension formatting, validated against Style Dictionary output.
- [ ] **v0.3: integration.** Snapshot support with [tokensnap](https://codeberg.org/), CI recipes, a parallel-run mode against an existing Style Dictionary pipeline.
- [ ] **Later, if justified:** `$ref` and `$extends`, further output formats, Style Dictionary to DTCG migration helper.

## Install

Not yet available. Planned: prebuilt binaries via GoReleaser, `go install`, and a Homebrew tap.

## Development

```sh
just check   # vet, test, lint
```

See [NEXT.md](NEXT.md) for the current state of work and [docs/decisions.md](docs/decisions.md) for design decisions.

## Licence

To be decided before the first public release.
