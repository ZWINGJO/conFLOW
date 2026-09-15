# conFLOW - SAP Workflows easy built

{% hint style="info" %}
**Deutsch:** diese Dokumentation gibt es auch [auf Deutsch](../README.md).
{% endhint %}

{% hint style="success" %}
**conFLOW** implements approval and release workflows in your SAP system — without SAP workflow know-how, without the Workflow Builder, without workflow Customizing in SPRO.

Instead of a classic SAP workflow with tasks, rules and step groups, you define the process in **Customizing tables** and implement the business logic in **a single BAdI class**. The framework takes care of the rest: work item creation, agent determination, deadlines, escalation, email notifications and status tracking.
{% endhint %}

<figure><img src=".gitbook/assets/conflow.png" alt="conFLOW overview"><figcaption><p>conFLOW - map workflows in a simple way</p></figcaption></figure>

## What conFLOW can do

| Feature | Description |
| --- | --- |
| **Customizing instead of development** | Steps, transitions, agents, deadlines and email notifications are maintained in tables — no Workflow Builder, no task definitions |
| **One BAdI class per workflow** | All business logic lives in one class with defined hooks — agent determination, description, navigation, follow-up processing |
| **Any number of workflows** | Each workflow definition has its own number and its own BAdI implementation. New processes don't require a new transport of the framework |
| **Background steps** | Automatic processing between decisions — enrichment, evaluation, status changes, document postings |
| **Deadlines and escalation** | Time-based forwarding via Customizing, no deadline agents |
| **Parallel approval** | Several agents on the same step — the results are merged |
| **SAP GUI and Fiori** | Work items appear in the Business Workplace (SBWP) and in Fiori My Inbox |
| **conMOBILE integration** | Mobile display of every workflow step via the conMOBILE platform |

## A workflow in a day

Bringing a standard SAP workflow into production typically takes ten days: task definitions, rule resolution, step groups, container operations, binding definitions, agent determination. With conFLOW this comes down to Customizing and one ABAP class — the first working workflow is ready the same day.

<figure><img src=".gitbook/assets/conflow_eng.png" alt="conFLOW architecture"><figcaption><p>Architecture and position in the SAP system</p></figcaption></figure>

## Getting started

{% content-ref url="workflow-stories/" %}
[Examples of implemented workflows](workflow-stories/)
{% endcontent-ref %}

{% content-ref url="technical-documentation.md" %}
[Technical documentation](technical-documentation.md)
{% endcontent-ref %}

{% content-ref url="how-to/sick-leave-workflow/" %}
[How-to: sick leave workflow](how-to/sick-leave-workflow/)
{% endcontent-ref %}
