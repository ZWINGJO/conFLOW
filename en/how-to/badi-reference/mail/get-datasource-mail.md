# `get_datasource_mail`

| | |
|---|---|
| **When** | Before every mail dispatch, and also when the work item text is built (IV_WORKITEM_DESC = 'X'). |
| **In** | IS_DATA            the instance<br>IS_TEXT            the standard text currently being filled<br>IV_WORKITEM_DESC   'X' = this is about the work item text, not about a mail |
| **Out** | CT_APPLICATION_INPUT  the DATA STRUCTURES for the placeholders &STRUCTURE-FIELD&<br>CT_SO10_TEXT          entire text blocks for named placeholders such as &NOTE& and &WF_PROT& |

**THE PRINCIPLE: YOU SUPPLY DATA, NOT TEXT**

/C09/CFL_CL_HELPER_0101=>ADD_DATASOURCE_MAIL accepts any structure and turns it into placeholders - one per field, named TABLE-FIELD. If you pass EKKO, you can write &EKKO-LIFNR& in the standard text.

Which fields end up in the mail is therefore decided by the STANDARD TEXT, i.e. by the customer. No transport, no developer.

**THE PITFALL THAT CATCHES EVERYONE ONCE**

ADD_DATASOURCE_MAIL works via RTTI and needs a DDIC HEADER - it reads the table name to build the placeholder names from it. A LOCAL structure (TYPES BEGIN OF ...) has none. It is silently ignored: no placeholders, no error message, a mail with gaps.

If you want calculated or formatted values in the mail - an amount in user format, a document number without leading zeros, a composed text - you need your OWN DDIC STRUCTURE in the Data Dictionary for them. That is not a detour, that is how it is built.

**THE TWO NAMED TEXT BLOCKS**

```
&NOTE&     the notes that agents captured along the way -
           the conversation history
&WF_PROT&  the workflow log as an HTML table: who decided
           what and when
```

The product helper delivers both ready-made. Building them yourself is not worth it.

**WHAT THE FRAMEWORK ALREADY SUPPLIES - BEFORE THIS HOOK**

/C09/CFL_C06T (VTEXT, OBJTEXT, with language fallback) and, with c08 TEMPLATE, the document row as &/C09/CFL_S_TPL_`<type>`-`<field>`&, amounts formatted by currency. Here only what is missing from that.

**LIMIT: ONLY THE FIRST FIVE STRUCTURES**

The mail dispatch reads only five entries from

**CT_APPLICATION_INPUT - the framework's ones count too.**

## The code

```abap
*--------------------------------------------------------------------*
* 1. The document data - here as an entire DDIC structure.
*
* Deliberately WITHOUT preselection: passing EKKO in full costs one of
* the five slots, and the customer can use any field in the standard
* text without anyone touching the code.
*--------------------------------------------------------------------*
    DATA lv_ebeln TYPE ekko-ebeln.

    lv_ebeln = is_data-instid.

    SELECT SINGLE * FROM ekko INTO @DATA(ls_ekko)            "#EC CI_ALL_FIELDS_NEEDED
      WHERE ebeln = @lv_ebeln.
    IF sy-subrc = 0.
      /c09/cfl_cl_helper_0101=>add_datasource_mail(
        EXPORTING is_datastruc         = ls_ekko
        CHANGING  ct_application_input = ct_application_input ).
    ENDIF.

*--------------------------------------------------------------------*
* 3. The notes of the previous agents.
*
* The entry point is the work item, not the instance - hence read
* /C09/CFL_S03 first. GET_PROT_WORKITEM_MAIL then collects all notes
* of the ENTIRE workflow, not just those of the one step.
*--------------------------------------------------------------------*
    SELECT SINGLE * FROM /c09/cfl_s03 INTO @DATA(ls_cfl_s03) "#EC CI_ALL_FIELDS_NEEDED
      WHERE id = @is_data-id.                                 "#EC CI_NOORDER
    IF sy-subrc = 0.

      APPEND INITIAL LINE TO ct_so10_text ASSIGNING FIELD-SYMBOL(<fs_so10>).
      <fs_so10>-tdname = '&NOTE&'.
      <fs_so10>-tlines = /c09/cfl_cl_helper_0101=>get_prot_workitem_mail(
                           iv_workitem = ls_cfl_s03-wi_id
                           iv_rfcdest  = space ).

*--------------------------------------------------------------------*
* Only set the heading if there are any notes at all - otherwise
* "Notes:" sits above an empty block.
*--------------------------------------------------------------------*
      IF <fs_so10>-tlines IS NOT INITIAL.
        INSERT INITIAL LINE INTO <fs_so10>-tlines ASSIGNING FIELD-SYMBOL(<fs_line>) INDEX 1.
        <fs_line>-tdline = '<b><u>Notes:</u></b><br><br>' ##NO_TEXT.
      ENDIF.

    ENDIF.

*--------------------------------------------------------------------*
* 4. The workflow log as an HTML table.
*
* Unlike the work item text, the MAIL is real HTML - here <b> and
* <table> are correct. The risk of confusing it with the ITF format
* of the work item text is real: both look the same in the code.
*--------------------------------------------------------------------*
    APPEND INITIAL LINE TO ct_so10_text ASSIGNING <fs_so10>.
    <fs_so10>-tdname = '&WF_PROT&'.

    /c09/cfl_cl_helper_0101=>get_wf_prot(
      EXPORTING is_cfl_s01   = is_data
      IMPORTING et_prot_html = <fs_so10>-tlines ).
```
