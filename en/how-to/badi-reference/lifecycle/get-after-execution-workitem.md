# `get_after_execution_workitem`

| | |
|---|---|
| **When** | After a work item has been completed - regardless of which UI it came from. |
| **In** | IS_DATA_STEP  the step<br>IS_SWR_WIHDR  the work item header<br>IV_KEY        the outcome |
| **Out** | nothing, and NO ABORT EITHER. Whatever happens here happens after the decision. |

**THE CLASSIC USE CASE: CLEANING UP PARALLEL WORK ITEMS**

When a step has several agents in parallel and one of them rejects, the other work items should disappear - otherwise people work on a case that has already been decided.

/C09/CFL_CL_HELPER_0101=>SET_WORKITEM_OBSOLET does that: it finds all open work items of the same top-level workflow and sets them to obsolete - except its own.

**FIRST CHECK WHETHER YOU NEED THIS HOOK AT ALL**

Veto, "first decision counts" and "majority decides" are a setting on the step: column Decision rule in Approval steps (C01). Then this hook stays empty. The example here is for rules that none of the settings cover.

**WHY THE COMMIT IS HERE**

SET_WORKITEM_OBSOLET calls SAP_WAPI_WORKITEM_COMPLETE with DO_COMMIT = FALSE, so that not every work item is committed separately. The COMMIT therefore has to come from the caller. Without this line the work items stay open - with no error message.

**DIFFERENCE FROM GET_AFTER_EXECUTION**

GET_AFTER_EXECUTION runs BEFORE completion and can prevent it. This one runs AFTERWARDS. If you want to check, use the other one.

## The code

```abap
IF iv_key <> /c09/cfl_cl_workflow_0101=>mc_decision-nok.
  RETURN.
ENDIF.

/c09/cfl_cl_helper_0101=>set_workitem_obsolet( is_data_step = is_data_step ).

COMMIT WORK AND WAIT.
```
