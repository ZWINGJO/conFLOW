# `get_factory_calendar`

> **This hook stays empty in the reference class.** The reason is below.

| | |
|---|---|
| **When** | When conFLOW calculates a deadline - i.e. when a work item with a deadline in /C09/CFL_C02 is created. |
| **In** | IS_CFL_S01           the instance |
| **Out** | CV_FACTORY_CALENDAR  the factory calendar (SCAL-FCALID) |

**PURPOSE** So that "a two-day deadline" does not expire on Friday afternoon because Saturday and Sunday were counted.

**NOT FILLED IN ANY OF THE PRODUCTION IMPLEMENTATIONS EXAMINED.**

That does NOT mean deadlines are correct without a calendar - it means the processes examined calculate deadlines in hours or have none at all.

**WHEN YOU NEED IT**

As soon as a deadline runs in DAYS and the escalation has an effect that annoys someone. A work item that escalates over Easter because four public holidays were counted is the classic first production bug.

The calendar then usually comes from the plant or the company code assignment of the document - i.e. from the document data, not from a constant.

**STAYS EMPTY.**
