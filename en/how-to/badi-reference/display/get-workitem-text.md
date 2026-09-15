# `get_workitem_text`

| | |
|---|---|
| **When** | On opening the work item. |
| **In** | IS_DATA           the instance |
| **Out** | CT_WORKITEM_TEXT  the text block, line by line |

**THE FORMAT IS SAPSCRIPT-ITF, NOT HTML**

The parameter type is called /C09/CFL_HTML_TABLE_TT, and the delivered sample implementation appends `<br>` to it. Both are misleading. What the work item viewer evaluates are SAPscript CHARACTER FORMATS:

```
<H>ORDER</>      correct - bold
<b>ORDER</b>     silently REMOVED
```

ITF reads `<b>` as a character format named 'b', does not know it, and deletes the brackets without comment. No bold, no visible tag, no error message - the most misleading outcome imaginable, because it looks like "HTML is not supported".

ALWAYS close with `</>`, never with `</H>`.

**A LAYOUT THAT HAS PROVEN ITSELF**

Finding first, then blocks with a bold heading. The agent should know after two lines what it is about, and only then read the details.

**WHAT DOES NOT BELONG HERE**

Calculations. Severity and recommendation belong in a BACKGROUND STEP and from there into the container. Only then does the audit trail show WHAT THE SYSTEM RECOMMENDED - and whether the agent deviated from it. If the display does the calculation itself, that information is gone as soon as the work item is closed.

## The code

```abap
    CLEAR ct_workitem_text.

    DATA(lv_id) = is_data-id.

*--------------------------------------------------------------------*
* Finding
*--------------------------------------------------------------------*
    APPEND |<H>{ get_val( iv_element = zcl_cfl_const_00900=>mc_prop-severity
                          iv_id      = lv_id ) }</> - | &&
           |{ fmt_amount( iv_element = zcl_cfl_const_00900=>mc_prop-net_value
                          iv_id      = lv_id ) } | &&
           |{ get_val( iv_element = zcl_cfl_const_00900=>mc_prop-currency
                       iv_id      = lv_id ) }|
      TO ct_workitem_text.

    APPEND space TO ct_workitem_text.

*--------------------------------------------------------------------*
* Block "Document"
*--------------------------------------------------------------------*
    APPEND '<H>DOCUMENT</>' TO ct_workitem_text.

    APPEND |Number   : { fmt_doc( get_val( iv_element = zcl_cfl_const_00900=>mc_prop-doc_number
                                           iv_id      = lv_id ) ) }|
      TO ct_workitem_text.

    APPEND |Item     : { get_val( iv_element = zcl_cfl_const_00900=>mc_prop-doc_item
                                  iv_id      = lv_id ) }|
      TO ct_workitem_text.

    APPEND |Vendor   : { fmt_doc( get_val( iv_element = zcl_cfl_const_00900=>mc_prop-vendor
                                           iv_id      = lv_id ) ) }|
      TO ct_workitem_text.

    APPEND space TO ct_workitem_text.

*--------------------------------------------------------------------*
* Block "Rule"
*
* Why the rule is on the work item and not just its result: the
* agent should be able to see why they are being asked. A work item
* that only says "please decide" generates follow-up questions.
*--------------------------------------------------------------------*
    APPEND '<H>RULE</>' TO ct_workitem_text.

    IF is_true( get_val( iv_element = zcl_cfl_const_00900=>mc_prop-limit_hit
                         iv_id      = lv_id ) ) = abap_true.
      APPEND |Value exceeds the approval limit of { zcl_cfl_const_00900=>mc_limit_value } - decision required.|
        TO ct_workitem_text.
    ELSE.
      APPEND 'Value within the approval limit.' TO ct_workitem_text.
    ENDIF.
```
