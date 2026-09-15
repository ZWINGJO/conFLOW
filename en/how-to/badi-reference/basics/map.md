# The map

All 26 hooks of the interface, in the order in which they occur
in the process. The **Code** column says whether the reference class
fills them - six stay empty on purpose.

## Start

| Hook | Code | Purpose |
|---|---|---|
| [`get_wi_create_swe2`](../start/get-wi-create-swe2.md) | yes | Does the workflow start at all? The veto point |
| [`get_actors`](../start/get-actors.md) | yes | Who receives the work item |
| [`get_number_actors_rel`](../start/get-number-actors-rel.md) | yes | Parallel steps, and the flag for email notification |

## Display

| Hook | Code | Purpose |
|---|---|---|
| [`get_description`](../display/get-description.md) | yes | The one line in the result list (CHAR100) |
| [`get_workitem_text`](../display/get-workitem-text.md) | yes | The text block in the opened work item |
| [`get_object_info`](../display/get-object-info.md) | yes | Label of the object link in **Fiori** |
| [`get_new_preview_descr`](../display/get-new-preview-descr.md) | yes | The same label in **SAP GUI** |
| [`get_wf_definition_text`](../display/get-wf-definition-text.md) | empty | Title of the overall workflow |
| [`default_attribute_value`](../display/default-attribute-value.md) | yes | The default attribute of the instance - one line, always the same |
| [`execute_default_method`](../display/execute-default-method.md) | yes | Double-click on the object shows the document |

## Work item lifecycle

| Hook | Code | Purpose |
|---|---|---|
| [`get_after_creation_workitem`](../lifecycle/get-after-creation-workitem.md) | yes | Priority, attachments, attaching a custom UI |
| [`get_before_execution_workitem`](../lifecycle/get-before-execution-workitem.md) | empty | Preparation on opening - cannot prevent anything |
| [`get_before_decision_workitem`](../lifecycle/get-before-decision-workitem.md) | yes | Removing and coloring buttons |
| [`get_fiori_task_dec_op_act`](../lifecycle/get-fiori-task-dec-op-act.md) | yes | The same for the Fiori inbox - a different table |
| [`get_after_execution`](../lifecycle/get-after-execution.md) | yes | Follow-up after the decision, cancel without a message |
| [`get_after_execution_mobile`](../lifecycle/get-after-execution-mobile.md) | yes | Cancel **with** a message - the only real gate |
| [`get_after_execution_workitem`](../lifecycle/get-after-execution-workitem.md) | yes | After completion: cleaning up parallel work items |
| [`get_event_raised_workitem`](../lifecycle/get-event-raised-workitem.md) | empty | Event on a running work item |

## The switch

| Hook | Code | Purpose |
|---|---|---|
| [`get_status_dynamic`](../switch/get-status-dynamic.md) | empty | Next status at runtime instead of from Customizing |

## Mail

| Hook | Code | Purpose |
|---|---|---|
| [`get_send_mail_user`](../mail/get-send-mail-user.md) | empty | Adding or removing recipients |
| [`get_mail_language`](../mail/get-mail-language.md) | empty | Language of the email |
| [`get_status_mail_dynamic`](../mail/get-status-mail-dynamic.md) | empty | Which group of agents is notified |
| [`get_datasource_mail`](../mail/get-datasource-mail.md) | yes | The data for the mail text and the work item text |
| [`get_add_attachments`](../mail/get-add-attachments.md) | yes | Attachments - archive, document files, SAP shortcut |

## Framework

| Hook | Code | Purpose |
|---|---|---|
| [`get_factory_calendar`](../framework/get-factory-calendar.md) | empty | Factory calendar for deadline calculation |
| [`release`](../framework/release.md) | empty | Cleaning up what you created yourself |

## The six empty ones

They are not forgotten. Each chapter says what the hook is meant
for and how to tell whether your own case belongs there:

- `get_before_execution_workitem` - cannot prevent anything
- `get_event_raised_workitem` - needs a "while the workflow is running"
- `get_status_dynamic` - hides the branching that Customizing makes visible
- `get_send_mail_user` - the group of recipients belongs in Customizing
- `get_mail_language` - the standard already does the right thing
- `get_factory_calendar` - only needed once deadlines run in days

In addition, `get_wf_definition_text` and `release` stay empty in this
example, but are needed more often than the six above.
