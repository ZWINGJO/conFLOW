# `get_after_execution`

| | |
|---|---|
| **When** | After the agent has decided in the SAP GUI, before the work item is completed. |
| **In** | IV_WI_ID     the work item<br>IV_ALTKEY    the CLICKED outcome - the value that matters<br>IV_ALT_TEXT  its text<br>IV_MSELNOTE  default for the note dialog |
| **Out** | CV_SUBRC      1 aborts (other values do not)<br>CS_OBJECT_ID  reference to a captured note |

**TWO DIFFERENT TASKS THAT COINCIDE HERE**

1. FOLLOW-UP PROCESSING - update the container, record who decided. This is the usual case.

2. FORCING A NOTE - first c09-COMMENT_REQ on the outcome: works in SAP GUI and Fiori, without code. Here only if the obligation depends on the document - then via SWU_INTERN_DECI_NOTE_POPUP with MSELNOTE = '2' (IV_MSELNOTE is usually empty = optional) and only if CS_OBJECT_ID is still empty.

**THE ABORT CANNOT SAY WHY**

CV_SUBRC = 1 stops the process, but there is no message parameter. The agent clicks and nothing happens - the worst feedback there is. If you need a check WITH a reason, call the same check additionally in GET_AFTER_EXECUTION_MOBILE, the only hook with CS_T100MSG.

**FIRST LINE: CHECK IV_ALTKEY**

The hook also runs for actions that are not a decision (forward, postpone). Then IV_ALTKEY is empty, and any logic that assumes an outcome reaches into nothing.

## The code

```abap
    IF iv_altkey IS INITIAL.
      RETURN.
    ENDIF.

    SELECT SINGLE * FROM /c09/cfl_s03 INTO @DATA(ls_cfl_s03) "#EC CI_ALL_FIELDS_NEEDED
      WHERE wi_id = @iv_wi_id.                                "#EC CI_NOORDER
    IF sy-subrc <> 0.
      RETURN.
    ENDIF.

*--------------------------------------------------------------------*
* Record WHO decided.
*
* This is also in the work item log (/C09/CFL_S03 with user and
* time) - but in the container it is READABLE for the following
* steps without them having to evaluate the log. The next step can,
* for example, derive its agent from it (see GET_ACTORS, variant B).
*--------------------------------------------------------------------*
    set_val( iv_element = zcl_cfl_const_00900=>mc_prop-decision_by
             iv_id      = ls_cfl_s03-id
             iv_value   = CONV #( sy-uname ) ).

*--------------------------------------------------------------------*
* Require a reason on rejection.
*
* The popup belongs to the workflow standard, not to conFLOW. If the
* agent cancels it (RETURNCODE 'A'), the FM raises an exception -
* and then the decision does not take effect either.
*
* MSELNOTE = '2' makes the note mandatory. If CS_OBJECT_ID is already
* filled, c09-COMMENT_REQ has already required the note - no second
* popup.
*--------------------------------------------------------------------*
    IF iv_altkey = /c09/cfl_cl_workflow_0101=>mc_decision-nok AND
       cs_object_id IS INITIAL.

      CALL FUNCTION 'SWU_INTERN_DECI_NOTE_POPUP'
        EXPORTING  wi_id          = iv_wi_id
                   alt_text       = iv_alt_text
                   mselnote       = '2'
        IMPORTING  ex_object_id   = cs_object_id
        EXCEPTIONS user_cancelled = 1
                   OTHERS         = 2.

      IF sy-subrc <> 0.
        cv_subrc = 1.
        RETURN.
      ENDIF.

    ENDIF.
```
