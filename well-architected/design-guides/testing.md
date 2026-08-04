---
title: Build confidence in Azure workloads with effective testing practices
description: Learn how to implement testing practices for Azure workloads, including test planning, automation, and continuous validation.
author: simipaul
ms.author: simipaul
ms.reviewer: simipaul
ms.date: 07/27/2026
ms.topic: concept-article
ms.subservice: waf-design-guide
ms.update-cycle: 1095-days  
---

# How to build confidence in Azure workloads with effective testing

Testing is a continuous process that validates the changes you introduce to a workload. It catches regressions and maintains quality as the workload evolves.

This guide builds on [OE:09 Architecture strategies for testing](../operational-excellence/testing.md), which covers the high-level principles you should review first. It doesn't cover performance, reliability, and security testing in depth. For those topics, see the [Performance testing guide](../performance-efficiency/performance-test.md), [Reliability testing guide](../reliability/reliability-test.md), and [Security testing guide](../security/test.md).

Plan and design testing alongside the architecture, and evolve it as the architecture changes. Testing has four phases that overlap and iterate rather than run in strict sequence:

:::image type="content" source="images/testing/testing-phases.svg" alt-text="Diagram of four testing phases: Planning, Preparation, Execution, and Analysis with key activities listed under each.":::

- **Planning**: Decide *what* to validate and *why*. These decisions govern test cost and coverage for the lifetime of the workload. Capture them in a test strategy and a test plan.
- **Preparation**: Set up the conditions that simulate real-world scenarios. Provision environments, create test data, build test cases, mock dependencies, and set up automation frameworks.
- **Execution**: Integrate tests into the CI/CD pipeline, run them across multiple layers and quality dimensions, safely run some tests in production, and retest defects.
- **Analysis**: Analyze the results and report on quality. Track defects, measure coverage, evaluate quality metrics, and feed improvements back into development.

The examples throughout this guide follow a single e-commerce workload that adds Apple Pay as a checkout payment option.

## Terminology

| Term | Definition |
|---|---|
| **Acceptance criteria** | Specific conditions that a feature or user story must meet to be considered complete and acceptable to stakeholders. |
| **Synthetic data** | Artificially generated test data that represents real-world scenarios without using actual production data, reducing security and privacy risks. |
| **Ephemeral environment** | A temporary test environment created on demand for a specific purpose and destroyed after use to reduce costs. |
| **Mock service** | A simulated component that mimics the behavior of a real service or dependency, allowing isolated testing without relying on external systems. |
| **Contract testing** | A testing approach that verifies interactions between components based on a shared contract, ensuring they communicate correctly. |
| **Code coverage** | A metric indicating what percentage of code paths, branches, or statements run during test runs. |
| **Flaky test** | A test that inconsistently passes or fails without code changes, often due to timing issues, environmental dependencies, or poor test design. |
| **Test debt** | Accumulated maintenance burden from flaky tests, duplicate coverage, obsolete tests, or poor test design that undermines test suite effectiveness. |
| **Test pyramid** | A layered testing model that prioritizes fast unit tests at the base, integration tests in the middle, and slower end-to-end tests at the top. |
| **Regression tests** | Tests that validate existing functionality still works correctly after changes, preventing unintended side effects. |
| **Quality gate** | An automated checkpoint in a pipeline that a change must pass before it progresses to the next stage. |

## Create the test strategy

Planning starts with a strategy that sets the direction for everything that follows. 

A test strategy is a long-lived agreement on *what* you test and *why*, across multiple releases. Architects, engineers, and product owners agree on it early, before development begins, and revisit it as the workload evolves. Start by gathering business requirements, identifying critical user flows and risk areas, and deciding how your architecture supports validation over time.

A strategy typically covers these elements. The exact list varies with team structure and organizational practices.

- **Objectives and scope.** Testing goals, critical user flows, and the SLOs that drive acceptance criteria. Define which layers, components, and scenarios are in scope, and which are excluded.
- **Methods and techniques.** The test types you run, such as functional, security, performance, and user acceptance, and the balance of manual versus automated testing.
- **Roles and responsibilities.** Who owns each test type and how teams coordinate.
- **Environments and test data.** The environments you need, the tests that run in each, their parity with production, and the test data sources and residency requirements.
- **Risks and limitations.** Potential risks such as resource constraints, environment availability, or test data challenges.
- **Entry and exit criteria.** Conditions to meet before testing begins and conditions to consider it complete.
- **Tools and processes.** Tools for test management, execution, reporting, and defect tracking with severity and priority definitions.

