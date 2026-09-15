# The helpers

Eleven methods that are not hooks. They are the reason
the hooks above stay readable: without them, every hook
would contain the same loop.

## `get_val`

A container element can hold SEVERAL values - that is why GET_ATTRIBUT_VALUE returns a table. Nine times out of ten you want the first and only one.

These four lines are the reason the hooks above are readable. Without them, every hook contains the same loop.

```abap
DATA(lt_value) = /c09/cfl_cl_workflow_0101=>get_attribut_value(
                   iv_element = iv_element
                   iv_id      = iv_id ).

READ TABLE lt_value ASSIGNING FIELD-SYMBOL(<fs_value>) INDEX 1.
IF sy-subrc = 0.
  rv_val = <fs_value>-value.
ENDIF.
```

## `set_val`

The counterpart. SET_ATTRIBUT_VALUE REPLACES the content of the element - it does not append. If you want several values, build the table yourself and call the framework method directly.

SET_ATTRIBUT_VALUE first looks up the ID in /C09/CFL_S01. If the instance is not there, the value is silently discarded: no error, no entry in S04, and reading it returns empty. The element itself does not have to be declared anywhere.

```abap
DATA lt_value TYPE /c09/cfl_value_s04_tt.

APPEND INITIAL LINE TO lt_value ASSIGNING FIELD-SYMBOL(<fs_value>).
<fs_value>-value = iv_value.

/c09/cfl_cl_workflow_0101=>set_attribut_value(
  iv_element = iv_element
  iv_id      = iv_id
  it_value   = lt_value ).
```

## `is_true`

The container has no booleans, only strings. What comes back from SET_VAL( abap_true ) is an 'X' - but depending on who set the value, it can also be 'x', 'true' or '1'.

A central evaluation is therefore not a luxury: otherwise one place checks for 'X' and the next for abap_true, and with lowercase input they disagree.

```abap
DATA(lv_upper) = to_upper( condense( iv_value ) ).

rv_yes = xsdbool( lv_upper = 'X'    OR
                  lv_upper = 'TRUE' OR
                  lv_upper = '1' ).
```

## `read_document`

The ONLY method that knows the document. If you adapt the class to a different document type, you change it here - and in no hook.

**EV_FOUND INSTEAD OF SY-SUBRC TO THE CALLER**

The caller should not need to know how many SELECTs the method consists of. A meaningful flag is more robust than a SY-SUBRC that the next statement overwrites.

**THE SUM ACROSS THE ITEMS**

For an approval, the value of the document counts, not that of a single item. SELECT SUM returns SY-SUBRC 4 and an initial value for a document without items - that is why the existence check is on EKKO and not on the sum.

```abap
CLEAR: ev_net_value, ev_currency, ev_vendor, ev_created_by.
ev_found = abap_false.

DATA lv_ebeln TYPE ekko-ebeln.
lv_ebeln = iv_instid.

IF lv_ebeln IS INITIAL.
  RETURN.
ENDIF.

SELECT SINGLE waers, lifnr, ernam
  FROM ekko
  INTO ( @ev_currency, @ev_vendor, @ev_created_by )
  WHERE ebeln = @lv_ebeln.

IF sy-subrc <> 0.
  RETURN.
ENDIF.

ev_found = abap_true.

SELECT SUM( netwr )
  FROM ekpo
  INTO @ev_net_value
  WHERE ebeln = @lv_ebeln
    AND loekz = @space.                  " do not count deleted items
```

## `classify`

The business rule, in exactly one place.

It is deliberately NOT in the hook, even though it would only be three lines there. The reason is not aesthetics: as soon as the rule exists in two places - once for the display, once for the decision - the two drift apart at some point, and then the work item shows something different from what the workflow does.

In a real installation this method belongs in a SEPARATE RULES CLASS that also serves the value help of the UI. Then display, recommendation and validation demonstrably come from the same source.

```abap
IF iv_net_value > zcl_cfl_const_00900=>mc_limit_value * 5.
  rv_severity = zcl_cfl_const_00900=>mc_severity-red.

ELSEIF iv_net_value > zcl_cfl_const_00900=>mc_limit_value.
  rv_severity = zcl_cfl_const_00900=>mc_severity-yellow.

ELSE.
  rv_severity = zcl_cfl_const_00900=>mc_severity-green.
ENDIF.
```

## `recommended_key`

Which outcome the system recommends - for the green highlight in both UIs.

