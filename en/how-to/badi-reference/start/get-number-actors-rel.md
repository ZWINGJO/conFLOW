# `get_number_actors_rel`

| | |
|---|---|
| **When** | Before GET_ACTORS, once per step. |
| **In** | IS_CFL_S01  the instance<br>IV_PROCESS  the context - 'MAIL' for mail dispatch |
| **Out** | CT_ACTORS   the AGENT GROUPS (/C09/CFL_C05_TT), not the agents. That is exactly the difference from GET_ACTORS. |

**PURPOSE** Two things that are not possible anywhere else:

1. PARALLEL WORK ITEMS. If you turn one entry into several, you get several work items on the same step. This is how approval levels come about whose NUMBER is only known at runtime - four signatures for this document, two for the next one.

2. SETTING THE FLAG FOR GET_ACTORS. IV_PROCESS only exists here. If you want to distinguish between work item and mail in GET_ACTORS, you need this line.

**DO YOU NEED IT**

Not in the normal case. One step, one agent group, any number of people in it - GET_ACTORS handles that on its own. This hook is for the case where the NUMBER OF STEPS is variable.

**WHAT TO WATCH OUT FOR**

The field WF_DEFINITION_OK in the entries is used as a pass counter for parallel steps. That is a misuse of the field, but it is the established solution - if you duplicate the entries without this index, you get work items that cannot be told apart.

## The code

```abap
    mv_process = iv_process.

*--------------------------------------------------------------------*
* For mail dispatch, only include the info recipient if it is
* activated in customizing. That way the customer can switch a
* notification on and off without touching the code.
*--------------------------------------------------------------------*
    IF iv_process = 'MAIL'.

      SELECT SINGLE objid FROM /c09/cfl_c03 INTO @DATA(lv_objid)
        WHERE wf_definition = @is_cfl_s01-wf_definition
          AND gen_stat_user = @zcl_cfl_const_00900=>mc_gsu-mail_info
          AND objid         = @abap_true.

      IF sy-subrc <> 0.
        DELETE ct_actors WHERE gen_stat_user = zcl_cfl_const_00900=>mc_gsu-mail_info.
      ENDIF.

    ENDIF.
```