## Develop the test plan

The test plan translates the strategy into an actionable document for a specific release or sprint. It targets the testing team and the contributors who run the tests. Create it after the strategy, when requirements are defined and testing tasks need detail.

The plan covers the same elements as the strategy but at release-level depth. It adds milestones, deliverables, timelines, and the sign-off process, so you can track progress and finish on time.

| Aspect | Test strategy | Test plan |
|---|---|---|
| Audience | Architects, product, security, QA leads | Testing team, release contributors |
| Scope | Workload lifetime, multiple releases | Single release or sprint |
| Lifespan | Long-lived, revisited periodically | Short-lived, per release |
| Owner | Test lead with stakeholder sign-off | Release test lead |
| Detail | Principles and approach | Cases, environments, schedule, sign-off |

### Example: test strategy and plan

The Apple Pay payment method introduces specific use cases, such as device setup, authentication, and transaction handling.

The **test strategy** doesn't change much for this release. It already mandates that payment flows are the highest testing priority, names the standard tools such as Playwright for UI tests and Azure Load Testing for load tests, and assigns the backend team to own payment integration tests and the platform team to own load tests.

The **test plan** for the Apple Pay release fills in the specifics:

| Plan element | Apple Pay release |
|---|---|
| Scope | Apple Pay checkout, refunds, and chargebacks on iOS web and native clients |
| Acceptance criteria | All payment-flow SLOs met; no open Sev-1 or Sev-2 defects |
| Resources | Two engineers; three iOS test devices |
| Environments | Pre-prod with Apple Pay sandbox; isolated test data with no personally identifiable information (PII) |
| Schedule | Four weeks, with a security review in week three |
| Entry criteria | Code complete and Apple Pay sandbox configured |
| Exit criteria | All planned tests pass; sign-off from payments and security leads |

Without a strategy and plan, each release takes an ad hoc approach that leads to inconsistent coverage, missed critical flows, and last-minute delays. Together they ensure consistent testing and a smooth rollout.

## Choose the right environment

Preparation is where you implement the strategy. Start by provisioning the right environments, which vary by test type. Match each environment to the testing you plan to run, along with its infrastructure, data, and security requirements.

- **Lower environments** (dev, integration): Use smaller-scale infrastructure and mock services. Run unit, integration, and regression tests here.
- **Pre-production environments**: Mirror production infrastructure and dependencies as closely as possible. Run performance, reliability, and security tests here.
- **Ephemeral environments**: Create them on demand for short-lived needs, such as validating a feature branch or running an isolated test suite. Tear them down to control costs.
- **Production environment**: In some cases, you run tests in production, such as validating a new feature with a small percentage of users or running stress tests during off-peak hours. Use guardrails to isolate the test and limit user exposure.

Automate environment provisioning for consistency and faster setup. Use Azure Resource Manager (ARM) templates, Bicep, or Terraform to define and deploy your test infrastructure. Configuration drift is a risk across multiple environments, so run a pipeline stage that compares the deployed configuration against the infrastructure as code (IaC) definition before running tests. For example, deploy infrastructure that mirrors production but substitutes the live payment gateway with the Apple Pay sandbox.

## Use mocks strategically

Your environments rarely replicate production exactly, so use mocks to stand in for real dependencies. Mocks replicate the services they replace, including expected responses, error conditions, and latency. Tools such as WireMock, Mountebank, or Azure API Management mock policies let you define and serve mock responses without writing custom code.

Good candidates for mocking are third-party APIs, non-deterministic services, and services that are slow, expensive, or unavailable in lower environments. Integration tests for the checkout service can use a mock payment gateway to simulate successful and failed transactions, while performance tests use the real gateway to capture actual latency and throughput. Never mock the component you're actually testing.

Design for testability so you can swap dependencies without code changes. For example, use dependency injection to replace the real Apple Pay client with a mock during testing.

> :::image type="icon" source="../_images/trade-off.svg"::: **Tradeoff:** Dependency injection can increase architectural complexity and make code harder to understand. Weigh the benefit of testability against code clarity to avoid long-term maintenance challenges.

**Use contract testing to keep mocks accurate.** A contract test verifies that the mock's request and response shapes match the real service's API. Run contract tests whenever the real service API changes. Without them, mocks silently diverge from real behavior, causing tests to pass in lower environments but fail in production.

## Generate realistic test data and manage it

