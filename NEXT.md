# NEXT

Update this before closing the laptop. Five lines of content, no more.

## Status

- **Last session:** YYYY-MM-DD
- **Card in progress:** none
- **Done so far:** nothing yet
- **Next step:** card 0.1, write `docs/inventory.md`
- **Mid-decision / open question:** none

## Re-entry

1. `git status`, then `git log --oneline -5`.
2. `just check`. It should be green. If it is not, fix that first.
3. Run the one failing or skipped test named under "Next step". That is where you stopped.
4. After a gap of more than a month, take a small mechanical card first (session rules).

## Example of a filled-in header

- **Last session:** 2027-02-14
- **Card in progress:** 2.1 Aliases
- **Done so far:** resolver handles plain and nested aliases
- **Next step:** `TestResolve_Cycle` is failing; cycle error needs file:line
- **Mid-decision / open question:** error type per finding, or one aggregate? (see docs/decisions.md)
