# `get_actors`

| | |
|---|---|
| **When** | On every work item creation, and a second time before every mail dispatch. The most important hook of the whole interface. |
| **In** | IS_CFL_C05  the agent group - GEN_STAT_USER is the key the code branches on<br>IS_CFL_S01  the instance (optional, but practically always set) |
| **Out** | CT_ACTORS   the agents |

**CHECK FIRST WHETHER YOU NEED IT**

/C09/CFL_C03 holds OTYPE and OBJID directly per GEN_STAT_USER - for example US/MEIER, US/WF-BATCH, US/WF_INITIATOR. Only USER_BADI = 'X' switches to this hook. For PoCs, demos and fixed assignments GET_ACTORS therefore stays EMPTY, and you need neither a role nor code.

**THE FORMAT IS THE MOST COMMON MISTAKE**

An entry in CT_ACTORS is always a TYPED org object, never a bare user name:

```
US<uname>      user
S<position>    position
O<orgunit>     organizational unit
AC<role>       role
```

'MEIER' creates no work item and no error message. The work item ends up with nobody and is only noticed when someone asks where it went.

**WHEN NOBODY IS FOUND**

A work item without an agent goes into error status and stays there. A defined fallback agent is better: conFLOW provides the entry 'C09_NO_USER' for this. The process keeps running and the gap is visible instead of silently standing still. See the very bottom of this method.

## The code

```abap
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
```
