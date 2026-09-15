# `get_before_execution_workitem`

> **This hook stays empty in the reference class.** The reason is below.

| | |
|---|---|
| **When** | Immediately before a work item is executed - i.e. after the double-click, before the user interface is built. |
| **In** | IS_SWR_WIHDR  the work item header |
| **Out** | nothing. |

```
PURPOSE   Preparations that are needed exactly when someone actually
       opens the work item - setting locks, filling a cache,
       incrementing counters.
```

**NOT FILLED IN ANY OF THE PRODUCTION IMPLEMENTATIONS EXAMINED.**

The reason: it cannot prevent anything. There is no return parameter and no way to abort - whatever happens here happens on the side. If you are looking for a check, go to GET_BEFORE_DECISION_WORKITEM (remove the button) or GET_AFTER_EXECUTION_MOBILE (abort with a message).

STAYS EMPTY. That is the decision, not a leftover.
