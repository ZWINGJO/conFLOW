# Display

What the agent sees before deciding

Five hooks for four places on the screen. They are regularly mixed up, so here is the map up front:

```
GET_DESCRIPTION        the ONE line in the result list
                       (CHAR100 - no more is possible)
GET_WORKITEM_TEXT      the BLOCK you read after opening it
GET_OBJECT_INFO        the label of the object link in the
                       FIORI inbox
GET_NEW_PREVIEW_DESCR  the same label in the SAP GUI
```

GET_WF_DEFINITION_TEXT the title of the overall workflow

GET_OBJECT_INFO and GET_NEW_PREVIEW_DESCR label THE SAME thing and are still both needed: the Fiori inbox fills its "Links" tab from SAP_WAPI_GET_OBJECTS and never sees GET_NEW_PREVIEW_DESCR. If you maintain only one of the two, the other user interface shows the framework class name.

## The hooks

- [`get_description`](get-description.md) - The one line in the result list (CHAR100)
- [`get_workitem_text`](get-workitem-text.md) - The text block in the opened work item
- [`get_object_info`](get-object-info.md) - Label of the object link in **Fiori**
- [`get_new_preview_descr`](get-new-preview-descr.md) - The same label in **SAP GUI**
- [`get_wf_definition_text`](get-wf-definition-text.md) - Title of the overall workflow
- [`default_attribute_value`](default-attribute-value.md) - The default attribute of the instance - one line, always the same
- [`execute_default_method`](execute-default-method.md) - Double-click on the object shows the document
