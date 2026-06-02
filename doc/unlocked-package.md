# Building the Unlocked Package (2GP)

This guide explains how the project's [`sfdx-project.json`](../sfdx-project.json) is configured
for an **unlocked package** (Salesforce *second-generation packaging*, "2GP") and how to
create, version, install, and promote it.

> **Tooling:** examples use the modern `sf` CLI (Salesforce CLI v2). The legacy `sfdx force:…`
> equivalents still work but are deprecated.

---

## 1. What changed from the previous configuration

The project historically shipped as a **managed** package. The `sfdx-project.json` has been
rewritten for an **unlocked** package. The meaningful differences:

| Setting | Before | After | Why |
| --- | --- | --- | --- |
| `package` | `Schedul-o-matic 9000` | `Schedul-o-matic 9000 Unlocked` | A package's *type* is fixed at creation; an unlocked package must be a **new** package, not the existing managed one. A distinct name avoids the alias collision. |
| `ancestorId` | `04t1Q0000016YgjQAE` | **removed** | Ancestry (`ancestorId` / `ancestorVersion`) is supported **only** by *managed* 2GP. Leaving it in an unlocked package definition is invalid. |
| `sourceApiVersion` | `48.0` | `60.0` | 48.0 (Spring '20) is long out of support; 60.0 matches this org's metadata cache. |
| `versionName` / `versionDescription` | — | added | Required/recommended metadata for each package version. |
| `versionNumber` | `0.10.0.0` | `0.11.0.NEXT` | `.NEXT` lets the CLI auto-increment the build number on every `package version create`. |
| `definitionFile` | — | `config/project-scratch-def.json` | Defines the org shape used to build/validate the package version. |
| `namespace` | `dcstuff` | `""` (removed) | This is a **fork**, and the original `dcstuff` namespace is registered to an org the maintainer doesn't control. A 2GP namespace is globally unique and can't be re-registered or relinked, so the package is built **namespace-less** (see note below). |

The historical managed-package version aliases are preserved in `packageAliases` purely as
lineage; they are not used to build the unlocked package.

> ### A note on the namespace
> The upstream project shipped under the `dcstuff` namespace. Because a 2GP namespace is
> globally unique and permanently bound to the one namespace-registry org it was created in,
> a fork without access to that org **cannot reuse `dcstuff`** (and Salesforce won't let anyone
> re-register a taken namespace). This fork therefore builds as a **namespace-less** unlocked
> package — no namespace org or registry link is required, and any Dev Hub will do.
>
> To make this work, the three former namespace dependencies were de-namespaced:
> - `SOM_Scheduler.hasPermission` now matches the permission set by `Name` only (installed
>   components have `NamespacePrefix = null`).
> - `SOM_Scheduler.getClasses` now excludes only the local (`null`-namespace) `SOM_Scheduler` class
>   (`AND (NamespacePrefix != null OR Name != 'SOM_Scheduler')`).
> - `scheduler.js` navigates to `SchedulomaticEntry__c` / list view `All` without a prefix.
>
> If you would rather ship under **your own** namespace, register a new unique namespace in a
> Developer Edition org, link it to your Dev Hub, set `"namespace": "<yours>"` here, and
> reverse the three changes above to prefix with your namespace.

---

## 2. Prerequisites

1. **Salesforce CLI** installed and up to date: `sf --version`.
2. A **Dev Hub** org with *Unlocked Packages and Second-Generation Managed Packages* enabled
   (Setup → Packaging Settings). Authorize it:
   ```bash
   sf org login web --set-default-dev-hub --alias DevHub
   ```

Because this fork is **namespace-less**, no namespace org and no Namespace Registry link are
required — any Dev Hub works.

---

## 3. One-time: create the package

This registers the package with the Dev Hub and writes the `0Ho…` package id into
`packageAliases` under the name in `sfdx-project.json`.

```bash
sf package create \
  --name "Schedul-o-matic 9000 Unlocked" \
  --package-type Unlocked \
  --path force-app \
  --target-dev-hub DevHub
```

Confirm it was registered:

```bash
sf package list --target-dev-hub DevHub
```

> If you ever need to start completely fresh (no lineage), you can delete the historical
> `packageAliases` entries — they are only kept for reference.

---

## 4. Create a package version (build)

```bash
sf package version create \
  --package "Schedul-o-matic 9000 Unlocked" \
  --installation-key-bypass \
  --code-coverage \
  --wait 30 \
  --target-dev-hub DevHub
```

- `--installation-key-bypass` builds an unprotected version. To require a key on install,
  replace it with `--installation-key "<your-key>"`.
- `--code-coverage` computes Apex coverage (required before a version can be **promoted**).
- Because `versionNumber` ends in `.NEXT`, the build number auto-increments (e.g.
  `0.11.0.1`, `0.11.0.2`, …). Bump the major/minor manually in `sfdx-project.json` for a new
  release line.

The command appends the new `04t…` **subscriber package version id** to `packageAliases`,
e.g. `"Schedul-o-matic 9000 Unlocked@0.11.0-1": "04t…"`.

List versions:

```bash
sf package version list --target-dev-hub DevHub
```

---

## 5. Install into a target org

Install by alias or by the `04t…` id. For a scratch/sandbox first:

```bash
sf package install \
  --package "Schedul-o-matic 9000 Unlocked@0.11.0-1" \
  --wait 20 \
  --publish-wait 20 \
  --target-org MyTestOrg
```

Add `--installation-key "<your-key>"` if the version was built with one. After install, assign
the permission set so a user can actually open the app:

```bash
sf org assign permset --name Schedulomatic9000User --target-org MyTestOrg
```

> **Remote site / callouts:** because the app makes self-callouts (flow listing + anonymous
> Apex), ensure callouts to the org's My Domain URL are permitted in the target org.

---

## 6. Promote a version to released

Beta versions can be installed for testing. Promote to make a version installable in
production and immutable:

```bash
sf package version promote \
  --package "Schedul-o-matic 9000 Unlocked@0.11.0-1" \
  --target-dev-hub DevHub
```

Promotion requires Apex test coverage to meet the platform threshold (≥ 75%).

---

## 7. Upgrading subscribers

Re-running `sf package install` with a newer version id performs an in-place upgrade. Unlocked
packages allow subscribers to change packaged metadata in the org; the next package upgrade
will reconcile those components back toward the packaged definition, so treat the package as
the source of truth.

---

## Quick reference

| Action | Command |
| --- | --- |
| Create package (once) | `sf package create --name "Schedul-o-matic 9000 Unlocked" --package-type Unlocked --path force-app` |
| Build a version | `sf package version create -p "Schedul-o-matic 9000 Unlocked" --installation-key-bypass --code-coverage -w 30` |
| List versions | `sf package version list` |
| Install | `sf package install -p "Schedul-o-matic 9000 Unlocked@0.11.0-1" -w 20` |
| Assign perm set | `sf org assign permset -n Schedulomatic9000User` |
| Promote | `sf package version promote -p "Schedul-o-matic 9000 Unlocked@0.11.0-1"` |