Test data drives your tests, so it should reflect the diversity of real-world data, including edge cases, boundary conditions, and variations in user behavior. Consider a sign-in scenario and its positive, negative, and edge cases: a valid account, an invalid password, and a locked account.

Give each scenario its own unique data set. A shared data set is a common source of flaky tests. If the locked account and valid account tests run against the same credentials, one test can cause the other to fail intermittently. Dedicated credentials let tests run in parallel without interfering with each other.

Build variety into the data and parameterize it, so the same test logic validates multiple behaviors. For the sign-in scenario, parameterize the credentials so one test validates a valid account, an invalid password, a locked account, an expired password, and an unverified email. Append unique identifiers such as a timestamp or GUID to usernames and email addresses to avoid collisions.

When a scenario genuinely needs production data, anonymize it first. Mask all sensitive information, such as names, addresses, and payment details. Use data generation tools like Faker or Mockaroo to create realistic synthetic data sets.

Manage test data as it grows with your coverage:

- **Automate creation and deletion.** Run a prerequisite step that puts your test in the right state before execution, and a teardown stage that deletes data afterward. Keep test data only for the duration of your test runs.
- **Version and secure persistent data.** When data must persist, store it in your test repository under version control, separate from production data. Never hardcode credentials, API keys, or certificates in test scripts. Store them in a secure vault and retrieve them at run time.
- **Keep data current.** Update test data to reflect changes in the workload, user behavior, and business requirements.

## Design effective test cases

Drive your test scenarios from user flows and business requirements. Cover functional behavior, edge cases, and non-functional attributes. Rank what to test by the likelihood a defect occurs and the impact if it reaches production. Critical areas such as sign-in, payment, and checkout flows deserve far more coverage than low-risk informational pages.

Balance coverage across the layers of the test pyramid. Set coverage targets for each layer based on the critical functions, risk, and maintenance cost of the tests.

:::image type="content" source="images/testing/test-pyramid.svg" alt-text="Diagram of the test pyramid with base layer as unit, middle layer as integration, and top layer as end-to-end tests.":::

Use the following criteria to decide which layer a test case belongs in:

| Test type | Scope and dependencies | Choose this layer when | Typical coverage target | Example |
|---|---|---|---|---|
| **Unit tests** | A single function in isolation, with dependencies replaced by mock data and mock services. | The logic is self-contained and deterministic, such as calculations, validation rules, branching, and error handling. Prefer this layer for any case you can verify without crossing a component boundary. | High (for example, 80% of core business logic) | Test a shopping-cart total calculation with various item prices and quantities to verify the math is correct. |
| **Integration tests** | Two or more components and their real interactions, using a mix of real and mock services depending on the dependency and environment. | The behavior depends on how components exchange data, such as service-to-service calls, database access, or message handling. | Moderate (for example, 50% of component interactions) | Verify that the order service stores a completed payment transaction returned by the payment service. |
| **End-to-end tests** | A complete user journey across the full system, using real service integrations and realistic production-like test data. | The flow is business critical and only meaningful end to end, such as checkout or sign-in. Keep this layer small because the tests are slow and costly to maintain. | Low (for example, 10% of critical user journeys) | Simulate a user who browses products, adds items to the cart, checks out, and completes a purchase, then verify the UI, backend services, payment processing, and order confirmation all work together. |
| **Exploratory tests** | Manual, unscripted exploration of the application with no predefined cases. | The area is new, ambiguous, or hard to script, such as a complex UI or unpredictable user behavior, and you want to surface issues automated tests miss. | Not coverage-driven; run alongside scripted tests | Probe product search with various terms, filters, and sort options to uncover problems with results or experience. |

A single flow usually needs tests at more than one layer, with each layer covering a different part of it. The checkout flow might use unit tests for the cart calculation logic, integration tests for the payment service interaction, and end-to-end tests for the full user journey. Use your coverage targets to spread coverage across critical flows instead of over-investing in a single one.

Look beyond functionality. Include non-functional cases such as performance, security, and reliability, and draw each one's acceptance criteria from the relevant requirements. If the checkout flow has an SLO of 2 seconds for payment confirmation under peak load, the acceptance criteria is that payment confirmation returns within 2 seconds for 95% of requests during a simulated peak load of 5,000 concurrent shoppers.

Structure each case clearly with a starting condition (*given*), an action or event (*when*), and an expected outcome (*then*). For example: *Given* a user with a valid Apple Pay account and a credit card with insufficient funds, *when* the user attempts to complete a purchase, *then* the checkout fails with an appropriate error message and the order isn't processed.

