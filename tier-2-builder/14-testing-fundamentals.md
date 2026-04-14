# Testing Fundamentals

## What is it?

Testing is the practice of writing code that verifies your code works correctly. You write a separate piece of code that exercises your application, then checks that the output matches your expectations.

Think of it like a quality inspector on an assembly line. The inspector doesn't build the product — they test it. They have a checklist: "Does this button respond to clicks? Does this form reject invalid email addresses? Does the search return relevant results?" Tests are that checklist, automated.

The testing pyramid organizes tests by scope and speed:

- **Unit tests** test a single function or component in isolation. Fast to run (thousands per second), no external dependencies.
- **Integration tests** test how multiple pieces work together — a feature that spans a database, an API route that calls a service.
- **End-to-end (E2E) tests** test the entire application from the user's perspective, simulating a real browser or HTTP client.

The pyramid shape matters because each level is slower and more expensive to run. You want many fast unit tests at the base. Fewer integration tests in the middle. A small number of E2E tests at the top that verify critical user paths.

## Why it matters for vibe coding

AI generates code quickly. That speed is a liability without tests — AI will confidently produce broken code that appears to work, because you never exercised the edge cases. Here are the specific failure modes:

**Without tests, AI regressions are invisible.** You ask AI to "add a discount feature." It modifies three files. Two weeks later you discover that existing orders now calculate shipping wrong. You have no idea what changed or why. Tests catch this automatically.

**Without tests, you cannot safely edit AI-generated code.** AI-generated code often needs modification — fixing a bug, adding a field, changing a business rule. Without a test suite, you have no way to know if your edit broke something. With tests, you run them and know immediately. This is why tests are not optional when working with AI — they are the safety net that makes iteration possible.

**AI generates tests that cover the wrong thing.** AI loves writing tests for getters and setters — code that is already correct by definition. It often skips tests for business logic, edge cases, and error conditions. High coverage from useless tests gives you a false sense of security.

**AI generates snapshot tests without judgment.** Snapshot testing saves a rendered output and compares future runs against it. AI will generate snapshot tests for everything, and then when the output changes (intentionally or not), it just updates the snapshot file rather than telling you whether the change is correct. You end up with tests that pass but don't verify anything real.

**AI generates tests that pass but don't test the right thing.** A test that always passes tells you nothing. AI will write tests where the assertion is too loose, the test doesn't actually exercise the code path, or the mock is wrong. You need to understand what a good test looks like to catch this.

## The 20% you need to know

### What each test type is actually testing

**Unit tests** are for pure logic — functions that take input and produce output with no side effects. A unit test calls that function with specific arguments and asserts the return value is what you expect. If your function queries a database, that's not a unit test — that's an integration test.

```python
# Unit test — this function is pure logic
def calculate_discount(price: float, discount_percent: float) -> float:
    return price * (1 - discount_percent / 100)

def test_discount_calculation():
    assert calculate_discount(100, 20) == 80
    assert calculate_discount(100, 100) == 0
    assert calculate_discount(50, 15) == 42.50
```

**Integration tests** verify that pieces work together. This means actual database calls, real HTTP calls, file system access. If it touches more than one unit, it's an integration test. Integration tests are slower but catch bugs that unit tests cannot — wrong SQL queries, misconfigured HTTP clients, transaction boundaries.

```python
# Integration test — this actually hits the database
def test_order_creation(db):
    order = OrderService.create(user_id=1, items=[{"product_id": 1, "qty": 2}])
    assert order.id is not None
    assert order.total == calculate_discount(products[1].price * 2, user.discount)
```

**E2E tests** simulate a real user. They open a browser and click through the application, or make real HTTP requests to a running server. E2E tests catch the bugs that unit and integration tests miss: routing issues, frontend state bugs, authentication flows, rendering problems.

```javascript
// E2E test with Playwright
test('user can checkout', async ({ page }) => {
  await page.goto('/products');
  await page.click('[data-testid="add-to-cart"]');
  await page.click('[data-testid="checkout"]');
  await expect(page.locator('.order-confirmation')).toBeVisible();
});
```

### The testing pyramid in practice

Write unit tests for:
- Pure functions with business logic
- Data transformations
- Validation logic
- Algorithm correctness

Write integration tests for:
- Database operations
- API routes
- Service-to-service communication
- Authentication flows

Write E2E tests for:
- Critical user journeys (signup, checkout, login)
- Flows that span multiple pages
- Anything that involves the browser

The right ratio is roughly 70% unit, 20% integration, 10% E2E. If you have too many E2E tests, they run slow and break often. If you have too few unit tests, your business logic has no safety net.

