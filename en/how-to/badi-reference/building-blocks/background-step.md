# The background step

Not a BAdI hook - the second contract conFLOW knows. It is entered in /C09/CFL_C01 on step B1, with class name and method name. The call is DYNAMIC: a wrong signature does not show up at activation but at runtime - and then the workflow gets stuck.

---

The step where the work happens:

1. Read the document 2. Write the values to the container 3. Classify 4. Use EV_DECISION_KEY to say how the process continues

**WHY THE VALUES GO INTO THE CONTAINER INSTEAD OF JUST BEING READ**

The container is the basis for the decision, and it is frozen. If the agent decides tomorrow and the document was changed tonight, the agent still has the figures in front of them that the decision is about - and the audit trail shows afterwards which figures those were.

If you read fresh from EKKO in the work item text instead, you get a display that shifts under the agent, and afterwards no way of telling what they actually saw.

**WHY THE CLASSIFICATION IS HERE AND NOT IN THE DISPLAY**

For the same reason. SEVERITY and the recommendation are CALCULATED values - if they are in the container, you can see afterwards what the system recommended and whether the agent deviated from it. If the display does the calculation, that information is gone as soon as the work item is closed.

**THE RETURN VALUE IS THE SWITCH**

EV_DECISION_KEY is evaluated exactly like a human decision - except that here the code sets it. /C09/CFL_C02 then contains:

```
B1 + OK   -> 01   (decision needed)
B1 + UC1  -> X3   (nothing to do, workflow ends)
```

This keeps the branch VISIBLE IN CUSTOMIZING. That is why this approach is preferable to the hook GET_STATUS_DYNAMIC: there the same switch would be invisible.

IF EV_DECISION_KEY STAYS EMPTY, the workflow does not continue. This is the most common reason for "the workflow is stuck in the background step".

**ERRORS GO INTO ET_BAPIRET2, NOT INTO AN EXCEPTION**

conFLOW writes the table to the application log and evaluates it. An uncaught exception, on the other hand, puts the workflow step into error status, and the reason then only shows up in the dump.

## The code

```abap
    CLEAR: et_bapiret2, ev_decision_key.

*--------------------------------------------------------------------*
* The instance - only it knows the document. IS_CFL_S03 is the STEP
* and does not have the INSTID.
*--------------------------------------------------------------------*
    SELECT SINGLE * FROM /c09/cfl_s01 INTO @DATA(ls_cfl_s01) "#EC CI_ALL_FIELDS_NEEDED
      WHERE id = @is_cfl_s03-id.
    IF sy-subrc <> 0.
      add_msg( EXPORTING iv_type     = 'E'
                         iv_text     = |Workflow instance { is_cfl_s03-id } not found|
               CHANGING  ct_bapiret2 = et_bapiret2 ).
      RETURN.
    ENDIF.

*--------------------------------------------------------------------*
* 1. Read the document
*--------------------------------------------------------------------*
    read_document( EXPORTING iv_instid     = ls_cfl_s01-instid
                   IMPORTING ev_net_value  = DATA(lv_net_value)
                             ev_currency   = DATA(lv_currency)
                             ev_vendor     = DATA(lv_vendor)
                             ev_created_by = DATA(lv_created_by)
                             ev_found      = DATA(lv_found) ).

    IF lv_found = abap_false.
      add_msg( EXPORTING iv_type     = 'E'
                         iv_text     = |Purchase order { ls_cfl_s01-instid } not found|
               CHANGING  ct_bapiret2 = et_bapiret2 ).
      RETURN.
    ENDIF.

*--------------------------------------------------------------------*
* 2. Into the container
*
* CONV #( ) on every value: SET_VAL expects a string, the document
* fields are not strings. Without the conversion the compiler reports
* nothing - it converts on its own, but for packed numbers not the way
* you would expect. Being explicit is better here.
*--------------------------------------------------------------------*
    set_val( iv_element = zcl_cfl_const_00900=>mc_prop-doc_number
             iv_id      = ls_cfl_s01-id
             iv_value   = CONV #( ls_cfl_s01-instid ) ).

    set_val( iv_element = zcl_cfl_const_00900=>mc_prop-net_value
             iv_id      = ls_cfl_s01-id
             iv_value   = |{ lv_net_value }| ).

    set_val( iv_element = zcl_cfl_const_00900=>mc_prop-currency
             iv_id      = ls_cfl_s01-id
             iv_value   = CONV #( lv_currency ) ).

    set_val( iv_element = zcl_cfl_const_00900=>mc_prop-vendor
             iv_id      = ls_cfl_s01-id
             iv_value   = CONV #( lv_vendor ) ).

    set_val( iv_element = zcl_cfl_const_00900=>mc_prop-decision_by
             iv_id      = ls_cfl_s01-id
             iv_value   = CONV #( lv_created_by ) ).

*--------------------------------------------------------------------*
* 3. Classify
*--------------------------------------------------------------------*
    DATA(lv_severity) = classify( lv_net_value ).

    set_val( iv_element = zcl_cfl_const_00900=>mc_prop-severity
             iv_id      = ls_cfl_s01-id
             iv_value   = lv_severity ).

    DATA(lv_limit_hit) = COND string(
      WHEN lv_net_value > zcl_cfl_const_00900=>mc_limit_value
      THEN CONV string( abap_true )
      ELSE space ).

    set_val( iv_element = zcl_cfl_const_00900=>mc_prop-limit_hit
             iv_id      = ls_cfl_s01-id
             iv_value   = lv_limit_hit ).

*--------------------------------------------------------------------*
* 4. The switch - and the log entry for it
*
* Both branches write to the log. The branch in which NOTHING happens
* needs the line most: otherwise the log shows a workflow that ended
* itself for no visible reason.
*--------------------------------------------------------------------*
    IF lv_limit_hit IS INITIAL.

      DATA(lv_text) = |No approval required - { lv_net_value } { lv_currency } | &&
                      |is within the limit of { zcl_cfl_const_00900=>mc_limit_value }|.

      ev_decision_key = /c09/cfl_cl_workflow_0101=>mc_decision-uc1.

    ELSE.

      lv_text = |Approval required - { lv_net_value } { lv_currency } | &&
                |exceeds the limit of { zcl_cfl_const_00900=>mc_limit_value } | &&
                |({ lv_severity })|.

      ev_decision_key = /c09/cfl_cl_workflow_0101=>mc_decision-ok.

    ENDIF.

    add_msg( EXPORTING iv_text     = lv_text
             CHANGING  ct_bapiret2 = et_bapiret2 ).

    update_witext( lv_text ).
```
