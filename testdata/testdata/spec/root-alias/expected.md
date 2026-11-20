# Root token alias

## Purpose

Verify that an alias can reference a group's root token using an explicit
`$root` path segment.

A group and its root token are distinct entries. Referencing the group must
not implicitly select its root token.

## Input

`input.tokens.json`

## Expected groups, in declaration order

1. `color`
2. `color.accent`

## Expected tokens, in declaration order

| Path | Effective type | Raw value |
|------|----------------|-----------|
| `color.accent.$root` | `color` | `{"colorSpace":"srgb","components":[0.2,0.4,0.8]}` |
| `color.accent.light` | `color` | `{"colorSpace":"srgb","components":[0.7,0.8,1]}` |
| `color.brand` | `color` | `"{color.accent.$root}"` |

## Type inheritance

- `color.accent.$root` inherits `color` from the enclosing `color` group.
- `color.accent.light` inherits `color` from the enclosing `color` group.
- `color.brand` has the resolved type of its referenced token: `color`.

## Alias resolution

Exactly one reference is present:

`color.brand` → `color.accent.$root`

The resolved value of `color.brand` is:

    {"colorSpace":"srgb","components":[0.2,0.4,0.8]}

Resolution must preserve the alias's original raw value for explanation.

## Root token behaviour

- `color.accent` remains a group.
- `color.accent.$root` is a token.
- `color.accent.light` is a separate child token.
- The root token's path retains the explicit `$root` segment.
- No root token is created for `color`.

## Definition provenance

All groups, tokens and the alias reference originate in
`input.tokens.json`.

There are no overrides. Each token has one definition.

## Diagnostics

None.

The alias target exists, is a token and does not participate in a cycle.

## CLI expectations

- `filo check` succeeds with exit code `0`.
- `filo explain color.brand` succeeds with exit code `0`.
- The explanation identifies `color.accent.$root` as the alias target and
  shows its resolved value.
- Neither command writes to stderr.

Exact output formatting is outside this fixture's expectations.

## Scope

This is the valid root-alias case.

An alias to `{color.accent}` belongs in a separate negative fixture:
it targets a group and must produce `FILO041`, not resolve implicitly
to the group's root token.
