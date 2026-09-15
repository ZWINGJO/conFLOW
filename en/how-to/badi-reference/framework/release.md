# `release`

> **This hook stays empty in the reference class.** The reason is below.

| | |
|---|---|
| **When** | When the BAdI instance is released - at the end of processing. |

**PURPOSE** Cleanup. Specifically, of whatever you CREATED in GET_OBJECT_INFO or GET_AFTER_CREATION_WORKITEM.

**THE CONNECTION THAT IS EASILY OVERLOOKED**

If you attach your own display to the work item in the SAP GUI (a docking control with document details, the usual pattern), you create a singleton instance for it in GET_OBJECT_INFO. That instance then lives longer than the work item.

Without the counterpart here, the agent sees the data of the FIRST work item when opening the SECOND one. No error, no dump - just wrong numbers. That is why both examined implementations with a docking control have exactly one line here: DEL_INSTANCE( ).

Rule of thumb: RELEASE is empty, OR it is the counterpart to something you created yourself. There is no third case.

STAYS EMPTY HERE, because this example does not come with its own SAP GUI display.

## The code

```abap
*   zcl_cfl_workflow_00900_doc=>del_instance( ).
```
