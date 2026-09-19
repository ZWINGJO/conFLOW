# `execute_default_method`

| | |
|---|---|
| **When** | Double-click on the object in the SAP GUI work item. |
| **In** | nothing - the instance is in /C09/CFL_CL_WORKFLOW_0101=>MS_INSTANCES-INSTANCE->MS_DATA |

```
PURPOSE   "Show me the document". With c06t-OBJTEXT and c08
       TEMPLATE the framework opens the document itself (default
       method of the template's object type) - leave it empty then,
       otherwise it opens twice. Without both, nothing happens on
       double-click, and the agent has to copy down the number.
```

MS_INSTANCES is class-wide - with several work items in one session not necessarily your own instance.

**WITH THE DISPLAY TRANSACTION, NOT THE CHANGE TRANSACTION**

ME23N, not ME22N. The agent should SEE the document while deciding. If you let them change it here, you have a document that moves underneath the running workflow - and an audit trail that no longer matches.

**AND SKIP FIRST SCREEN**

Skips the initial screen. For the Enjoy transactions (ME23N, VA03) the addition has no effect or gets in the way - there SET PARAMETER ID is enough.

**IF SEVERAL DOCUMENT TYPES RUN ON THE SAME CLASS**

Branch on IS_DATA-TYPEID. With a separate BOR type per workflow (the normal case) this is not needed.

## The code

```abap
* Only needed without c06t-OBJTEXT and c08 TEMPLATE - otherwise the
* framework already opens the document, and this method stays empty.
    DATA lv_ebeln TYPE ekko-ebeln.

    lv_ebeln = /c09/cfl_cl_workflow_0101=>ms_instances-instance->ms_data-instid.

    SET PARAMETER ID 'BES' FIELD lv_ebeln.
    CALL TRANSACTION 'ME23N'.
```
