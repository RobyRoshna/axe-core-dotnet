!!! Actual tests and more explanations coming soon
# Accessibility Testing in .NET with axe-core and Playwright 

This is a practical guide to help set up automated accessibility testing in your .NET project. 

NOTE: Automated acscessibility tests do not identify all barriers on your app, it should just act as a starting point and not a replacement for targeted accessibility audits.

**Disclaimer:** This is a step-by-step setup guide, not a replacement for the official documentation. All accessibility scanning is powered by [axe-core](https://github.com/dequelabs/axe-core) and the official [`Deque.AxeCore.Playwright`](https://www.nuget.org/packages/Deque.AxeCore.Playwright) NuGet package maintained by [Deque Systems](https://www.deque.com/). Head there for the full API reference, changelog, and support.
---

## Table of Contents

- [What is axe-core and why should you care](#what-is-axe-core-and-why-should-you-care)
- [How it fits into your testing strategy](#how-it-fits-into-your-testing-strategy)
- [Prerequisites](#prerequisites)
- [Setup](#setup)
- [Writing your test](#writing-your-test)
- [Scoping your scans](#scoping-your-scans)
- [Adding to an existing test suite](#adding-to-an-existing-test-suite)
- [Organising tests with Categories](#organising-tests-with-categories)
- [Running in CI/CD](#running-in-cicd)
- [When tests fail](#when-tests-fail)
- [Useful links](#useful-links)

---

## What is axe-core and why should you care

[axe-core](https://github.com/dequelabs/axe-core) is an open source accessibility rules engine built by [Deque](https://www.deque.com/), one of the industry leaders in a11y training and audit tooling. You point it at a page, it scans the DOM, and it tells you what's broken and why — things like missing alt text, insufficient colour contrast, form inputs without labels, and so on.

It can't catch everything — some accessibility issues require human judgement. It's good at catching the mechanical stuff automatically, and it maps violations directly to [WCAG](https://www.w3.org/WAI/standards-guidelines/wcag/) success criteria so you know exactly what standard you're failing.

The .NET package — [Deque.AxeCore.Playwright](https://www.nuget.org/packages/Deque.AxeCore.Playwright) wraps axe-core so you can call it from your C# E2E tests. Playwright loads the page in a real browser, axe injects itself into the DOM, scans it, and hands back structured results you can assert on.

---

## How it fits into your testing strategy

It can be a part of:

- **E2E (End-to-End):** simulates a real user in a real browser going through your actual app
-  and later **Regression:** any test you re-run to make sure something that worked before still works

Axe slots into the same layer as your other E2E tests.

```
Your test suite
├── Unit tests           
├── Integration tests   
└── E2E tests            
      ├── functional assertions  → "the button submits the form"
      └── axe assertion          → "the page has no violations" 
```

---

## Prerequisites

- [.NET 8 SDK](https://dotnet.microsoft.com/download) or later
- A .NET web app to test (or just a URL: staging, prod, localhost)

---

## Setup

### 1. Create a test project

If you're adding to an existing solution:

```bash
dotnet new nunit -n MyApp.AccessibilityTests
dotnet sln add MyApp.AccessibilityTests
```

If you want a standalone project that can point at any app:

```bash
dotnet new nunit -n AccessibilityTests
cd AccessibilityTests
```

### 2. Add the packages

```bash
dotnet add package Microsoft.Playwright.NUnit
dotnet add package Deque.AxeCore.Playwright
```

That's it. `Deque.AxeCore.Playwright` pulls in `Deque.AxeCore.Commons` and `Microsoft.Playwright` automatically as dependencies so you don't need to add those separately.

Your `.csproj` should end up looking roughly like this:

```xml
<ItemGroup>
  <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.10.0" />
  <PackageReference Include="NUnit" Version="4.1.0" />
  <PackageReference Include="NUnit3TestAdapter" Version="4.5.0" />
  <PackageReference Include="Microsoft.Playwright.NUnit" Version="1.44.0" />
  <PackageReference Include="Deque.AxeCore.Playwright" Version="4.11.1" />
</ItemGroup>
```

### 3. Install Playwright browsers

Playwright needs to download the actual browser binaries. This is a one-time step (and a separate step in CI — more on that below).

```bash
dotnet build
pwsh bin/Debug/net8.0/playwright.ps1 install
```

On Mac or Linux:

```bash
./bin/Debug/net8.0/playwright.sh install
```

---

## Writing your test

`Microsoft.Playwright.NUnit` gives you a `PageTest` base class that handles all the browser lifecycle boilerplate — launching the browser, creating a context, opening a page, tearing it all down after. You inherit from it and just write tests.

```csharp
using Deque.AxeCore.Playwright;
using Microsoft.Playwright.NUnit;
using NUnit.Framework;

[TestFixture]
public class HomePageTests : PageTest
{
    [Test]
    public async Task HomePage_ShouldHaveNoAccessibilityViolations()
    {
        await Page.GotoAsync("https://your-app.com");

        var results = await Page.RunAxe();

        Assert.That(results.Violations, Is.Empty);
    }
}
```

`Page.RunAxe()` is the entire axe integration — one method call. It scans the full page and returns an `AxeResult` object containing violations, passes, incomplete checks, and inapplicable rules.

### Making failure messages useful

The default NUnit failure message when `Violations` isn't empty is not very helpful. Add a formatted message so you know what actually failed:

```csharp
var results = await Page.RunAxe();

var message = string.Join("\n", results.Violations.Select(v =>
    $"[{v.Impact?.ToUpper()}] {v.Id}: {v.Description}\n" +
    $"  Help: {v.HelpUrl}\n" +
    string.Join("\n", v.Nodes.Select(n => $"  Element: {n.Target?.FirstOrDefault()}"))
));

Assert.That(results.Violations, Is.Empty, message);
```

### Filtering by WCAG level

By default `RunAxe()` runs all rules. You can narrow it to a specific compliance target using `AxeRunOptions`:

```csharp
using Deque.AxeCore.Commons;

var options = new AxeRunOptions
{
    RunOnly = new RunOnlyOptions
    {
        Type = "tag",
        // Common options: wcag2a, wcag2aa, wcag21aa, wcag22aa, best-practice
        Values = new List<string> { "wcag2a", "wcag21aa" }
    }
};

var results = await Page.RunAxe(null, options);
```

WCAG 2.1 AA (`wcag21aa`) is the most common compliance target — it's what most accessibility laws and procurement requirements reference.

---

## Scoping your scans

By default axe scans the entire page. Sometimes that's more than you want — for example, you might want to exclude a third-party chat widget you don't control, or scan only a specific component.

### Scan only part of a page

```csharp
// Using a Playwright Locator — axe scans only inside that element
var results = await Page.Locator("#main-content").RunAxe();
var results = await Page.Locator("nav").RunAxe();
```

### Exclude elements

```csharp
var context = new AxeRunContext
{
    Exclude = new List<AxeSelector>
    {
        new AxeSelector("#third-party-widget"),
        new AxeSelector(".cookie-banner")
    }
};

var results = await Page.RunAxe(context);
```

### Scan dynamic states

This is clear advantage over axe's own browser extension. Static page scanners can only see the initial DOM — but your users interact with modals, validation errors, tooltips, and menus. Playwright puts the page in those states first, then axe scans them:

```csharp
[Test]
public async Task CheckoutForm_AllStates_AreAccessible()
{
    await Page.GotoAsync("/checkout");

    // Scan the initial state
    Assert.That((await Page.RunAxe()).Violations, Is.Empty, "Initial page load");

    // Open a modal and scan again
    await Page.GetByRole(AriaRole.Button, new() { Name = "Add coupon" }).ClickAsync();
    Assert.That((await Page.RunAxe()).Violations, Is.Empty, "With coupon modal open");

    // Trigger validation errors and scan again
    await Page.GetByRole(AriaRole.Button, new() { Name = "Pay now" }).ClickAsync();
    Assert.That((await Page.RunAxe()).Violations, Is.Empty, "With validation errors visible");
}
```

---

## Adding to an existing test suite

If you already have Playwright E2E tests, the simplest approach is to add an axe assertion to the end of tests that already navigate somewhere. No new infrastructure needed:

```csharp
[Test]
public async Task ProductPage_AddToCart()
{
    await Page.GotoAsync("/products/widget");
    await Page.GetByRole(AriaRole.Button, new() { Name = "Add to cart" }).ClickAsync();
    await Expect(Page.GetByText("Added to cart")).ToBeVisibleAsync();

    // Just add this — axe scans the post-interaction state for free
    var results = await Page.RunAxe();
    Assert.That(results.Violations, Is.Empty, FormatViolations(results));
}
```

### Starting on an app that already has violations

If you point axe at an existing app, it will almost certainly find violations immediately. Don't let that block you from committing — start in audit mode (log but don't fail) while you work through the backlog, then flip to hard assertions once you're clean:

```csharp
// Audit mode — commit this while you're fixing existing issues
var results = await Page.RunAxe();
if (results.Violations.Any())
    Console.WriteLine($"[AUDIT] {results.Violations.Length} violations found:\n{FormatViolations(results)}");

// Hard mode — uncomment this once the backlog is clear
// Assert.That(results.Violations, Is.Empty, FormatViolations(results));
```

---

## Organising tests with Categories

NUnit's `[Category]` attribute is just a label — it doesn't change how tests run, it just lets you filter them. You can put it on a whole fixture or individual tests, and stack multiple categories. This is exceptionally useful if you do not want to run your a11y tests everytime there is a minor change.

```csharp
[TestFixture]
[Category("Accessibility")]
[Category("E2E")]
public class AccessibilityTests : PageTest { }
```

```csharp
[TestFixture]
public class CheckoutTests : PageTest
{
    [Test]
    [Category("Functional")]
    [Category("Smoke")]
    public async Task Checkout_SubmitsOrder() { }

    [Test]
    [Category("Accessibility")]
    public async Task Checkout_IsAccessible() { }
}
```

Then filter on the CLI:

```bash
# Run everything
dotnet test

# Run only accessibility tests
dotnet test --filter "Category=Accessibility"

# Run smoke tests only (e.g. quick sanity check after a deploy)
dotnet test --filter "Category=Smoke"

# Combine categories
dotnet test --filter "Category=Accessibility|Category=Smoke"

# Exclude slow tests
dotnet test --filter "Category!=Slow"
```

A common pattern in CI is to run fast unit/integration tests on every commit, and gate E2E + accessibility tests on pull requests to `main` only — they take longer and need a running app.

---

## Running in CI/CD

The main thing to know: **Playwright browsers aren't included in the NuGet package** — you have to install them as a separate step in your pipeline.

### GitHub Actions

```yaml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.x'

      - name: Build
        run: dotnet build

      - name: Install Playwright browsers
        run: pwsh MyApp.AccessibilityTests/bin/Debug/net8.0/playwright.ps1 install --with-deps chromium

      - name: Start app
        run: dotnet run --project MyApp &
        # Give the app a moment to start
        # For a more robust solution look at health check polling

      - name: Run accessibility tests
        env:
          # If your tests read a base URL from an env var
          TEST_BASE_URL: http://localhost:5000
        run: dotnet test --filter "Category=Accessibility" --logger "trx;LogFileName=results.trx"

      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()  # upload even if tests fail
        with:
          name: test-results
          path: "**/*.trx"
```

### Pointing tests at different environments

Rather than hardcoding URLs, read the base URL from an environment variable:

```csharp
var baseUrl = Environment.GetEnvironmentVariable("TEST_BASE_URL") ?? "https://localhost:5001";
await Page.GotoAsync($"{baseUrl}/checkout");
```

Then in CI you set `TEST_BASE_URL` per environment — localhost for PR checks, your staging URL for nightly runs, etc.

---

## When tests fail

axe violations have four impact levels, roughly:

| Impact | What it means |
|--------|--------------|
| `critical` | Blocks users entirely — e.g. an image button with no accessible name |
| `serious` | Very hard to use — e.g. text with insufficient colour contrast |
| `moderate` | Confusing or frustrating — e.g. a form input with no label |
| `minor` | Best practice deviation — e.g. a redundant ARIA role |

Each violation links to a detailed explanation on [deque's rule reference](https://dequeuniversity.com/rules/axe/) — the `HelpUrl` in the result takes you straight there. These pages explain the rule, why it matters, and how to fix it. They're genuinely good documentation and worth reading when you hit a failure you don't recognise.

---

## Useful links

- [Deque.AxeCore.Playwright on NuGet](https://www.nuget.org/packages/Deque.AxeCore.Playwright) — the package
- [axe-core-nuget on GitHub](https://github.com/dequelabs/axe-core-nuget) — source, changelog, issue tracker
- [axe-core rules reference](https://dequeuniversity.com/rules/axe/) — what each rule checks and how to fix it
- [Microsoft.Playwright.NUnit docs](https://playwright.dev/dotnet/docs/test-runners) — PageTest base class and browser config
- [WCAG 2.1 Quick Reference](https://www.w3.org/WAI/WCAG21/quickref/) — the actual standard axe is checking against
- [axe-core on GitHub](https://github.com/dequelabs/axe-core) — the underlying JS engine, useful for understanding rule tags

---

*Contributions welcome. If something's out of date or unclear, open an issue.*