The method returns a KEY, not a text. That is intentional: the button texts come from /C09/CFL_C02T and /C09/CFL_C09T and are translated. Comparing texts here would give you a recommendation that works in English and not in German.

```abap
IF is_true( get_val( iv_element = zcl_cfl_const_00900=>mc_prop-limit_hit
                     iv_id      = iv_id ) ) = abap_true.
  CLEAR rv_key.                          " above the limit: no recommendation
ELSE.
  rv_key = /c09/cfl_cl_workflow_0101=>mc_decision-ok.
ENDIF.
```

## `fmt_doc`

Strip leading zeros. '0004500001234' becomes '4500001234'.

**A WARNING THAT COSTS MONEY IN PRACTICE**

The result is NOT usable as a key for a SELECT. If you put a document number formatted like this into a WHERE condition, you find nothing - and you get no error message, just an empty table. A read routine that formats for display is not a source of keys.

```abap
rv_out = iv_value.
SHIFT rv_out LEFT DELETING LEADING '0'.
CONDENSE rv_out.
```

## `fmt_amount`

Amounts in the USER's format, not in the internal format.

WRITE ... TO is the right statement for this - it respects the user settings for decimal and thousands separators. Assigning to a string does not.

**THE TRY IS NOT DECORATION**

The container holds a string. Whether it is a number, you cannot know: the element may be empty, or an earlier version wrote text into it. A conversion that fails would abort the DISPLAY of the work item - that is, exactly when someone is looking.

```abap
DATA lv_amount TYPE p LENGTH 13 DECIMALS 2.
DATA lv_out(30) TYPE c.

DATA(lv_raw) = condense( get_val( iv_element = iv_element
                                  iv_id      = iv_id ) ).

IF lv_raw IS INITIAL.
  RETURN.
ENDIF.

TRY.
    lv_amount = lv_raw.
  CATCH cx_sy_conversion_error.
    rv_out = lv_raw.                     " not convertible: show raw value
    RETURN.
ENDTRY.

WRITE lv_amount TO lv_out LEFT-JUSTIFIED.
rv_out = lv_out.
CONDENSE rv_out.
```

## `add_msg`

One line for the application log.

**WHY THE DETOUR VIA MESSAGE 00/398**

SLG1 only displays a line if ID and NUMBER are filled. Plain free text in the field MESSAGE disappears without a trace - no error, no line, nothing. You can spend hours looking for that.

00/398 is the standard message '&1&2&3&4' - four variables of 50 characters each. That fits 200 characters of free text, and SLG1 displays them.

**WHAT YOU SHOULD DO INSTEAD WHEN IT GETS SERIOUS**

A message class of your own with meaningful numbers. Then the messages can be translated and evaluated. 00/398 is the approach that needs no new object - good for getting started, not good for the long run.

**SO THAT IT ENDS UP IN SLG1 AT ALL**

Object and subobject for the workflow must be maintained in /C09/CFL_C08, AND both must be created in SLG0. If that is missing, conFLOW collects the messages and writes them nowhere.

```abap
CONSTANTS lc_var_len TYPE i VALUE 50.

DATA ls_return TYPE bapiret2.

DATA(lv_text) = iv_text.
DATA(lv_len)  = strlen( lv_text ).

ls_return-type       = iv_type.
ls_return-id         = '00'.
ls_return-number     = '398'.
ls_return-message    = lv_text.
ls_return-message_v1 = lv_text.

IF lv_len > lc_var_len.
  ls_return-message_v2 = lv_text+lc_var_len.
ENDIF.
IF lv_len > 100.
  ls_return-message_v3 = lv_text+100.
ENDIF.
IF lv_len > 150.
  ls_return-message_v4 = lv_text+150.
ENDIF.

APPEND ls_return TO ct_bapiret2.
```

## `set_priority`

Set the priority - in two stages, and both stages are needed.

**THE OBVIOUS APPROACH DOES NOT WORK**

SAP_WAPI_CHANGE_WORKITEM_PRIO reads SWWWIHEAD from the database. In the after-create hook the work item is not there yet. The call silently does nothing - no error, the priority stays at the default value. Measured.

**STAGE 1 - the work item manager of the running transaction**

CL_SWF_RUN_WIM_FACTORY knows the work items that are being created RIGHT NOW. The search goes by WI_ID, not by type: the hook means one specific work item, not just any.

**STAGE 2 - SWW_WI_PRIORITY_CHANGE with checks switched off**

The same function that sits under the WAPI - but with AUTHORIZATION_CHECKED and PRECONDITIONS_CHECKED set to 'X'. Exactly these checks are the blocker, because the work item does not yet have a status they would accept.

