# Portfolio Contribution Rebalancing Advisor Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a stateless, manual-entry web calculator that allocates one contribution to at most two purchases while minimizing post-contribution allocation error.

**Architecture:** Use a small client-side TypeScript application with a pure domain layer and a vanilla DOM interface. Validation and optimization remain independent of the UI so the financial logic is deterministic, directly testable, and reusable. No server, account system, persistence, brokerage integration, or LLM decision-making is included.

**Tech Stack:** TypeScript, Vite, Vitest, jsdom, vanilla DOM APIs, and CSS.

**Spec:** `docs/superpowers/specs/2026-09-08-portfolio-contribution-rebalancing-advisor-design.md`

## Global Constraints

- Primary user: individual investor.
- Contribution model: one upcoming contribution at a time.
- Portfolio entry: manual entry only.
- Allocation items: either broad asset classes or specific funds/ETFs.
- Purchase model: fractional shares and exact dollar allocations.
- Trading invariant: existing holdings are never sold or recommended for sale.
- Purchase limit: the advisor automatically chooses one or two items, never more than two.
- Optimization: choose the mathematically closest post-contribution allocation.
- Legacy holdings: holdings with a 0% target are allowed and never receive new contributions.
- Persistence: each calculation starts fresh; no saved portfolio or account is required.
- The full contribution is allocated among the selected purchase items.
- The optimizer minimizes `sum((p_i - t_i)^2)` across all portfolio items.
- The result must show purchases, before/after percentages, remaining drift, and an explanation.

---

## File Map

The implementation should create these focused files:

- `package.json` - scripts and development dependencies.
- `tsconfig.json` - strict TypeScript compiler settings.
- `vite.config.ts` - Vite and Vitest configuration.
- `index.html` - application shell.
- `src/domain/types.ts` - shared domain interfaces and result types.
- `src/domain/validation.ts` - input validation and validation issue construction.
- `src/domain/optimizer.ts` - buy-only one/two-item optimization.
- `src/domain/explanations.ts` - deterministic explanation text derived from the result.
- `src/main.ts` - DOM wiring and application lifecycle.
- `src/ui/form.ts` - form rendering and input parsing.
- `src/ui/form.test.ts` - form parsing and row management tests.
- `src/ui/results.ts` - recommendation and before/after rendering.
- `src/ui/results.test.ts` - validation and result rendering tests.
- `src/styles.css` - focused layout and accessible visual styles.
- `src/domain/validation.test.ts` - validation tests.
- `src/domain/optimizer.test.ts` - optimizer tests and golden scenarios.
- `src/ui/app.test.ts` - jsdom-level user-flow test.
- `README.md` - local setup and calculator behavior.

---

### Task 1: Bootstrap the TypeScript testable application

**Files:**
- Create: `package.json`
- Create: `tsconfig.json`
- Create: `vite.config.ts`
- Create: `index.html`
- Create: `src/main.ts`
- Create: `src/styles.css`
- Create: `.gitignore`

**Interfaces:**
- Produces the commands `npm test`, `npm run build`, and `npm run lint` for every later task.

- [ ] **Step 1: Create the package manifest and scripts**

Use these scripts and development dependencies:

```json
{
  "name": "portfolio-contribution-rebalancing-advisor",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "test": "vitest run",
    "test:watch": "vitest",
    "build": "tsc --noEmit && vite build",
    "lint": "tsc --noEmit"
  },
  "devDependencies": {
    "jsdom": "latest",
    "typescript": "latest",
    "vite": "latest",
    "vitest": "latest"
  }
}
```

- [ ] **Step 2: Install the declared dependencies**

Run:

```powershell
npm install
```

Expected: `package-lock.json` is created and the command exits successfully.

- [ ] **Step 3: Configure strict TypeScript and Vitest**

`tsconfig.json` must enable strict checking and include `src`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "useDefineForClassFields": true,
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "allowImportingTsExtensions": false,
    "verbatimModuleSyntax": true,
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noEmit": true,
    "types": ["vitest/globals"]
  },
  "include": ["src", "vite.config.ts"]
}
```

`vite.config.ts` must configure jsdom for tests:

```ts
import { defineConfig } from "vite";