### Why high coverage ≠ good coverage

Code coverage measures what percentage of your code is exercised by tests. 80% coverage means 80% of your lines were executed during the test run. This sounds good. It is not.

Coverage tells you what code ran — not whether the assertions were meaningful. AI can write tests that call every line of a getter function:

```python
def test_user_model():
    user = User(name="Alice", email="alice@example.com")
    assert user.name == "Alice"  # covers the getter
    assert user.email == "alice@example.com"  # covers the getter
    # 100% coverage on a getter/setter, 0% coverage on business logic
```

This test has 100% coverage on those lines and proves nothing about whether your application works. Good tests cover *behavior*, not *lines*.

Target coverage on business logic functions (the 20% that contains your actual rules). Do not target 100% overall coverage — it incentivizes useless tests.

### Test doubles: mocks, stubs, and fakes

When testing, you often need to replace real dependencies with controlled stand-ins. These are test doubles:

**Mocks** verify interactions happened — that a method was called with specific arguments. Use mocks to verify that your code called the right functions with the right data. Do not use mocks to fake return values; use stubs for that.

**Stubs** provide predetermined responses to calls — replacing a real service with a fake one that returns known data. Use stubs when you want to test your code's behavior given a specific response, without depending on the real service.

**Fakes** are simplified working implementations — an in-memory database instead of a real one, a fake email sender that just logs instead of sending. Fakes are useful when you need something that behaves like the real thing but is faster and more controlled.

```python
# Stub — provides fake response
def test_order_total_with_tax():
    fake_pricing = StubPricingService(prices={"SKU1": 100})
    order = Order(pricing_service=fake_pricing)
    assert order.total == 110  # 100 + 10% tax

# Mock — verifies the call happened
def test_order_sends_confirmation_email():
    mock_email = Mock()
    order = Order(email_service=mock_email)
    order.confirm()
    mock_email.send.assert_called_once_with(
        to=order.user.email,
        subject="Order Confirmed"
    )

# Fake — in-memory implementation
class FakeDatabase:
    def __init__(self):
        self.data = {}
    def insert(self, table, row):
        self.data.setdefault(table, []).append(row)
    def select(self, table, filters=None):
        rows = self.data.get(table, [])
        if filters:
            rows = [r for r in rows if all(r[k] == v for k, v in filters.items())]
        return rows
```

**Mocks can hide bugs.** If you mock a database call but write the wrong assertion, you have a test that passes while your real code is broken. The mock hides the bug. Prefer stubs and fakes over mocks when possible. Use mocks only when you need to verify that a specific interaction occurred — not just that your code ran without errors.

### TDD basics for constraining AI

Test-Driven Development (TDD) is a workflow where you write the test before you write the code:

1. Write a failing test that describes the behavior you want.
2. Write the minimal code to make it pass.
3. Refactor to clean up.

In vibe coding, TDD is useful because it *constrains* what AI produces. When you write the test first, you define exactly what the code must do before AI writes it. AI cannot deviate from the spec because the test is the spec.

```bash
# TDD workflow with AI
# 1. You write the test
def test_discount_applies_to_order():
    order = create_order_with_items(total=100, discount_percent=20)
    assert order.final_total() == 80

# 2. You give AI the test and say "make this pass"
# 3. AI writes code to satisfy the test
# 4. You run the test to verify
# 5. If it passes, the feature is done
```

The test defines the contract. AI fills in the implementation. This reverses the normal vibe coding flow — instead of AI producing code you then test, you produce the test and AI produces the code that satisfies it.

You do not need to do full TDD (write all tests before any code). Even writing a few key tests before asking AI to implement a feature gives you a safety net.

### Property-based testing

Property-based testing generates random inputs to test that your code satisfies certain properties. Instead of writing specific test cases, you describe a property that must always hold true.

```python
from hypothesis import given, strategies as st

# Property: reversing a list twice returns the original list
@given(st.lists(st.integers()))
def test_reverse_twice_returns_original(items):
    assert reverse(reverse(items)) == items

# Property: sorting produces a non-decreasing sequence
@given(st.lists(st.integers()))
def test_sort_is_monotonic(items):
    sorted_items = sorted(items)
    for i in range(len(sorted_items) - 1):
        assert sorted_items[i] <= sorted_items[i + 1]
```

This is powerful because it finds edge cases you would never think to test manually: empty lists, very large numbers, negative numbers, special characters. AI tools almost never generate property-based tests, but they are particularly valuable for catching AI-introduced bugs — AI often handles the happy path correctly but fails on unusual inputs.

### Snapshot testing

