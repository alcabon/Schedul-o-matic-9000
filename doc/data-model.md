# Data Model

## `SchedulomaticEntry__c`

One record represents one scheduled job. The record both **configures** the job (what to run
and when) and **records** its outcome (executions so far, last error, the live batch job id).

- **Label / Plural:** Schedul-o-matic Entry / Schedul-o-matic Entries
- **Sharing model:** Private (org-wide default), `ReadWrite` internal sharing
- **History tracking:** enabled
- **Reports / Search / Bulk API / Streaming API:** enabled

> The `Schedulomatic9000User` permission set (namespace-less) grants the object and field access
> required to use the tool; without it the UI shows the "No permiso!" panel.

### Fields

| API Name | Label | Type | Role |
| --- | --- | --- | --- |
| `Class__c` | Class | Text(40) | API name of the Apex class to run (optionally `namespace__Name`). |
| `IsBatchable__c` | Is Batchable | Checkbox | Run `Class__c` as a `Database.Batchable`. |
| `IsSchedulable__c` | Is Schedulable | Checkbox | Run `Class__c` as a `Schedulable`. |
| `BatchSize__c` | Batch Size | Number | Optional batch scope size; blank uses the Salesforce default. |
| `Flow__c` | Flow | Text | API name of an autolaunched flow to run. |
| `AnonymousCode__c` | Anonymous Code | Long Text Area | Block of anonymous Apex to execute. |
| `Start__c` | Start | Date/Time | When the (next) run fires. Required. |
| `End__c` | End | Date/Time | Optional. Stops a repeating/daily job once passed. |
| `RepeatInterval__c` | Repeat Interval | Number | Minutes between runs. Blank = run once. |
| `RescheduleInterval__c` | Reschedule Interval | Number | Minutes to wait before retrying a batch when the queue is full / prior run is unfinished. |
| `IsDaily__c` | Is Daily | Checkbox | Re-run at the same time each day. |
| `DailyStartDateTime__c` | Daily Start Date & Time | Date/Time | The daily run time; advanced by a day on each reschedule. |
| `DailyEnd__c` | Daily End | Date | Final day for a daily schedule. |
| `NumberOfExecutions__c` | Number of Executions | Number | Incremented on each successful run. |
| `AsyncApexJobId__c` | Async Apex Job Id | Text | Id of the most recent batch job; used to check completion. |
| `ExecutionError__c` | Execution Error | Text | Why the last run was skipped (`Inactive owner`, `Owner missing permissions`, `Invalid class`). |

### Field grouping by purpose

- **What to run** — `Class__c` (+ `IsBatchable__c` / `IsSchedulable__c` / `BatchSize__c`), `Flow__c`, or `AnonymousCode__c`. Exactly one of the three is required.
- **When to run** — `Start__c`, `End__c`, `RepeatInterval__c`, `RescheduleInterval__c`, `IsDaily__c`, `DailyStartDateTime__c`, `DailyEnd__c`.
- **Run state (system-maintained)** — `NumberOfExecutions__c`, `AsyncApexJobId__c`, `ExecutionError__c`. These are read-only in the permission set.

## Validation rules

| Rule | Enforces |
| --- | --- |
| `Start_date_time_valid` | `Start__c` is required (`ISBLANK(Start__c)` is an error). |
| `End_date_time_valid` | `Start__c` must be before `End__c`. |
| `Daily_end_date_valid` | `Start__c` must be before the daily final end date. |
| `Repeat_interval_valid` | `RepeatInterval__c` is empty or a positive whole number. |
| `Reschedule_interval_valid` | `RescheduleInterval__c` is a positive whole number. |
| `Batch_size_valid` | `BatchSize__c` is blank (use default) or a positive whole number. |
| `Is_Class_Flow_or_Code` | An entry must specify a class, a flow, **or** an anonymous code block. |
| `Is_Class_and_Batchable_or_Schedulable` | A class entry must be marked batchable or schedulable. |

> These rules run on `createRecord` in the LWC, so most bad input is rejected before a
> `CronTrigger` is ever created. Owner-active and permission checks, by contrast, are enforced
> at *run time* by `SOM_Scheduler.start` (recorded in `ExecutionError__c`).
