# HR sick leave notification

## The requirement

An employee calls in sick. The information must reliably reach the HR department and be processed and documented there. Without a workflow this happens by phone or email — unstructured, not traceable, and dependent on whether the right contact person is available.

## What conFLOW does here

| Step | Who | What happens |
| --- | --- | --- |
| Notification | Employee | Enters the sick leave notification — the workflow starts |
| Processing | HR department | Receives a work item, checks and confirms the notification |
| Completion | System | The workflow is completed, the case is documented |

The whole process is traceable: who reported when, who processed it when, and what the result was. The HR department finds the notification in its inbox (SAP GUI or Fiori My Inbox), not in an email among a hundred others.

### User story

**Daily routine:** HR employee **Jessica** processes the sick leave and return-to-work notifications of the staff. In her mid-sized company there are several per week. Thanks to employee self-service, all that is left for her is checking and administration.

**conFLOW for sick leave:** **Anne** has broken her leg and calls in sick from bed via an SAP app. The workflow lands with **Jessica** in HR, who orders flowers and adds this as a note to the workflow. Anne's manager **Bernd** then receives a notification.

### Workflow

<figure><img src="../.gitbook/assets/folie4.png" alt="Workflow HR sick leave"><figcaption><p>Process flow of the workflow in conFLOW (diagram labels in German)</p></figcaption></figure>

{% hint style="success" %}
**This workflow is also the basis for the [how-to](../how-to/sick-leave-workflow/README.md)** — it shows step by step how it is built.
{% endhint %}