Snapshot testing saves the rendered output of a component or data structure to a file, then compares future runs against that saved snapshot. If the output changed, the test fails and shows you the diff.

Snapshot tests are useful for:
- UI components where visual regression matters
- Serialized data structures with known formats
- API responses that should be stable

Snapshot tests are dangerous when used without judgment because they accept any change without evaluating whether the change is correct. When a snapshot test fails, you must manually review the diff and decide if the new output is correct. AI will often just auto-update the snapshot without telling you whether the change was intentional.

## Hands-on exercise

**Write a test suite that catches an AI-introduced regression.**

You will use Python with pytest. You will write business logic, add tests, then simulate an AI edit that breaks the logic — and verify that your tests catch it.

**Time: 15 minutes**

1. Create a project and install dependencies:
```bash
mkdir testing-lab && cd testing-lab
python -m venv venv && source venv/bin/activate  # Windows: venv\Scripts\activate
pip install pytest hypothesis
```

2. Create the business logic file `pricing.py`:
```python
# pricing.py
def calculate_line_total(price: float, quantity: int, discount_percent: float = 0) -> float:
    """Calculate total for a line item, applying quantity discount."""
    subtotal = price * quantity
    if quantity >= 10:
        discount_percent += 5  # bulk discount
    discount_amount = subtotal * (discount_percent / 100)
    return subtotal - discount_amount

def calculate_order_total(line_totals: list[float], tax_percent: float = 10) -> float:
    """Calculate order total from line items, adding tax."""
    subtotal = sum(line_totals)
    tax = subtotal * (tax_percent / 100)
    return subtotal + tax
```

3. Write tests that cover the business logic:
```python
# test_pricing.py
import pytest
from pricing import calculate_line_total, calculate_order_total

def test_line_total_basic():
    assert calculate_line_total(100, 1) == 100

def test_line_total_with_discount():
    assert calculate_line_total(100, 1, discount_percent=20) == 80

def test_bulk_discount_applies_at_10_units():
    # 10 units gets 5% bulk discount (on top of any other discount)
    # price 100, qty 10, no other discount: 1000 - 5% = 950
    assert calculate_line_total(100, 10) == 950

def test_bulk_discount_stacks_with_other_discount():
    # price 100, qty 10, 10% other discount: 1000 - 15% = 850
    assert calculate_line_total(100, 10, discount_percent=10) == 850

def test_order_total_with_tax():
    line_totals = [100, 200, 50]
    assert calculate_order_total(line_totals) == 385  # 350 + 10% tax

def test_order_total_empty():
    assert calculate_order_total([]) == 0
```

4. Run the tests to verify they pass:
```bash
pytest test_pricing.py -v
```

5. Simulate an AI edit that introduces a bug. Edit `pricing.py` to remove the bulk discount logic:
```python
# pricing.py — AI "helpfully" removed the bulk discount
def calculate_line_total(price: float, quantity: int, discount_percent: float = 0) -> float:
    subtotal = price * quantity
    # AI removed the bulk discount because it "simplified the code"
    discount_amount = subtotal * (discount_percent / 100)
    return subtotal - discount_amount
```

6. Run the tests again:
```bash
pytest test_pricing.py -v
```

The bulk discount tests will fail. Your test suite caught the regression. This is the workflow: write tests that cover business rules, then use AI to implement features, then run tests to verify nothing broke.

## Common mistakes

**Mistake 1: Testing implementation details instead of behavior.**

What happens: You write tests that check internal state, private methods, or the specific way a function is implemented. When you refactor the internals (which AI often does), the tests break even though the behavior is correct.

Why it happens: It is easier to assert on internal variables than to think about what the user actually sees and experiences.

How to fix: Write tests that call the public interface and check observable outcomes. Test what the function returns, what it writes to the database, what it sends to the user. Do not test that a private `_process_items` method was called — test that the correct order total was returned.

**Mistake 2: Not testing edge cases and error conditions.**

What happens: Tests cover the happy path (valid input produces valid output) but never test boundary conditions: empty input, very large numbers, negative values, malformed data. AI-introduced bugs often live in these edge cases.

Why it happens: Happy path tests are easier to write and more obvious. Edge cases require thinking about what could go wrong.

How to fix: After writing happy path tests, add tests for: empty collections, zero, negative numbers, extremely large values, None/null values, malformed input. Property-based testing automates this.

**Mistake 3: Tests that depend on execution order.**

What happens: Test B depends on state created by Test A. When you run tests individually they pass; when you run the full suite they fail. Or tests pass on your machine but fail in CI.

Why it happens: Tests share mutable global state (a database, a global variable, a file system) without resetting it between runs.

