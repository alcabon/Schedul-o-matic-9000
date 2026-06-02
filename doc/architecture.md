# Architecture

This document describes how the Schedul-o-matic 9000 is put together and what happens, step
by step, when a user schedules and then runs a job.

## 1. Components

### Lightning Web Components (`force-app/main/default/lwc`)

| Component | Responsibility |
| --- | --- |
| `scheduler` | The main UI. Loads the permitted user's flows, drives the form, builds a `SchedulomaticEntry__c`, and calls the Apex `schedule` method. |
| `classLookup` | Type-ahead lookup that calls `SOM_Scheduler.getClasses` and renders matches. |
| `classLookupOption` | A single, highlighted lookup result row (pre/mark/post text segments). |
| `schedulerUtils` | Shared helper module (e.g. `showToast`). |

The `scheduler` component is exposed through a custom tab, the App Launcher app, the Utility
Bar, and can be dropped onto any Lightning page via App Builder.

### Apex (`force-app/main/default/classes`)

| Class | Role |
| --- | --- |
| `SOM_Scheduler` | The heart of the system. Implements `Schedulable`; exposes `@AuraEnabled` entry points; performs scheduling, validation, and job execution. |
| `SOM_Dao` | A tiny per-transaction query cache used by `getClasses`. |
| `SOM_SchedulableContext` | A hand-rolled `SchedulableContext` so a scheduled class can be invoked directly from a `@future` method. |
| `SOM_MockHttpResponse` | Test double for HTTP callouts. |
| `SOM_TestUtils` | Shared test data/setup helpers. |
| `*_Test` | Apex unit tests for the classes above. |

See [apex-reference.md](apex-reference.md) for method-level detail.

### Visualforce (`force-app/main/default/pages`)

| Page | Role |
| --- | --- |
| `SchedulerHelper` | Renders `{!$Api.Session_ID}` as JSON. The LWC fetches this to obtain a session id usable for REST/Tooling callouts back into the same org (the standard LWC pattern for self-callouts). |

### Other metadata

- **Custom object** `SchedulomaticEntry__c` — one record per scheduled job. See [data-model.md](data-model.md).
- **Permission set** `Schedulomatic9000User` — gates all access (object, fields, Apex, page, tabs, app).
- **Custom labels** (`CustomLabels.labels-meta.xml`) — every visible string, so the UI is fully translatable.
- **Static resources** — the scheduler icon and the "missing permissions" pickle.
- **App / tabs / flexipages** — surface the component throughout the org.

## 2. Why the org calls itself over HTTP

Two operations cannot be done with ordinary synchronous Apex DML/SOQL and therefore use a
REST/Tooling callout back into the *same* org, authenticated with the session id from
`SchedulerHelper.page`:

1. **Listing active autolaunched flows** — queried from `FlowDefinitionView` via the REST
   Query API (`SOM_Scheduler.getFlows`).
2. **Executing anonymous Apex** — submitted to the Tooling API `executeAnonymous` endpoint
   (`SOM_Scheduler.executeAnonymous`, a `@future(callout=true)` method).

The API version for the callout endpoint is resolved at runtime from the highest installed
`ApexClass.ApiVersion` (`SOM_Scheduler.getApiVersion`), so the package does not hard-code a REST
version in the URL.

> **Note:** because these are self-callouts, the org's domain URL
> (`Url.getOrgDomainUrl()`) must be allowed as a Remote Site / trusted URL, or callouts must
> otherwise be permitted in the target org.

## 3. Scheduling flow (design time)

```
User opens "Schedul-o-matic 9000"
        │
        ▼
scheduler LWC  ──init()──▶  SOM_Scheduler.init()
        │                         │ hasPermission(currentUser)?  ── no ──▶ AuraHandledException('No permiso!')
        │                         │ yes
        │                         ▼
        │                    getFlows()  (REST query of FlowDefinitionView)
        │◀── list of flows ───────┘
        │
   user types a class name
        │
classLookup ──getClasses(term)──▶ SOM_Scheduler.getClasses()
        │   (reflection determines Batchable / Schedulable, inside a rolled-back Savepoint)
        │◀── matching classes ────┘
        │
   user configures timing + clicks Schedule
        │
        ▼
createRecord(SchedulomaticEntry__c)   ── platform validation rules run here
        │  (returns the new entry Id)
        ▼
schedule(jobName, startDatetime, entryId) ──▶ SOM_Scheduler.schedule()
                                                 System.schedule(name, cron, new SOM_Scheduler(entryId))
```

