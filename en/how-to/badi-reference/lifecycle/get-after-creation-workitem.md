# `get_after_creation_workitem`

| | |
|---|---|
| **When** | After EVERY work item is created - including the background steps - and before it is first displayed. |
| **In** | IS_DATA_STEP  the step (/C09/CFL_S03)<br>IS_SWR_WIHDR  the work item header, including WI_ID |
| **Out** | nothing. Effect only through side effects. |

```
PURPOSE   Everything that should happen ONCE PER WORK ITEM: priority,
       attachments, notes, preparing the container, attaching a
       custom user interface.
```

**THE HOOK ALSO RUNS FOR BACKGROUND STEPS**

And that is almost always unwanted. A priority on a work item that nobody sees only costs runtime. That is why the method starts with a check that lets only the dialog steps through - the line looks like a trifle and is not.

**THE WORK ITEM IS NOT YET ON THE DATABASE HERE**

The central point of this hook, and the cause of the most common disappointment: every API that READS SWWWIHEAD comes back empty. SAP_WAPI_CHANGE_WORKITEM_PRIO does exactly that - it reports no error, it just has no effect. The right way goes through the work item manager of the running transaction, see SET_PRIORITY( ).

## The code

```abap
    IF is_data_step-gen_stat <> zcl_cfl_const_00900=>mc_stat-approve AND
       is_data_step-gen_stat <> zcl_cfl_const_00900=>mc_stat-escalate.
      RETURN.
    ENDIF.

    IF is_swr_wihdr-wi_id IS INITIAL.
      RETURN.
    ENDIF.

*--------------------------------------------------------------------*
* The escalation is always urgent - that is exactly why it ended up
* with the manager. Otherwise the severity calculated in B1
* decides.
*--------------------------------------------------------------------*
    DATA lv_prio TYPE sww_prio.

    IF is_data_step-gen_stat = zcl_cfl_const_00900=>mc_stat-escalate.
      lv_prio = zcl_cfl_const_00900=>mc_prio-high.

    ELSEIF get_val( iv_element = zcl_cfl_const_00900=>mc_prop-severity
                    iv_id      = is_data_step-id ) = zcl_cfl_const_00900=>mc_severity-red.
      lv_prio = zcl_cfl_const_00900=>mc_prio-high.

    ELSE.
      lv_prio = zcl_cfl_const_00900=>mc_prio-medium.
    ENDIF.

    set_priority( iv_wi_id = is_swr_wihdr-wi_id
                  iv_prio  = lv_prio ).
```
