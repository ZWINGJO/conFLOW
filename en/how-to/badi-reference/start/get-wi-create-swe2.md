# `get_wi_create_swe2`

| | |
|---|---|
| **When** | On event-driven start via SWE2, before conFLOW creates the instance. Only on this path - if you start the workflow via START_WORKFLOW_INT( ) from your own code, it does not pass through here. |
| **In** | IT_EVENT_CONTAINER_TAB  the event container<br>CS_CFL_C10              the type linkage that matched |
| **Out** | CS_SENDER  object type and instance key of the new instance |

**PURPOSE** This is the VETO point. Clearing CS_SENDER means: no workflow. This filters events that are raised but should not trigger a process from a business point of view - document type, plant, amount below the de minimis limit.

**WHY HERE AND NOT IN B1**

A workflow that starts and ends itself in the first background step leaves behind an instance, a log entry and a line in every report. With ten documents that does not matter, with ten thousand it does. What is rejected here never existed.

The price: there is also no trace that a check took place. If you have to prove WHY a document did not get a workflow, you are better off filtering in B1 and ending there with a log entry.

## The code

```abap
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
```
