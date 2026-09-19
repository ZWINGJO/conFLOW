# `get_fiori_task_dec_op_act`

| | |
|---|---|
| **When** | The Fiori inbox fetches its decision options. The path goes through the task gateway handler, not through the work item exit - which is why GET_BEFORE_DECISION_WORKITEM alone is not enough. |
| **In** | IV_INSTANCE_ID  the WI_ID |
| **Out** | CT_DEC_OPT      the options of the inbox |

**FIXED COLOR AND MANDATORY COMMENT**

They come from c09-NATURE and c09-COMMENT_REQ, without code. The framework sets them right after this hook and only where the hook has set nothing.

**THIS HOOK LOOKS DEAD IN THE WHERE-USED LIST - AND STILL RUNS**

/C09/CL_TGW_RFC_HANDLER is not a class of its own, but a conFLOW ENHANCEMENT on the task gateway handler. That is why where-used finds nothing. The hook runs anyway, and it is the same place where the button texts from /C09/CFL_C09T are set.

**MATCH ON THE KEY, NEVER ON THE TEXT**

The obvious choice would be a comparison on DECISION_TEXT. It cannot work: at this point it already holds the TRANSLATED text from /C09/CFL_C09T, no longer the raw key. DECISION_KEY, on the other hand, is NUMC4 with the same numbering as SWR_DECIKEY (0001 = OK, 0002 = NOK, 0003 = UC1 ...) and is therefore stable.

**DO NOT TOUCH DECISION_TEXT**

The framework checks right AFTER this call whether the text still contains a '-', and otherwise skips the C09T translation. If you write to the text here, you end up with UNLABELED buttons - and look for the error in the wrong place.

## The code

```abap
CONSTANTS lc_positive TYPE /iwwrk/wf_decision_nature VALUE 'POSITIVE' ##NO_TEXT.
CONSTANTS lc_negative TYPE /iwwrk/wf_decision_nature VALUE 'NEGATIVE' ##NO_TEXT.

IF zcl_cfl_const_00900=>mc_fiori_nature = abap_false.
  RETURN.
ENDIF.

SELECT SINGLE * FROM /c09/cfl_s03 INTO @DATA(ls_cfl_s03) "#EC CI_ALL_FIELDS_NEEDED
  WHERE wi_id = @iv_instance_id.                          "#EC CI_NOORDER
IF sy-subrc <> 0.
  RETURN.
ENDIF.

DATA(lv_recommended) = recommended_key( ls_cfl_s03-id ).

LOOP AT ct_dec_opt ASSIGNING FIELD-SYMBOL(<fs_opt>).
  IF lv_recommended IS NOT INITIAL AND <fs_opt>-decision_key = lv_recommended.
    <fs_opt>-nature = lc_positive.
  ELSEIF <fs_opt>-decision_key = /c09/cfl_cl_workflow_0101=>mc_decision-nok.
    <fs_opt>-nature = lc_negative.
  ENDIF.
ENDLOOP.
```
