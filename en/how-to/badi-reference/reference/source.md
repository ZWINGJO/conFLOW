# The whole class

For copying. Everything explained in the previous chapters is here
in one piece - with the same comments, because both come from the
same file.

> Before activating, replace `00900` with your own workflow number,
> and maintain `TEXT-001` in the text pool.

```abap
CLASS zcl_cfl_workflow_00900 DEFINITION
  PUBLIC
  FINAL
  CREATE PUBLIC.

*----------------------------------------------------------------------*
* REFERENCE IMPLEMENTATION /C09/CFL_IF_BADI_0101
*
* All 26 hooks of the conFLOW BAdI, each with the same header block:
* when it runs, what it receives, what it may change, and whether you
* need it at all.
*
* Six hooks are INTENTIONALLY EMPTY here. That is not a leftover that
* someone still has to fill in - it is the statement that a normal
* workflow does not need them. The header block says in each case what
* they would be for and how you can tell that your own case needs them.
*
* The example process is a purchase order approval:
*
*   X0  Start
*   B1  Background: read document, check amount against the limit
*   01  Dialog:     Purchasing decides
*   02  Dialog:     Manager decides (escalation)
*   B2  Background: write back to the document
*   B3  Background: notification
*   X1  End approved / X2 rejected / X3 nothing to do
*
* Deliberately built on an SAP standard document (purchasing document,
* BUS2012), so that every reader knows the data part and can focus on
* the workflow part.
*
* HOW TO COPY IT
*   1. Copy the class, replace 00900 with your own WF number
*      (three places: class name, constants class, BAdI filter)
*   2. Copy ZCL_CFL_CONST_00900 as well and adapt it to your own
*      customizing - it is the only place with keys
*   3. LEAVE hooks you do not need EMPTY. Do not delete them:
*      the interface requires them, and an empty body with a header
*      block is the documentation that the decision was made
*
* The filter of the BAdI implementation is WF_DEFINITION = '00900'.
* Without this filter the class runs for EVERY workflow in the system.
*----------------------------------------------------------------------*

  PUBLIC SECTION.

    INTERFACES /c09/cfl_if_badi_0101.
    INTERFACES if_badi_interface.

*--------------------------------------------------------------------*
* BACKGROUND STEP
*
* Not a BAdI hook, but the second contract conFLOW knows: a method
* that is entered in /C09/CFL_C01 as a background task
* (ATTRIBUT = 'BACK_BATCH', class and method name in customizing).
*
* The signature is FIXED and must look exactly like this:
*
*     IMPORTING is_cfl_s03      TYPE /c09/cfl_s03
*     EXPORTING et_bapiret2     TYPE bapiret2_t
*               ev_decision_key TYPE swr_decikey
*
* If it differs, conFLOW does not find the method at runtime - and
* without a syntax error, because the call is dynamic. The workflow
* then gets stuck in the background step.
*
* EV_DECISION_KEY is the outcome. It is evaluated just like in a
* dialog step, except that the code sets it here instead of a
* person. If it stays empty, the workflow does not continue.
*
* Why the example includes this method even though it is not part of
* the BAdI: the BAdI hooks do not DECIDE anything and do not DO
* anything, they display and steer. The work happens here. An example
* without a background step shows a workflow without content.
*--------------------------------------------------------------------*
    CLASS-METHODS background_classify
      IMPORTING is_cfl_s03      TYPE /c09/cfl_s03
      EXPORTING et_bapiret2     TYPE bapiret2_t
                ev_decision_key TYPE swr_decikey.

  PRIVATE SECTION.

*--------------------------------------------------------------------*
* Flag for GET_ACTORS.
*
* GET_NUMBER_ACTORS_REL runs BEFORE GET_ACTORS and is the only hook
* that learns in which context agent determination is currently
* taking place (IV_PROCESS = 'MAIL' for mail dispatch). If you want
* to distinguish between "create work item" and "send mail" in
* GET_ACTORS, you have to remember the value here - the signature of
* GET_ACTORS does not provide it.
*--------------------------------------------------------------------*
    DATA mv_process TYPE char10.

*--------------------------------------------------------------------*
* All helpers are CLASS METHODS.
*
* Not for reasons of style: the background step BACKGROUND_CLASSIFY
* is itself a class method (conFLOW requires that), and a class
* method cannot call an instance method. If the helpers were
* instance methods, the background step would have to build itself
* an instance of the BAdI class - which works, but nobody expects it.
*--------------------------------------------------------------------*

*--------------------------------------------------------------------*
* Container access
*
* /C09/CFL_CL_WORKFLOW_0101=>GET_ATTRIBUT_VALUE returns a TABLE -
* a container element can have several values. In the vast majority
* of cases you want the first and only one. GET_VAL wraps that so
* the caller does not have to write a loop every time.
*
* These three helpers are the reason why the hooks below are so
* short. Leave them out and you write the same four lines twenty times.
*--------------------------------------------------------------------*
    CLASS-METHODS get_val
      IMPORTING iv_element    TYPE /c09/cfl_s04-element
                iv_id         TYPE /c09/cfl_s01-id
      RETURNING VALUE(rv_val) TYPE string.

    CLASS-METHODS set_val
      IMPORTING iv_element TYPE /c09/cfl_s04-element
                iv_id      TYPE /c09/cfl_s01-id
                iv_value   TYPE string.

    CLASS-METHODS is_true
      IMPORTING iv_value      TYPE string
      RETURNING VALUE(rv_yes) TYPE abap_bool.

*--------------------------------------------------------------------*
* Document access and classification
*
* Kept separate because they are the only methods that know the
* document. If you adapt the class to a different document type, you
* change exactly this part - and nothing in the hooks.
*--------------------------------------------------------------------*
    CLASS-METHODS read_document
      IMPORTING iv_instid        TYPE /c09/cfl_s01-instid
      EXPORTING ev_net_value     TYPE ekpo-netwr
                ev_currency      TYPE ekko-waers
                ev_vendor        TYPE ekko-lifnr
                ev_created_by    TYPE ekko-ernam
                ev_found         TYPE abap_bool.

    CLASS-METHODS classify
      IMPORTING iv_net_value       TYPE ekpo-netwr
      RETURNING VALUE(rv_severity) TYPE string.

*--------------------------------------------------------------------*
* Display formatting
*
* Document numbers are stored in the database with leading zeros
* (0004500001234). On the work item that looks like a technical
* slip. Quantities and amounts, in turn, have the internal format in
* the database - an agent expects them in THEIR own notation.
*
* IMPORTANT: a read routine that formats for DISPLAY is NOT suitable
* as a key source for a SELECT. If you apply FMT_DOC( ) to a
* document number and search with it, you find nothing - and you
* get no error, but an empty table.
*--------------------------------------------------------------------*
    CLASS-METHODS fmt_doc
      IMPORTING iv_value      TYPE string
      RETURNING VALUE(rv_out) TYPE string.

    CLASS-METHODS fmt_amount
      IMPORTING iv_element    TYPE /c09/cfl_s04-element
                iv_id         TYPE /c09/cfl_s01-id
      RETURNING VALUE(rv_out) TYPE string.

*--------------------------------------------------------------------*
* Log
*
* A background step that does something must be able to say what.
* For this, conFLOW takes the BAPIRET2 table that every background
* method returns and writes it to the application log (SLG1) -
* provided that an object/subobject is maintained in /C09/CFL_C08
* and that it has been created in SLG0.
*
* PITFALL: SLG1 only shows a line if ID and NUMBER are filled.
* Pure free text in MESSAGE disappears without a trace - no error,
* no line. That is why the text goes through the generic message
* 00/398 ('&1&2&3&4') here, four variables of 50 characters each.
* A dedicated message class is cleaner and the next expansion
* step.
*--------------------------------------------------------------------*
    CLASS-METHODS add_msg
      IMPORTING iv_type      TYPE bapiret2-type DEFAULT 'S'
                iv_text      TYPE string
      CHANGING  ct_bapiret2  TYPE bapiret2_t.

*--------------------------------------------------------------------*
* Set the work item priority.
*
* This is a separate method because the obvious way does not
* work - see the comment in the implementation. The effort pays off:
* the priority is the only way to FILTER by the finding instead of
* just seeing it.
*--------------------------------------------------------------------*
    CLASS-METHODS set_priority
      IMPORTING iv_wi_id TYPE sww_wiid
                iv_prio  TYPE sww_prio.

*--------------------------------------------------------------------*
* The decision key that the system recommends.
*
* Needed in TWO places (GET_BEFORE_DECISION_WORKITEM for the
* SAP GUI, GET_FIORI_TASK_DEC_OP_ACT for the Fiori inbox), and the
* two tables are not the same. A shared method prevents the two
* user interfaces from highlighting different buttons.
*--------------------------------------------------------------------*
    CLASS-METHODS recommended_key
      IMPORTING iv_id         TYPE /c09/cfl_s01-id
      RETURNING VALUE(rv_key) TYPE swr_decikey.

*--------------------------------------------------------------------*
* Update the text of the running BACKGROUND work item.
*
* Without this, the workflow log shows the customizing text for every
* background step - so the same one for all five. With the update it
* shows what the step found out. That is the difference between a
* log and a list.
*--------------------------------------------------------------------*
    CLASS-METHODS update_witext
      IMPORTING iv_text TYPE string.

*--------------------------------------------------------------------*
* Is a SAP GUI attached to this session?
*
* Used in GET_BEFORE_DECISION_WORKITEM: the HTML route for coloring
* may only be taken if the text really ends up on a GUI screen.
* The Fiori inbox takes its button labels from the same hook - there
* the <span> would otherwise appear as text on the button.
*--------------------------------------------------------------------*
    CLASS-METHODS is_sapgui
      RETURNING VALUE(rv_gui) TYPE abap_bool.

*--------------------------------------------------------------------*
* Color a button text for the SAP GUI.
*
* The decision screen in the Business Workplace renders ALTTEXT as
* HTML, so the color is created there IN THE TEXT, not via a field.
*--------------------------------------------------------------------*
    CLASS-METHODS gui_colour
      IMPORTING iv_text        TYPE clike
                iv_color       TYPE clike
      RETURNING VALUE(rv_text) TYPE string.

ENDCLASS.


CLASS zcl_cfl_workflow_00900 IMPLEMENTATION.

*======================================================================*
*
*   GROUP 1 - START
*   Who gets the work item, and is one created at all?
*
*======================================================================*

  METHOD /c09/cfl_if_badi_0101~get_wi_create_swe2.
*--------------------------------------------------------------------*
* WHEN   On event-driven start via SWE2, before conFLOW creates the
*        instance. Only on this path - if you start the workflow via
*        START_WORKFLOW_INT( ) from your own code, it does not pass
*        through here.
*
* IN     IT_EVENT_CONTAINER_TAB  the event container
*        CS_CFL_C10              the type linkage that matched
*
* OUT    CS_SENDER  object type and instance key of the new instance
*
* PURPOSE  This is the VETO point. Clearing CS_SENDER means: no
*        workflow. This filters events that are raised but should
*        not trigger a process from a business point of view -
*        document type, plant, amount below the de minimis limit.
*
* WHY HERE AND NOT IN B1
*        A workflow that starts and ends itself in the first
*        background step leaves behind an instance, a log entry and
*        a line in every report. With ten documents that does not
*        matter, with ten thousand it does.
*        What is rejected here never existed.
*
*        The price: there is also no trace that a check took place.
*        If you have to prove WHY a document did not get a workflow,
*        you are better off filtering in B1 and ending there with a
*        log entry.
*--------------------------------------------------------------------*

    IF cs_sender-typeid <> zcl_cfl_const_00900=>mc_objecttype.
      RETURN.
    ENDIF.

    read_document( EXPORTING iv_instid    = CONV #( cs_sender-instid )
                   IMPORTING ev_net_value = DATA(lv_net_value)
                             ev_found     = DATA(lv_found) ).

*--------------------------------------------------------------------*
* Document not readable or trivial amount: no workflow.
*
* The first case is the more important one. An event can arrive for
* a document that does not exist (yet) - for example because the
* event was raised before the COMMIT. Without this check, instances
* are created for document numbers that nobody can find.
*--------------------------------------------------------------------*
    IF lv_found = abap_false.
      CLEAR cs_sender.
      RETURN.
    ENDIF.

    IF lv_net_value IS INITIAL.
      CLEAR cs_sender.
    ENDIF.

  ENDMETHOD.


  METHOD /c09/cfl_if_badi_0101~get_actors.
*--------------------------------------------------------------------*
* WHEN   On every work item creation, and a second time before every
*        mail dispatch. The most important hook of the whole interface.
*
* IN     IS_CFL_C05  the agent group - GEN_STAT_USER is the key
*                    the code branches on
*        IS_CFL_S01  the instance (optional, but practically always set)
*
* OUT    CT_ACTORS   the agents
*
* CHECK FIRST WHETHER YOU NEED IT
*        /C09/CFL_C03 holds OTYPE and OBJID directly per GEN_STAT_USER -
*        for example US/MEIER, US/WF-BATCH, US/WF_INITIATOR. Only
*        USER_BADI = 'X' switches to this hook. For PoCs, demos and
*        fixed assignments GET_ACTORS therefore stays EMPTY, and you
*        need neither a role nor code.
*
* THE FORMAT IS THE MOST COMMON MISTAKE
*        An entry in CT_ACTORS is always a TYPED org object, never a
*        bare user name:
*
*          US<uname>      user
*          S<position>    position
*          O<orgunit>     organizational unit
*          AC<role>       role
*
*        'MEIER' creates no work item and no error message. The work
*        item ends up with nobody and is only noticed when someone
*        asks where it went.
*
* WHEN NOBODY IS FOUND
*        A work item without an agent goes into error status and
*        stays there. A defined fallback agent is better: conFLOW
*        provides the entry 'C09_NO_USER' for this. The process keeps
*        running and the gap is visible instead of silently standing
*        still. See the very bottom of this method.
*--------------------------------------------------------------------*

    CASE is_cfl_c05-gen_stat_user.

*--------------------------------------------------------------------*
* VARIANT A - role
*
* The most common case. The customer maintains the role, and the code
* stays unchanged when people change.
*
* The call does NOT belong here but in a central class
* ZCL_CFL_GET_ACTORS. Reason: the same role is needed by several
* workflows, and TWP_GET_ROLE_USER_ASSIGNMENT with sorting and
* deduplication is the same block every time. The class is shown as
* a pattern in the chapter on this hook.
*--------------------------------------------------------------------*
      WHEN zcl_cfl_const_00900=>mc_gsu-buyer.
        ct_actors = zcl_cfl_get_actors=>buyer( ).

*--------------------------------------------------------------------*
* VARIANT B - agent from the container
*
* When an earlier step has determined who gets the next one: the
* creator of the document, the substitute, the person selected in
* step 01.
*
* Watch the prefix - GET_VAL returns the bare user name, 'US' has to
* go in front.
*--------------------------------------------------------------------*
      WHEN zcl_cfl_const_00900=>mc_gsu-creator.
        DATA(lv_uname) = get_val( iv_element = zcl_cfl_const_00900=>mc_prop-decision_by
                                  iv_id      = is_cfl_s01-id ).
        IF lv_uname IS NOT INITIAL.
          APPEND |US{ lv_uname }| TO ct_actors.
        ENDIF.

*--------------------------------------------------------------------*
* VARIANT C - organizational structure
*
* The manager, the cost center owner, the next approval level.
* RH_GET_ACTORS evaluates an SWO1 role (AC...) against the
* organizational structure.
*
* THE PITFALL: the result is usually POSITIONS or PERSONS
* (OTYPE 'S' or 'P'), not users. A work item sent to a person without
* a user assignment reaches nobody. Hence the detour via
* PTRV_CONVERT_PERNR_TO_USERID. If you leave it out, you have a
* workflow that runs in the test system (everything is assigned
* there) and gets stuck in the production system.
*
* The role number is client-dependent customizing and therefore does
* NOT belong in the code as a literal - it is only here so that
* the example is complete.
*--------------------------------------------------------------------*
      WHEN zcl_cfl_const_00900=>mc_gsu-supervisor.

        DATA lt_container TYPE TABLE OF swcont.
        DATA lt_swhactor  TYPE TABLE OF swhactor.

        APPEND INITIAL LINE TO lt_container ASSIGNING FIELD-SYMBOL(<fs_cont>).
        <fs_cont>-element = 'OBJECT'.
        <fs_cont>-value   = 'US'.

        CALL FUNCTION 'RH_GET_ACTORS'
          EXPORTING  act_object      = CONV rhobjects-object( 'AC00000168' )
                     search_date     = sy-datum
          TABLES     actor_container = lt_container
                     actor_tab       = lt_swhactor
          EXCEPTIONS OTHERS          = 1.
        IF sy-subrc <> 0.
          CLEAR lt_swhactor.
        ENDIF.

*--------------------------------------------------------------------*
* LV_USER_ID has to be declared BEFOREHAND. An inline declaration
* DATA(...) is not allowed on a classic CALL FUNCTION ... IMPORTING
* - the compiler rejects it. This affects all calls of this kind,
* and there are many of them in a conFLOW BAdI.
*--------------------------------------------------------------------*
        DATA lv_user_id TYPE usr02-bname.

        LOOP AT lt_swhactor ASSIGNING FIELD-SYMBOL(<fs_actor>).

          IF <fs_actor>-otype = 'P'.
            CLEAR lv_user_id.
            CALL FUNCTION 'PTRV_CONVERT_PERNR_TO_USERID'
              EXPORTING  personnel_number  = CONV p0001-pernr( <fs_actor>-objid )
              IMPORTING  user_id           = lv_user_id
              EXCEPTIONS user_id_not_found = 1
                         OTHERS            = 2.
            IF sy-subrc = 0.
              APPEND |US{ lv_user_id }| TO ct_actors.
            ENDIF.
          ELSE.
            APPEND |{ <fs_actor>-otype }{ <fs_actor>-objid }| TO ct_actors.
          ENDIF.

        ENDLOOP.

      WHEN OTHERS.
    ENDCASE.

*--------------------------------------------------------------------*
* FOLLOW UP 1 - handle mail dispatch differently from work item creation
*
* The same hook runs for both. For mail dispatch you often want a
* different group: not everyone who MAY decide also wants a mail.
* MV_PROCESS comes from GET_NUMBER_ACTORS_REL, which runs before.
*
* The flag is cleared afterwards - otherwise it carries over into the
* next call, and that one is no longer a mail.
*--------------------------------------------------------------------*
    IF mv_process = 'MAIL'.
      CLEAR mv_process.
      RETURN.
    ENDIF.

*--------------------------------------------------------------------*
* FOLLOW UP 2 - fallback agent
*
* Only for work items, not for mails: a mail to nobody is harmless,
* a work item to nobody stays stuck.
*--------------------------------------------------------------------*
    IF ct_actors IS INITIAL.
      APPEND 'C09_NO_USER' TO ct_actors.
    ENDIF.

  ENDMETHOD.


  METHOD /c09/cfl_if_badi_0101~get_number_actors_rel.
*--------------------------------------------------------------------*
* WHEN   Before GET_ACTORS, once per step.
*
* IN     IS_CFL_S01  the instance
*        IV_PROCESS  the context - 'MAIL' for mail dispatch
*
* OUT    CT_ACTORS   the AGENT GROUPS (/C09/CFL_C05_TT), not
*                    the agents. That is exactly the difference
*                    from GET_ACTORS.
*
* PURPOSE  Two things that are not possible anywhere else:
*
*        1. PARALLEL WORK ITEMS. If you turn one entry into several,
*           you get several work items on the same step. This is how
*           approval levels come about whose NUMBER is only known at
*           runtime - four signatures for this document, two for the
*           next one.
*
*        2. SETTING THE FLAG FOR GET_ACTORS. IV_PROCESS only exists
*           here. If you want to distinguish between work item and
*           mail in GET_ACTORS, you need this line.
*
* DO YOU NEED IT
*        Not in the normal case. One step, one agent group, any
*        number of people in it - GET_ACTORS handles that on its
*        own. This hook is for the case where the NUMBER OF STEPS
*        is variable.
*
* WHAT TO WATCH OUT FOR
*        The field WF_DEFINITION_OK in the entries is used as a pass
*        counter for parallel steps. That is a misuse of the field,
*        but it is the established solution - if you duplicate the
*        entries without this index, you get work items that cannot
*        be told apart.
*--------------------------------------------------------------------*

    mv_process = iv_process.

*--------------------------------------------------------------------*
* For mail dispatch, only include the info recipient if it is
* activated in customizing. That way the customer can switch a
* notification on and off without touching the code.
*--------------------------------------------------------------------*
    IF iv_process = 'MAIL'.

      SELECT SINGLE objid FROM /c09/cfl_c03 INTO @DATA(lv_objid)
        WHERE wf_definition = @is_cfl_s01-wf_definition
          AND gen_stat_user = @zcl_cfl_const_00900=>mc_gsu-mail_info
          AND objid         = @abap_true.

      IF sy-subrc <> 0.
        DELETE ct_actors WHERE gen_stat_user = zcl_cfl_const_00900=>mc_gsu-mail_info.
      ENDIF.

    ENDIF.

  ENDMETHOD.


*======================================================================*
*
*   GROUP 2 - DISPLAY
*   What the agent sees before deciding
*
*   Five hooks for four places on the screen. They are regularly mixed
*   up, so here is the map up front:
*
*     GET_DESCRIPTION        the ONE line in the result list
*                            (CHAR100 - no more is possible)
*     GET_WORKITEM_TEXT      the BLOCK you read after opening it
*     GET_OBJECT_INFO        the label of the object link in the
*                            FIORI inbox
*     GET_NEW_PREVIEW_DESCR  the same label in the SAP GUI
*     GET_WF_DEFINITION_TEXT the title of the overall workflow
*
*   GET_OBJECT_INFO and GET_NEW_PREVIEW_DESCR label THE SAME thing and
*   are still both needed: the Fiori inbox fills its "Links" tab from
*   SAP_WAPI_GET_OBJECTS and never sees GET_NEW_PREVIEW_DESCR. If you
*   maintain only one of the two, the other user interface shows the
*   framework class name.
*
*======================================================================*

  METHOD /c09/cfl_if_badi_0101~get_description.
*--------------------------------------------------------------------*
* WHEN   On every display of the instance - hit list, header line,
*        workflow log.
*
* IN     IS_DATA         the instance
* OUT    CV_DESCRIPTION  the line. CHAR100, HARD LIMIT.
*
* TWO THINGS YOU NEED TO KNOW
*
* 1. THE TECHNICAL PREFIX
*    The framework passes CV_DESCRIPTION in already filled. The text
*    is built in /C09/CFL_CL_WORKFLOW_0101 like this:
*
*        CONCATENATE ms_data-wf_definition '|' ms_data-gen_stat '-'
*                    ms_cfl_c01t-vtext INTO ev_description
*                    SEPARATED BY space.
*
*    Worked out, that gives
*
*        00900 | 01 - <c01t-vtext>
*        |<--- 13 --->|
*
*    i.e. 5 (WF_DEFINITION) + 1 + 1 + 1 + 2 (GEN_STAT) + 1 + 1 + 1.
*    That explains the line `cv_description = cv_description+13`,
*    which looks like a random value without this paragraph.
*
*    CAREFUL WHEN COMPARING WITH OLDER CODE: in practice you often
*    find `+12`. That is not wrong, just imprecise - it leaves a
*    leading space that nobody notices on screen. Take 13 and you
*    can skip the CONDENSE.
*
*    DO NOT CUT WITHOUT CHECKING: if the text is shorter than the
*    prefix, the offset runs past the end and the short dump comes
*    at display time - exactly when someone is watching.
*
* 2. THE PLACEHOLDERS
*    The text after the prefix comes from /C09/CFL_C01T and can
*    contain placeholders of the form §{name}. The customer maintains
*    them in Customizing, this hook replaces them. That way the line
*    changes without a transport.
*
* WHAT BELONGS AT THE FRONT
*    The first screen column is the most expensive one. It is where
*    the FINDING goes, not the document number - that is in the text
*    after it anyway. A symbol at the very front (traffic light)
*    turns the list into a work list you can scan at a glance.
*--------------------------------------------------------------------*

    CONSTANTS lc_prefix_len TYPE i VALUE 13.
    CONSTANTS lc_max_len    TYPE i VALUE 100.

    DATA lv_text TYPE string.

    IF strlen( cv_description ) > lc_prefix_len.
      lv_text = cv_description+lc_prefix_len.
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

  ENDMETHOD.


  METHOD /c09/cfl_if_badi_0101~get_workitem_text.
*--------------------------------------------------------------------*
* WHEN   On opening the work item.
*
* IN     IS_DATA           the instance
* OUT    CT_WORKITEM_TEXT  the text block, line by line
*
* THE FORMAT IS SAPSCRIPT-ITF, NOT HTML
*
*        The parameter type is called /C09/CFL_HTML_TABLE_TT, and the
*        delivered sample implementation appends <br> to it. Both are
*        misleading. What the work item viewer evaluates are
*        SAPscript CHARACTER FORMATS:
*
*            <H>ORDER</>      correct - bold
*            <b>ORDER</b>     silently REMOVED
*
*        ITF reads <b> as a character format named 'b', does not know
*        it, and deletes the brackets without comment. No bold, no
*        visible tag, no error message - the most misleading outcome
*        imaginable, because it looks like "HTML is not supported".
*
*        ALWAYS close with </>, never with </H>.
*
* A LAYOUT THAT HAS PROVEN ITSELF
*        Finding first, then blocks with a bold heading. The agent
*        should know after two lines what it is about, and only then
*        read the details.
*
* WHAT DOES NOT BELONG HERE
*        Calculations. Severity and recommendation belong in a
*        BACKGROUND STEP and from there into the container. Only then
*        does the audit trail show WHAT THE SYSTEM RECOMMENDED - and
*        whether the agent deviated from it. If the display does the
*        calculation itself, that information is gone as soon as the
*        work item is closed.
*--------------------------------------------------------------------*

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

  ENDMETHOD.


  METHOD /c09/cfl_if_badi_0101~get_object_info.
*--------------------------------------------------------------------*
* WHEN   The FIORI inbox builds its "Links" tab. The path there goes
*        through SAP_WAPI_GET_OBJECTS, not through
*        GET_NEW_PREVIEW_DESCR - which is why this hook works there and
*        the other one does not.
*
* IN     IS_LPOR    the object to be labeled
* OUT    CV_RETURN  the prefix of the label
*
* WHAT HAPPENS IF YOU LEAVE IT OUT
*        In the Fiori inbox, "Objects and attachments" shows the class
*        name of the framework:
*
*            CON: conFLOW Worflow V.0101: 0004500001234
*
*        With the hook:
*
*            Purchase order: 0004500001234
*
*        The framework appends the instance ID itself - CV_RETURN only
*        replaces the part in front of it.
*
* THE TEXT BELONGS IN A TEXT SYMBOL, NOT IN THE CODE
*        TEXT-001 belongs to the text pool of the class. That makes it
*        translatable, and the same label for both user interfaces
*        lives in exactly one place.
*
*        PITFALL WHEN TRANSPORTING: text symbols belong to the text
*        pool, not to the code. If you move the class via abapGit or a
*        transport and forget the text pool, the label is lost - and
*        you only notice in the inbox of the target system.
*
* THIS HOOK LOOKS DEAD IN THE WHERE-USED LIST
*        Where-used does not find it, because it is called through an
*        enhancement. It runs anyway. Rule of thumb for conFLOW BAdI
*        hooks in general: TRY IT OUT INSTEAD OF TRUSTING WHERE-USED.
*--------------------------------------------------------------------*

    cv_return = text-001.

  ENDMETHOD.


  METHOD /c09/cfl_if_badi_0101~get_new_preview_descr.
*--------------------------------------------------------------------*
* WHEN   The SAP GUI builds the object list of the work item
*        ("Objects and attachments").
*
* IN     IV_WI_ID            the work item
* OUT    CT_PREVIEW_OBJECTS  the object list, changeable
*
* PURPOSE   Same as GET_OBJECT_INFO, just for the other user interface.
*        Maintain both, otherwise one of the two looks ugly.
*
* THE CHECK ON OBJTYPE IS NOT OPTIONAL
*        The list has several entries: the conFLOW instance object,
*        notes (SOFM), attachments. If you loop over the table without
*        a CHECK, you label everything the same - including the notes,
*        which then lose their own name.
*
* DELETING IS ALLOWED TOO
*        The table is CHANGING. A DELETE removes an entry from the
*        display - the usual case is a technical note that is none of
*        the agent's business. The example below is commented out,
*        because it needs a key that only exists in your own
*        project.
*--------------------------------------------------------------------*

    LOOP AT ct_preview_objects ASSIGNING FIELD-SYMBOL(<fs_object>).
      CHECK <fs_object>-objtype = '/C09/CFL_CL_WORKFLOW_0101'.
      <fs_object>-descript = text-001.
    ENDLOOP.

*   DELETE ct_preview_objects
*     WHERE objtype    = 'SOFM'
*       AND def_attrib = zcl_cfl_const_00900=>mc_def_attrib_intern.

  ENDMETHOD.


  METHOD /c09/cfl_if_badi_0101~get_wf_definition_text.
*--------------------------------------------------------------------*
* WHEN   Building the title for the ENTIRE workflow - not for a
*        single step.
*
* IN     IS_DATA    the instance
* OUT    CV_WITEXT  the title (SWW_WITEXT)
*
* DO YOU NEED IT AT ALL
*        Usually not. The standard text comes from /C09/CFL_C06T and
*        can be maintained in Customizing - without a transport, with
*        translation. That is the better way.
*
* CASES WHERE YOU DO
*        When the title should contain document data that the
*        Customizing text does not know. "Purchase order approval" is
*        impossible to find in the workflow log next to twenty others,
*        "Purchase order approval 4500001234 / Example Corp." is not.
*
*        Stays EMPTY here, because the sample process gets by with the
*        Customizing text - and because a reference example should
*        show that you do not fill hooks just because they exist.
*--------------------------------------------------------------------*
  ENDMETHOD.


  METHOD /c09/cfl_if_badi_0101~default_attribute_value.
*--------------------------------------------------------------------*
* WHEN   The workflow framework asks for the default attribute of the
*        instance - when building object lists, attachments and
*        anywhere an object should "name itself".
*
* OUT    RESULT  a DATA REFERENCE to the value
*
* THE ONE LINE THAT IS THE SAME IN EVERY IMPLEMENTATION
*
*        Of all 26 hooks, this is the only one that was filled in
*        every production implementation examined - and every time
*        with exactly the same line. Copy it and you have it right.
*
* WHY GET REFERENCE OF AND NOT AN ASSIGNMENT
*        RESULT is REF TO DATA. The framework dereferences it later.
*        A local variable would be long gone by then - hence the
*        reference to the instance data of the framework, which lives
*        the whole time.
*
* PITFALL  If you return a reference to a METHOD-LOCAL variable, you
*        get no error, but garbled data later. Always reference
*        MS_INSTANCES-INSTANCE->MS_DATA.
*--------------------------------------------------------------------*

    GET REFERENCE OF /c09/cfl_cl_workflow_0101=>ms_instances-instance->ms_data-instid INTO result.

  ENDMETHOD.


  METHOD /c09/cfl_if_badi_0101~execute_default_method.
*--------------------------------------------------------------------*
* WHEN   Double-click on the object in the SAP GUI work item.
*
* IN     nothing - the instance is in
*        /C09/CFL_CL_WORKFLOW_0101=>MS_INSTANCES-INSTANCE->MS_DATA
*
* PURPOSE   "Show me the document". Without this hook, nothing happens
*        on double-click, and the agent has to copy down the document
*        number and call the transaction themselves.
*
* WITH THE DISPLAY TRANSACTION, NOT THE CHANGE TRANSACTION
*        ME23N, not ME22N. The agent should SEE the document while
*        deciding. If you let them change it here, you have a document
*        that moves underneath the running workflow - and an audit
*        trail that no longer matches.
*
* AND SKIP FIRST SCREEN
*        Skips the initial screen. For the Enjoy transactions (ME23N,
*        VA03) the addition has no effect or gets in the way - there
*        SET PARAMETER ID is enough.
*
* IF SEVERAL DOCUMENT TYPES RUN ON THE SAME CLASS
*        Branch on IS_DATA-TYPEID. With a separate BOR type per
*        workflow (the normal case) this is not needed.
*--------------------------------------------------------------------*

    DATA lv_ebeln TYPE ekko-ebeln.

    lv_ebeln = /c09/cfl_cl_workflow_0101=>ms_instances-instance->ms_data-instid.

    SET PARAMETER ID 'BES' FIELD lv_ebeln.
    CALL TRANSACTION 'ME23N'.

  ENDMETHOD.


*======================================================================*
*
*   GROUP 3 - THE WORK ITEM LIFECYCLE
*   Create, open, decide, complete
*
*   This group holds the three places where you can STOP the process.
*   They differ in what they can do, and that is the most important
*   difference in this whole group:
*
*     GET_BEFORE_DECISION_WORKITEM
*         can REMOVE buttons. No message, no abort.
*         "Not allowed" here means: the button is gone.
*
*     GET_AFTER_EXECUTION
*         can ABORT (CV_SUBRC), but WITHOUT a message. The agent only
*         sees that nothing happens - useless on its own.
*
*     GET_AFTER_EXECUTION_MOBILE
*         can ABORT AND SAY WHY (CS_T100MSG). The only hook with
*         both.
*
*   Rule of thumb: what the agent is NOT ALLOWED to do, you take away
*   beforehand. What they DID WRONG, you tell them afterwards.
*
*======================================================================*

  METHOD /c09/cfl_if_badi_0101~get_after_creation_workitem.
*--------------------------------------------------------------------*
* WHEN   After EVERY work item is created - including the
*        background steps - and before it is first displayed.
*
* IN     IS_DATA_STEP  the step (/C09/CFL_S03)
*        IS_SWR_WIHDR  the work item header, including WI_ID
*
* OUT    nothing. Effect only through side effects.
*
* PURPOSE   Everything that should happen ONCE PER WORK ITEM: priority,
*        attachments, notes, preparing the container, attaching a
*        custom user interface.
*
* THE HOOK ALSO RUNS FOR BACKGROUND STEPS
*        And that is almost always unwanted. A priority on a work item
*        that nobody sees only costs runtime. That is why the method
*        starts with a check that lets only the dialog steps through -
*        the line looks like a trifle and is not.
*
* THE WORK ITEM IS NOT YET ON THE DATABASE HERE
*        The central point of this hook, and the cause of the most
*        common disappointment: every API that READS SWWWIHEAD comes
*        back empty. SAP_WAPI_CHANGE_WORKITEM_PRIO does exactly that -
*        it reports no error, it just has no effect. The right way
*        goes through the work item manager of the running
*        transaction, see SET_PRIORITY( ).
*--------------------------------------------------------------------*

    IF is_data_step-gen_stat <> zcl_cfl_const_00900=>mc_stat-approve AND
       is_data_step-gen_stat <> zcl_cfl_const_00900=>mc_stat-escalate.
      RETURN.
    ENDIF.

    IF is_swr_wihdr-wi_id IS INITIAL.
      RETURN.
    ENDIF.

*--------------------------------------------------------------------*
* The escalation is always urgent - that is exactly why it ended up
* with the manager. Otherwise the severity calculated in B1
* decides.
*--------------------------------------------------------------------*
    DATA lv_prio TYPE sww_prio.

    IF is_data_step-gen_stat = zcl_cfl_const_00900=>mc_stat-escalate.
      lv_prio = zcl_cfl_const_00900=>mc_prio-high.

    ELSEIF get_val( iv_element = zcl_cfl_const_00900=>mc_prop-severity
                    iv_id      = is_data_step-id ) = zcl_cfl_const_00900=>mc_severity-red.
      lv_prio = zcl_cfl_const_00900=>mc_prio-high.

    ELSE.
      lv_prio = zcl_cfl_const_00900=>mc_prio-medium.
    ENDIF.

    set_priority( iv_wi_id = is_swr_wihdr-wi_id
                  iv_prio  = lv_prio ).

  ENDMETHOD.


  METHOD /c09/cfl_if_badi_0101~get_before_execution_workitem.
*--------------------------------------------------------------------*
* WHEN   Immediately before a work item is executed - i.e. after the
*        double-click, before the user interface is built.
*
* IN     IS_SWR_WIHDR  the work item header
* OUT    nothing.
*
* PURPOSE   Preparations that are needed exactly when someone actually
*        opens the work item - setting locks, filling a cache,
*        incrementing counters.
*
* NOT FILLED IN ANY OF THE PRODUCTION IMPLEMENTATIONS EXAMINED.
*
*        The reason: it cannot prevent anything. There is no return
*        parameter and no way to abort - whatever happens here happens
*        on the side. If you are looking for a check, go to
*        GET_BEFORE_DECISION_WORKITEM (remove the button) or
*        GET_AFTER_EXECUTION_MOBILE (abort with a message).
*
*        STAYS EMPTY. That is the decision, not a leftover.
*--------------------------------------------------------------------*
  ENDMETHOD.


  METHOD /c09/cfl_if_badi_0101~get_before_decision_workitem.
*--------------------------------------------------------------------*
* WHEN   The work item exit builds the decision options - i.e.
*        before the buttons are drawn.
*
* IN/OUT     CM_WORKITEM_CONTEXT - the work item context. You get the
*            header and the options from it, and write the changed
*            ones back into it.
*
* THREE THINGS WORK HERE, AND ONLY HERE
*
* 1. REMOVE A BUTTON (the "guard")
*    If an action is not allowed from a business point of view, it
*    disappears - instead of being rejected afterwards. That is the
*    friendlier design: the agent only sees what they may do.
*
*    The hook CANNOT force an error message. "Not allowed" here means
*    button gone, not an error afterwards.
*
* 2. COLOR A BUTTON - SAP GUI, VIA HTML IN ALTTEXT
*    The decision screen in the Business Workplace renders ALTTEXT
*    as HTML. So the color is created INSIDE THE TEXT:
*
*        <span style="color:green;font-size:120%">Approve</span>
*
*    Color and size, no font-weight - both hit the same buttons, and a
*    third attribute would not add a third signal.
*    ALTTEXT is CHAR255, the wrapper costs about 50 characters; the
*    c09t texts fit with room to spare.
*
*    GUI GUARD ONLY. The Fiori inbox takes its button texts from
*    exactly this hook - without the guard, the <span> would show up
*    there as the label on the button. So ask GUI_IS_AVAILABLE and
*    take the HTML route only when there is a real GUI.
*
* 3. COLOR THE SAME BUTTON - FIORI, VIA ALTNATURE
*    SWR_DECIALTS has the field ALTNATURE. It works through the
*    workflow DEFINITION and is NOT ENOUGH ON ITS OWN: the color the
*    Fiori inbox actually draws comes from NATURE in
*    GET_FIORI_TASK_DEC_OP_ACT. Setting it here does no harm and it
*    stays as a second track - but if you write only this line, you
*    see no color in Fiori and look for it in the wrong hook.
*
*    There are EXACTLY TWO values - POSITIVE and NEGATIVE. That is not
*    a palette but a statement, and it is handed out sparingly:
*
*        POSITIVE   the calculated recommendation
*        NEGATIVE   the one action that discards something for good
*        neutral    everything else
*
*    If the system itself recommends the hard action for once, the
*    recommendation wins. Two signals on the same button would be
*    none.
*
* WHY THE DECISION IS STILL MADE ONLY ONCE
*        Two user interfaces, two MECHANISMS - the GUI colors via HTML
*        in the text, Fiori via NATURE from the task gateway, and the
*        two hooks work on different tables. If you maintain only one
*        route, one of the two user interfaces has no color.
*
*        So you answer the question "which button is positive, which
*        negative" ONCE - below in the loop over LV_ROLE - and serve
*        both routes from it. Write the rule down twice and the two
*        drift apart at the next additional outcome, and nobody
*        notices, because hardly anyone opens both user interfaces
*        side by side.
*
*        Confirmed at runtime in two systems.
*
* STARTING FROM THE WI_ID IS MANDATORY
*        The hook does NOT receive the conFLOW instance. The only way
*        to it goes through the work item header and /C09/CFL_S03.
*--------------------------------------------------------------------*

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

  ENDMETHOD.


  METHOD /c09/cfl_if_badi_0101~get_fiori_task_dec_op_act.
*--------------------------------------------------------------------*
* WHEN   The Fiori inbox fetches its decision options. The path goes
*        through the task gateway handler, not through the work item
*        exit - which is why GET_BEFORE_DECISION_WORKITEM alone is not
*        enough.
*
* IN     IV_INSTANCE_ID  the WI_ID
* OUT    CT_DEC_OPT      the options of the inbox
*
* THIS HOOK LOOKS DEAD IN THE WHERE-USED LIST - AND STILL RUNS
*        /C09/CL_TGW_RFC_HANDLER is not a class of its own, but a
*        conFLOW ENHANCEMENT on the task gateway handler. That is why
*        where-used finds nothing. The hook runs anyway, and it is the
*        same place where the button texts from /C09/CFL_C09T are
*        set.
*
* MATCH ON THE KEY, NEVER ON THE TEXT
*        The obvious choice would be a comparison on DECISION_TEXT. It
*        cannot work: at this point it already holds the TRANSLATED
*        text from /C09/CFL_C09T, no longer the raw key. DECISION_KEY,
*        on the other hand, is NUMC4 with the same numbering as
*        SWR_DECIKEY (0001 = OK, 0002 = NOK, 0003 = UC1 ...) and is
*        therefore stable.
*
* DO NOT TOUCH DECISION_TEXT
*        The framework checks right AFTER this call whether the text
*        still contains a '-', and otherwise skips the C09T
*        translation. If you write to the text here, you end up with
*        UNLABELED buttons - and look for the error in the wrong
*        place.
*--------------------------------------------------------------------*

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

  ENDMETHOD.


  METHOD /c09/cfl_if_badi_0101~get_after_execution.
*--------------------------------------------------------------------*
* WHEN   After the agent has decided in the SAP GUI, before the work
*        item is completed.
*
* IN     IV_WI_ID     the work item
*        IV_ALTKEY    the CLICKED outcome - the value that matters
*        IV_ALT_TEXT  its text
*        IV_MSELNOTE  default for the note dialog
*
* OUT    CV_SUBRC      <> 0 aborts
*        CS_OBJECT_ID  reference to a captured note
*
* TWO DIFFERENT TASKS THAT COINCIDE HERE
*
* 1. FOLLOW-UP PROCESSING - update the container, record who
*    decided. This is the usual case.
*
* 2. FORCING A NOTE - via SWU_INTERN_DECI_NOTE_POPUP. This is the
*    standard way for "please justify the rejection".
*
* THE ABORT CANNOT SAY WHY
*        CV_SUBRC <> 0 stops the process, but there is no message
*        parameter. The agent clicks and nothing happens - the worst
*        feedback there is. If you need a check WITH a reason, call
*        the same check additionally in GET_AFTER_EXECUTION_MOBILE,
*        the only hook with CS_T100MSG.
*
* FIRST LINE: CHECK IV_ALTKEY
*        The hook also runs for actions that are not a decision
*        (forward, postpone). Then IV_ALTKEY is empty, and any logic
*        that assumes an outcome reaches into nothing.
*--------------------------------------------------------------------*

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
*--------------------------------------------------------------------*
    IF iv_altkey = /c09/cfl_cl_workflow_0101=>mc_decision-nok.

      CALL FUNCTION 'SWU_INTERN_DECI_NOTE_POPUP'
        EXPORTING  wi_id          = iv_wi_id
                   alt_text       = iv_alt_text
                   mselnote       = iv_mselnote
        IMPORTING  ex_object_id   = cs_object_id
        EXCEPTIONS user_cancelled = 1
                   OTHERS         = 2.

      IF sy-subrc <> 0.
        cv_subrc = sy-subrc.
        RETURN.
      ENDIF.

    ENDIF.

  ENDMETHOD.


  METHOD /c09/cfl_if_badi_0101~get_after_execution_mobile.
*--------------------------------------------------------------------*
* WHEN   After execution from conMOBILE or from the BSP path.
*
* IN     IV_WI_ID   the work item
*        IV_ALTKEY  the clicked outcome
*
* OUT    CV_SUBRC     <> 0 aborts. 9 is the usual value.
*        CS_T100MSG   the matching message
*
* THE ONLY HOOK THAT CAN ABORT AND SAY WHY.
*
*        That makes it the most important gate in the whole
*        interface - more important than its name suggests.
*
* THE NAME IS MISLEADING
*        "MOBILE" suggests it only runs for conMOBILE. According to
*        the documentation it is tied to the conMOBILE/BSP path;
*        whether a particular Fiori inbox calls it must be CHECKED in
*        your own system, not assumed. GET_FIORI_TASK_DEC_OP_ACT
*        looked just as dead, and the hook ran.
*
*        If it does not run in your own UI, the gate has no effect
*        there. The only option left: put the same check additionally
*        into a BACKGROUND STEP after the decision - that always runs.
*
* FREE TEXT AS A T100 MESSAGE
*        CS_T100MSG wants an ID and a number, not a string. 00/398 is
*        '&1&2&3&4' - four variables of 50 characters each. That fits
*        200 characters of free text. A message class of your own is
*        cleaner, but this way needs no new object.
*--------------------------------------------------------------------*

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

  ENDMETHOD.


  METHOD /c09/cfl_if_badi_0101~get_after_execution_workitem.
*--------------------------------------------------------------------*
* WHEN   After a work item has been completed - regardless of which
*        UI it came from.
*
* IN     IS_DATA_STEP  the step
*        IS_SWR_WIHDR  the work item header
*        IV_KEY        the outcome
*
* OUT    nothing, and NO ABORT EITHER. Whatever happens here happens
*        after the decision.
*
* THE CLASSIC USE CASE: CLEANING UP PARALLEL WORK ITEMS
*
*        When a step has several agents in parallel and one of them
*        rejects, the other work items should disappear - otherwise
*        people work on a case that has already been decided.
*
*        /C09/CFL_CL_HELPER_0101=>SET_WORKITEM_OBSOLET does that: it
*        finds all open work items of the same top-level workflow
*        and sets them to obsolete - except its own.
*
* WHY THE COMMIT IS HERE
*        SET_WORKITEM_OBSOLET calls SAP_WAPI_WORKITEM_COMPLETE with
*        DO_COMMIT = FALSE, so that not every work item is committed
*        separately. The COMMIT therefore has to come from the
*        caller. Without this line the work items stay open - with no
*        error message.
*
* DIFFERENCE FROM GET_AFTER_EXECUTION
*        GET_AFTER_EXECUTION runs BEFORE completion and can prevent
*        it. This one runs AFTERWARDS. If you want to check, use the
*        other one.
*--------------------------------------------------------------------*

    IF iv_key <> /c09/cfl_cl_workflow_0101=>mc_decision-nok.
      RETURN.
    ENDIF.

    /c09/cfl_cl_helper_0101=>set_workitem_obsolet( is_data_step = is_data_step ).

    COMMIT WORK AND WAIT.

  ENDMETHOD.


  METHOD /c09/cfl_if_badi_0101~get_event_raised_workitem.
*--------------------------------------------------------------------*
* WHEN   When an event hits a RUNNING work item - not at workflow
*        start, but while it runs.
*
* IN     IM_EVENT_NAME - which event
* IN/OUT     CM_WORKITEM_CONTEXT - the context of the affected work item
*
* PURPOSE  Reacting to changes to the document WHILE the workflow is
*        open. The document is cancelled while someone is deciding on
*        it - then the work item should disappear instead of staying
*        in the inbox.
*
* NOT FILLED IN ANY OF THE PRODUCTION IMPLEMENTATIONS EXAMINED.
*
*        Not because it is useless, but because the case is rare and
*        the alternative is closer at hand: the change is checked at
*        the next step instead of taking effect immediately.
*
*        You know you need it when there is a requirement of the form
*        "if X happens WHILE the workflow is running, then ...".
*        Without that "while" it is an ordinary background step.
*
*        STAYS EMPTY.
*--------------------------------------------------------------------*
  ENDMETHOD.


*======================================================================*
*
*   GROUP 4 - THE SWITCH
*   Where to go next when the customizing does not know
*
*======================================================================*

  METHOD /c09/cfl_if_badi_0101~get_status_dynamic.
*--------------------------------------------------------------------*
* WHEN   On every status change, after conFLOW has determined the next
*        status from /C09/CFL_C02 - and before it uses it.
*
* IN/OUT     CS_DATA - the instance WITH the intended next status.
*            Overwriting CS_DATA-GEN_STAT here overrides the
*            customizing. CS_DATA_OLD holds the state before.
*
* PURPOSE  Branches whose target is only known at runtime: approval
*        levels by amount, skipping a level when it does not apply
*        for business reasons, returning to the point from which the
*        item was forwarded.
*
* THE RELATIONSHIP TO /C09/CFL_C02
*        C02 says "after step 01 with outcome OK comes step B2". This
*        hook says "unless ...". If you use it, you have a process
*        flow that is NO LONGER VISIBLE in the customizing - that is
*        the price, and it is high.
*
*        HENCE THE ORDER OF QUESTIONS:
*        1. Does an additional status in C02 do the job?
*        2. Does a background step that branches via
*           EV_DECISION_KEY do the job? (That is the clean way - the
*           branch stays visible in C02.)
*        3. Only then this hook.
*
*        The sample process gets by with 2 - see
*        BACKGROUND_CLASSIFY( ) at the very bottom, which does exactly
*        that.
*
* WHAT TO KEEP IN MIND
*        The hook runs on EVERY status change, including those that
*        are none of your business. Without a switch at the start you
*        rewrite cases you never meant to touch.
*
*        STAYS EMPTY HERE - on purpose, see above. The pattern is
*        commented out below, so you have it when you need it.
*--------------------------------------------------------------------*

*   " Example: an amount limit decides whether to escalate
*   IF cs_data-gen_stat <> zcl_cfl_const_00900=>mc_stat-approve.
*     RETURN.
*   ENDIF.
*
*   IF is_true( get_val( iv_element = zcl_cfl_const_00900=>mc_prop-limit_hit
*                        iv_id      = cs_data-id ) ) = abap_true.
*     cs_data-gen_stat = zcl_cfl_const_00900=>mc_stat-escalate.
*   ELSE.
*     cs_data-gen_stat = zcl_cfl_const_00900=>mc_stat-post.
*   ENDIF.

  ENDMETHOD.


*======================================================================*
*
*   GROUP 5 - MAIL
*   Who gets a notification, in which language, with which
*   content
*
*   conFLOW sends mails from an SO10 standard text. It contains
*   placeholders of the form &STRUCTURE-FIELD&. GET_DATASOURCE_MAIL
*   fills them by passing entire STRUCTURES - not individual values.
*
*   This is the core and the cause of most misunderstandings: you
*   supply data, not text. Which of it ends up in the mail is decided
*   by the standard text - that is, by the customer, without a transport.
*
*======================================================================*

  METHOD /c09/cfl_if_badi_0101~get_send_mail_user.
*--------------------------------------------------------------------*
* WHEN   Before mail dispatch, once the recipient group is fixed.
*
* IN     IS_DATA   the instance
* OUT    CT_USER   the recipients as users
*        CT_MAIL   the recipients as mail addresses
*
* PURPOSE  Add or remove recipients that agent determination cannot
*        supply: a distribution list address, an external contact, a
*        shared mailbox.
*
* NOT FILLED IN ANY OF THE PRODUCTION IMPLEMENTATIONS EXAMINED.
*
*        For a good reason: the recipient group comes from
*        /C09/CFL_C05 and /C09/CFL_C03, i.e. from the customizing. If
*        you add to it in code, you have a recipient that nobody
*        reading the configuration will find - and who stays when
*        the person leaves the company.
*
*        The clean way for "one address should always be included" is
*        a separate GEN_STAT_USER in C05, set to the address in the
*        customizing. Then it is where people look for it.
*
*        STAYS EMPTY.
*--------------------------------------------------------------------*
  ENDMETHOD.


  METHOD /c09/cfl_if_badi_0101~get_mail_language.
*--------------------------------------------------------------------*
* WHEN   Without wave 1: never. The hook is in the interface, but the
*        framework does not call it (checked 2026-09-11). With wave
*        1: per recipient, right before dispatch.
*
* IN     IT_USER / IT_MAIL  the one recipient of this dispatch
* OUT    CS_DATA-WI_LANG    the language for this recipient
*
* WITHOUT WAVE 1 EVERY MAIL GOES OUT IN THE SENDER'S LANGUAGE
*        That is, in the logon language of whoever triggers the
*        dispatch - dialog user or WF-BATCH -, not in the language of
*        the instance (S01-WI_LANG). That is why the hook is not
*        filled in any of the production implementations examined: it
*        had no effect.
*
* WITH WAVE 1 WI_LANG ARRIVES WITH THE SENDER'S LANGUAGE
*        Not with that of the instance - if you want that one, read
*        it via CS_DATA-ID from /C09/CFL_S01. If the hook leaves
*        WI_LANG as it is, nothing changes. The language is only
*        switched if it is installed and the subject (SO10) exists
*        in it.
*
* WHEN YOU NEED IT
*        When recipients should be addressed in their own language,
*        typically the one from the user master (USR01-LANGU for
*        IT_USER).
*
*        STAYS EMPTY HERE: the reference does not require wave 1.
*--------------------------------------------------------------------*
  ENDMETHOD.


  METHOD /c09/cfl_if_badi_0101~get_status_mail_dynamic.
*--------------------------------------------------------------------*
* WHEN   During mail dispatch, once it is fixed WHICH agent group is
*        notified.
*
* IN     IS_DATA - the instance
* IN/OUT     CS_CFL_C05 (the intended group) and CT_CFL_C05 (the
*            list of groups) - here you can turn one into several
*            or replace it
*
* PURPOSE  The counterpart to GET_STATUS_DYNAMIC, but for mail
*        dispatch: "Who gets a mail depends on what has just
*        happened."
*
* THE CHECK FOR BACKGROUND STEPS IS THE TRICK
*        `is_data-gen_stat+0(1) = 'B'` tells dialog from batch - that
*        is the reason for the naming convention that background
*        steps start with B. Without this line you send
*        notifications about steps that nobody has seen.
*
*        STAYS EMPTY HERE, because the sample process has a fixed
*        recipient group per step. The pattern is below.
*--------------------------------------------------------------------*

*   " Example: on escalation, also inform the original
*   " agent
*   IF is_data-gen_stat+0(1) = 'B'.
*     RETURN.                      " background step - no mail
*   ENDIF.
*
*   IF is_data-gen_stat <> zcl_cfl_const_00900=>mc_stat-escalate.
*     RETURN.
*   ENDIF.
*
*   CLEAR ct_cfl_c05.
*   APPEND cs_cfl_c05 TO ct_cfl_c05.
*   APPEND INITIAL LINE TO ct_cfl_c05 ASSIGNING FIELD-SYMBOL(<fs_c05>).
*   <fs_c05>               = cs_cfl_c05.
*   <fs_c05>-gen_stat_user = zcl_cfl_const_00900=>mc_gsu-buyer.

  ENDMETHOD.


  METHOD /c09/cfl_if_badi_0101~get_datasource_mail.
*--------------------------------------------------------------------*
* WHEN   Before every mail dispatch, and also when the work item text
*        is built (IV_WORKITEM_DESC = 'X').
*
* IN     IS_DATA            the instance
*        IS_TEXT            the standard text currently being filled
*        IV_WORKITEM_DESC   'X' = this is about the work item text, not
*                           about a mail
*
* OUT    CT_APPLICATION_INPUT  the DATA STRUCTURES for the
*                              placeholders &STRUCTURE-FIELD&
*        CT_SO10_TEXT          entire text blocks for named
*                              placeholders such as &NOTE& and &WF_PROT&
*
* THE PRINCIPLE: YOU SUPPLY DATA, NOT TEXT
*
*        /C09/CFL_CL_HELPER_0101=>ADD_DATASOURCE_MAIL accepts any
*        structure and turns it into placeholders - one per field,
*        named TABLE-FIELD. If you pass EKKO, you can write
*        &EKKO-LIFNR& in the standard text.
*
*        Which fields end up in the mail is therefore decided by the
*        STANDARD TEXT, i.e. by the customer. No transport, no
*        developer.
*
* THE PITFALL THAT CATCHES EVERYONE ONCE
*
*        ADD_DATASOURCE_MAIL works via RTTI and needs a DDIC HEADER -
*        it reads the table name to build the placeholder names from
*        it. A LOCAL structure (TYPES BEGIN OF ...) has none. It is
*        silently ignored: no placeholders, no error message, a mail
*        with gaps.
*
*        If you want calculated or formatted values in the mail
*        - an amount in user format, a document number without
*        leading zeros, a composed text - you need your OWN DDIC
*        STRUCTURE in the Data Dictionary for them. That is not a
*        detour, that is how it is built.
*
* THE TWO NAMED TEXT BLOCKS
*
*        &NOTE&     the notes that agents captured along the way -
*                   the conversation history
*        &WF_PROT&  the workflow log as an HTML table: who decided
*                   what and when
*
*        The product helper delivers both ready-made. Building them
*        yourself is not worth it.
*--------------------------------------------------------------------*

*--------------------------------------------------------------------*
* 1. The header text of the workflow from the customizing.
*
* That way the mail shows the same name as everywhere else - and it
* is translated, because C06T is language-dependent.
*--------------------------------------------------------------------*
    SELECT SINGLE * FROM /c09/cfl_c06t INTO @DATA(ls_c06t)  "#EC CI_ALL_FIELDS_NEEDED
      WHERE wf_definition = @is_data-wf_definition
        AND lang          = @sy-langu.
    IF sy-subrc = 0.
      /c09/cfl_cl_helper_0101=>add_datasource_mail(
        EXPORTING is_datastruc         = ls_c06t
        CHANGING  ct_application_input = ct_application_input ).
    ENDIF.

*--------------------------------------------------------------------*
* 2. The document data - here as entire DDIC structures.
*
* Deliberately WITHOUT preselection: passing EKKO and EKPO in full
* costs nothing, and the customer can use any field in the standard
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

  ENDMETHOD.


  METHOD /c09/cfl_if_badi_0101~get_add_attachments.
*--------------------------------------------------------------------*
* WHEN   During mail dispatch, after the text is in place.
*
* IN     IS_DATA   the instance
*        IT_SMTP   the recipients
* OUT    CT_ATTACHMENT  the attachments
*
* PURPOSE  Attach documents to the mail that the recipient would
*        otherwise have to look up in the system: the archived invoice
*        image, the files attached to the document, an SAP shortcut.
*
* THE SAP SHORTCUT IS THE UNDERRATED CASE
*        A .SAP file in the attachment opens the right system, the
*        right client and the right transaction directly on the
*        recipient's side - with a double click from the mail.
*        For approvers who rarely use the system, that is the
*        difference between "gets done" and "sits there".
*
*        IMPORTANT: the shortcut is generated PER RECIPIENT, because
*        it contains the user name. Hence the loop over IT_SMTP - and
*        hence it is wrong to build it once and send it to everyone.
*
* THREE SOURCES, THREE PRODUCT METHODS
*        GET_SHORTCUT   the SAP shortcut
*        GET_ARCHIVE    documents from the archive (ArchiveLink)
*        GET_GOS        document attachments (Generic Object Services)
*
*        All three live in /C09/CFL_CL_HELPER_0101 and return
*        ready-made attachment lines. Building them yourself is work
*        without gain.
*
* KEEP AN EYE ON SIZE
*        Attachments go into mail dispatch as SOLIX. A 20 MB PDF to
*        ten recipients is 200 MB in the send request. Where size can
*        be an issue, the shortcut is the better answer than the
*        document.
*--------------------------------------------------------------------*

    LOOP AT it_smtp INTO DATA(ls_smtp) WHERE uname IS NOT INITIAL.

      /c09/cfl_cl_helper_0101=>get_shortcut(
        EXPORTING iv_user        = CONV syuname( ls_smtp-uname )
                  iv_transaction = CONV tcode( 'SBWP' )
                  iv_parameter   = space
        CHANGING  ct_attachment  = ct_attachment ).

*--------------------------------------------------------------------*
* Only for the first recipient with a user ID. If you leave out the
* EXIT, the mail gets as many shortcuts as there are recipients - and
* everyone sees the user IDs of the others.
*--------------------------------------------------------------------*
      EXIT.

    ENDLOOP.

  ENDMETHOD.


*======================================================================*
*
*   GROUP 6 - FRAMEWORK
*   Deadlines and cleanup
*
*======================================================================*

  METHOD /c09/cfl_if_badi_0101~get_factory_calendar.
*--------------------------------------------------------------------*
* WHEN   When conFLOW calculates a deadline - i.e. when a work item
*        with a deadline in /C09/CFL_C02 is created.
*
* IN     IS_CFL_S01           the instance
* OUT    CV_FACTORY_CALENDAR  the factory calendar (SCAL-FCALID)
*
* PURPOSE  So that "a two-day deadline" does not expire on Friday
*        afternoon because Saturday and Sunday were counted.
*
* NOT FILLED IN ANY OF THE PRODUCTION IMPLEMENTATIONS EXAMINED.
*
*        That does NOT mean deadlines are correct without a calendar -
*        it means the processes examined calculate deadlines in hours
*        or have none at all.
*
* WHEN YOU NEED IT
*        As soon as a deadline runs in DAYS and the escalation has an
*        effect that annoys someone. A work item that escalates over
*        Easter because four public holidays were counted is the
*        classic first production bug.
*
*        The calendar then usually comes from the plant or the
*        company code assignment of the document - i.e. from the
*        document data, not from a constant.
*
*        STAYS EMPTY.
*--------------------------------------------------------------------*
  ENDMETHOD.


  METHOD /c09/cfl_if_badi_0101~release.
*--------------------------------------------------------------------*
* WHEN   When the BAdI instance is released - at the end of
*        processing.
*
* PURPOSE  Cleanup. Specifically, of whatever you CREATED in
*        GET_OBJECT_INFO or GET_AFTER_CREATION_WORKITEM.
*
* THE CONNECTION THAT IS EASILY OVERLOOKED
*        If you attach your own display to the work item in the SAP
*        GUI (a docking control with document details, the usual
*        pattern), you create a singleton instance for it in
*        GET_OBJECT_INFO. That instance then lives longer than the
*        work item.
*
*        Without the counterpart here, the agent sees the data of the
*        FIRST work item when opening the SECOND one. No error, no
*        dump - just wrong numbers. That is why both examined
*        implementations with a docking control have exactly one line
*        here: DEL_INSTANCE( ).
*
*        Rule of thumb: RELEASE is empty, OR it is the counterpart to
*        something you created yourself. There is no third case.
*
*        STAYS EMPTY HERE, because this example does not come with
*        its own SAP GUI display.
*--------------------------------------------------------------------*

*   zcl_cfl_workflow_00900_doc=>del_instance( ).

  ENDMETHOD.


*======================================================================*
*
*   THE BACKGROUND STEP
*
*   Not a BAdI hook - the second contract conFLOW knows. It is entered
*   in /C09/CFL_C01 on step B1, with class name and method name. The
*   call is DYNAMIC: a wrong signature does not show up at activation
*   but at runtime - and then the workflow gets stuck.
*
*======================================================================*

  METHOD background_classify.
*--------------------------------------------------------------------*
* The step where the work happens:
*
*   1. Read the document
*   2. Write the values to the container
*   3. Classify
*   4. Use EV_DECISION_KEY to say how the process continues
*
* WHY THE VALUES GO INTO THE CONTAINER INSTEAD OF JUST BEING READ
*
*     The container is the basis for the decision, and it is frozen.
*     If the agent decides tomorrow and the document was changed
*     tonight, the agent still has the figures in front of them that
*     the decision is about - and the audit trail shows afterwards
*     which figures those were.
*
*     If you read fresh from EKKO in the work item text instead, you
*     get a display that shifts under the agent, and afterwards no way
*     of telling what they actually saw.
*
* WHY THE CLASSIFICATION IS HERE AND NOT IN THE DISPLAY
*
*     For the same reason. SEVERITY and the recommendation are
*     CALCULATED values - if they are in the container, you can see
*     afterwards what the system recommended and whether the agent
*     deviated from it. If the display does the calculation, that
*     information is gone as soon as the work item is closed.
*
* THE RETURN VALUE IS THE SWITCH
*
*     EV_DECISION_KEY is evaluated exactly like a human decision -
*     except that here the code sets it. /C09/CFL_C02 then contains:
*
*         B1 + OK   -> 01   (decision needed)
*         B1 + UC1  -> X3   (nothing to do, workflow ends)
*
*     This keeps the branch VISIBLE IN CUSTOMIZING. That is why this
*     approach is preferable to the hook GET_STATUS_DYNAMIC: there the
*     same switch would be invisible.
*
*     IF EV_DECISION_KEY STAYS EMPTY, the workflow does not continue.
*     This is the most common reason for "the workflow is stuck in the
*     background step".
*
* ERRORS GO INTO ET_BAPIRET2, NOT INTO AN EXCEPTION
*
*     conFLOW writes the table to the application log and evaluates
*     it. An uncaught exception, on the other hand, puts the workflow
*     step into error status, and the reason then only shows up in the
*     dump.
*--------------------------------------------------------------------*

    CLEAR: et_bapiret2, ev_decision_key.

*--------------------------------------------------------------------*
* The instance - only it knows the document. IS_CFL_S03 is the STEP
* and does not have the INSTID.
*--------------------------------------------------------------------*
    SELECT SINGLE * FROM /c09/cfl_s01 INTO @DATA(ls_cfl_s01) "#EC CI_ALL_FIELDS_NEEDED
      WHERE id = @is_cfl_s03-id.
    IF sy-subrc <> 0.
      add_msg( EXPORTING iv_type     = 'E'
                         iv_text     = |Workflow instance { is_cfl_s03-id } not found|
               CHANGING  ct_bapiret2 = et_bapiret2 ).
      RETURN.
    ENDIF.

*--------------------------------------------------------------------*
* 1. Read the document
*--------------------------------------------------------------------*
    read_document( EXPORTING iv_instid     = ls_cfl_s01-instid
                   IMPORTING ev_net_value  = DATA(lv_net_value)
                             ev_currency   = DATA(lv_currency)
                             ev_vendor     = DATA(lv_vendor)
                             ev_created_by = DATA(lv_created_by)
                             ev_found      = DATA(lv_found) ).

    IF lv_found = abap_false.
      add_msg( EXPORTING iv_type     = 'E'
                         iv_text     = |Purchase order { ls_cfl_s01-instid } not found|
               CHANGING  ct_bapiret2 = et_bapiret2 ).
      RETURN.
    ENDIF.

*--------------------------------------------------------------------*
* 2. Into the container
*
* CONV #( ) on every value: SET_VAL expects a string, the document
* fields are not strings. Without the conversion the compiler reports
* nothing - it converts on its own, but for packed numbers not the way
* you would expect. Being explicit is better here.
*--------------------------------------------------------------------*
    set_val( iv_element = zcl_cfl_const_00900=>mc_prop-doc_number
             iv_id      = ls_cfl_s01-id
             iv_value   = CONV #( ls_cfl_s01-instid ) ).

    set_val( iv_element = zcl_cfl_const_00900=>mc_prop-net_value
             iv_id      = ls_cfl_s01-id
             iv_value   = |{ lv_net_value }| ).

    set_val( iv_element = zcl_cfl_const_00900=>mc_prop-currency
             iv_id      = ls_cfl_s01-id
             iv_value   = CONV #( lv_currency ) ).

    set_val( iv_element = zcl_cfl_const_00900=>mc_prop-vendor
             iv_id      = ls_cfl_s01-id
             iv_value   = CONV #( lv_vendor ) ).

    set_val( iv_element = zcl_cfl_const_00900=>mc_prop-decision_by
             iv_id      = ls_cfl_s01-id
             iv_value   = CONV #( lv_created_by ) ).

*--------------------------------------------------------------------*
* 3. Classify
*--------------------------------------------------------------------*
    DATA(lv_severity) = classify( lv_net_value ).

    set_val( iv_element = zcl_cfl_const_00900=>mc_prop-severity
             iv_id      = ls_cfl_s01-id
             iv_value   = lv_severity ).

    DATA(lv_limit_hit) = COND string(
      WHEN lv_net_value > zcl_cfl_const_00900=>mc_limit_value
      THEN CONV string( abap_true )
      ELSE space ).

    set_val( iv_element = zcl_cfl_const_00900=>mc_prop-limit_hit
             iv_id      = ls_cfl_s01-id
             iv_value   = lv_limit_hit ).

*--------------------------------------------------------------------*
* 4. The switch - and the log entry for it
*
* Both branches write to the log. The branch in which NOTHING happens
* needs the line most: otherwise the log shows a workflow that ended
* itself for no visible reason.
*--------------------------------------------------------------------*
    IF lv_limit_hit IS INITIAL.

      DATA(lv_text) = |No approval required - { lv_net_value } { lv_currency } | &&
                      |is within the limit of { zcl_cfl_const_00900=>mc_limit_value }|.

      ev_decision_key = /c09/cfl_cl_workflow_0101=>mc_decision-uc1.

    ELSE.

      lv_text = |Approval required - { lv_net_value } { lv_currency } | &&
                |exceeds the limit of { zcl_cfl_const_00900=>mc_limit_value } | &&
                |({ lv_severity })|.

      ev_decision_key = /c09/cfl_cl_workflow_0101=>mc_decision-ok.

    ENDIF.

    add_msg( EXPORTING iv_text     = lv_text
             CHANGING  ct_bapiret2 = et_bapiret2 ).

    update_witext( lv_text ).

  ENDMETHOD.


*======================================================================*
*
*   PRIVATE HELPERS
*
*======================================================================*

  METHOD get_val.
*--------------------------------------------------------------------*
* A container element can hold SEVERAL values - that is why
* GET_ATTRIBUT_VALUE returns a table. Nine times out of ten you want
* the first and only one.
*
* These four lines are the reason the hooks above are readable.
* Without them, every hook contains the same loop.
*--------------------------------------------------------------------*

    DATA(lt_value) = /c09/cfl_cl_workflow_0101=>get_attribut_value(
                       iv_element = iv_element
                       iv_id      = iv_id ).

    READ TABLE lt_value ASSIGNING FIELD-SYMBOL(<fs_value>) INDEX 1.
    IF sy-subrc = 0.
      rv_val = <fs_value>-value.
    ENDIF.

  ENDMETHOD.


  METHOD set_val.
*--------------------------------------------------------------------*
* The counterpart. SET_ATTRIBUT_VALUE REPLACES the content of the
* element - it does not append. If you want several values, build the
* table yourself and call the framework method directly.
*
* SET_ATTRIBUT_VALUE first looks up the ID in /C09/CFL_S01. If the
* instance is not there, the value is silently discarded: no error,
* no entry in S04, and reading it returns empty. The element itself
* does not have to be declared anywhere.
*--------------------------------------------------------------------*

    DATA lt_value TYPE /c09/cfl_value_s04_tt.

    APPEND INITIAL LINE TO lt_value ASSIGNING FIELD-SYMBOL(<fs_value>).
    <fs_value>-value = iv_value.

    /c09/cfl_cl_workflow_0101=>set_attribut_value(
      iv_element = iv_element
      iv_id      = iv_id
      it_value   = lt_value ).

  ENDMETHOD.


  METHOD is_true.
*--------------------------------------------------------------------*
* The container has no booleans, only strings. What comes back from
* SET_VAL( abap_true ) is an 'X' - but depending on who set the value,
* it can also be 'x', 'true' or '1'.
*
* A central evaluation is therefore not a luxury: otherwise one place
* checks for 'X' and the next for abap_true, and with lowercase input
* they disagree.
*--------------------------------------------------------------------*

    DATA(lv_upper) = to_upper( condense( iv_value ) ).

    rv_yes = xsdbool( lv_upper = 'X'    OR
                      lv_upper = 'TRUE' OR
                      lv_upper = '1' ).

  ENDMETHOD.


  METHOD read_document.
*--------------------------------------------------------------------*
* The ONLY method that knows the document. If you adapt the class to
* a different document type, you change it here - and in no hook.
*
* EV_FOUND INSTEAD OF SY-SUBRC TO THE CALLER
*     The caller should not need to know how many SELECTs the method
*     consists of. A meaningful flag is more robust than a SY-SUBRC
*     that the next statement overwrites.
*
* THE SUM ACROSS THE ITEMS
*     For an approval, the value of the document counts, not that of
*     a single item. SELECT SUM returns SY-SUBRC 4 and an initial value
*     for a document without items - that is why the existence check
*     is on EKKO and not on the sum.
*--------------------------------------------------------------------*

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

  ENDMETHOD.


  METHOD classify.
*--------------------------------------------------------------------*
* The business rule, in exactly one place.
*
* It is deliberately NOT in the hook, even though it would only be
* three lines there. The reason is not aesthetics: as soon as the rule
* exists in two places - once for the display, once for the decision -
* the two drift apart at some point, and then the work item shows
* something different from what the workflow does.
*
* In a real installation this method belongs in a SEPARATE RULES
* CLASS that also serves the value help of the UI. Then display,
* recommendation and validation demonstrably come from the same
* source.
*--------------------------------------------------------------------*

    IF iv_net_value > zcl_cfl_const_00900=>mc_limit_value * 5.
      rv_severity = zcl_cfl_const_00900=>mc_severity-red.

    ELSEIF iv_net_value > zcl_cfl_const_00900=>mc_limit_value.
      rv_severity = zcl_cfl_const_00900=>mc_severity-yellow.

    ELSE.
      rv_severity = zcl_cfl_const_00900=>mc_severity-green.
    ENDIF.

  ENDMETHOD.


  METHOD recommended_key.
*--------------------------------------------------------------------*
* Which outcome the system recommends - for the green highlight in
* both UIs.
*
* The method returns a KEY, not a text. That is intentional: the
* button texts come from /C09/CFL_C02T and /C09/CFL_C09T and are
* translated. Comparing texts here would give you a recommendation
* that works in English and not in German.
*--------------------------------------------------------------------*

    IF is_true( get_val( iv_element = zcl_cfl_const_00900=>mc_prop-limit_hit
                         iv_id      = iv_id ) ) = abap_true.
      CLEAR rv_key.                          " above the limit: no recommendation
    ELSE.
      rv_key = /c09/cfl_cl_workflow_0101=>mc_decision-ok.
    ENDIF.

  ENDMETHOD.


  METHOD fmt_doc.
*--------------------------------------------------------------------*
* Strip leading zeros. '0004500001234' becomes '4500001234'.
*
* A WARNING THAT COSTS MONEY IN PRACTICE
*     The result is NOT usable as a key for a SELECT. If you put a
*     document number formatted like this into a WHERE condition, you
*     find nothing - and you get no error message, just an empty
*     table. A read routine that formats for display is not a source
*     of keys.
*--------------------------------------------------------------------*

    rv_out = iv_value.
    SHIFT rv_out LEFT DELETING LEADING '0'.
    CONDENSE rv_out.

  ENDMETHOD.


  METHOD fmt_amount.
*--------------------------------------------------------------------*
* Amounts in the USER's format, not in the internal format.
*
* WRITE ... TO is the right statement for this - it respects the user
* settings for decimal and thousands separators. Assigning to a string
* does not.
*
* THE TRY IS NOT DECORATION
*     The container holds a string. Whether it is a number, you cannot
*     know: the element may be empty, or an earlier version wrote text
*     into it. A conversion that fails would abort the DISPLAY of the
*     work item - that is, exactly when someone is looking.
*--------------------------------------------------------------------*

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

  ENDMETHOD.


  METHOD add_msg.
*--------------------------------------------------------------------*
* One line for the application log.
*
* WHY THE DETOUR VIA MESSAGE 00/398
*     SLG1 only displays a line if ID and NUMBER are filled. Plain free
*     text in the field MESSAGE disappears without a trace - no error,
*     no line, nothing. You can spend hours looking for that.
*
*     00/398 is the standard message '&1&2&3&4' - four variables of 50
*     characters each. That fits 200 characters of free text, and SLG1
*     displays them.
*
* WHAT YOU SHOULD DO INSTEAD WHEN IT GETS SERIOUS
*     A message class of your own with meaningful numbers. Then the
*     messages can be translated and evaluated. 00/398 is the approach
*     that needs no new object - good for getting started, not good
*     for the long run.
*
* SO THAT IT ENDS UP IN SLG1 AT ALL
*     Object and subobject for the workflow must be maintained in
*     /C09/CFL_C08, AND both must be created in SLG0. If that is
*     missing, conFLOW collects the messages and writes them nowhere.
*--------------------------------------------------------------------*

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

  ENDMETHOD.


  METHOD set_priority.
*--------------------------------------------------------------------*
* Set the priority - in two stages, and both stages are needed.
*
* THE OBVIOUS APPROACH DOES NOT WORK
*     SAP_WAPI_CHANGE_WORKITEM_PRIO reads SWWWIHEAD from the database.
*     In the after-create hook the work item is not there yet. The
*     call silently does nothing - no error, the priority stays at
*     the default value. Measured.
*
* STAGE 1 - the work item manager of the running transaction
*     CL_SWF_RUN_WIM_FACTORY knows the work items that are being
*     created RIGHT NOW. The search goes by WI_ID, not by type: the
*     hook means one specific work item, not just any.
*
* STAGE 2 - SWW_WI_PRIORITY_CHANGE with checks switched off
*     The same function that sits under the WAPI - but with
*     AUTHORIZATION_CHECKED and PRECONDITIONS_CHECKED set to 'X'.
*     Exactly these checks are the blocker, because the work item
*     does not yet have a status they would accept.
*
*     DO_COMMIT STAYS EMPTY. The COMMIT belongs to the framework - if
*     you commit here yourself, you cut the running transaction in
*     half.
*
* THE TRY BLOCKS ARE INTENTIONAL
*     The hook runs in the middle of creating a work item. An uncaught
*     exception because of a PRIORITY would be a remarkably high price
*     to pay for a display detail.
*--------------------------------------------------------------------*

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

  ENDMETHOD.


  METHOD update_witext.
*--------------------------------------------------------------------*
* Update the text of the running background work item.
*
* The framework sets the text from Customizing BEFORE the background
* method runs. If you want to see a RESULT in the log, you have to
* change it yourself afterwards.
*
* The difference in the workflow log:
*
*     without:  "Classification"         (the same text five times)
*     with:     "Approval required - 12.500,00 EUR exceeds ..."
*
* THE SEARCH GOES BY WI_TYPE = 'B'
*     Unlike SET_PRIORITY, there is no WI_ID here - the background
*     method does not know its own work item. 'B' is the background
*     work item type, and during a background step exactly one of
*     them is registered.
*--------------------------------------------------------------------*

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

  ENDMETHOD.


  METHOD is_sapgui.
*--------------------------------------------------------------------*
* GUI_IS_AVAILABLE is the standard approach: in the OData/RFC context
* of the Fiori inbox there is no front end, so ' ' comes back. The HTML
* approach to coloring needs exactly this distinction.
*--------------------------------------------------------------------*

    DATA lv_return TYPE c LENGTH 1.

    CALL FUNCTION 'GUI_IS_AVAILABLE'
      IMPORTING
        return = lv_return.

    rv_gui = xsdbool( lv_return = abap_true ).

  ENDMETHOD.


  METHOD gui_colour.
*--------------------------------------------------------------------*
* Produces exactly the form that runs in production:
*
*     <span style="color:green;font-size:120%">Approve</span>
*
* The size is in ZCL_CFL_CONST_00900=>MC_GUI_FONT_SIZE and may be
* empty; then only the color remains.
*
* THE PROTECTION AGAINST A SECOND PASS is cheap and honest: the
* framework exit does rebuild ALTTEXT fresh from c09t right before,
* but a text wrapped twice would be an error you cannot see on the
* screen.
*--------------------------------------------------------------------*

    IF iv_text CS '<span'.
      rv_text = iv_text.
      RETURN.
    ENDIF.

    DATA(lv_size) = COND string(
      WHEN zcl_cfl_const_00900=>mc_gui_font_size IS INITIAL THEN ``
      ELSE |;font-size:{ zcl_cfl_const_00900=>mc_gui_font_size }| ).

    rv_text = |<span style="color:{ iv_color }{ lv_size }">{ iv_text }</span>|.

  ENDMETHOD.

ENDCLASS.
```
