# `get_description`

| | |
|---|---|
| **When** | On every display of the instance - hit list, header line, workflow log. |
| **In** | IS_DATA         the instance |
| **Out** | CV_DESCRIPTION  the line. CHAR100, HARD LIMIT. |

**TWO THINGS YOU NEED TO KNOW**

**1. THE TECHNICAL PREFIX**

   The framework passes CV_DESCRIPTION in already filled. The text is built in /C09/CFL_CL_WORKFLOW_0101 like this:

   ```abap
       CONCATENATE ms_data-wf_definition '|' ms_data-gen_stat '-'
                   ms_cfl_c01t-vtext INTO ev_description
                   SEPARATED BY space.
   ```

   Worked out, that gives

   ```
       00900 | 01 - <c01t-vtext>
       |<--- 13 --->|
   ```

   i.e. 5 (WF_DEFINITION) + 1 + 1 + 1 + 2 (GEN_STAT) + 1 + 1 + 1. That explains the line `cv_description = cv_description+13`, which looks like a random value without this paragraph.

   CAREFUL WHEN COMPARING WITH OLDER CODE: in practice you often find `+12`. That is not wrong, just imprecise - it leaves a leading space that nobody notices on screen. Take 13 and you can skip the CONDENSE.

   DO NOT CUT BLINDLY: the prefix is built from the instance and compared. If it does not match, the text stays as it is. If c06t-OBJTEXT is maintained, the framework already removes the prefix BEFORE this hook - a blind +13 would then cut off the first 13 characters of the real text.

**2. THE PLACEHOLDERS**

   The text after the prefix comes from /C09/CFL_C01T and can contain placeholders of the form §{name}. The customer maintains them in Customizing, this hook replaces them. That way the line changes without a transport. With c06t-OBJTEXT and c08 TEMPLATE the framework resolves §{field} from the template itself - only replace your own placeholders here.

**WHAT BELONGS AT THE FRONT**

The first screen column is the most expensive one. It is where the FINDING goes, not the document number - that is in the text after it anyway. A symbol at the very front (traffic light) turns the list into a work list you can scan at a glance.

## The code

```abap
    CONSTANTS lc_max_len TYPE i VALUE 100.

    DATA lv_text TYPE string.

    DATA(lv_prefix) = |{ is_data-wf_definition } \| { is_data-gen_stat } - |.
    DATA(lv_plen)   = strlen( lv_prefix ).

    IF strlen( cv_description ) > lv_plen AND
       cv_description(lv_plen) = lv_prefix.
      lv_text = cv_description+lv_plen.
    ELSE.
      lv_text = cv_description.
    ENDIF.

    REPLACE ALL OCCURRENCES OF '§{doc}' IN lv_text
      WITH fmt_doc( get_val( iv_element = zcl_cfl_const_00900=>mc_prop-doc_number
                             iv_id      = is_data-id ) ).

    REPLACE ALL OCCURRENCES OF '§{value}' IN lv_text
      WITH fmt_amount( iv_element = zcl_cfl_const_00900=>mc_prop-net_value
                       iv_id      = is_data-id ).

    REPLACE ALL OCCURRENCES OF '§{severity}' IN lv_text
      WITH get_val( iv_element = zcl_cfl_const_00900=>mc_prop-severity
                    iv_id      = is_data-id ).

*--------------------------------------------------------------------*
* Truncate to CHAR100 - and do it YOURSELF.
*
* Leave it to the assignment operator and you get a line that ends
* in the middle of a word. Three dots tell the reader that there was
* more.
*--------------------------------------------------------------------*
    CONDENSE lv_text.

    IF strlen( lv_text ) > lc_max_len.
      lv_text = |{ lv_text(97) }...|.
    ENDIF.

    cv_description = lv_text.
```