export default defineConfig({
  test: {
    environment: "jsdom",
    globals: true,
    include: ["src/**/*.test.ts"]
  }
});
```

- [ ] **Step 4: Add the minimal application shell**

`index.html` should contain a page title, an empty `<main id="app"></main>`, and the module entry `/src/main.ts`. `src/main.ts` should render a temporary heading into `#app`; later tasks replace that heading with the calculator. `src/styles.css` should contain only base box-sizing, body typography, and form defaults.

- [ ] **Step 5: Add the ignore file and run the baseline checks**

`.gitignore` must ignore `node_modules`, `dist`, coverage output, and editor-local files.

Run:

```powershell
npm run lint
npm test
npm run build
```

Expected: all commands pass; the test command reports zero test files without failing.

- [ ] **Step 6: Commit the bootstrap checkpoint**

```powershell
git add package.json package-lock.json tsconfig.json vite.config.ts index.html src/main.ts src/styles.css .gitignore
git commit -m "chore: bootstrap calculator application"
```

### Task 2: Define the domain model and input validation

**Files:**
- Create: `src/domain/types.ts`
- Create: `src/domain/validation.ts`
- Create: `src/domain/validation.test.ts`

**Interfaces:**
- Produces `PortfolioItem`, `CalculatorInput`, `ValidationIssue`, and `validateInput(input): ValidationIssue[]` for later tasks.

Use these types in `src/domain/types.ts`:

```ts
export type PortfolioItem = {
  id: string;
  name: string;
  currentValue: number;
  targetPercent: number;
};

export type CalculatorInput = {
  items: PortfolioItem[];
  contribution: number;
};

export type ValidationIssue = {
  field: string;
  message: string;
};

export type Purchase = {
  itemId: string;
  amount: number;
};

export type AllocationRow = {
  itemId: string;
  name: string;
  currentValue: number;
  targetPercent: number;
  currentPercent: number;
  projectedValue: number;
  projectedPercent: number;
  currentDrift: number;
  projectedDrift: number;
};

export type CalculationResult = {
  purchases: Purchase[];
  allocatedContribution: number;
  currentTotal: number;
  projectedTotal: number;
  errorScore: number;
  rows: AllocationRow[];
  explanation: string[];
};
```

The validation test file must define this explicit fixture helper:

```ts
function validInput(overrides: {
  contribution?: number;
  targetPercent?: number[];
} = {}): CalculatorInput {
  const targets = overrides.targetPercent ?? [60, 40];
  return {
    contribution: overrides.contribution ?? 100,
    items: [
      { id: "a", name: "A", currentValue: 600, targetPercent: targets[0] },
      { id: "b", name: "B", currentValue: 400, targetPercent: targets[1] }
    ]
  };
}
```

- [ ] **Step 1: Write failing validation tests**

Cover these exact cases:

```ts
it("accepts a valid portfolio and contribution", () => {
  expect(validateInput(validInput())).toEqual([]);
});

it("rejects target percentages that do not total 100", () => {
  const issues = validateInput(validInput({ targetPercent: [60, 30] }));
  expect(issues).toContainEqual({
    field: "items",
    message: "Target percentages must total 100%."
  });
});

it("rejects negative values, zero contributions, duplicate names, and empty names", () => {
  const issues = validateInput({
    contribution: 0,
    items: [
      { id: "a", name: "", currentValue: -1, targetPercent: 50 },
      { id: "b", name: "Same", currentValue: 100, targetPercent: 50 },
      { id: "c", name: "Same", currentValue: 100, targetPercent: 0 }
    ]
  });

  expect(issues.map((issue) => issue.message)).toEqual(expect.arrayContaining([
    "Contribution amount must be greater than zero.",
    "Item name cannot be empty.",
    "Current value cannot be negative.",
    "Item names must be unique."
  ]));
});

it("allows a 0%-target legacy holding but requires a positive-target item", () => {
  const input = {
    contribution: 100,
    items: [{ id: "legacy", name: "Legacy", currentValue: 100, targetPercent: 0 }]
  };
  expect(validateInput(input)).toContainEqual({
    field: "items",
    message: "At least one item must have a positive target percentage."
  });
});
```

The test helper must construct the valid input explicitly and allow targeted overrides; do not hide validation rules in a shared fixture.

- [ ] **Step 2: Run the validation tests to verify they fail**

Run:

```powershell
npm test -- src/domain/validation.test.ts
```

Expected: FAIL because `validateInput` does not exist yet.

- [ ] **Step 3: Implement exact validation behavior**

