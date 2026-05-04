---
name: audit-tautological-tests
description: Find tests that pass without exercising the code under test — over-mocked tests where the mock is the implementation, snapshot tests with no behavioral checks, tests that assert on the mock's return value. Use proactively after a bug ships to "well-tested" code, or when test coverage is high but bug rate is too.
---

# Audit Tautological Tests

A tautological test is one where you delete the implementation and the test still passes — because the test is asserting against its own mocks, not against real behavior. These tests give false confidence and let bugs ship.

## When to use

- A bug shipped to code that had "100% coverage."
- High test coverage + high bug rate is a smell.
- Reviewing a PR with many new tests that look suspicious.
- Refactor risk-assessment: which tests would actually catch a regression?

## When NOT to use

- Test suite is genuinely good and bugs are escaping for other reasons (test environment doesn't match prod).
- Looking for slow tests, flaky tests, or coverage gaps — different audits.

## The five test smells

### 1. The mock is the implementation

```ts
// Tautological
test("createUser saves user", async () => {
  const mockSave = jest.fn().mockResolvedValue({ id: 1 });
  const repo = { save: mockSave };
  const result = await createUser(repo, { email: "a@b.com" });
  expect(result.id).toBe(1);                  // asserting on the mock's value
  expect(mockSave).toHaveBeenCalled();        // asserting it was called
});
```

What does this test? That `createUser` returns whatever `repo.save` returns. If `createUser`'s body is `return repo.save(...)`, the test passes. If it's `return null` (bug!), the test... still passes? No — the test asserts `result.id === 1`, but the mock returned `{ id: 1 }`, so `result` is `null` and the assertion fails.

OK, this one isn't quite tautological — but it's *close*. It tests pass-through, nothing else.

```ts
// Worse — fully tautological
test("createUser saves user", async () => {
  const repo = { save: jest.fn() };
  await createUser(repo, { email: "a@b.com" });
  expect(repo.save).toHaveBeenCalled();
});
```

Delete the implementation entirely (`async function createUser() {}`). Test fails (`save` not called). OK. Now make the implementation `repo.save({})`. Test passes. The test has no opinion on what gets saved or whether the email is validated.

**Fix:** assert on the *content* of what `save` was called with, and assert the result has the right shape:
```ts
expect(repo.save).toHaveBeenCalledWith(expect.objectContaining({
  email: "a@b.com",
  createdAt: expect.any(Date),
  status: "pending",
}));
```

### 2. Snapshot tests with no behavior contract

```ts
test("renders user card", () => {
  const tree = render(<UserCard user={fakeUser} />).toJSON();
  expect(tree).toMatchSnapshot();
});
```

Snapshot tests pass as long as the rendered output matches what was *recorded last time*. They catch *unintended* output changes — but only if someone reviews the snapshot diff, which often nobody does ("just run `jest -u` to update").

Snapshot tests for entire components rarely test the thing you care about: that clicking "Save" actually saves.

**Fix:** keep snapshots only for stable UI primitives (icons, brand assets). For interactive components, write behavior tests:
```ts
test("clicking save submits the form", async () => {
  const onSave = jest.fn();
  render(<UserCard onSave={onSave} />);
  await userEvent.click(screen.getByRole("button", { name: /save/i }));
  expect(onSave).toHaveBeenCalledWith(expect.objectContaining({ email: "a@b.com" }));
});
```

### 3. Mocking the system under test

```ts
test("PaymentService charges customer", () => {
  const svc = new PaymentService();
  svc.charge = jest.fn().mockResolvedValue({ id: "pay_1" });   // mocked itself!
  // ... rest of test
});
```

Or, more subtly:
```ts
jest.spyOn(svc, "validate").mockReturnValue(true);              // bypassing the validation
expect(await svc.charge(args)).toEqual({ id: "pay_1" });
```

The test confirms the *non-validate path* works — which is the path that doesn't exist in production. Real payments go through `validate`.

**Fix:** mock at the *boundary* (database, external API), not internal methods of the unit under test.

### 4. "Round-trip the mock"

```ts
test("API returns user", async () => {
  fetchMock.get("/api/users/1", { id: 1, email: "a@b.com" });
  const user = await client.getUser(1);
  expect(user).toEqual({ id: 1, email: "a@b.com" });
});
```

This tests that `client.getUser` makes an HTTP request and returns the JSON. It does *not* test that the API actually returns this shape. Production `/api/users/:id` could return `{ user_id, mail }` and this test would still pass.

**Fix:** if you're mocking the API, you're testing the client's *parsing*. Use a contract test or schema validator that lives near the actual API definition.

### 5. Tests that import production data

```ts
test("calculator sums correctly", () => {
  expect(sum(2, 2)).toBe(4);
});
test("calculator with real data", () => {
  const inputs = require("./fixtures/yesterday-prod-export.json");
  expect(sum(inputs.a, inputs.b)).toBe(inputs.expected);
});
```

The second test "passes" because the fixture was generated from the same code. There's no independent ground truth. If the implementation has the same bug as when the fixture was generated, the test never catches it.

**Fix:** ground truth must come from outside the system — the spec, the user requirement, a hand-computed example.

## The mutation test

The killer test for tautology: **mutate the implementation and see if any test fails.**

- Delete the function body entirely. Run tests. Any pass? Those tests are tautological.
- Replace return value with a constant. Run tests. Any still pass that shouldn't?
- Invert a conditional (`if (x)` → `if (!x)`). Run tests. Any still pass?

Tools like [Stryker](https://stryker-mutator.io) (JS/TS, .NET) or [mutmut](https://github.com/boxed/mutmut) (Python) automate this. Run them. The "mutation score" is the real signal of test quality, not coverage.

## Symptoms in your codebase

```bash
# Find tests that only assert "was called" without arguments
grep -r "toHaveBeenCalled()" tests/ | wc -l
# vs
grep -r "toHaveBeenCalledWith" tests/ | wc -l
```

If the first is much higher than the second, you have a tautology problem.

```bash
# Find snapshot tests
find . -name "__snapshots__" | xargs -I {} ls {} | wc -l
```

How many of these would survive deletion of the component's logic?

## What good tests look like

1. **Test the contract**, not the implementation. "When I call `chargeCustomer` with valid input, a `payments` row is written and a Razorpay order is created" — not "the function calls `db.save` and `razorpay.orders.create` in that order."
2. **Mock at the boundary**, not internally. The DB, the HTTP client, the file system. Not the service's own private methods.
3. **Prefer integration tests** for business logic; reserve unit tests for pure functions and tricky algorithms.
4. **Hand-write fixtures** that represent real shapes, not whatever the impl currently produces.
5. **Aim for mutation score** > 70% on critical code (payments, auth, data).

## Anti-patterns

- **`toHaveBeenCalled()` without `With`** — checks the function ran, not what it did.
- **Mocking everything** — leaves nothing real to test.
- **Snapshot tests as "fast and easy" coverage** — coverage without contract.
- **Mocking the unit under test's own methods** — bypasses the code you're testing.
- **`expect(actual).toEqual(actual)` (with renamed variables)** — yes, this happens.
- **`try { ... } catch { /* test passes */ }`** — error-swallowing tests.
- **Empty assertion blocks** — `test("works", () => { doThing(); });` — no `expect`. Linter rule should catch.
- **Tests with no DB / no HTTP for code that does both** — too pure to be useful.
- **Refusing to delete tautological tests because "coverage will drop"** — coverage was lying.
- **Adding tests after the bug ships, asserting the now-correct behavior, never asserting the wrong behavior wouldn't pass** — write the test before the fix; watch it fail; fix; watch it pass.

## Verify it worked

- [ ] Ran a mutation test (Stryker / mutmut) on critical modules; mutation score > 70%.
- [ ] Found tests where deleting the implementation didn't break them; they're either fixed or deleted.
- [ ] `toHaveBeenCalled()` count is much lower than `toHaveBeenCalledWith()` count.
- [ ] Snapshot tests cover only stable primitives, not interactive components.
- [ ] No test mocks methods of the unit under test.
- [ ] Critical path code (payments, auth) has integration tests against real DB / HTTP, not pure mocks.
- [ ] When a real bug shipped, you can identify which test *should* have caught it; that test is added or the gap is acknowledged.
- [ ] PR review surfaces tautology smells; `toHaveBeenCalled()` without args is a flag.
