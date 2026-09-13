# Portfolio Contribution Rebalancing Advisor - Design

**Date:** 2026-09-08  
**Status:** Brainstormed design; not implemented

## Purpose

Help an individual investor direct one upcoming contribution toward a target portfolio allocation using a practical number of purchases. The advisor is a strictly buy-only calculator, not a brokerage or trading system.

The recommendation should move the portfolio as close as possible to the target allocation while limiting the contribution to at most two purchased items. The full contribution remains eligible for investment; the advisor does not need to distribute it across every underweight item.

## Confirmed scope

- Primary user: individual investor.
- Primary goal: approach a user-defined target allocation.
- Contribution model: one upcoming contribution at a time.
- Portfolio entry: manual entry only.
- Allocation items: either broad asset classes or specific funds/ETFs.
- Purchase model: fractional shares and exact dollar allocations.
- Trading invariant: existing holdings are never sold or recommended for sale.
- Purchase limit: the advisor automatically chooses one or two items, never more than two.
- Optimization: choose the mathematically closest post-contribution allocation.
- Legacy holdings: holdings with a 0% target are allowed and never receive new contributions.
- Persistence: each calculation starts fresh; no saved portfolio or account is required.

## Out of scope for version one

- Any sale or sell recommendation, including selling 0%-target legacy holdings.
- Brokerage integrations, account authentication, or transaction execution.
- Whole-share limits, minimum order sizes, or brokerage-specific fees.
- Tax consequences, tax-loss harvesting, or account-location advice.
- Recurring contributions or automated scheduling.
- Portfolio persistence, cloud storage, or user accounts.
- More than two purchases for a single contribution.

The two-purchase maximum is the initial mechanism for limiting transaction burden. Actual brokerage costs are not modeled.

The buy-only rule is absolute: the advisor may recommend new purchases only. Existing positions remain unchanged, including positions with a 0% target allocation.

## User flow

1. The investor adds one or more allocation items.
2. For each item, the investor enters its current dollar value and target percentage.
3. The investor enters the amount of the upcoming contribution.
4. The advisor validates the inputs.
5. The calculation engine evaluates every valid one-item and two-item purchase combination.
6. The advisor presents the best contribution allocation.
7. The results show the recommendation and a before/after allocation comparison.

### Example input fields

Each item contains:

- Name, such as `US Total Market`, `International Stocks`, or `Bonds`.
- Current dollar value.
- Target percentage, including 0% for a legacy holding that should receive no new money.

The calculation also receives one positive contribution amount.

### Results

The result should show:

- The one or two selected items.
- The dollar amount directed to each selected item.
- The total contribution allocated.
- The current allocation percentage for every item.
- The projected allocation percentage after the contribution for every item.
- The remaining allocation drift.
- A plain-language explanation of why the selected items were chosen.

The result should make zero-dollar recommendations clear for items that should not receive the contribution.

## Allocation and optimization model

Let each portfolio item have:

- Current value `v_i`.
- Target weight `t_i`, where all target weights sum to 100%.
- Contribution amount `C`.

For a candidate purchase plan, `x_i` is the dollar amount contributed to item `i`. The constraints are:

- `x_i >= 0` for every item.
- The sum of all `x_i` equals `C`.
- At most two `x_i` values are non-zero.
- Items with a 0% target cannot have a positive `x_i`.
- Existing values `v_i` are never reduced; the plan contains no sell operation.

After the contribution, the projected portfolio value is `V + C`, where `V` is the current total value. The projected weight for item `i` is:

`p_i = (v_i + x_i) / (V + C)`

The advisor minimizes the total squared allocation error:

`sum((p_i - t_i)^2)`

This gives a precise, deterministic definition of "closest to target" while considering every portfolio item, including 0%-target legacy holdings.

The engine evaluates all eligible one-item and two-item combinations. For each combination it calculates the best non-negative dollar split whose total equals the contribution, then selects the candidate with the lowest error. A mathematically better two-item result wins even when the improvement is small. If candidates are exactly tied, the deterministic tie-breaker is fewer purchases, followed by stable item-name order.

The engine should normally select underweight items naturally through the objective. It must never contribute to a 0%-target item. If the portfolio is already exactly at target, the engine still selects the mathematically best eligible one- or two-item plan for the upcoming contribution.

## System boundaries and data flow

The first version is a stateless calculator with three logical components:

### Input layer

Collects the manually entered item rows and contribution amount. It should support adding, editing, and removing rows before calculation.

### Calculation engine

Accepts a validated input object and returns a recommendation plus metrics. It should not depend on UI state, browser APIs, brokerage APIs, or persistence.

### Results layer

Formats the recommendation, before/after percentages, remaining drift, and explanation for the investor.

The flow is:

`manual input -> validation -> candidate generation -> candidate optimization -> best recommendation -> before/after explanation`

The calculation engine is the key reusable boundary. It should be independently testable and usable by a future interface, scenario planner, or brokerage integration without changing the optimization rules.

## Validation and error handling

The calculator must reject invalid input rather than silently changing it.

Required validation rules:

- At least one allocation item exists.
- Item names are non-empty and unique.
- Current values are finite and zero or greater.
- Target percentages are finite and zero or greater.
- Target percentages sum to exactly 100% within a documented numeric tolerance for input parsing.
- At least one item has a positive target percentage.
- The contribution amount is finite and greater than zero.
- The current portfolio total is greater than zero.

Errors should identify the affected field and explain how to correct it. Examples include:

- "Target percentages must total 100%."
- "Contribution amount must be greater than zero."
- "Current value cannot be negative."
- "Item names must be unique."

Calculation failures should be surfaced as an actionable message, not a blank result. The result view should not display a recommendation until validation succeeds.

## Testing strategy

The calculation engine should have focused unit tests for:

- One underweight item receiving the full contribution.
- Several underweight items where the best result uses two items.
- A case where the best two-item split is uneven.
- A case where the mathematically closest result uses only one item.
- Zero-target legacy holdings receiving zero contribution.
- A portfolio already at target before a new contribution.
- A very small contribution that cannot materially remove drift.
- A large contribution that still must be allocated to no more than two items.
- Equal target distributions and deterministic tie-breaking.
- Invalid totals, negative values, duplicate names, empty names, zero contribution, and empty portfolio totals.

Tests should verify both the selected item set and the resulting dollar allocation, not only a final rounded percentage display.

## Success criteria

The design is successful when an investor can manually describe a portfolio, enter one contribution, and receive a deterministic recommendation that:

1. Allocates the full contribution.
2. Uses no more than two purchased items.
3. Never allocates new money to a 0%-target holding.
4. Produces the lowest post-contribution squared allocation error among all valid one- and two-item buy-only plans.
5. Clearly shows the projected result and remaining drift.
6. Gives actionable validation feedback for incorrect input.