`validateInput` must return an issue for each invalid condition and must not mutate the input. Check, in order:

1. At least one item exists.
2. Names are non-empty after trimming.
3. Names are unique after trimming, case-insensitively.
4. Current values are finite and at least zero.
5. Target percentages are finite and between 0 and 100.
6. Target percentages sum to 100 within `1e-9`.
7. At least one target percentage is positive.
8. Contribution is finite and greater than zero.
9. Current portfolio total is greater than zero.

- [ ] **Step 4: Run the validation tests to verify they pass**

```powershell
npm test -- src/domain/validation.test.ts
```

Expected: PASS with every validation case covered.

- [ ] **Step 5: Commit the validation checkpoint**

```powershell
git add src/domain/types.ts src/domain/validation.ts src/domain/validation.test.ts
git commit -m "feat: add calculator input validation"
```

### Task 3: Implement the buy-only one/two-item optimizer

**Files:**
- Create: `src/domain/optimizer.ts`
- Create: `src/domain/explanations.ts`
- Create: `src/domain/optimizer.test.ts`

**Interfaces:**
- Consumes `CalculatorInput` and the types from `src/domain/types.ts`.
- Produces `calculateRecommendation(input: CalculatorInput): CalculationResult`.

- [ ] **Step 1: Write failing optimizer tests with explicit golden inputs**

Use fixtures with stable item IDs and assert exact purchase IDs, amounts, and projected rows. Include these cases:

```ts
it("allocates the full contribution to the best single purchase when that is optimal", () => {
  const result = calculateRecommendation({
    contribution: 100,
    items: [
      { id: "stocks", name: "Stocks", currentValue: 900, targetPercent: 90 },
      { id: "bonds", name: "Bonds", currentValue: 100, targetPercent: 10 }
    ]
  });

  expect(result.purchases).toEqual([{ itemId: "stocks", amount: 100 }]);
  expect(result.allocatedContribution).toBe(100);
});

it("chooses no more than two purchases and can split a contribution", () => {
  const result = calculateRecommendation({
    contribution: 300,
    items: [
      { id: "a", name: "A", currentValue: 100, targetPercent: 50 },
      { id: "b", name: "B", currentValue: 100, targetPercent: 30 },
      { id: "c", name: "C", currentValue: 800, targetPercent: 20 }
    ]
  });

  expect(result.purchases).toHaveLength(2);
  expect(result.purchases.reduce((sum, purchase) => sum + purchase.amount, 0)).toBeCloseTo(300);
  expect(result.purchases.every((purchase) => purchase.itemId !== "c")).toBe(true);
});

it("never buys a 0%-target legacy holding", () => {
  const result = calculateRecommendation({
    contribution: 200,
    items: [
      { id: "legacy", name: "Legacy", currentValue: 400, targetPercent: 0 },
      { id: "target", name: "Target", currentValue: 600, targetPercent: 100 }
    ]
  });

  expect(result.purchases).toEqual([{ itemId: "target", amount: 200 }]);
});

it("never reduces existing values and scores every item after the purchase", () => {
  const input: CalculatorInput = {
    contribution: 300,
    items: [
      { id: "a", name: "A", currentValue: 100, targetPercent: 50 },
      { id: "b", name: "B", currentValue: 300, targetPercent: 30 },
      { id: "legacy", name: "Legacy", currentValue: 600, targetPercent: 20 }
    ]
  };
  const result = calculateRecommendation(input);
  const purchased = new Map(result.purchases.map((purchase) => [purchase.itemId, purchase.amount]));

  for (const item of input.items) {
    const purchase = purchased.get(item.id) ?? 0;
    const row = result.rows.find((candidate) => candidate.itemId === item.id);
    expect(row?.projectedValue).toBe(item.currentValue + purchase);
  }
});
```

Add tests for an uneven two-item split, a portfolio already at target, equal target distributions, deterministic tie-breaking, and invalid input passed directly to the optimizer. The invalid-input test must assert that the thrown error message is exactly `Cannot calculate recommendation for invalid input.`.

- [ ] **Step 2: Run optimizer tests to verify they fail**

```powershell
npm test -- src/domain/optimizer.test.ts
```

Expected: FAIL because `calculateRecommendation` does not exist yet.

- [ ] **Step 3: Implement candidate generation and scoring**