How to fix: Each test must set up its own state and clean up after itself. Use fixtures or setup methods to create fresh state for each test. Tests should be independent — any test should be able to run in isolation.

**Mistake 4: Asserting on floating-point values with ==.**

What happens: `assert 0.1 + 0.2 == 0.3` fails in most languages because floating-point arithmetic is imprecise.

Why it happens: 0.1 + 0.2 in binary floating point is actually 0.30000000000000004.

How to fix: When testing floating-point calculations, use approximate equality: `assert abs(result - expected) < 0.001` or use language-specific tolerance assertions like `pytest.approx()` in Python.

**Mistake 5: Mocking everything, including things that should not be mocked.**

What happens: You mock a database, a service, and an API call, then test that your code ran the mock. The test passes but the real integration is broken.

Why it happens: Mocks make tests fast and reliable, but they also remove the real behavior. If you mock everything, you test nothing.

How to fix: Mock at the boundary of your system — external HTTP calls, third-party services. Do not mock internal components that you are actually testing. Integration tests should use real databases, not in-memory fakes, when testing database logic.

## AI-specific pitfalls

**AI generates tests that cover getters and setters.** AI will write `test_user_get_name()` and `test_user_set_name()` but never test the actual business logic. When reviewing AI-generated tests, ask: "Does this test verify something the user cares about?" If the answer is no, the test is noise.

**AI generates snapshot tests without reviewing the diff.** AI will add snapshot tests to your codebase, and when the snapshot no longer matches (because the UI changed), it will auto-update the snapshot file. The test passes but nobody checked whether the UI change was correct. Always review snapshot diffs manually before approving changes.

**AI generates tests that pass but do not test the right thing.** This shows up as tests where the assertion is always true, the mock always returns the expected value regardless of input, or the test does not actually call the function being tested. Read each AI-generated test and trace through what it actually does.

**AI does not write tests for error paths.** AI tends to test happy paths and skip error conditions (invalid input, missing data, service timeouts). Ask AI explicitly: "Write tests for the error cases — what happens when the input is empty, when the database is unavailable, when the user is not authenticated."

**AI writes brittle tests tied to exact strings.** AI will write tests that assert on specific error message text: `assert "Username must be at least 3 characters" in response.text`. If a designer changes the message, the test breaks even though the behavior is correct. Test the behavior (validation rejects short usernames), not the exact phrasing.

## Quick Reference

**Testing pyramid:**

```
        /\
       /  \
      / E2E \       ← Few, slow, high value
     /--------\
    /Integration\  ← Some, medium speed
   /--------------\
  /    Unit        \ ← Many, fast, low level
 /------------------\
```

**What to test at each level:**

| Level | What to test | What to mock |
|---|---|---|
| Unit | Pure logic, calculations, transformations | Nothing — pure functions need no doubles |
| Integration | Database queries, API routes, service calls | External services (payment gateway, email) |
| E2E | Critical user flows, browser interactions | Nothing — test the whole system |

**Good test checklist:**
- Does this test verify user-visible behavior?
- Does it fail when the code is broken?
- Does it pass when the code is correct?
- Is it independent of other tests?
- Does it cover an edge case, not just the happy path?

**When to use test doubles:**

| Double | When to use | Example |
|---|---|---|
| Stub | Control input (provide fake responses) | Stub a pricing API to return $100 |
| Mock | Verify output (check that a call happened) | Mock an email service to verify it was called |
| Fake | Replace a complex real dependency | In-memory database for faster integration tests |

**TDD constraint workflow:**
1. Write the test first (describe the behavior you want)
2. Give AI the test + "implement the feature so this test passes"
3. Run the test — if it passes, the feature is done
4. If it fails, AI must fix the implementation, not the test

## Go deeper

- [Pytest Documentation](https://docs.pytest.org/) (verified 2026-04) — The standard Python testing framework. Best practices for fixtures, parametrization, and plugins.
- [Playwright: Writing Tests](https://playwright.dev/docs/writing-tests) (verified 2026-04) — Cross-browser E2E testing. The官方 docs cover best practices for stable, fast E2E tests.
- [Python Hitchhiker's Guide to Testing](https://docs.python-guide.org/writing/tests/) (verified 2026-04) — Practical testing patterns for Python: mocking, fixtures, test organization.
- [Property-Based Testing with Hypothesis](https://hypothesis.readthedocs.io/) (verified 2026-04) — Official docs for Hypothesis, the Python property-based testing library.
- [Martin Fowler: Test Pyramid](https://martinfowler.com/articles/practical-test-pyramid.html) (verified 2026-04) — The canonical explanation of the testing pyramid with concrete examples in Java.
