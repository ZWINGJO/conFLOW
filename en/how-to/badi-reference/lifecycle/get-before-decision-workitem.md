# `get_before_decision_workitem`

| | |
|---|---|
| **When** | The work item exit builds the decision options - i.e. before the buttons are drawn. |
| **In and out** | CM_WORKITEM_CONTEXT - the work item context. You get the header and the options from it, and write the changed ones back into it. |

**CUSTOMIZING FIRST**

Fixed colour per outcome: c09-NATURE (P/N). Mandatory comment: c09-COMMENT_REQ. Hide: c09-NODISPLAY. All of it works in SAP GUI and Fiori, without code. This hook only for what depends on the document, as in the example below.

**THREE THINGS WORK HERE, AND ONLY HERE**

1. REMOVE A BUTTON (the "guard") If an action is not allowed from a business point of view, it disappears - instead of being rejected afterwards. That is the friendlier design: the agent only sees what they may do.

   The hook CANNOT force an error message. "Not allowed" here means button gone, not an error afterwards.

**2. COLOR A BUTTON - SAP GUI, VIA HTML IN ALTTEXT**

   The decision screen in the Business Workplace renders ALTTEXT as HTML. So the color is created INSIDE THE TEXT:

   ```
       <span style="color:green;font-size:120%">Approve</span>
   ```

   Color and size, no font-weight - both hit the same buttons, and a third attribute would not add a third signal. ALTTEXT is CHAR255, the wrapper costs about 50 characters; the c09t texts fit with room to spare.

   GUI GUARD ONLY. The Fiori inbox takes its button texts from exactly this hook - without the guard, the `<span>` would show up there as the label on the button. So ask GUI_IS_AVAILABLE and take the HTML route only when there is a real GUI.

**3. COLOR THE SAME BUTTON - FIORI, VIA ALTNATURE**

   SWR_DECIALTS has the field ALTNATURE. It works through the workflow DEFINITION and is NOT ENOUGH ON ITS OWN: the color the Fiori inbox actually draws comes from NATURE in GET_FIORI_TASK_DEC_OP_ACT. Setting it here does no harm and it stays as a second track - but if you write only this line, you see no color in Fiori and look for it in the wrong hook.

   There are EXACTLY TWO values - POSITIVE and NEGATIVE. That is not a palette but a statement, and it is handed out sparingly:

   ```
   POSITIVE   the calculated recommendation
   NEGATIVE   the one action that discards something for good
   neutral    everything else
   ```

   If the system itself recommends the hard action for once, the recommendation wins. Two signals on the same button would be none.

**WHY THE DECISION IS STILL MADE ONLY ONCE**

Two user interfaces, two MECHANISMS - the GUI colors via HTML in the text, Fiori via NATURE from the task gateway, and the two hooks work on different tables. If you maintain only one route, one of the two user interfaces has no color.

So you answer the question "which button is positive, which negative" ONCE - below in the loop over LV_ROLE - and serve both routes from it. Write the rule down twice and the two drift apart at the next additional outcome, and nobody notices, because hardly anyone opens both user interfaces side by side.

Confirmed at runtime in two systems.

**STARTING FROM THE WI_ID IS MANDATORY**

The hook does NOT receive the conFLOW instance. The only way to it goes through the work item header and /C09/CFL_S03.

## The code

```abap
    CONSTANTS lc_positive TYPE swr_nature VALUE 'POSITIVE' ##NO_TEXT.
    CONSTANTS lc_negative TYPE swr_nature VALUE 'NEGATIVE' ##NO_TEXT.

    DATA lt_decialts TYPE if_wapi_workitem_context=>swrtdecialts.

    DATA(ls_wihdr) = cm_workitem_context->get_header( ).

    SELECT SINGLE * FROM /c09/cfl_s03 INTO @DATA(ls_cfl_s03) "#EC CI_ALL_FIELDS_NEEDED
      WHERE wi_id = @ls_wihdr-wi_id.                          "#EC CI_NOORDER
    IF sy-subrc <> 0.
      RETURN.
    ENDIF.

    cm_workitem_context->get_decision_alts( IMPORTING et_decialts = lt_decialts ).

*--------------------------------------------------------------------*
* GUARD - only someone who can escalate may reject.
*
* In the example: the buyer on step 01 should not be able to reject a
* document above the limit alone. The manager on step 02 may.
*--------------------------------------------------------------------*
    IF ls_cfl_s03-gen_stat = zcl_cfl_const_00900=>mc_stat-approve AND
       is_true( get_val( iv_element = zcl_cfl_const_00900=>mc_prop-limit_hit
                         iv_id      = ls_cfl_s03-id ) ) = abap_true.

      DELETE lt_decialts WHERE altkey = /c09/cfl_cl_workflow_0101=>mc_decision-nok.

    ENDIF.

*--------------------------------------------------------------------*
* COLOR - one statement, two user interfaces, two mechanisms
*
* The HTML route is only taken when a GUI is really attached.
*--------------------------------------------------------------------*
    DATA(lv_recommended) = recommended_key( ls_cfl_s03-id ).

    DATA(lv_gui_color) = xsdbool( zcl_cfl_const_00900=>mc_gui_html_color = abap_true AND
                                  is_sapgui( )                           = abap_true ).

    LOOP AT lt_decialts ASSIGNING FIELD-SYMBOL(<fs_alt>).

*     The role of the button - determined ONCE, then served twice.
      DATA(lv_role) = COND char1(
        WHEN lv_recommended IS NOT INITIAL AND <fs_alt>-altkey = lv_recommended
          THEN 'P'
        WHEN <fs_alt>-altkey = /c09/cfl_cl_workflow_0101=>mc_decision-nok
          THEN 'N' ).

      IF lv_role IS INITIAL.
        CONTINUE.
      ENDIF.

      IF zcl_cfl_const_00900=>mc_fiori_nature = abap_true.
        <fs_alt>-altnature = COND swr_nature( WHEN lv_role = 'P' THEN lc_positive
                                              ELSE lc_negative ).
      ENDIF.

      IF lv_gui_color = abap_true.
        <fs_alt>-alttext = gui_colour(
          iv_text  = <fs_alt>-alttext
          iv_color = COND string( WHEN lv_role = 'P' THEN zcl_cfl_const_00900=>mc_gui_color-positive
                                  ELSE zcl_cfl_const_00900=>mc_gui_color-negative ) ).
      ENDIF.

    ENDLOOP.

    cm_workitem_context->set_decision_alts( it_decialts = lt_decialts ).
```