Implement `calculateRecommendation` using this deterministic algorithm:

1. Call `validateInput`; if issues exist, throw an `Error` whose message is `Cannot calculate recommendation for invalid input.` The UI validates before calling, while direct callers cannot receive a silent result.
2. Compute `currentTotal = sum(currentValue)` and `projectedTotal = currentTotal + contribution`.
3. Build eligible candidates from items with `targetPercent > 0`.
4. Generate every one-item subset and every two-item subset from eligible candidates.
5. For a one-item subset, assign the full contribution to that item.
6. For a two-item subset `(a, b)`, compute desired purchase dollars:

   ```ts
   desiredA = (a.targetPercent / 100) * projectedTotal - a.currentValue;
   desiredB = (b.targetPercent / 100) * projectedTotal - b.currentValue;
   amountA = clamp((contribution + desiredA - desiredB) / 2, 0, contribution);
   amountB = contribution - amountA;
   ```

7. For each candidate, set all non-selected purchase amounts to zero and calculate every projected percentage.
8. Score a candidate with `sum((projectedPercent - targetPercent) ** 2)` across all items.
9. Choose the lowest score. For an exact score tie, choose fewer purchases; for a remaining tie, choose the candidate whose sorted item names come first lexicographically.
10. Drop zero-dollar purchases, preserve stable item order in the result, and return all rows and the score.

Define the helper used in step 6 as:

```ts
function clamp(value: number, minimum: number, maximum: number): number {
  return Math.min(Math.max(value, minimum), maximum);
}
```

No branch may subtract money from `currentValue`, create a purchase for a 0%-target item, or return more than two purchases.

- [ ] **Step 4: Add deterministic explanation output**

Create `buildExplanation(items: PortfolioItem[], purchases: Purchase[]): string[]` in `src/domain/explanations.ts`, call it from the optimizer, and return explanation strings with these exact facts:

- `The contribution was allocated across N purchase(s).`
- `Selected item names because this plan has the lowest post-contribution allocation error.`
- `Existing holdings are unchanged; this recommendation is buy-only.`
- If a 0%-target item exists: `0%-target holdings receive no new contribution.`

- [ ] **Step 5: Run the optimizer tests to verify they pass**

```powershell
npm test -- src/domain/optimizer.test.ts
npm run lint
```

Expected: PASS with exact purchase amounts within a documented currency tolerance such as `toBeCloseTo(value, 8)`.

- [ ] **Step 6: Commit the optimizer checkpoint**

```powershell
git add src/domain/optimizer.ts src/domain/optimizer.test.ts src/domain/explanations.ts
git commit -m "feat: add buy-only contribution optimizer"
```

### Task 4: Build the manual-entry form

**Files:**
- Create: `src/ui/form.ts`
- Modify: `src/main.ts`
- Modify: `src/styles.css`
- Create: `src/ui/form.test.ts`

**Interfaces:**
- `renderCalculator(root: HTMLElement, onCalculate: (input: CalculatorInput) => void): void`
- The form emits `CalculatorInput` only after parsing fields; validation remains in the domain layer.

- [ ] **Step 1: Write the form test**

Render the form into a jsdom root, populate two item rows and the contribution field, click the calculate button, and assert the callback receives:

```ts
{
  contribution: 1000,
  items: [
    { id: "item-1", name: "Stocks", currentValue: 6000, targetPercent: 70 },
    { id: "item-2", name: "Bonds", currentValue: 3000, targetPercent: 30 }
  ]
}
```

Also assert that the add-item button creates a new row and that removing a row updates the emitted items.

- [ ] **Step 2: Run the form test to verify it fails**

```powershell
npm test -- src/ui/form.test.ts
```

Expected: FAIL because `renderCalculator` does not exist yet.

- [ ] **Step 3: Implement accessible form controls**

Render:

- A labelled contribution input with `inputmode="decimal"`.
- A table of item rows with labelled name, current value, and target percentage inputs.
- Add-item and remove-item buttons.
- A calculate button.
- An empty validation region with `role="alert"`.

Each row must use a stable generated ID, preserve user input while adding/removing rows, and parse numeric inputs without rounding. Empty numeric fields must become `NaN` so domain validation can report them as invalid instead of treating them as zero.

- [ ] **Step 4: Run the form test to verify it passes**

```powershell
npm test -- src/ui/form.test.ts
```

