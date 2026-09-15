# Work item lifecycle

Create, open, decide, complete

This group holds the three places where you can STOP the process. They differ in what they can do, and that is the most important difference in this whole group:

**GET_BEFORE_DECISION_WORKITEM**

can REMOVE buttons. No message, no abort. "Not allowed" here means: the button is gone.

**GET_AFTER_EXECUTION**

can ABORT (CV_SUBRC), but WITHOUT a message. The agent only sees that nothing happens - useless on its own.

**GET_AFTER_EXECUTION_MOBILE**

```
      can ABORT AND SAY WHY (CS_T100MSG). The only hook with
      both.
```

Rule of thumb: what the agent is NOT ALLOWED to do, you take away beforehand. What they DID WRONG, you tell them afterwards.

## The hooks

- [`get_after_creation_workitem`](get-after-creation-workitem.md) - Priority, attachments, attaching a custom UI
- [`get_before_execution_workitem`](get-before-execution-workitem.md) - Preparation on opening - cannot prevent anything
- [`get_before_decision_workitem`](get-before-decision-workitem.md) - Removing and coloring buttons
- [`get_fiori_task_dec_op_act`](get-fiori-task-dec-op-act.md) - The same for the Fiori inbox - a different table
- [`get_after_execution`](get-after-execution.md) - Follow-up after the decision, cancel without a message
- [`get_after_execution_mobile`](get-after-execution-mobile.md) - Cancel **with** a message - the only real gate
- [`get_after_execution_workitem`](get-after-execution-workitem.md) - After completion: cleaning up parallel work items
- [`get_event_raised_workitem`](get-event-raised-workitem.md) - Event on a running work item
