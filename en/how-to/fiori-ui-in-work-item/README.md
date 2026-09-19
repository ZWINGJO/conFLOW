# Custom Fiori UI in the work item

*A small UI5 app in the detail area of My Inbox, attached through conFLOW, without rebuilding the workflow.*

## What you end up with

```
┌─────────────────────────────────────────┬────────────────────────┐
│  Work item title                        │ Comments │ Attachments │
│                                         │ More ∨                 │
│  Work item text                         │                        │
│  (finding, document, quantities, dates) │  ← My Inbox standard,  │
│                                         │    via "Show Details"  │
│  Reason     [ Select             ▾ ]    │                        │
│  Note       [                      ]    │                        │
│  Saved 09:26:03                         │                        │
├─────────────────────────────────────────┴────────────────────────┤
│  Partial delivery │ Accept new date │ Escalate │ Show Details │ ⋯ │
└──────────────────────────────────────────────────────────────────┘
      custom app (left)               conFLOW buttons (bottom)
```

{% hint style="info" %}
**The app decides nothing.** Buttons, agent determination, deadline, priority and log stay with conFLOW. The app shows the context and records *why* a decision is made: reason and note go into the conFLOW container. The decision itself is still made with the buttons below.
{% endhint %}

## The chain at a glance

```
conFLOW
  parameter VISU in C08   (special case: BAdI set_inbox_ui( ))
    └─ sets three container elements on the work item
                           │
Customizing SWFVMD1        ▼   (once per system)
  Task TS00388601  →  Intent  #ZCFLOrderPromiseV2-openInInbox?CFLQueryObject00=<instance>
                           │
Launchpad                  ▼   (per workflow)
  Semantic object · target mapping (openMode!) · catalog · role
                           │
App                        ▼   (per workflow)
  UI5 freestyle  →  OData V2 (SEGW)  →  conFLOW container /C09/CFL_S04
```

**Each work item decides which app appears**, not the task. conFLOW wires the visualization generically: at runtime the task reads the container elements set by conFLOW. If they are missing, My Inbox shows the text block as before.

## What you create

| Area | What | How often | Page |
| --- | --- | --- | --- |
| conFLOW Customizing | parameter `VISU` in `C08` (semantic object) | per workflow | [conFLOW](conflow.md) |
| conFLOW BAdI | only for special cases, e.g. a different app per step | optional | [conFLOW](conflow.md) |
| SWFVMD1 | dynamic visualization for task `TS00388601` | **once per system** | [Customizing](customizing.md) |
| Launchpad | semantic object, target mapping, catalog, role | per workflow | [Customizing](customizing.md) |
| Gateway | register the OData service | per workflow | [Customizing](customizing.md) |
| App | SEGW model, two ABAP classes, UI5 app | per workflow | [The app](app.md) |

The deployment order and the check after each step are described under [Check](check.md).

## Prerequisites

| | |
| --- | --- |
| ABAP | SAP_BASIS **7.50**, SAP_GWFND 750 — no RAP required |
| UI5 | SAPUI5 **1.71** or later |
| Inbox | Fiori My Inbox (BSP `CA_FIORI_INBOX`) |

{% hint style="info" %}
**All names are examples** from a reference workflow (number `00500`, a delivery date deviation in a sales order). For your own workflow, replace number and names — the pattern stays the same.
{% endhint %}
