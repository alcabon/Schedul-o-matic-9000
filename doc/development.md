# Development Guide

## Prerequisites

- [Salesforce CLI](https://developer.salesforce.com/tools/salesforcecli) (`sf`)
- Node.js (for the LWC Jest tests and linting)
- An authorized **Dev Hub** (see [unlocked-package.md](unlocked-package.md))

Install Node dependencies:

```bash
npm install
```

## Scratch org

The scratch org shape is defined in [`config/project-scratch-def.json`](../config/project-scratch-def.json)
(Developer edition with `DebugApex`).

```bash
# Create and open a scratch org
sf org create scratch --definition-file config/project-scratch-def.json --alias scratch --set-default --duration-days 7
sf org open --target-org scratch

# Push source and assign the permission set
sf project deploy start --target-org scratch
sf org assign permset --name Schedulomatic9000User --target-org scratch
```

> This fork is **namespace-less**, so a plain scratch org works — no namespace registry link is
> needed. See [unlocked-package.md](unlocked-package.md#a-note-on-the-namespace) for why the
> original `dcstuff` namespace can't be reused.

## Tests

### Apex

```bash
sf apex run test --target-org scratch --code-coverage --result-format human --wait 10
```

The Apex tests are designed to run without a live external endpoint: callouts are stubbed via
`SOM_MockHttpResponse`, and reflection-heavy branches short-circuit on `Test.isRunningTest()`.

### LWC (Jest)

```bash
npm run test:unit            # one-off run
npm run test:unit:watch      # watch mode
npm run test:unit:coverage   # with coverage
```

Jest config lives in [`jest.config.js`](../jest.config.js); Lightning stubs are under
`force-app/test/jest-mocks`.

## Linting & formatting

```bash
npm run lint                 # ESLint over the LWC
npm run prettier             # format all source
npm run prettier:verify      # check formatting without writing
```

> The toolchain pinned in `package.json` is several years old. If you modernize it, the most
> impactful upgrades are `eslint`, `@salesforce/eslint-config-lwc`, `@salesforce/sfdx-lwc-jest`,
> and `prettier` (v3 changes some formatting defaults). Treat that as a separate, tested change
> so any formatting churn lands in its own commit.

## Source layout

```
force-app/main/default/
├── applications/        Schedulomatic9000 app
├── classes/             Apex (SOM_Scheduler, SOM_Dao, helpers, *_Test)
├── contentassets/       brand/logo assets
├── documents/icons/     tab/app icons
├── flexipages/          record & home pages
├── labels/              CustomLabels (translatable strings)
├── layouts/             page layouts
├── lwc/                 scheduler, classLookup, classLookupOption, schedulerUtils
├── objects/             SchedulomaticEntry__c (fields, list views, validation rules)
├── pages/               SchedulerHelper Visualforce page
├── permissionsets/      Schedulomatic9000User
├── staticresources/     scheduler icon, missing-permissions image
└── tabs/                app + object tabs
```

## Housekeeping notes (applied in this pass)

- **API versions unified to `60.0`** across all Apex `*-meta.xml`, LWC `*.js-meta.xml`, and the
  Visualforce page (one LWC was lingering at `47.0`; the rest were `48.0`). `sourceApiVersion`
  in `sfdx-project.json` was bumped to match.
- **Typo fix:** the object plural label `Schedul-o-matic Entriess` → `Schedul-o-matic Entries`.
- **De-namespaced** the fork (removed the `dcstuff` namespace, which can't be reused — see
  [unlocked-package.md](unlocked-package.md#a-note-on-the-namespace)). Updated `SOM_Scheduler`
  (permission check + class filter), `scheduler.js` (object/list-view navigation), `SOM_TestUtils`,
  and the affected Apex/LWC tests to match.
- **`sfdx-project.json`** rewritten for a namespace-less unlocked package — see
  [unlocked-package.md](unlocked-package.md) for the full rationale.

### Suggested follow-ups (not yet applied)

- Modernize the npm devDependencies (ESLint/Prettier/Jest) as a standalone change.
- Consider migrating the session-id-via-Visualforce callout pattern; on current API versions
  the same self-callouts can often be simplified, though the Visualforce approach remains valid.
