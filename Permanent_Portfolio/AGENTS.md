# Portfolio Contribution Rebalancing Advisor

## Mission

Build a deterministic, client-side contribution allocation calculator for an individual investor.

## Product invariants

- Buy-only: never sell or recommend selling any existing holding.
- A contribution may be allocated to at most two items.
- The full contribution is allocated among the selected purchase items.
- Holdings with a 0% target may remain in the portfolio but must never receive new money.
- Use manual portfolio entry only.
- Use fractional-share, dollar-based calculations.
- Do not add brokerage integration, persistence, accounts, taxes, or recurring contributions without explicit approval.
- The optimizer minimizes the total squared post-contribution allocation error.

## Engineering workflow

- Use `superpowers:brainstorming` before new behavior, features, or architectural changes.
- Do not implement before the design is approved and an implementation plan exists.
- Use `superpowers:test-driven-development` before writing implementation code.
- Keep the calculation engine pure and independent of the UI.
- Prefer small, independently testable modules with explicit interfaces.
- Run focused tests after each task and the full verification suite before completion.
- Review the diff and test output before committing.
- Do not broaden scope to financial advice, brokerage execution, or data persistence.

## Expected commands

- `npm test` - run the full test suite.
- `npm run build` - type-check and build the client application.
- `npm run lint` - run static checks when configured.

## Completion standard

Before claiming completion, verify the buy-only invariant, the two-purchase maximum, 0%-target behavior, validation errors, and the before/after allocation output with automated tests.