`SOM_Scheduler.schedule` converts the chosen start `Datetime` into a one-shot CRON expression
(`'s m H d M ? yyyy'`) and registers a `CronTrigger`.

## 4. Execution flow (run time)

When the platform fires the `CronTrigger`, it calls `SOM_Scheduler.execute(SchedulableContext)`,
which delegates to the private `start(jobId)` method:

```
execute(sc) ──▶ start(sc.getTriggerId())
        │
        ▼
System.abortJob(jobId)            ← the one-shot trigger has fired; tidy it up
        │
        ▼
load SchedulomaticEntry__c
        │  Owner inactive?            ── yes ──▶ log "Inactive owner", stop
        │  Owner lacks permission?    ── yes ──▶ log "Owner missing permissions", stop
        ▼
isBeforeOrNoEndDateTime()?
        │  no  ──▶ if IsDaily__c: rescheduleForTomorrow(); else stop
        │  yes
        ▼
resolve & validate the target (Class / Flow / Anonymous)
        │  Class invalid / not Batchable|Schedulable as declared ──▶ log "Invalid class", stop
        ▼
canStartMore()?   (batch queue capacity / previous batch finished?)
        │  no  ──▶ if RescheduleInterval__c set: bump Start__c and reschedule()
        │  yes
        ▼
executeJob()                       ← batch | schedulable | flow | anonymous
        │
        ▼
RepeatInterval__c set?  ── yes ──▶ Start__c += interval; incrementExecutions(); reschedule()
IsDaily__c?             ── yes ──▶ incrementExecutions(); rescheduleForTomorrow()
else                    ──────────▶ incrementExecutions(); updateEntry()  (one-and-done)
```

### Job dispatch (`executeJob`)

| Entry contents | How it runs |
| --- | --- |
| `Class__c` + `IsBatchable__c` | `Database.executeBatch(...)`, honoring `BatchSize__c` if set. The returned `AsyncApexJobId__c` is stored so the next run can check completion. |
| `Class__c` + `IsSchedulable__c` | `@future` `executeSchedulable` reflects the type and calls `execute(new SOM_SchedulableContext())`. |
| `Flow__c` | `@future` `executeFlow` builds a `Flow.Interview` (namespaced or not) and `start()`s it. |
| `AnonymousCode__c` | `@future(callout=true)` `executeAnonymous` posts the body to the Tooling `executeAnonymous` REST endpoint. |

### Recurrence model

- **Run once:** no `RepeatInterval__c`, `IsDaily__c = false`. Executes a single time.
- **Repeat every N minutes:** `RepeatInterval__c = N`. Re-schedules itself `N` minutes after each run, until `End__c` (if any) passes.
- **Daily:** `IsDaily__c = true` with `DailyStartDateTime__c` / `DailyEnd__c`. Re-schedules for the same time tomorrow.
- **Combined window:** a repeat interval *plus* daily — e.g. every 30 minutes between 2 pm and 5 pm every day until next Thursday.
- **Backpressure:** for batch jobs, `RescheduleInterval__c` defers the next attempt when the
  Holding batch queue is full (`MAX_BATCH_JOBS = 99`) or the previous batch has not completed.

## 5. Security model

- **UI gate:** `SOM_Scheduler.init` throws `AuraHandledException('No permiso!')` unless the current
  user holds the `Schedulomatic9000User` permission set.
- **Execution gate:** at run time, `start` re-checks that the *entry owner* is active and still
  holds the permission set; otherwise it records the reason in `ExecutionError__c` and stops.
- **Least surprise for anonymous code:** anonymous Apex runs under the *current user's* session
  id and therefore the current user's permissions.
- **Sharing:** `SOM_Scheduler` and `SOM_SchedulableContext` are declared `with sharing`.
  `SOM_Dao` is `without`-sharing-neutral (it simply runs the query it is handed) — see the
  reference doc for the rationale and the SOQL-injection guard in `getClasses`.

> **Namespace:** this fork ships as a **namespace-less** unlocked package. Installed components
> have no prefix (`NamespacePrefix = null`), which is why `hasPermission` matches the permission
> set by `Name` alone and `getClasses` excludes only the local (`null`-namespace) `SOM_Scheduler`
> class. See [unlocked-package.md](unlocked-package.md#a-note-on-the-namespace).
