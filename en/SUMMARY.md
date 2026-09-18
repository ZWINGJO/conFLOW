# Table of contents

* [conFLOW - SAP Workflows Made Easy](README.md)
* [conFLOW in action ↗](https://story.conflow-help.com)

## Workflow Stories

* [Examples of implemented workflows](workflow-stories/README.md)
  * [HR sick leave notification](workflow-stories/hr-sick-leave.md)
  * [HR onboarding](workflow-stories/hr-onboarding.md)
  * [SD billing request](workflow-stories/sd-billing-request.md)
  * [SD business partner synchronization](workflow-stories/sd-business-partner-sync.md)
  * [BC alert monitor](workflow-stories/bc-alert-monitor.md)
  * [BC master data distribution](workflow-stories/bc-master-data-distribution.md)
  * [CA award proposal](workflow-stories/ca-award-proposal.md)

## Technical Documentation

* [Technical documentation](technical-documentation.md)

## How-To

* [Example: sick leave workflow](how-to/sick-leave-workflow/README.md)
  * [Step 1: Create the Customizing](how-to/sick-leave-workflow/step-1-customizing.md)
  * [Step 2: Extend the Customizing](how-to/sick-leave-workflow/step-2-extend-customizing.md)
  * [Step 3: BAdI implementation](how-to/sick-leave-workflow/step-3-badi-implementation.md)
  * [Step 4: Start and test the workflow](how-to/sick-leave-workflow/step-4-start-and-test.md)
<!-- BEGIN howto-fiori-workitem (erzeugt, nicht von Hand aendern) -->
* [Custom Fiori UI in the work item](how-to/fiori-ui-in-work-item/README.md)
  * [conFLOW: attaching the app to the work item](how-to/fiori-ui-in-work-item/conflow.md)
  * [Customizing and launchpad](how-to/fiori-ui-in-work-item/customizing.md)
  * [The app: OData V2 and UI5 freestyle](how-to/fiori-ui-in-work-item/app.md)
  * [Deploy, check, kill switch](how-to/fiori-ui-in-work-item/check.md)
<!-- END howto-fiori-workitem -->

<!-- conflow-badi-referenz:anfang -->
* [Reference: all 26 BAdI methods](how-to/badi-reference/README.md)
  * [Why this example](how-to/badi-reference/basics/why.md)
  * [The process](how-to/badi-reference/basics/process.md)
  * [The map](how-to/badi-reference/basics/map.md)
  * [The constants class](how-to/badi-reference/basics/constants.md)
  * [Start](how-to/badi-reference/start/README.md)
    * [get_wi_create_swe2](how-to/badi-reference/start/get-wi-create-swe2.md)
    * [get_actors](how-to/badi-reference/start/get-actors.md)
    * [get_number_actors_rel](how-to/badi-reference/start/get-number-actors-rel.md)
  * [Display](how-to/badi-reference/display/README.md)
    * [get_description](how-to/badi-reference/display/get-description.md)
    * [get_workitem_text](how-to/badi-reference/display/get-workitem-text.md)
    * [get_object_info](how-to/badi-reference/display/get-object-info.md)
    * [get_new_preview_descr](how-to/badi-reference/display/get-new-preview-descr.md)
    * [get_wf_definition_text](how-to/badi-reference/display/get-wf-definition-text.md)
    * [default_attribute_value](how-to/badi-reference/display/default-attribute-value.md)
    * [execute_default_method](how-to/badi-reference/display/execute-default-method.md)
  * [Work item lifecycle](how-to/badi-reference/lifecycle/README.md)
    * [get_after_creation_workitem](how-to/badi-reference/lifecycle/get-after-creation-workitem.md)
    * [get_before_execution_workitem](how-to/badi-reference/lifecycle/get-before-execution-workitem.md)
    * [get_before_decision_workitem](how-to/badi-reference/lifecycle/get-before-decision-workitem.md)
    * [get_fiori_task_dec_op_act](how-to/badi-reference/lifecycle/get-fiori-task-dec-op-act.md)
    * [get_after_execution](how-to/badi-reference/lifecycle/get-after-execution.md)
    * [get_after_execution_mobile](how-to/badi-reference/lifecycle/get-after-execution-mobile.md)
    * [get_after_execution_workitem](how-to/badi-reference/lifecycle/get-after-execution-workitem.md)
    * [get_event_raised_workitem](how-to/badi-reference/lifecycle/get-event-raised-workitem.md)
  * [The switch](how-to/badi-reference/switch/README.md)
    * [get_status_dynamic](how-to/badi-reference/switch/get-status-dynamic.md)
  * [Mail](how-to/badi-reference/mail/README.md)
    * [get_send_mail_user](how-to/badi-reference/mail/get-send-mail-user.md)
    * [get_mail_language](how-to/badi-reference/mail/get-mail-language.md)
    * [get_status_mail_dynamic](how-to/badi-reference/mail/get-status-mail-dynamic.md)
    * [get_datasource_mail](how-to/badi-reference/mail/get-datasource-mail.md)
    * [get_add_attachments](how-to/badi-reference/mail/get-add-attachments.md)
  * [Framework](how-to/badi-reference/framework/README.md)
    * [get_factory_calendar](how-to/badi-reference/framework/get-factory-calendar.md)
    * [release](how-to/badi-reference/framework/release.md)
  * [The background step](how-to/badi-reference/building-blocks/background-step.md)
  * [The helpers](how-to/badi-reference/building-blocks/helpers.md)
  * [Agent determination](how-to/badi-reference/building-blocks/agent-determination.md)
  * [Installation](how-to/badi-reference/reference/installation.md)
  * [Pitfalls](how-to/badi-reference/reference/pitfalls.md)
  * [The whole class](how-to/badi-reference/reference/source.md)
<!-- conflow-badi-referenz:ende -->
