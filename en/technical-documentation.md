# Technical documentation

This page describes the technical architecture of conFLOW: the Customizing model, the runtime data, the BAdI interface and the integration with SAP Business Workflow.

{% hint style="info" %}
**Where to go next.** For a hands-on walkthrough, see the [sick leave how-to](how-to/sick-leave-workflow/README.md). For every BAdI method in detail, see the BAdI reference in the how-to section.
{% endhint %}

---

## Architecture at a glance

conFLOW is built on top of SAP Business Workflow and controls it through Customizing tables (`/C09/CFL_C*`) and a BAdI interface (`/C09/CFL_IF_BADI_0101`). Runtime data is stored in the tables `/C09/CFL_S*`.

```
Customizing  /C09/CFL_C*          what the process looks like
      │                           steps · transitions · agents · deadlines · mail
      ▼
conFLOW framework                 creates work items, resolves agents,
      │                           monitors deadlines, sends mail, logs
      ├──▶ BAdI class ZCL_CFL_WORKFLOW_<nnnnn>   business logic, one class per workflow
      ▼
SAP Business Workflow             generic conFLOW tasks, SBWP, Fiori My Inbox
      │
      ▼
Runtime data  /C09/CFL_S*         instance · work item history · container
```

## The Customizing model

The workflow definition, its steps, transitions, agents and deadlines are maintained entirely in Customizing tables. Transaction `/C09/CONFLOW_C` (view cluster `/C09/CFL_C`) is the central entry point.

| Node (EN logon) | Table | Content |
| --- | --- | --- |
| Workflow definition | `/C09/CFL_C06` | one row per workflow number |
| Approval steps | `/C09/CFL_C01` | steps with status code `gen_stat`, type, texts, background method |
| Approval status - control | `/C09/CFL_C02` | transitions (`OK` / `NOK`), deadlines |
| Control additional status | `/C09/CFL_C09` | decision options `UC1`–`UC5` per step |
| User status definition | `/C09/CFL_C04` | roles (agent keys `gen_stat_user`) |
| User status assignment (current settings) | `/C09/CFL_C03` | users, positions or PFCG roles per role, or `user_badi = 'X'` |
| User status assignment (default - transport) | `/C09/CFL_C12` | transportable default for the assignment |
| Assignment of user status | `/C09/CFL_C05` | which role processes which step |
| Control mail setting | `/C09/CFL_C07` | who receives which email at which step |
| General parameters | `/C09/CFL_C08` | e.g. object and subobject of the application log |
| Type linkages standard | `/C09/CFL_C10` | links an event to a workflow definition |

## Workflow definition and steps

Every workflow has a unique number (e.g. `00208`) and is registered in `/C09/CFL_C06`. The steps are stored in `/C09/CFL_C01`, the transitions in `/C09/CFL_C02`.

| Type | Status code | Meaning |
| --- | --- | --- |
| Start | `X0` | the entry point of every workflow |
| Dialog | `01`, `02`, … | work item in the agent's inbox |
| Background | `B1`, `B2`, … | automatic processing, no work item |
| End | `X1`, `X2`, … | every status starting with `X` ends the workflow |

## Agent determination and roles

Steps are assigned to agents through agent keys (`gen_stat_user`) in `/C09/CFL_C04` and `/C09/CFL_C05`. The actual resolution — which user receives the work item — can be fixed in Customizing (`/C09/CFL_C03`) or determined dynamically by the BAdI method `GET_ACTORS`.

## Runtime data model

Runtime data lives in three tables:

| Table | One row per | Content |
| --- | --- | --- |
| `/C09/CFL_S01` | workflow instance | workflow number, object key (`instid`), current step, end flag |
| `/C09/CFL_S03` | work item, chronologically | step, agent key, work item ID, creator, time |
| `/C09/CFL_S04` | container element | attribute-value pairs of the instance |

The three tables are linked by the instance `id`.

## BAdI interface

The interface `/C09/CFL_IF_BADI_0101` defines the hooks where business logic plugs in. Each workflow implements the interface in its own class `ZCL_CFL_WORKFLOW_<nnnnn>`, restricted by a filter on the workflow number. The most important hooks:

| Hook | Purpose |
| --- | --- |
| `GET_ACTORS` | agent determination |
| `GET_DESCRIPTION` / `GET_WORKITEM_TEXT` | work item title and text |
| `GET_BEFORE_DECISION_WORKITEM` | control the buttons |
| `EXECUTE_DEFAULT_METHOD` | navigation to the business object |
| `GET_STATUS_DYNAMIC` | determine the next step in code |
| `GET_DATASOURCE_MAIL` | values for mail placeholders |

Hooks you don't need stay empty.

## Decision options

Every step offers up to seven outcomes: `OK` and `NOK` (controlled via `/C09/CFL_C02`) and `UC1` to `UC5` (controlled via `/C09/CFL_C09`). The button labels are maintained in the corresponding text tables.

## Background steps

Background steps execute a static method automatically — without a work item in an inbox. The method receives the current workflow instance and returns a decision key that determines the next step. This lets you model branches, enrichment and automatic postings within the process.

## Deadlines and escalation

Deadlines are maintained in Customizing (`/C09/CFL_C02`): value, unit and follow-up status when the deadline expires. No deadline agents, no workflow Customizing in SPRO — everything in one table row.

## Email notification

conFLOW sends emails per step, controlled by Customizing. The contents come from SO10 texts in which placeholders are replaced dynamically. The BAdI method `GET_DATASOURCE_MAIL` supplies the replacement values.

## Container and data transfer

The conFLOW container (`/C09/CFL_S04`) stores any attribute-value pairs per workflow instance. It is written and read with the framework methods `SET_ATTRIBUT_VALUE` and `GET_ATTRIBUT_VALUE`. Every value stored there can be traced later.

## Starting a workflow

There are several ways to start a conFLOW workflow: from an application BAdI (e.g. after a document is saved), via SAP status management (a status change raises a BOR event) or from a report. The key element is the **type linkage** in `/C09/CFL_C10`, which links the event to the workflow definition.

## SAP GUI and Fiori

Work items appear both in the SAP Business Workplace (transaction `SBWP`) and in SAP Fiori My Inbox. conFLOW controls button labels, the context block and navigation to the business object in both UIs through the same BAdI hooks.

## conMOBILE integration

The integration with conMOBILE provides a mobile display of every workflow step. The conMOBILE app reads the same conFLOW container and uses the same decision keys — one data source, one decision model.

## Parallel processing

conFLOW supports parallel agent paths: several agents receive a work item at the same time, and the workflow waits for all decisions before it continues. This is configured by assigning several agent keys to one approval step.

## Status management and audit trail

The workflow status is kept in `/C09/CFL_S01` (current step) and `/C09/CFL_S03` (history of all work items). Together with the container (`S04`) this results in a complete audit trail without a custom Z table: who processed which step when, with which outcome, and which data was available at the time of the decision.
