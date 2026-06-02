# Release Notes — Unlocked Package (2GP) Conversion

This release modernizes a long-dormant fork of the **Schedul-o-matic 9000** and repackages it
as a **namespace-less unlocked package** (Salesforce second-generation packaging, "2GP").
Everything builds, deploys, and tests green.

- **Branch:** `unlocked-package-2gp`
- **Package:** `Schedul-o-matic 9000 Unlocked` (Unlocked, namespace-less)
- **Package Id:** `0Hod20000003ft3CAA`
- **First version built:** `Schedul-o-matic 9000 Unlocked@0.1.0-1` → `04td2000000Q2ZFAA0` (beta)
- **Source API version:** `60.0`
- **Tests:** Apex passing at version-create time · LWC Jest **12/12 passing**

> Full technical documentation lives under [doc/](doc/) — start at [doc/README.md](doc/README.md).
> The end-to-end packaging guide is [doc/unlocked-package.md](doc/unlocked-package.md).

---

## Why this work was needed

The upstream project had been untouched for ~6 years and shipped under the **`dcstuff`**
managed-package namespace. As a fork:

- The `dcstuff` namespace is registered to an org we do **not** control. A 2GP namespace is
  globally unique and permanently bound to its registry org — it **cannot be reused or
  re-registered**. So the package had to be rebuilt **without** that namespace.
- Several metadata definitions (API versions, a flexipage schema) and one test had aged out of
  what current Salesforce API versions accept, and only surfaced once the package was actually
  built in a modern, non-English Dev Hub.

---

## What changed (by commit)

The branch is organized into focused, reviewable commits:

| Commit | Summary |
| --- | --- |
| `chore` | Modernize metadata to **API 60.0** across Apex/LWC/Visualforce meta; fix the `SchedulomaticEntry__c` plural-label typo (`Entriess` → `Entries`). |
| `refactor` | **Convert to namespace-less** unlocked package and **prefix every Apex class with `SOM_`**. |
| `fix` | Update the record-page **flexipage** to the current `itemInstances` schema. |
| `build` | Rewrite **`sfdx-project.json`** for the unlocked 2GP package. |
| `docs` | Add the **`doc/`** technical documentation set. |
| `build` | Bump **dev dependencies** (`npm audit fix --force`) and **modernize the Jest tests** for Jest 29. |

### 1. De-namespacing (`dcstuff` → none)

Installed components in a namespace-less package have `NamespacePrefix = null`. The three former
namespace dependencies were reworked accordingly:

- `SOM_Scheduler.hasPermission` matches the permission set by **`Name` only**.
- `SOM_Scheduler.getClasses` excludes only the local class: `AND (NamespacePrefix != null OR Name != 'SOM_Scheduler')`.
- `scheduler.js` navigates to `SchedulomaticEntry__c` / list view `All` with **no prefix**.

### 2. `SOM_` class prefix (the namespace substitute)

Without a namespace, all Apex classes share the subscriber org's global namespace, so generic
names would collide on install. Every class was prefixed:

| Before | After |
| --- | --- |
| `Scheduler` | `SOM_Scheduler` |
| `Dao` | `SOM_Dao` |
| `TestUtils` | `SOM_TestUtils` |
| `MockHttpResponse` | `SOM_MockHttpResponse` |
| `SchedulableContextInterface9000` | `SOM_SchedulableContext` |
| `*_Test` | `SOM_*_Test` |

All references were updated: `THIS_CLASS`, LWC `@salesforce/apex/...` imports (+ `jest.mock`
paths), the permission-set `classAccess`, and cross-class calls. The Visualforce page
`SchedulerHelper`, the LWC component folders, and the platform `Schedulable` /
`SchedulableContext` interfaces were intentionally left unchanged.

### 3. `sfdx-project.json` for an unlocked package

- Single package directory set as **`"default": true"`** (required for one-dir projects).
- Removed **`ancestorId`** — ancestry is supported only by *managed* 2GP, not unlocked.
- Raised **`sourceApiVersion` to 60.0**; switched to a **`.NEXT`** build counter.
- Historical managed-package aliases retained for lineage.

---

## Build issues encountered & resolved

These only appeared at `sf package version create` time and were each fixed at the source:

| Symptom | Root cause | Fix |
| --- | --- | --- |
| `SingleNonDefaultPackageError` | `sf package create` wrote `"default": false` | Set the package directory back to `"default": true`. |
| `Property 'componentInstances' not valid in version 60.0` (+ failed `View` override) | Flexipage used the **legacy `componentInstances`** schema | Convert to `itemInstances` → `componentInstance` with `identifier`s. |
| 9× `System.QueryException: List has no rows…` in tests | `SOM_TestUtils` queried the **localized** profile name `'Standard User'`; the build org is non-English | Reuse the running user's profile via `UserInfo.getProfileId()` (locale-independent; access is gated only by the permission set). |
| LWC Jest failures after `npm audit fix --force` | Major bumps to **Jest 24 → 29** removed the `setImmediate` global and changed fake-timer behavior | Replace `setImmediate` with `setTimeout`; use `runAllTimersAsync()` with behavioral assertions on `getClasses`. |

---

## How to build, install & promote

```bash
# Build a new version (auto-increments the build number via .NEXT)
sf package version create -p "Schedul-o-matic 9000 Unlocked" \
  --installation-key-bypass --code-coverage --wait 30 --target-dev-hub DevHub

# Install the beta into a scratch/sandbox/dev org, then grant access
sf package install -p "Schedul-o-matic 9000 Unlocked@0.1.0-1" --wait 20 --target-org <org>
sf org assign permset -n Schedulomatic9000User --target-org <org>

# Promote to make it installable in production (requires >=75% Apex coverage)
sf package version promote -p "Schedul-o-matic 9000 Unlocked@0.1.0-1" --target-dev-hub DevHub
```

> The current `0.1.0-1` is a **beta** — installable in scratch/sandbox/dev orgs. Promote it for
> production installs. See [doc/unlocked-package.md](doc/unlocked-package.md) for the full guide,
> including the namespace rationale and how to ship under your own namespace instead.

---

## Verification

- ✅ `sf package version create` — **Success** (version `0.1.0-1`).
- ✅ Apex tests — pass during version create (locale bug fixed).
- ✅ LWC Jest — `npm run test:unit` → **12 passed, 3 suites**.

## Known follow-ups (not in this release)

- **ESLint config migration.** The bump to ESLint 10 / `eslint-config-lwc` 4 likely needs a flat
  config (`eslint.config.js`) to replace the legacy `.eslintrc`; `npm run lint` was not
  re-validated. Best done as its own change.
- **Promotion** to a released version once coverage is confirmed ≥ 75%.

---

*Generated alongside the conversion work. Security note: the dev-dependency audit fixes affect
local tooling only — they do not ship inside the package or change its runtime behavior.*
