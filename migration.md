# .NET 8 to .NET 10 Migration Plan

## Objective

Migrate the .NET 8 Web API to .NET 10 within **20 working days**, using a three-person team, while preserving existing behavior, protecting the ten external integrations, and avoiding any architectural refactoring.

.NET 8 reaches end of support on **November 10, 2026**; .NET 10 is a Long Term Support release supported until **November 14, 2028**. [dotnet.microsoft](https://dotnet.microsoft.com/en-us/platform/support/policy)

## Scope

### In scope

- Upgrade all projects from `net8.0` to `net10.0`
- Upgrade NuGet packages and resolve compatibility issues
- Validate ASP.NET Core, EF Core, authentication, serialization, logging, and hosting behavior
- Validate all ten external integrations
- Improve the safety of the appsettings-based mock configuration
- Execute regression, contract, performance, and deployment validation
- Prepare a rollback plan

### Out of scope

- Hexagonal Architecture migration
- DDD refactoring
- Rewriting anemic domain classes
- Moving business rules out of API services
- New features
- Unrelated bug fixes
- Database schema redesign

## Initial Assessment

Before committing to the 20-day schedule, complete a technical inventory during the first 1–3 days.

| Measurement | Why it matters | Red flag |
|---|---|---|
| Number of `.csproj` files | Determines upgrade surface area | Many projects or shared libraries |
| Lines of code and service count | Indicates complexity and test scope | Very large service classes |
| NuGet dependencies | Major source of migration risk | Unsupported or abandoned packages |
| EF Core version | Must be compatible with .NET 10 | EF Core 6/7 or custom database providers |
| External integrations | Highest-risk area due to ten connected systems | Native SDKs without .NET 10 support |
| Test coverage | Determines confidence in detecting regressions | Little or no integration testing |
| Deployment model | Affects runtime, images, CI/CD, and rollback | Custom Docker images or legacy hosting |
| Mock configuration | May hide real integration failures | Mocks controlled by scattered appsettings flags |

Use the .NET Upgrade Assistant to analyze and upgrade projects, but do not rely on it to validate external integrations. 

## Effort Estimate

| Workstream | Estimated person-days | Notes |
|---|---:|---|
| Inventory, branch, CI baseline | 2–3 | Establish a known-good .NET 8 build and deployment |
| Project and package upgrades | 5–8 | `net10.0`, NuGet updates, compiler errors and warnings |
| Runtime and API breaking changes | 3–6 | ASP.NET Core, DI, hosting, serialization, logging |
| EF Core and database validation | 3–6 | Provider compatibility, migrations, query behavior |
| Integration validation | 6–10 | Highest-risk area: ten external applications |
| Contract and regression testing | 5–8 | Must cover real integration contracts, not only mocks |
| Performance, security, observability | 2–4 | Startup, latency, memory, authentication, logging |
| Staging deployment and UAT | 3–5 | Can partially overlap with other work |
| Production rollout and contingency | 2–3 | Phased or blue/green deployment preferred |
| **Total** | **31–53** | **Approximately 10–18 elapsed days with three developers** |

### Estimate interpretation

- **Feasible:** 35–50 person-days, assuming no unsupported critical dependencies and at least basic integration tests.
- **At risk:** 50–60 person-days, if several packages or SDKs require replacement.
- **Not feasible in 20 days:** 60+ person-days, especially if there is weak test coverage, unsupported third-party SDKs, or no reliable rollback process.

## Team Allocation

| Role | Primary responsibilities |
|---|---|
| Migration lead | Branch strategy, package compatibility, build/runtime fixes, risk register, final merge |
| Integration developer | Integration inventory, contract tests, adapter validation, mock configuration |
| Quality and deployment developer | Test suite, CI/CD, environments, observability, rollout and rollback |

Avoid having all developers modify the same projects simultaneously. Partition work by project and integration boundary, with one person owning the release branch.

## 20-Day Execution Plan

### Days 1–3: Baseline and analysis

- Create a protected upgrade branch.
- Confirm the current .NET 8 build, deployment process, and rollback procedure.
- Run .NET Upgrade Assistant analysis.
- Audit NuGet packages for .NET 10 compatibility.
- Inventory all ten external integrations.
- Document every mock switch in `appsettings.json`.
- Identify packages with no compatible .NET 10 version.

**Exit criteria:** known-good .NET 8 baseline, complete dependency and integration inventory, initial risk register.

### Days 4–8: Mechanical upgrade

- Change `TargetFramework` to `net10.0` in all projects.
- Upgrade Microsoft, EF Core, authentication, serialization, and observability packages.
- Resolve compile errors, obsolete APIs, and runtime warnings.
- Run the application locally using the existing mock configuration.
- Do not refactor business logic or architecture.

**Exit criteria:** solution builds and runs on .NET 10 locally; existing tests pass or known gaps are documented.

### Days 9–14: Integration hardening

- Validate every external integration in a dedicated environment.
- Replace broad mocks with recorded contracts where possible.
- Test authentication, retries, timeouts, serialization, error responses, and idempotency.
- Run performance smoke tests.
- Compare startup time, latency, memory usage, and database query behavior against the .NET 8 baseline.

**Exit criteria:** all integrations validated or explicitly accepted with a documented risk.

### Days 15–17: Staging certification

- Deploy to staging using production-like configuration.
- Execute end-to-end business scenarios.
- Validate all ten integrations.
- Obtain sign-off from dependent application owners where possible.
- Freeze the codebase except for critical migration defects.

**Exit criteria:** staging certification complete; release candidate approved.

### Days 18–20: Release and contingency

- Deploy using a phased, blue/green, or canary strategy where infrastructure allows.
- Monitor error rates, latency, dependency failures, and business KPIs.
- Retain the .NET 8 deployment artifact and rollback plan.
- Do not bundle unrelated bug fixes, refactoring, or feature work into the release.

**Exit criteria:** production deployment stable and rollback plan verified.

## Mock Configuration Hardening

The current appsettings-based mock system should be made explicit and safe without redesigning the application.

Recommended actions:

- Replace scattered configuration reads with a strongly typed options class, for example `IntegrationOptions`.
- Select real or mock implementations through dependency injection.
- Validate configuration at startup.
- Prevent mock usage in production unless explicitly approved.
- Log a startup warning whenever any mock is enabled.
- Identify exactly which integration each mock replaces.
- Add tests that verify both real and mock paths where practical.

This creates a clear integration seam without changing the system’s overall architecture.

## Risk Register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Third-party SDK lacks .NET 10 support | Medium | High | Identify early; isolate behind an adapter; retain .NET 8 sidecar only if unavoidable |
| EF Core provider or migration behavior changes | Medium | High | Test migrations and representative queries against a production-like database |
| JSON serialization changes affect consumers | Medium | High | Add contract tests and use staged rollout |
| Mocks hide real integration defects | High | High | Validate against real or recorded contracts before release |
| No tested rollback path | Medium | Critical | Retain prior deployment artifact and verify rollback procedure |
| Scope creep into refactoring | High | High | Enforce a strict no-refactoring rule during migration |
| Performance regression | Medium | Medium | Capture .NET 8 baseline metrics and compare after upgrade |
| Integration owner unavailable for UAT | Medium | High | Schedule validation early and use contract tests as fallback |

## Go / No-Go Criteria

Proceed with the 20-day migration only if the following are true by day 3:

- All critical dependencies support .NET 10 or have a viable replacement.
- The current .NET 8 application can be built, deployed, and rolled back reliably.
- Every external integration has at least a smoke-level test or recorded contract.
- The team has access to staging environments and integration credentials.
- Product and stakeholders agree that no architectural refactoring will occur during the migration.
- A rollback strategy exists and has been reviewed.

If these conditions are not met, use a two-phase approach:

1. **Phase 1:** Stabilize tests, integration contracts, mock seams, and deployment reliability on .NET 8.
2. **Phase 2:** Perform the .NET 10 upgrade once the safety net exists.

## Recommended Communication

> “This is not another large refactoring initiative. The goal is to upgrade the runtime safely while preserving current behavior and protecting all external integrations. We will first establish a reliable baseline, validate every integration boundary, and maintain a tested rollback plan. Any future architectural improvements will be incremental and separate from this migration.”

## Success Criteria

The migration is successful when:

- The application runs on .NET 10 in production.
- All ten integrations pass their defined validation scenarios.
- Existing API contracts remain compatible.
- Performance is within agreed tolerances compared with .NET 8.
- The rollback plan has been tested or clearly validated.
- No unrelated architectural or business-logic changes were introduced.
- The system remains supportable after .NET 8 end of support. [dotnet.microsoft](https://dotnet.microsoft.com/en-us/platform/support/policy)
