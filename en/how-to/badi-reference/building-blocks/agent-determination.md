# Agent determination

Agent determination - central, not in the workflow class.

**CUSTOMIZING FIRST**

conFLOW resolves a PFCG role itself: C03 with OTYPE AG and the column AGR_NAME - dialog users only, locked users are left out. Use this class only for cases this does not cover. TWP_GET_ROLE_USER_ASSIGNMENT filters neither validity nor lock nor user type.

**WHY A SEPARATE CLASS**

Role resolution is always the same block: call the function module, sort, remove duplicates, add the prefix. Six lines that have nothing to do with the individual workflow.

If it sits in the WF class, by the third workflow it is in the system three times - and at the first bug you fix two of them. Also: the same role is typically needed by several workflows ("Purchasing", "Accounting"). A shared class is the place where you look up who that actually is.

**WHAT DOES NOT BELONG HERE**

The DECISION which agent group gets which helper. That lives in GET_ACTORS of the workflow class, because it is workflow-specific. This class only says HOW to get to the people - not WHEN.

**THE PREFIX IS THE MOST COMMON MISTAKE**

An entry in the agent list is always a typed org object:

```
US<uname>      User
S<position>    Position
O<orgunit>     Organizational unit
AC<role>       Role
```

A bare user name creates no work item and no error message. That is why all methods here set the prefix themselves - the caller should never get into a position where it can forget it.

## The whole class

```abap
CLASS zcl_cfl_get_actors DEFINITION
  PUBLIC
  FINAL
  CREATE PUBLIC.

*----------------------------------------------------------------------*
* Agent determination - central, not in the workflow class.
*
* CUSTOMIZING FIRST
*
* conFLOW resolves a PFCG role itself: C03 with OTYPE AG and the
* column AGR_NAME - dialog users only, locked users are left out.
* Use this class only for cases this does not cover.
* TWP_GET_ROLE_USER_ASSIGNMENT filters neither validity nor lock
* nor user type.
*
* WHY A SEPARATE CLASS
*
* Role resolution is always the same block: call the function module,
* sort, remove duplicates, add the prefix. Six lines that have nothing
* to do with the individual workflow.
*
* If it sits in the WF class, by the third workflow it is in the
* system three times - and at the first bug you fix two of them.
* Also: the same role is typically needed by several workflows
* ("Purchasing", "Accounting"). A shared class is the place where you
* look up who that actually is.
*
* WHAT DOES NOT BELONG HERE
*
* The DECISION which agent group gets which helper. That lives in
* GET_ACTORS of the workflow class, because it is workflow-specific.
* This class only says HOW to get to the people - not WHEN.
*
* THE PREFIX IS THE MOST COMMON MISTAKE
*
* An entry in the agent list is always a typed org object:
*
*     US<uname>      User
*     S<position>    Position
*     O<orgunit>     Organizational unit
*     AC<role>       Role
*
* A bare user name creates no work item and no error message. That is
* why all methods here set the prefix themselves - the caller should
* never get into a position where it can forget it.
*----------------------------------------------------------------------*

  PUBLIC SECTION.

*--------------------------------------------------------------------*
* The role names.
*
* They are constants here and not literals in the methods - that way
* you see in one place which roles this workflow toolkit requires.
* That is the list the Basis colleague needs when setting up a new
* system.
*
* In a larger installation they belong in a customizing table - role
* names differ between development and production systems more often
* than you would think.
*--------------------------------------------------------------------*
    CONSTANTS mc_role_buyer      TYPE agr_name VALUE 'Z_CFL_BUYER' ##NO_TEXT.
    CONSTANTS mc_role_supervisor TYPE agr_name VALUE 'Z_CFL_SUPERVISOR' ##NO_TEXT.

    CLASS-METHODS buyer
      RETURNING VALUE(rt_actors) TYPE /c09/cfl_wf_tt_actors.

    CLASS-METHODS supervisor
      RETURNING VALUE(rt_actors) TYPE /c09/cfl_wf_tt_actors.

*--------------------------------------------------------------------*
* The one building block everyone uses.
*
* Public, so that a workflow class with a special role can call it
* without a new method being added here for every individual case.
*--------------------------------------------------------------------*
    CLASS-METHODS by_role
      IMPORTING iv_role          TYPE agr_name
      RETURNING VALUE(rt_actors) TYPE /c09/cfl_wf_tt_actors.

ENDCLASS.


CLASS zcl_cfl_get_actors IMPLEMENTATION.

  METHOD buyer.
    rt_actors = by_role( mc_role_buyer ).
  ENDMETHOD.


  METHOD supervisor.
    rt_actors = by_role( mc_role_supervisor ).
  ENDMETHOD.


  METHOD by_role.
*--------------------------------------------------------------------*
* All users of a role as agents.
*
* TWP_GET_ROLE_USER_ASSIGNMENT is the standard way. Two parameters
* deserve attention:
*
*   NO_USERS_FROM_COMPOSITE_ROLES = SPACE
*       Composite roles are INCLUDED. That is almost always right
*       and almost never obvious: in many installations the users
*       are assigned to composite roles, and the single role would
*       be empty. If you set the parameter to 'X', you get a
*       workflow that runs in the development system and finds
*       nobody in the production system.
*
*   The exceptions
*       ROLE_NOT_FOUND is the case that actually occurs in the
*       production system - the role was not created or has a
*       different name. It must NOT lead to a short dump, but to an
*       empty list. The fallback agent in GET_ACTORS then catches
*       it, and the work item shows that nobody was found.
*
* SORTING AND REMOVING DUPLICATES
*       A user can come in through several role assignments. Without
*       DELETE ADJACENT DUPLICATES they get the same work item
*       several times - which looks like a bug in the SAP GUI and
*       like three open tasks in the Fiori inbox.
*--------------------------------------------------------------------*

    DATA lt_users TYPE TABLE OF twpagruser.

    IF iv_role IS INITIAL.
      RETURN.
    ENDIF.

    CALL FUNCTION 'TWP_GET_ROLE_USER_ASSIGNMENT'
      EXPORTING  role_name                     = iv_role
                 no_users_from_composite_roles = space
      TABLES     role_user_assignment          = lt_users
      EXCEPTIONS role_not_found                = 1
                 no_role_user_assignment_found = 2
                 not_supported                 = 3
                 no_role_name_specified        = 4
                 OTHERS                        = 5.

    IF sy-subrc <> 0.
      RETURN.                                " empty list, no dump
    ENDIF.

    SORT lt_users BY uname.
    DELETE ADJACENT DUPLICATES FROM lt_users COMPARING uname.

    LOOP AT lt_users INTO DATA(ls_user).
      IF ls_user-uname IS NOT INITIAL.
        APPEND |US{ ls_user-uname }| TO rt_actors.
      ENDIF.
    ENDLOOP.

  ENDMETHOD.

ENDCLASS.
```
