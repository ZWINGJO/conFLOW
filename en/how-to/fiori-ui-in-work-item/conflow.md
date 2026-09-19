# conFLOW: attaching the app to the work item

*One method in the BAdI and three constants. The conFLOW Customizing does not change.*

## The hook

[`get_after_creation_workitem`](../badi-reference/lifecycle/get-after-creation-workitem.md) runs once per work item, after it is created and before it is first displayed. This is exactly where the UI is attached.

```abap
  METHOD /c09/cfl_if_badi_0101~get_after_creation_workitem.
    " ... whatever else the workflow does here (priority etc.)

    set_inbox_ui( iv_wi_id = is_swr_wihdr-wi_id
                  iv_id    = is_data_step-id ).
  ENDMETHOD.
```

## `set_inbox_ui( )`

Three container elements are set **on the work item**. The third is the key the app uses to find its instance: the conFLOW instance as a 32-character hex string, so that it fits into the URL.

```abap
  METHOD set_inbox_ui.
    DATA lv_key TYPE c LENGTH 32.

    IF zcl_cfl_const_00500=>mc_inbox_ui = abap_false.      " kill switch
      RETURN.
    ENDIF.

    IF iv_wi_id IS INITIAL OR iv_id IS INITIAL.
      RETURN.
    ENDIF.

    lv_key = iv_id.                                        " RAW16 -> hex

    TRY.
        DATA(lo_container) = cl_swf_run_workitem_context=>get_instance(
                               im_wiid = iv_wi_id )->if_wapi_workitem_context~get_wi_container( ).

        lo_container->set( name  = zcl_cfl_const_00500=>mc_visu-semantic_object
                           value = zcl_cfl_const_00500=>mc_ui_semantic_object ).
        lo_container->set( name  = zcl_cfl_const_00500=>mc_visu-action
                           value = zcl_cfl_const_00500=>mc_ui_action ).
        lo_container->set( name  = zcl_cfl_const_00500=>mc_visu-query_obj00
                           value = lv_key ).

      CATCH cx_swf_ifs_exception.
        RETURN.
    ENDTRY.
  ENDMETHOD.
```

{% hint style="success" %}
**The silent `CATCH` is intentional.** If anything fails here, the work item keeps the standard text block. A UI that does not show up must never prevent a work item.
{% endhint %}

## The constants

```abap
    CONSTANTS mc_inbox_ui           TYPE abap_bool VALUE abap_true.
    CONSTANTS mc_ui_semantic_object TYPE string    VALUE 'ZCFLOrderPromiseV2' ##NO_TEXT.
    CONSTANTS mc_ui_action          TYPE string    VALUE 'openInInbox'        ##NO_TEXT.

    CONSTANTS:
      BEGIN OF mc_visu,
        semantic_object TYPE swfdname VALUE '/C09/CFL_VISU_SEMANTIC_OBJECT',
        action          TYPE swfdname VALUE '/C09/CFL_VISU_ACTION',
        query_obj00     TYPE swfdname VALUE '/C09/CFL_VISU_QUERY_OBJ00',
      END OF mc_visu.
```

{% hint style="danger" %}
**Spelling matters, character by character.** `mc_ui_semantic_object` must match the semantic object in the launchpad exactly, including upper and lower case. If a single letter differs, the intent does not resolve and the work item silently falls back to the text block.
{% endhint %}

The three `/C09/CFL_VISU_*` names are defined by conFLOW and are the same in every workflow. `/C09/CFL_VISU_QUERY_OBJ01` to `…05` are available for further parameters if an app needs more than the key.

## The decision is made at creation time

The values are stored in the work item container and stay there. As a consequence:

- **An existing work item does not switch its UI.** If you change `mc_inbox_ui` or the BAdI, you need a **new** work item to test.
- A different app for a specific step is an `IF` on `is_data_step-gen_stat` before the `set`.

## The simple way: parameter `VISU` in Customizing

You do not have to call `set_inbox_ui` yourself. conFLOW sets the three container elements on its own as soon as the parameter `VISU` is maintained on the workflow.

Maintain it in **c08 (general parameters)** per workflow definition:

| Parameter | Value |
| --- | --- |
| `VISU` | the semantic object of the app, e.g. `ZCFLOrderPromiseV2` |

From it the framework sets, at each work item's creation, `/C09/CFL_VISU_SEMANTIC_OBJECT` (from `VISU`), `/C09/CFL_VISU_ACTION` (`openInInbox`) and `/C09/CFL_VISU_QUERY_OBJ00` (the workflow instance). **Empty = as before**, the work item shows the text block. Not inherited via `WF_DEF` — maintain per definition. Fiori only.

{% hint style="info" %}
**Existing open work items** only get the elements at creation. The report `/C09/CFL_MIGRATE_VISU` retrofits them — simulation by default, it only writes once the flag is set, via `SAP_WAPI_WRITE_CONTAINER`.
{% endhint %}

The BAdI route above remains for **special cases**: a different app per step or a computed semantic object. It overrides `VISU`, because `get_after_creation_workitem` runs after the framework.

## What the app needs from the workflow

The app reads and writes **only through conFLOW**. It needs no tables of its own.

| Table | What the app uses it for |
| --- | --- |
| `/C09/CFL_S01` | header: does the instance belong to **this** workflow (`WF_DEFINITION`)? |
| `/C09/CFL_S03` | steps: which work item (`WI_ID`) is on which step (`GEN_STAT`) |
| `/C09/CFL_S04` | container: the values, read and written |

**The app fetches the work item text instead of rebuilding it.** The BAdI already builds the context block in `get_workitem_text`. Through `SAP_WAPI_WORKITEM_DESCRIPTION` the app receives exactly this block — the same one SAP GUI and the standard inbox show. If the text changes in the BAdI, the app follows without a single line of change.

**Writes go through the helpers of the workflow class**, for example by making the app's data class a `GLOBAL FRIEND`. Then there is only one way to compute texts and values.

{% hint style="danger" %}
**A container value is at most 132 characters long** (`/C09/CFL_S04-VALUE`). Anything longer is truncated without a message. Limit the note to 132 in the model, the input field and the server check.
{% endhint %}

## Optional: enforcing checks at decision time

The app **always** saves, even incomplete input — otherwise nobody could work. Whether the process may **continue** is checked by conFLOW after a button is clicked:

| Hook | Cancel | Message to the agent |
| --- | --- | --- |
| [`get_after_execution_mobile`](../badi-reference/lifecycle/get-after-execution-mobile.md) | `cv_subrc = 9` | yes, `cs_t100msg` |
| [`get_after_execution`](../badi-reference/lifecycle/get-after-execution.md) | `cv_subrc = 1` — **not 9**, otherwise the decision still goes through in SAP GUI | no |

Both receive `iv_altkey`, the outcome that was actually clicked. Both hooks should call the same check method, so that one condition applies on both paths.

{% hint style="info" %}
**The rule:** the app checks whether a **work state** may be saved. conFLOW checks whether the **process** may continue. A missing justification is a completion condition, not a save condition.
{% endhint %}
