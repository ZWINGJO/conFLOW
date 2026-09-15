# `get_event_raised_workitem`

> **This hook stays empty in the reference class.** The reason is below.

| | |
|---|---|
| **When** | When an event hits a RUNNING work item - not at workflow start, but while it runs. |
| **In** | IM_EVENT_NAME - which event |
| **In and out** | CM_WORKITEM_CONTEXT - the context of the affected work item |

**PURPOSE** Reacting to changes to the document WHILE the workflow is open. The document is cancelled while someone is deciding on it - then the work item should disappear instead of staying in the inbox.

**NOT FILLED IN ANY OF THE PRODUCTION IMPLEMENTATIONS EXAMINED.**

Not because it is useless, but because the case is rare and the alternative is closer at hand: the change is checked at the next step instead of taking effect immediately.

You know you need it when there is a requirement of the form "if X happens WHILE the workflow is running, then ...". Without that "while" it is an ordinary background step.

**STAYS EMPTY.**
