# `get_after_execution_mobile`

| | |
|---|---|
| **When** | After execution from conMOBILE or from the BSP path. |
| **In** | IV_WI_ID   the work item<br>IV_ALTKEY  the clicked outcome |
| **Out** | CV_SUBRC     <> 0 aborts. 9 is the usual value.<br>CS_T100MSG   the matching message |

**THE ONLY HOOK THAT CAN ABORT AND SAY WHY.**

That makes it the most important gate in the whole interface - more important than its name suggests.

**THE NAME IS MISLEADING**

"MOBILE" suggests it only runs for conMOBILE. According to the documentation it is tied to the conMOBILE/BSP path; whether a particular Fiori inbox calls it must be CHECKED in your own system, not assumed. GET_FIORI_TASK_DEC_OP_ACT looked just as dead, and the hook ran.

If it does not run in your own UI, the gate has no effect there. The only option left: put the same check additionally into a BACKGROUND STEP after the decision - that always runs.

**FREE TEXT AS A T100 MESSAGE**

CS_T100MSG wants an ID and a number, not a string. 00/398 is '&1&2&3&4' - four variables of 50 characters each. That fits 200 characters of free text. A message class of your own is cleaner, but this way needs no new object.

## The code

```abap
    CONSTANTS lc_var_len TYPE i VALUE 50.

    IF iv_altkey IS INITIAL.
      RETURN.
    ENDIF.

    SELECT SINGLE * FROM /c09/cfl_s03 INTO @DATA(ls_cfl_s03) "#EC CI_ALL_FIELDS_NEEDED
      WHERE wi_id = @iv_wi_id.                                "#EC CI_NOORDER
    IF sy-subrc <> 0.
      RETURN.
    ENDIF.

*--------------------------------------------------------------------*
* The check: there is no rejection without a note.
*
* Deliberately the same business rule as in the guard further up -
* but effective at a different point. The guard removes what is not
* allowed at all; this gate checks what is still missing for the
* allowed action.
*--------------------------------------------------------------------*
    DATA lv_error TYPE string.

    IF iv_altkey = /c09/cfl_cl_workflow_0101=>mc_decision-nok AND
       get_val( iv_element = zcl_cfl_const_00900=>mc_prop-note
                iv_id      = ls_cfl_s03-id ) IS INITIAL.

      lv_error = 'A rejection needs a reason. Please fill in the note.' ##NO_TEXT.

    ENDIF.

    IF lv_error IS INITIAL.
      RETURN.
    ENDIF.

    DATA(lv_len) = strlen( lv_error ).

    cs_t100msg-msgid = '00'.
    cs_t100msg-msgno = '398'.
    cs_t100msg-msgty = 'E'.
    cs_t100msg-msgv1 = lv_error.

    IF lv_len > lc_var_len.
      cs_t100msg-msgv2 = lv_error+lc_var_len.
    ENDIF.
    IF lv_len > 100.
      cs_t100msg-msgv3 = lv_error+100.
    ENDIF.
    IF lv_len > 150.
      cs_t100msg-msgv4 = lv_error+150.
    ENDIF.

*--------------------------------------------------------------------*
* 9 means: do not continue. The work item stays open, the agent
* corrects and clicks again.
*--------------------------------------------------------------------*
    cv_subrc = 9.
```
