---
title: What a green browser test does not prove
date: 2026-09-19 09:00:00 +0100
categories: [Testing]
tags: [Playwright, browser testing, quality]
description: A passing browser test proves that one scripted path satisfied its assertions in one environment. It does not prove that the product is correct everywhere.
image:
  path: /assets/img/posts/what-a-green-browser-test-does-not-prove.webp
  alt: A green browser check sits beside several untested user paths.
---

A green Playwright run is useful evidence. It is not a certificate that the feature works.

The test exercised a particular build, browser, viewport, account state, network condition, and sequence of actions. It passed the assertions that were written. Everything outside that boundary remains an untested assumption.

## Assert what the user can observe

Browser tests are strongest when they check user-visible behavior. The test should submit a form and verify the visible confirmation, not only that a request returned 200. It should verify the heading, status, accessible name, or rendered value that tells a user the operation completed.

Playwright's [web-first assertions](https://playwright.dev/docs/test-assertions) retry while the page changes. A check such as `await expect(page.getByTestId('status')).toHaveText('Submitted')` expresses an observable result and waits for it. That is better than sleeping for a fixed duration and inspecting a transient implementation detail.

User-visible assertions do not make the test complete. Add the important negative behavior too: an unauthorized user cannot see the action, a failed request shows a useful error, and a retry does not create a second visible record. The right assertions depend on the product's contract, not on what is easiest to select in the DOM.

## Selectors are an interface

A selector is a dependency. A CSS class used only for layout is a poor dependency because a harmless redesign can break the test. Playwright's [locator guidance](https://playwright.dev/docs/locators) recommends user-facing attributes and explicit test IDs where appropriate. Roles and accessible names often align with how a person or assistive technology identifies the control.

This does not mean every test should use text. Text can change with copy edits or localization. It means the selector should express why the element is the target. A role plus accessible name can express a button's contract. A test ID can express a stable testing hook. A deep CSS path usually expresses nothing except the current markup tree.

Selector stability is not product correctness. A test can be perfectly stable while selecting the wrong element. Keep at least one assertion about the page state that would be wrong if the user-visible behavior were wrong.

## Screenshots are evidence, not proof

Screenshots help diagnose layout regressions and can catch changes that a few assertions miss. Playwright supports screenshot comparisons through `toHaveScreenshot`, as described in its [visual comparison guide](https://playwright.dev/docs/test-snapshots). A reference image is still a comparison under a particular browser, operating system, font set, viewport, and rendering configuration.

A matching screenshot does not prove that keyboard navigation works, that the text is accessible to a screen reader, or that a hidden overflow failure does not appear at another width. A different screenshot does not always mean a product regression; font rendering, animation, time, and nondeterministic content can change pixels.

Use screenshots for visual contracts, keep dynamic regions controlled, and review baseline changes as code. Pair them with semantic assertions and accessibility checks. Do not turn every page into a pixel snapshot. That creates noisy maintenance without improving coverage.

## Know the environment boundary

A browser test may stub APIs, use a seeded database, run with an admin account, or bypass a real payment and email provider. Those choices make the test repeatable. They also narrow what it proves. If the application depends on a proxy, a third-party callback, a real queue, or a different cookie policy, a mocked browser test cannot validate that integration.

Run tests across the environments that matter: representative browsers, mobile and desktop viewports, localization settings, permission levels, and meaningful data states. Keep the matrix purposeful. Ten copies of the same happy path do not cover a failed network response or a stale session.

A green test also says nothing about paths it never reached. If a user-visible flow depends on an asynchronous job, assert the job's observable result or test the job separately. If a commit can be replayed, cover the duplicate case. If a model chooses a tool, record and validate the tool boundary rather than assuming the browser result proves the choice was authorized.

The right conclusion from a green browser test is precise: this build satisfied these assertions for this path in this environment. That is enough to trust the tested contract. It is not a reason to stop looking for the contracts the test did not name.
