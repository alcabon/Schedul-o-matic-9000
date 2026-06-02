# Apex Reference

All classes live in [`force-app/main/default/classes`](../force-app/main/default/classes) and
are compiled at API version `60.0`.

---

## `SOM_Scheduler`

`public with sharing class SOM_Scheduler implements Schedulable`

The single entry point for both the UI and the platform scheduler. It plays three roles:

1. **UI service** — `@AuraEnabled` methods consumed by the `scheduler` LWC.
2. **Scheduler** — a static `schedule(...)` helper that registers a `CronTrigger`.
3. **Runnable** — an instance `execute(SchedulableContext)` that the trigger invokes.

### Constants

| Constant | Value | Purpose |
| --- | --- | --- |
| `THIS_CLASS` | `'SOM_Scheduler'` | Excluded from the class lookup so the tool can't schedule itself. |
| `PERMISSION_SET` | `'Schedulomatic9000User'` | The gating permission set (matched by `Name`; the package is namespace-less). |
| `BATCHABLE_TYPE` / `SCHEDULABLE_TYPE` | `Type.forName(...)` | Cached `Type`s for `isAssignableFrom` reflection. |
| `MAX_BATCH_JOBS` / `MAX_SCHEDULED_JOBS` | `99` | Platform-imposed ceilings used for backpressure. |
| `ASYNC_JOB_COMPLETED_STATUSES` | `'~Completed~Failed~Aborted~'` | Membership-test string for batch completion. |

### `@AuraEnabled` API

#### `init() : Object` *(cacheable)*
Permission gate + bootstrap. Returns the list of schedulable flows for the current user, or
throws `AuraHandledException('No permiso!')` when the user lacks the permission set. Returns
the string `'Flow error!'` if flow retrieval fails (the LWC handles this sentinel).

#### `getClasses(String searchTerm) : List<Object>`
Returns Apex classes whose name matches `searchTerm` and that implement `Batchable` and/or
`Schedulable`. Notable details:

- The user input is escaped with `String.escapeSingleQuotes` before being used in the SOQL
  `LIKE` clause (SOQL-injection guard).
- The current package's own `SOM_Scheduler` class is filtered out.
- Each candidate is instantiated via reflection (`Type.forName(...).newInstance()`) inside a
  `Database.setSavepoint()` / `Database.rollback(sp)` envelope so that no instantiation side
  effect can persist. Classes without a no-arg constructor or that are private are skipped.
- Each match is decorated (`pre` / `mark` / `post`) so the LWC can highlight the matched text.

#### `schedule(String jobName, Datetime startDatetime, Id entryId) : String`
Formats `startDatetime` into a one-shot CRON expression and calls `System.schedule(...)`,
returning the new `CronTrigger` id.

### `Schedulable` implementation

#### `execute(SchedulableContext sc)`
Delegates to `start(sc.getTriggerId())`.

#### `start(Id jobId)` *(private)*
The run-time orchestrator. In order:
1. `System.abortJob(jobId)` — clears the spent one-shot trigger.
2. Loads the `SchedulomaticEntry__c`; verifies the owner is active and still permitted,
   otherwise records the reason in `ExecutionError__c` and returns.
3. If still within the schedule window, resolves and validates the target, checks capacity,
   executes, then re-schedules according to the recurrence model.

See [architecture.md](architecture.md#4-execution-flow-run-time) for the full decision tree.

### Execution helpers

| Method | Notes |
| --- | --- |
| `executeBatch()` | `Database.executeBatch`, honoring `BatchSize__c`; stores `AsyncApexJobId__c`. |
| `executeSchedulable(String)` | `@future` — reflects the type and calls `execute(new SOM_SchedulableContext())`. |
| `executeFlow(String)` | `@future(callout=true)` — builds a namespaced or local `Flow.Interview` and starts it. |
| `executeAnonymous(String)` | `@future(callout=true)` — POSTs to the Tooling `executeAnonymous` endpoint with the current user's session id. |
| `canStartMore()` | Returns `false` when the prior batch hasn't completed or the Holding queue is at `MAX_BATCH_JOBS`. |
| `isBeforeOrNoEndDateTime()` | Window check honoring `End__c`, `IsDaily__c`, and `DailyEnd__c`. |
| `reschedule()` / `rescheduleForTomorrow()` | Persist the entry then re-register the trigger. |
| `getApiVersion()` | Highest installed `ApexClass.ApiVersion`, used to build callout URLs. |
| `getLwcSessionId()` | Reads the session id from the `SchedulerHelper` Visualforce page (empty during tests). |
| `logExecutionError(Id, String)` | Writes a human-readable reason to `ExecutionError__c`. |

### Testability hooks
Many members are annotated `@TestVisible`, and several branches check `Test.isRunningTest()`
so that reflection and callouts can be short-circuited (substituting `BATCHABLE_TYPE`) during
unit tests. This keeps the tests deterministic without a live org.

---

## `SOM_Dao`

`public class SOM_Dao`

A minimal per-transaction query cache:

```apex
public List<SObject> getRecords(String query) {
    if (!recordsMap.containsKey(query)) {
        recordsMap.put(query, Database.query(query));
    }
    return recordsMap.get(query);
}
```

It memoizes `Database.query` results in a static `Map`, so the same dynamic SOQL string within
a transaction executes only once. The static map is `@TestVisible`, letting tests seed
results. Because it executes whatever query string it is given, callers (here, `getClasses`)
are responsible for escaping any user input before it reaches `getRecords` — which `SOM_Scheduler`
does with `String.escapeSingleQuotes`.

---

## `SOM_SchedulableContext`

`public with sharing class SOM_SchedulableContext implements SchedulableContext`

A hand-built `SchedulableContext`. You cannot construct the platform's `SchedulableContext`
directly, but `executeSchedulable` needs one to invoke a user class's `execute` method from a
`@future` context. `getTriggerId()` returns a synthetic, well-formed `CronTrigger` key prefix
padded to 15 characters.

---

## Test classes

| Class | Covers |
| --- | --- |
| `SOM_Scheduler_Test` | The scheduling/execution paths of `SOM_Scheduler`. |
| `SOM_Dao_Test` | The query-cache behavior of `SOM_Dao`. |
| `SOM_SchedulableContext_Test` | `getTriggerId()`. |
| `SOM_MockHttpResponse` | `HttpCalloutMock` implementation for the REST/Tooling callouts. |
| `SOM_TestUtils` | Shared fixture/setup helpers. |

Run them with `sf apex run test` (see [development.md](development.md)).