Capture cases in Azure Test Plans, TestRail, or a similar tool so you can organize, track, and report on them. Link each case to its requirement or user story to keep traceability and coverage visible.

As your workload evolves, so should your test cases. Scenarios become outdated as user behavior, traffic patterns, and infrastructure change. Review your cases regularly, retire ones that no longer reflect the workload, and fold lessons from production incidents into new cases.

## Build your automation framework

Automated tests give you faster feedback and run more frequently than manual tests. They take upfront investment to design and maintain, but pay off through quicker releases and broader coverage over time.

> :::image type="icon" source="../_images/trade-off.svg"::: **Tradeoff:** A well-designed framework takes time to build. Weigh the automation investment against the risk of defects reaching production. Start small, balance automation with manual testing, and expand the framework as the workload grows.

Start by deciding what to automate. Favor test cases that are repeatable, critical, and stable. Leave exploratory work and fast-changing UIs to manual testing.

Based on your test strategy, choose tools that fit your workload and team. Consider workload compatibility, licensing, ease of use, community support, CI/CD integration, and the learning curve. Lean on established frameworks rather than building your own, such as Playwright or Selenium for UI tests and Postman or RestAssured for API tests.

Build the framework with maintainability, scalability, and security in mind so you can add tests without major refactoring:

- **Structure for scale.** Apply modular design, reusable components, and parameterization. Organize test configurations, cases, data, logs, and results. Split suites by test type, such as integration, load, and stress, so you can run targeted tests, compare results across runs, and maintain each suite independently. Avoid one monolithic suite, which is slow and makes root-cause analysis hard.
- **Version-control test assets.** Keep test data, configuration files, and scripts in GitHub or Azure DevOps. Enable pull request policies, build validation, and code review just like production code, so you catch defects in the tests themselves before they merge.
- **Add clear assertions.** Assertions validate that actual results match expected results. Use assertion libraries such as JUnit that provide clear, descriptive error messages to make failures easier to diagnose.
- **Build in observability.** Use your framework's reporting to capture structured logs and metrics. Record input parameters, expected and actual results, and exceptions. Don't log sensitive information such as authentication tokens, passwords, or error details that might leak information.
- **Design for isolation.** Design tests to run in any order with no shared state. Each test sets up and tears down its own preconditions, so tests run in parallel without interfering. Enforce sequential execution only when dependencies or business workflows require it, using framework features such as JUnit annotations to control order.
- **Secure the framework.** Automation frameworks often touch production data and systems, which introduces risk from imported libraries or vulnerable test code. Apply secure coding practices such as vulnerability scanning, input validation, and proper secret handling, and use tools like JFrog Artifactory to manage dependencies and artifacts securely.

## Execute tests in the pipeline

With an automation framework in place, run tests continuously so you get feedback on every code change and catch defects early. The base and middle layers of the test pyramid integrate most easily because they have minimal dependencies. You don't need a complete framework to begin. Start with a small set of tests, and migrate them into a unified framework as your capabilities mature.

Use pipeline stages to separate different test types, and define quality gates between stages so a change can't progress until it meets quality criteria. In the e-commerce application, you run checkout unit tests on every commit, progress to integration tests on pull requests only when the unit tests pass, and run regression tests when the pull request triggers the deployment pipeline.

:::image type="content" source="images/testing/ci-cd.svg" alt-text="An example CI/CD pipeline with testing integrated at different stages.":::

Set up nightly runs of your full suite in pre-production to catch flaky tests and regressions and to monitor workload behavior over time. These runs can include longer-running tests, such as load and performance tests, that aren't practical to run on every commit.

Run a mix of test types to get a complete picture. Combining tests reveals defects a single type would miss. A stress test might show the checkout service failing at 5,000 concurrent shoppers during a flash sale, while a 12-hour endurance test at normal load exposes a memory leak in the cart service that would crash the site overnight. Neither test alone catches both problems.

Keep feedback fast by managing execution time. Use parallel execution where possible. For example, in Gradle, set `maxParallelForks` to a value greater than 1. Apply a `fail-fast` mechanism for critical tests so the pipeline stops as soon as a critical test fails, without running the full regression suite.

Some tests must run in production. For example, run stress tests for the checkout service during off-peak hours to find the breaking point and confirm the system recovers gracefully, or validate a new feature with a small percentage of users. Implement guardrails that isolate the test and limit user exposure, such as feature flags for targeted rollout and automated stops when metrics like error rate or latency breach your SLOs.

### Retest defects

