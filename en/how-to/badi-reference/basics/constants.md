# The constants class

Constants class for reference workflow 00900.

It is the FIRST object you create when rebuilding the workflow - before the BAdI class. Reason: every key from customizing shows up again in the code, and each one is a two-character code with no meaning. `IF ls_s03-gen_stat = '02'` is not readable and cannot be found the next time customizing is reworked.

Create it later, and you have already typed the codes twenty times.

The class contains ONLY constants - no logic. That way the BAdI class, a conMOBILE model, the query provider of a custom Fiori app and reports can all use it without depending on each other.

## The whole class

```abap
CLASS zcl_cfl_const_00900 DEFINITION
  PUBLIC
  FINAL
  CREATE PUBLIC.

*----------------------------------------------------------------------*
* Constants class for reference workflow 00900.
*
* It is the FIRST object you create when rebuilding the workflow -
* before the BAdI class. Reason: every key from customizing shows up
* again in the code, and each one is a two-character code with no
* meaning. `IF ls_s03-gen_stat = '02'` is not readable and cannot be
* found the next time customizing is reworked.
*
* Create it later, and you have already typed the codes twenty times.
*
* The class contains ONLY constants - no logic. That way the BAdI
* class, a conMOBILE model, the query provider of a custom Fiori app
* and reports can all use it without depending on each other.
*----------------------------------------------------------------------*

  PUBLIC SECTION.

*--------------------------------------------------------------------*
* Identity - what defines this workflow
*
* MC_WF_DEFINITION must match the filter value of the BAdI
* implementation and the key in /C09/CFL_C06.
*
* MC_OBJECTTYPE is the BOR type under which the instance is kept.
* It does NOT have to be an object created in SWO1 - conFLOW treats
* TYPEID as a free key. A separate type per workflow (`ZCFL00900`) is
* therefore allowed and usually the better choice: it prevents two
* workflows on the same BOR type from seeing each other's instances.
* If you use the standard type (`BUS2012` = purchasing document), you
* inherit its behavior in EXECUTE_DEFAULT_METHOD and in the object
* display - in exchange, you share the space with everything else
* that runs on BUS2012.
*--------------------------------------------------------------------*
    CONSTANTS mc_wf_definition TYPE /c09/cfl_s01-wf_definition VALUE '00900' ##NO_TEXT.
    CONSTANTS mc_objecttype    TYPE swo_objtyp                 VALUE 'BUS2012' ##NO_TEXT.

*--------------------------------------------------------------------*
* Steps - the keys from /C09/CFL_C01
*
* Two conventions that conFLOW itself relies on:
*   - Background steps start with 'B'. The framework evaluates the
*     first character (see GET_STATUS_MAIL_DYNAMIC further below:
*     `is_data-gen_stat+0(1) = 'B'` tells dialog from batch).
*   - End steps usually start with 'X'. That is a convention, not a
*     check - but every conFLOW installation reads it that way.
*--------------------------------------------------------------------*
    CONSTANTS:
      BEGIN OF mc_stat,
        start        TYPE /c09/cfl_c01-gen_stat VALUE 'X0',   " Start
        classify     TYPE /c09/cfl_c01-gen_stat VALUE 'B1',   " Background: read and assess document
        approve      TYPE /c09/cfl_c01-gen_stat VALUE '01',   " Dialog: purchasing decides
        escalate     TYPE /c09/cfl_c01-gen_stat VALUE '02',   " Dialog: supervisor decides
        post         TYPE /c09/cfl_c01-gen_stat VALUE 'B2',   " Background: write back
        notify       TYPE /c09/cfl_c01-gen_stat VALUE 'B3',   " Background: notification
        end_ok       TYPE /c09/cfl_c01-gen_stat VALUE 'X1',   " End: approved
        end_rejected TYPE /c09/cfl_c01-gen_stat VALUE 'X2',   " End: rejected
        end_auto     TYPE /c09/cfl_c01-gen_stat VALUE 'X3',   " End: nothing to do
      END OF mc_stat.

*--------------------------------------------------------------------*
* Agent groups - the keys from /C09/CFL_C05
*
* GEN_STAT_USER is NOT the same as GEN_STAT. A step (GEN_STAT) can
* have several agent groups - that is the way to parallel work items.
* And the same agent group can be attached to several steps.
*
* GET_ACTORS receives the GEN_STAT_USER, not the GEN_STAT. If you
* branch on GEN_STAT in the hook, sooner or later you hit the case
* where it no longer fits.
*--------------------------------------------------------------------*
    CONSTANTS:
      BEGIN OF mc_gsu,
        buyer      TYPE /c09/cfl_c05-gen_stat_user VALUE '01',   " Buyer
        supervisor TYPE /c09/cfl_c05-gen_stat_user VALUE '02',   " Supervisor
        creator    TYPE /c09/cfl_c05-gen_stat_user VALUE '03',   " Creator of the document
        mail_info  TYPE /c09/cfl_c05-gen_stat_user VALUE 'M1',   " mail recipient only, no work item
      END OF mc_gsu.

*--------------------------------------------------------------------*
* Container attributes - the element names in /C09/CFL_S04
*
* Rule of thumb for the selection: the container holds what IDENTIFIES
* the document or feeds into the DECISION. Anything that is only
* displayed and is in the document is read from there.
*
* The reason is not storage space but truth: a container value is a
* copy and ages. If you store the vendor name along with it, you
* display it wrongly a year later. If you store the approved AMOUNT,
* you are right - because the decision was made on that amount, even
* if the document has changed since.
*--------------------------------------------------------------------*
    CONSTANTS:
      BEGIN OF mc_prop,
        " identifies the document
        doc_number  TYPE /c09/cfl_s04-element VALUE 'DOC_NUMBER' ##NO_TEXT,
        doc_item    TYPE /c09/cfl_s04-element VALUE 'DOC_ITEM' ##NO_TEXT,
        " feeds into the decision
        net_value   TYPE /c09/cfl_s04-element VALUE 'NET_VALUE' ##NO_TEXT,
        currency    TYPE /c09/cfl_s04-element VALUE 'CURRENCY' ##NO_TEXT,
        vendor      TYPE /c09/cfl_s04-element VALUE 'VENDOR' ##NO_TEXT,
        " calculated in B1 - the assessment, not the raw data
        severity    TYPE /c09/cfl_s04-element VALUE 'SEVERITY' ##NO_TEXT,
        limit_hit   TYPE /c09/cfl_s04-element VALUE 'LIMIT_HIT' ##NO_TEXT,
        " set by the agent
        decision_by TYPE /c09/cfl_s04-element VALUE 'DECISION_BY' ##NO_TEXT,
        note        TYPE /c09/cfl_s04-element VALUE 'NOTE' ##NO_TEXT,
      END OF mc_prop.

*--------------------------------------------------------------------*
* Business values
*--------------------------------------------------------------------*
    CONSTANTS:
      BEGIN OF mc_severity,
        green  TYPE string VALUE 'GREEN' ##NO_TEXT,
        yellow TYPE string VALUE 'YELLOW' ##NO_TEXT,
        red    TYPE string VALUE 'RED' ##NO_TEXT,
      END OF mc_severity.

*--------------------------------------------------------------------*
* The limit above which a human decides.
*
* It is a constant here because it should be easy to follow in a
* reference example. In a real installation, a value like this belongs
* in a rule: c09-BEDINGUNG on a background step, e.g.
* GESAMTWERT_RW > '10000.00' with c08 TEMPLATE = the template class
* for BUS2012 - changeable without development.
*--------------------------------------------------------------------*
    CONSTANTS mc_limit_value TYPE p LENGTH 8 DECIMALS 2 VALUE '10000.00'.

*--------------------------------------------------------------------*
* Work item priority (SWW_PRIO, 1-9, 1 = highest)
*
* Watch out for 1: SAP sends every agent an express message, a
* popup in SAP GUI. Only level 1 triggers that - 2 is the highest
* level without this effect. A fixed priority per step is c01-PRIO,
* without any code.
*--------------------------------------------------------------------*
    CONSTANTS:
      BEGIN OF mc_prio,
        high   TYPE sww_prio VALUE 4,
        medium TYPE sww_prio VALUE 5,
      END OF mc_prio.

*--------------------------------------------------------------------*
* Switches
*
* A reference example should show how to keep display details
* switchable. If a My Inbox version colors the buttons differently
* than expected, a switch is faster than a transport in a meeting.
*--------------------------------------------------------------------*
    CONSTANTS mc_fiori_nature TYPE abap_bool VALUE abap_true.

*--------------------------------------------------------------------*
* The same switch for the second coloring path - HTML in ALTTEXT,
* which the decision screen in the Business Workplace renders.
*
* Two switches instead of one, because the two UIs can cause trouble
* independently of each other: if something goes wrong in Fiori, the
* GUI should not notice, and vice versa.
*--------------------------------------------------------------------*
    CONSTANTS mc_gui_html_color TYPE abap_bool VALUE abap_true.

    CONSTANTS:
      BEGIN OF mc_gui_color,
        positive TYPE string VALUE 'green' ##NO_TEXT,
        negative TYPE string VALUE 'red' ##NO_TEXT,
      END OF mc_gui_color.

*--------------------------------------------------------------------*
* Font size for the same button. Color alone carries too little in
* the GUI - the bar is gray and the text is small. Size reinforces the
* same message and hits the same two buttons, so it does not open a
* second signal.
*
* Empty means "no size specified"; then only the color remains. This
* is the value to adjust when the button bar gets too wide.
*--------------------------------------------------------------------*
    CONSTANTS mc_gui_font_size TYPE string VALUE '120%' ##NO_TEXT.

ENDCLASS.


CLASS zcl_cfl_const_00900 IMPLEMENTATION.
ENDCLASS.
```
