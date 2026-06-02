# Schedul-o-matic 9000 — Technical Documentation

*Not your mother's Salesforce scheduler.*

The Schedul-o-matic 9000 lets a Salesforce admin or developer schedule, from a friendly
Lightning UI, any of the following without writing a line of CRON:

- An Apex class that implements `Schedulable` and/or `Database.Batchable`
- An autolaunched **Flow**
- An arbitrary block of **anonymous Apex**

Jobs can run once, on a repeating minute interval, daily within a time window, or any
combination thereof. Every scheduled job is backed by a `SchedulomaticEntry__c` record that
can be edited between runs.

---

## Documentation index

| Document | What's inside |
| --- | --- |
| [architecture.md](architecture.md) | Component map, runtime, and the end-to-end scheduling/execution flow. |
| [apex-reference.md](apex-reference.md) | Class-by-class Apex reference (`SOM_Scheduler`, `SOM_Dao`, helpers). |
| [data-model.md](data-model.md) | The `SchedulomaticEntry__c` object, every field, and all validation rules. |
| [unlocked-package.md](unlocked-package.md) | **How to build, version, install, and promote the unlocked (2GP) package.** |
| [development.md](development.md) | Local setup, scratch orgs, Apex + Jest tests, linting, and formatting. |

> Looking for the end-user / marketing overview? See the root [README.md](../README.md).

---

## At a glance

| Aspect | Value |
| --- | --- |
| Package name | `Schedul-o-matic 9000 Unlocked` |
| Package type | Unlocked (second-generation packaging / 2GP) |
| Namespace | None (namespace-less) |
| Source API version | `60.0` |
| Primary metadata root | [`force-app/main/default`](../force-app/main/default) |
| License | BSD-3-Clause (see [LICENSE.txt](../LICENSE.txt)) |

## High-level component map

```
Lightning UI (LWC)                Apex (server)                 Salesforce platform
─────────────────                 ─────────────                 ───────────────────
scheduler  ───────────────────▶  SOM_Scheduler.init()      ──────▶  PermissionSetAssignment
  classLookup                    SOM_Scheduler.getClasses()         ApexClass (Tooling/REST)
    classLookupOption            SOM_Scheduler.schedule()   ──────▶  System.schedule() / CronTrigger
  schedulerUtils                                                FlowDefinitionView
                                 SOM_Scheduler.execute()    ──────▶  Database.executeBatch()
                                   SOM_Dao (query cache)             Flow.Interview
                                   SchedulableContext…           executeAnonymous (Tooling REST)
                                                                 SchedulomaticEntry__c
SchedulerHelper.page  ─────────▶  session id for REST callouts
```

See [architecture.md](architecture.md) for the detailed flow.