A failing test signals a defect, which can come from code changes, configuration drift, or environmental issues. Log every defect with a clear title, reproduction steps, the failing test or environment, and supporting logs, images, or videos. Classify severity (critical, high, medium, low), link each defect to user impact and business risk, assign an owner, and track it to closure.

Don't disable the tests that identify defects. Instead, add logic to handle the defect and continue testing. For example, add a conditional check to skip a test in *dev* but run it in *pre-prod*, so you can keep testing other areas while the defect is fixed.

After a fix, retest in the same environment where the defect was found and run regression tests to confirm the fix works and doesn't break other parts of the workload.

## Analyze the results and report on quality

Test results are only valuable if you analyze them and report on quality. After each run, review the results to answer:

- *What did the tests reveal about workload quality?*
- *Where are the gaps in coverage?*
- *What defects were found, and how critical are they?*
- *How can we improve testing effectiveness?*

Track results, execution time, failure trends, and historical comparisons to monitor suite health over time. Use the built-in reporting in your test framework and CI/CD platform, and notify the right people on failure so they can investigate quickly. Monitor nightly runs for new failures, regressions, and recurring patterns to catch flaky tests and unstable areas before they affect release quality.

### Defect tracking

Maintain a defect dashboard that shows open defects, severity, status, ownership, and aging, and use it to prioritize fixes. For example, fix a critical defect in the checkout flow before a low-severity cosmetic issue on the help page. In Azure DevOps, use work items to track defects, link them to test cases, and visualize their status in dashboards.

### Coverage analysis

Measure code coverage to identify untested paths, but treat coverage as a signal rather than a target. High coverage of low-risk code is less valuable than focused coverage of critical flows. Use tools such as SonarQube or JaCoCo to generate coverage reports and find gaps in payment, checkout, and other high-risk areas, then add cases where the risk justifies the maintenance cost. When a defect escapes to production, check whether a test should have caught it and add coverage where it should have.

### Quality metrics and reporting
Track a small set of metrics that reflect quality and suite health.

| Metric | What it tells you |
|---|---|
| Test pass rate | The share of tests that pass per run; a sustained drop signals regressions or instability. |
| Defect escape rate | The share of defects found in production rather than testing; a rising rate signals coverage gaps. |
| Flakiness rate | The share of tests that fail intermittently; high flakiness erodes trust in results. |
| Execution time trend | How long the suite takes over time; growth slows feedback and signals a need to optimize. |
| Code coverage | The share of code paths exercised by tests; low coverage in critical areas signals risk. |

Report metrics to the audience that needs them, and tailor each dashboard so it answers that audience's questions. For the checkout flow, the same metric set serves different audiences in different ways:

| Audience | Metrics they track | What it tells them |
|---|---|---|
| Developers | Flakiness rate, code coverage | Which payment tests are unreliable and which checkout paths are still untested. |
| Operations | Test pass rate, execution time trend | Whether the checkout release is ready and whether feedback is slowing down. |
| Business stakeholders | Defect escape rate | Whether checkout quality is trending up or down across releases. |

Azure DevOps provides built-in dashboards and widgets for test results, code coverage, and work items that you can customize per audience.

### Feedback loops and continuous improvement

Create a test report at the end of each release as a deliverable in test plan sign-off. Include release details, test run results, defect summaries, and coverage information to inform decisions about release readiness, acceptable risks, and future priorities.

Use these insights to improve testing effectiveness. Review tests on a regular cadence, look for patterns in defect types and coverage gaps, and refine the strategy and cases accordingly. Because flaky tests, duplicate coverage, and obsolete tests erode confidence and slow delivery, schedule regular maintenance sprints to reduce test debt. With each phase, you gain valuable insights and refine your approach, building confidence in every release.

## How to avoid testing antipatterns

The following antipatterns commonly undermine what testing can do for workloads.