Expected: PASS with the exact emitted `CalculatorInput`.

- [ ] **Step 5: Add focused styling and wire the form into `main.ts`**

Use a single-column layout that remains usable on narrow screens. Keep the result area separate from the input form. Do not add persistence, navigation, or unrelated visual components.

- [ ] **Step 6: Commit the form checkpoint**

```powershell
git add src/ui/form.ts src/ui/form.test.ts src/main.ts src/styles.css
git commit -m "feat: add manual portfolio entry form"
```

### Task 5: Render validation and recommendation results

**Files:**
- Create: `src/ui/results.ts`
- Create: `src/ui/results.test.ts`
- Modify: `src/main.ts`
- Modify: `src/styles.css`

**Interfaces:**
- `renderValidationIssues(root: HTMLElement, issues: ValidationIssue[]): void`
- `renderCalculationResult(root: HTMLElement, result: CalculationResult): void`

- [ ] **Step 1: Write result-rendering tests**

Assert that validation issues render as readable messages and that a calculation result renders:

- One or two purchase instructions.
- The total contribution.
- A before/after table for every item.
- Target, current percentage, projected percentage, and projected drift.
- The buy-only explanation.
- A visible 0%-target legacy holding row with a zero purchase amount.

- [ ] **Step 2: Run result tests to verify they fail**

```powershell
npm test -- src/ui/results.test.ts
```

Expected: FAIL because the render functions do not exist yet.

- [ ] **Step 3: Implement result rendering**

Use semantic headings, tables, and lists. Format dollar amounts to two decimals and percentages to two decimals, but keep calculation values unrounded internally. Escape user-provided item names by assigning them through `textContent`, never by concatenating them into HTML.

- [ ] **Step 4: Wire the complete flow**

`main.ts` must:

1. Render the form.
2. Clear old validation and result content when calculation starts.
3. Call `validateInput`.
4. Render issues and stop when issues exist.
5. Call `calculateRecommendation` only for valid input.
6. Render the calculation result.

- [ ] **Step 5: Run the result tests and full checks**

```powershell
npm test -- src/ui/results.test.ts
npm test
npm run lint
npm run build
```

Expected: all tests, type checks, and the production build pass.

- [ ] **Step 6: Commit the results checkpoint**

```powershell
git add src/ui/results.ts src/ui/results.test.ts src/main.ts src/styles.css
git commit -m "feat: render allocation recommendations"
```

### Task 6: Add documentation and perform final verification

**Files:**
- Create: `README.md`
- Modify: `src/domain/validation.test.ts`
- Modify: `src/domain/optimizer.test.ts`
- Modify: `src/ui/app.test.ts`

**Interfaces:**
- No new production interfaces. This task verifies all interfaces from Tasks 2-5 together.

- [ ] **Step 1: Add a complete user-flow test**

In `src/ui/app.test.ts`, mount the application in jsdom, enter a portfolio containing a 0%-target legacy holding, enter a contribution, submit the form, and assert:

- The result contains no more than two purchase rows.
- The legacy holding has a zero purchase amount.
- The displayed total purchase amount equals the contribution.
- Every existing value is represented in the projected rows without reduction.

- [ ] **Step 2: Add README usage documentation**

Document:

- `npm install`
- `npm run dev`
- `npm test`
- `npm run build`
- The buy-only rule.
- The maximum of two purchases.
- The treatment of 0%-target holdings.
- The fact that this is an allocation calculator, not financial, tax, or brokerage advice.

- [ ] **Step 3: Run the complete verification suite**

```powershell
npm test
npm run lint
npm run build
```

Expected: all commands pass with no unhandled console errors.

- [ ] **Step 4: Perform a manual browser smoke test**

Run:

```powershell
npm run dev -- --host 127.0.0.1
```

Verify manually that a user can add rows, enter values, calculate a result, see validation errors, and see the buy-only recommendation. Stop the dev server after the check.

- [ ] **Step 5: Review the diff against the design spec**

Confirm that the diff contains no brokerage integration, selling behavior, persistence, accounts, recurring contributions, or more than two purchase recommendations. Confirm all user-entered names render safely.

- [ ] **Step 6: Commit the completed vertical slice**

```powershell
git add README.md src/domain/validation.test.ts src/domain/optimizer.test.ts src/ui/app.test.ts
git commit -m "test: verify contribution advisor end to end"
```