DO_COMMIT STAYS EMPTY. The COMMIT belongs to the framework - if you commit here yourself, you cut the running transaction in half.

**THE TRY BLOCKS ARE INTENTIONAL**

The hook runs in the middle of creating a work item. An uncaught exception because of a PRIORITY would be a remarkably high price to pay for a display detail.

```abap
DATA lt_instances TYPE swwtwihndl.
DATA ls_instances LIKE LINE OF lt_instances.
DATA lo_flow      TYPE REF TO if_swf_run_wim_internal.

TRY.
    DATA(lo_factory) = cl_swf_run_wim_factory=>get_instance( ).
    lt_instances = lo_factory->get_registered_workitems( ).

    LOOP AT lt_instances INTO ls_instances.
      TRY.
          lo_flow ?= ls_instances.
          IF lo_flow->m_sww_wihead-wi_id = iv_wi_id.
            lo_flow->if_swf_run_wim~change_priority( iv_prio ).
          ENDIF.
        CATCH cx_root.
      ENDTRY.
    ENDLOOP.

  CATCH cx_root.
ENDTRY.

CALL FUNCTION 'SWW_WI_PRIORITY_CHANGE'
  EXPORTING  wi_id                 = iv_wi_id
             priority              = iv_prio
             do_commit             = space
             authorization_checked = abap_true
             preconditions_checked = abap_true
  EXCEPTIONS no_authorization      = 1
             update_failed         = 2
             invalid_type          = 3
             invalid_status        = 4
             OTHERS                = 5.

IF sy-subrc <> 0.
  /c09/cfl_cl_workflow_0101=>ignore_subrc( ).
ENDIF.
```

## `update_witext`

Update the text of the running background work item.

The framework sets the text from Customizing BEFORE the background method runs. If you want to see a RESULT in the log, you have to change it yourself afterwards.

The difference in the workflow log:

```
without:  "Classification"         (the same text five times)
with:     "Approval required - 12.500,00 EUR exceeds ..."
```

**THE SEARCH GOES BY WI_TYPE = 'B'**

Unlike SET_PRIORITY, there is no WI_ID here - the background method does not know its own work item. 'B' is the background work item type, and during a background step exactly one of them is registered.

```abap
DATA lt_instances TYPE swwtwihndl.
DATA ls_instances LIKE LINE OF lt_instances.
DATA lo_flow      TYPE REF TO if_swf_run_wim_internal.
DATA lv_witext    TYPE sww_witext.

lv_witext = iv_text.

TRY.
    DATA(lo_factory) = cl_swf_run_wim_factory=>get_instance( ).
    lt_instances = lo_factory->get_registered_workitems( ).

    LOOP AT lt_instances INTO ls_instances.
      TRY.
          lo_flow ?= ls_instances.
          IF lo_flow->m_sww_wihead-wi_type = 'B'.
            lo_flow->if_swf_run_wim~change_witext( lv_witext ).
          ENDIF.
        CATCH cx_root.
      ENDTRY.
    ENDLOOP.

  CATCH cx_root.
ENDTRY.
```

## `is_sapgui`

GUI_IS_AVAILABLE is the standard approach: in the OData/RFC context of the Fiori inbox there is no front end, so ' ' comes back. The HTML approach to coloring needs exactly this distinction.

```abap
DATA lv_return TYPE c LENGTH 1.

CALL FUNCTION 'GUI_IS_AVAILABLE'
  IMPORTING
    return = lv_return.

rv_gui = xsdbool( lv_return = abap_true ).
```

## `gui_colour`

Produces exactly the form that runs in production:

```
    <span style="color:green;font-size:120%">Approve</span>
```

The size is in ZCL_CFL_CONST_00900=>MC_GUI_FONT_SIZE and may be empty; then only the color remains.

THE PROTECTION AGAINST A SECOND PASS is cheap and honest: the framework exit does rebuild ALTTEXT fresh from c09t right before, but a text wrapped twice would be an error you cannot see on the screen.

```abap
IF iv_text CS '<span'.
  rv_text = iv_text.
  RETURN.
ENDIF.

DATA(lv_size) = COND string(
  WHEN zcl_cfl_const_00900=>mc_gui_font_size IS INITIAL THEN ``
  ELSE |;font-size:{ zcl_cfl_const_00900=>mc_gui_font_size }| ).

rv_text = |<span style="color:{ iv_color }{ lv_size }">{ iv_text }</span>|.
```