| Antipattern | Guidance |
|---|---|
|**No formal test strategy or plan**<br><br>Testing is ad-hoc, unplanned, and disconnected from business objectives. Teams lack clarity on what to test and why.|**Formalize your test approach:**<br>- Define a test strategy aligned with business objectives.<br>- Create test plans with clear scope, resources, and timelines.<br>- Establish entry and exit criteria for testing phases.<br>- Document roles and responsibilities.|
|**Testing too late in the delivery cycle**<br><br>Testing is deferred until late stages, leading to missed defects, increased rework, and delayed releases.|**Start early:**<br>- Start testing during design and early development.<br>- Integrate tests into CI/CD pipelines for rapid feedback.<br>- Use unit and integration tests to catch defects early.<br>- Treat testing as continuous, not a phase.|
|**Inadequate test coverage**<br><br>Critical paths, edge cases, and integration points are untested. Coverage is skewed toward easy-to-test components rather than high-risk areas.|**Prioritize coverage strategically:**<br>- Use the test pyramid to balance unit, integration, and end-to-end tests.<br>- Focus on business-critical flows and high-risk scenarios.<br>- Measure and track coverage gaps.<br>- Add regression tests for production defects.|
|**Ignoring non-functional testing**<br><br>Testing focuses only on functional correctness, neglecting performance, security, reliability, and operational quality.|**Adopt multi-dimensional testing:**<br>- Include performance, load, and stress testing.<br>- Integrate security testing throughout the lifecycle.<br>- Validate resilience with chaos engineering.|
|**Flaky and unreliable tests**<br><br>Tests fail inconsistently without code changes, eroding trust and slowing delivery. Teams ignore failures or disable tests rather than fixing them.|**Maintain test reliability:**<br>- Identify and fix or remove flaky tests promptly.<br>- Design tests for independence and isolation.<br>- Use stable, deterministic test data.<br>- Monitor test suite health and address degradation.|
|**Including all possible tests in the build pipeline**<br><br>Including every possible test in the build pipeline can slow down release cycles and increase the risk of important tests being bypassed.|**Focus on critical tests:**<br>- Prioritize tests that protect critical workflows.<br>- Avoid overloading the pipeline with low-value tests.|
|**Test environments don't reflect production**<br><br>Tests pass in lower environments but fail in production due to configuration drift, missing dependencies, or infrastructure differences.|**Mirror production conditions:**<br>- Design test environments to closely resemble production.<br>- Automate environment provisioning with IaC.<br>- Validate configuration consistency across environments.<br>- Use mock services where full replication isn't feasible.|
|**Poor test data management**<br><br>Test data is inconsistent, stale, or contains sensitive information. Data setup is manual and error-prone.|**Manage test data deliberately:**<br>- Use synthetic data by default to reduce risk.<br>- Anonymize production data when necessary.<br>- Automate data setup and teardown.<br>- Version control test data alongside code.|
|**Neglecting test maintenance**<br><br>Test suites accumulate debt through obsolete tests, duplicate coverage, and poor design. Maintenance is reactive rather than planned.|**Treat tests as production assets:**<br>- Schedule regular test maintenance sprints.<br>- Apply code review and architectural principles to tests.<br>- Remove obsolete and duplicate tests.<br>- Refactor tests for clarity and reliability.|
|**No test observability**<br><br>Teams lack visibility into test execution, failures, coverage, and trends. Debugging test failures is time-consuming and uncertain.|**Extend observability to testing:**<br>- Implement structured logging in test code.<br>- Track execution time, failure rates, and flakiness.<br>- Generate coverage and quality reports.<br>- Use dashboards to visualize test suite health.|

## Azure facilitation

**Test management and planning:**

- [Azure Test Plans](/azure/devops/test/overview) provides browser-based test management for manual testing, user acceptance testing, exploratory testing, and stakeholder feedback. It includes [Test Analytics](/azure/devops/pipelines/test/test-analytics) to track test quality over time.

**Test automation and CI/CD integration:**

- [Azure Pipelines](/azure/devops/pipelines/get-started/what-is-azure-pipelines) enables test automation integrated into CI/CD workflows, with support for parallel execution, pipeline stages, and quality gates.

- [GitHub Actions](https://azure.github.io/actions/) provides similar CI/CD capabilities integrated with GitHub repositories and Azure services.

**Functional and performance testing:**

- [Azure App Testing](/azure/app-testing/overview-what-is-azure-app-testing) supports functional and performance testing with [Playwright Workspaces](/azure/app-testing/playwright-workspaces/overview-what-is-microsoft-playwright-workspaces) for end-to-end tests and [Azure Load Testing](/azure/app-testing/load-testing/overview-what-is-azure-load-testing) for performance validation.

**Reliability testing:**

- [Azure Chaos Studio](/azure/chaos-studio/chaos-studio-overview) is a managed service that uses chaos testing to help you measure, understand, and improve your cloud application and service resilience. 

## Related links

- [Recommendations for testing (OE:09)](../operational-excellence/testing.md)
- [Recommendations for performance testing](../performance-efficiency/performance-test.md)
- [Recommendations for reliability testing](../reliability/reliability-test.md)
- [Recommendations for security testing](../security/test.md)