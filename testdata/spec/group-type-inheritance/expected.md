# Group type inheritance

## Expected groups, in declaration order

1. spacing
2. spacing.compact
3. spacing.scale

## Expected tokens, in declaration order

| Path | Effective type | Value | Reason |
|------|----------------|-------|--------|
| spacing.small | dimension | {"value":4,"unit":"px"} | Inherits spacing's type. |
| spacing.compact.small | dimension | {"value":2,"unit":"px"} | compact declares no type, so inheritance continues from spacing. |
| spacing.scale.factor | number | 1.5 | The nearest parent group, scale, declares number. |
| spacing.label | string | "compact" | The token's explicit type overrides the parent group's type. |

## Diagnostics

None.

## Scope

All inheritance occurs within one file.
Cross-file and cross-layer non-inheritance are separate cases.