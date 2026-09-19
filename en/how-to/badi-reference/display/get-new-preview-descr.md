# `get_new_preview_descr`

| | |
|---|---|
| **When** | The SAP GUI builds the object list of the work item ("Objects and attachments"). |
| **In** | IV_WI_ID            the work item |
| **Out** | CT_PREVIEW_OBJECTS  the object list, changeable |

```
PURPOSE   Same as GET_OBJECT_INFO, just for the other user interface.
       Maintain both, otherwise one of the two looks ugly.
       With c06t-OBJTEXT the framework does both itself - leave it
       empty then.
```

**THE CHECK ON OBJTYPE IS NOT OPTIONAL**

The list has several entries: the conFLOW instance object, notes (SOFM), attachments. If you loop over the table without a CHECK, you label everything the same - including the notes, which then lose their own name.

**DELETING IS ALLOWED TOO**

The table is CHANGING. A DELETE removes an entry from the display - the usual case is a technical note that is none of the agent's business. The example below is commented out, because it needs a key that only exists in your own project.

## The code

```abap
    LOOP AT ct_preview_objects ASSIGNING FIELD-SYMBOL(<fs_object>).
      CHECK <fs_object>-objtype = '/C09/CFL_CL_WORKFLOW_0101'.
      <fs_object>-descript = text-001.
    ENDLOOP.

*   DELETE ct_preview_objects
*     WHERE objtype    = 'SOFM'
*       AND def_attrib = zcl_cfl_const_00900=>mc_def_attrib_intern.
```
