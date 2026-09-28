# Setting up input fields on the work item

*A form on the approval step. Four steps in Customizing, no code — in the SAP Business Workplace right away, in Fiori My Inbox after a one-time setup.*

## What it looks like

```
┌──────────────────────────────────────────────────────────────┐
│  ① Approval line manager Training request 166                │
│                                                              │
│  TRAINING REQUEST                                            │
│  Course: Fiori Inbox · Start: 10.10.2027 · Cost: 1,714.00 €  │
│                                                              │
│  Your entries                                                │
│    Course*      [ Fiori Inbox                             ]  │
│                   Required for every decision                │
│    Start date*  [ 10.10.2027                           📅 ]  │
│    Cost*        [         1714.00 ] EUR                      │
│    Currency*    [ EUR – Euro                            ▾ ]  │
│    Reason       [                                        ]  │
│                                                   Saved 09:26│
├──────────────────────────────────────────────────────────────┤
│                        ✔ Approve  │  ✖ Reject                │
└──────────────────────────────────────────────────────────────┘
```

{% hint style="info" %}
**The fields are in no user interface.** SAP GUI and Fiori build the same form from Customizing. What you set up here appears in both — and the values go into the conFLOW container, the same place the placeholders for work item texts and emails come from.
{% endhint %}

## The four steps

### 1 · Create the field set

Maintenance tree `/C09/CONFLOW_C`, node **Field sets** (`/C09/CFL_C13`) below the workflow definition.

| Field | What goes in |
| --- | --- |
| `VIEW_ID` | a name, unique **within this definition** — for example `Z_REQUEST_START` or `Z_REQUEST_01` |
| Heading | per language, in the text node (`C13T`); it appears above the form |

A set is a **form**, not a step. It may be used by several steps: the values belong to the case, not to the step.

### 2 · Enter the fields

Sub-node **Fields of the field set** (`/C09/CFL_C14`), one row per field.

| Field | What goes in |
| --- | --- |
| Sequence | in steps of ten, so something can be squeezed in later |
| Element | the name in the container, up to 32 characters — it reappears as the placeholder `&CFL-<ELEMENT>&` |
| Data element | **the only real decision**, see below |
| Mode | input or display |
| Mandatory | enforced at start and when deciding |
| Label | only needed if the one from the data element does not fit (text node `C14T`) |

**The data element decides the control.** You do not pick what the field "should be" — that is in the Dictionary:

| What you want | Data element | What appears |
| --- | --- | --- |
| short text | a `CHAR` element, e.g. `TEXT132` | input field, length from the Dictionary |
| long text | a `STRING` element | multi-line field, any length |
| yes/no | `XFELD` | checkbox |
| fixed choice | your own element with **domain fixed values** | drop-down, translated; from 20 entries with search |
| choice from Customizing | element with a **check table**, e.g. `WERKS_D`, `EKGRP` | drop-down for small tables, otherwise entry with a check |
| date | `DATUM` | date picker |
| time | `UZEIT` | time field |
| amount | `WRBTR` | amount field with currency |
| quantity | a `QUAN` element | number field with unit |
| a small table | your own **DDIC structure** | table with one column per field |

{% hint style="success" %}
**Your own data element pays off.** For a drop-down, create a domain with fixed values and translate the texts there — then the list is right in every logon language, the F4 help comes for free, and the check on save uses **the same** list the interface shows.
{% endhint %}

### 3 · Attach the set to the step

In the node **Approval steps** (`/C09/CFL_C01`), the field `VIEW_ID` holds which set a step shows.

**A step without an entry shows no form and runs unchanged.** So you can switch the feature on for one step and change nothing anywhere else.

The same element may appear in several sets — input on one step, display on the next. What one person requests becomes what the next person decides on, without copying the value.

### 4 · Switch on Fiori

**In the SAP GUI the area appears with no setup at all** — step 4 concerns Fiori My Inbox only. There, conFLOW delivers the app; it is set up **once per system**, after which every further workflow is one line of Customizing.

#### How it fits together

```
My Inbox, work item clicked
     │
     │  task TS00388601 is set to dynamic visualisation (SWFVMD1)
     │  and reads three container elements of the work item
     ▼
  conFLOW set them when the work item was created — from C08-VISU
     │     /C09/CFL_CL_VISU_0101, in the work item exit before the BAdI
     ▼
  Intent:  <SemanticObject>-openInInbox?CFLQueryObject00=<case>
     │
     │  the target mapping in the launchpad resolves it
     ▼
  App  c09.cfl.inbox   (BSP /C09/CFL_INBOX)
     │  OData V2
     ▼
  /C09/CFL_INBOX_SRV → data layer → field core → container /C09/CFL_S04
```

**Which app appears is decided by the individual work item**, not by the task. That is why workflows with and without a form can sit next to each other in the same inbox. And that is why **only new work items** get the redirection: it is written into the container when they are created.

#### Once per system

| # | What | Transaction |
| --- | --- | --- |
| 1 | Register the model: `/C09/CFL_INBOX_MDL`, model provider class `/C09/CFL_CL_INBOX_MPC` | `/IWBEP/REG_MODEL` |
| 2 | Register the service: `/C09/CFL_INBOX_SRV`, data provider class `/C09/CFL_CL_INBOX_DPC_EXT`, assign the model | `/IWBEP/REG_SERVICE` |
| 3 | Activate the service and call `$metadata` — if the model comes back, half the job is done | `/IWFND/MAINT_SERVICE` |
| 4 | Create the target mapping (**no tile**) | `/UI2/FLPD_CUST` |
| 5 | Put the catalog into the agents' role | `PFCG` |

{% hint style="danger" %}
**Steps 1 to 3 do not travel with the transport.** Classes and app do, the registration is a system setting and has to be repeated in every system. Without it the app reports *service not found* — and nobody knows why.
{% endhint %}

#### The names, and what has to match what

| Value | Where it is held | Where it comes from |
| --- | --- | --- |
| **Semantic object** | target mapping **and** `C08-VISU` | **freely chosen** — both places must read exactly the same. No `/` allowed, so write a namespace as a prefix |
| **Action** `openInInbox` | target mapping | **fixed in the product** (`/C09/CFL_CL_VISU_0101`) — do not change |
| **ID** `c09.cfl.inbox` | target mapping | the **component ID** of the app, not the BSP name |
| **URL** `/sap/bc/ui5_ui5/c09/cfl_inbox` | target mapping | the BSP application `/C09/CFL_INBOX` |
| **Parameter** `openMode` | target mapping, as **Value**, not as Default Value, mandatory ticked | value `embedIntoDetailsNestedRouter`. Without it the detail area stays **white** |
| *Allow additional parameters* | target mapping | ticked — the app reads the case ID from the URL itself |

{% hint style="info" %}
**The entry in `/UI2/SEMOBJ` is convenience, not a prerequisite.** The intent is resolved by the target mapping; the table provides the value help in the launchpad designer. Without an entry everything works — only the name has to be typed exactly when creating the target mapping, and a typo there is reported nowhere. A minute well spent.
{% endhint %}

#### Authorisations

| Who | What they need |
| --- | --- |
| The agent | Fiori My Inbox as before, **plus** the catalog with the target mapping in one of their roles — otherwise the navigation does not resolve and the work item looks as it did before |
| The agent | access to the OData service `/C09/CFL_INBOX_SRV`, as for any Gateway service |
| Nobody | **a conFLOW authorisation of their own.** Whoever may decide may enter |

**The last point matters more than it looks.** conFLOW does not check an authorisation object but the work item: writing is allowed for whoever is the **current agent of the open work item** — or their **active substitute**. Everyone else sees the same form as display only, with the note *Display only — this work item is not open for you*. So if you are looking for a role that grants "may write input fields", you will not find one; it does not exist, and it would be the wrong place.

#### Kill switch and rollback

| Goal | What you do | Affects |
| --- | --- | --- |
| gone at once, including open work items | delete the target mapping from the catalog | **all** work items, without a transport |
| new work items without a form | clear `VISU` in `C08` | **new** work items only |
| switch the form off on one step | clear `VIEW_ID` in `C01` | that step, in **both** interfaces |

{% hint style="success" %}
**The fallback is not an emergency measure but the normal case** — every work item without the three container elements shows the standard text block. That is why it is reliable: it is in constant use, not only when something goes wrong.
{% endhint %}

#### And then per workflow

Enter the semantic object in `C08-VISU`. That is all.

**The BAdI is not needed for this.** It remains for the special case where every step is to open a *different* app — then you set the container elements yourself in the hook `get_after_creation_workitem`.

## A request without a document

A leave request, a training request, a report — cases with no SAP document behind them. Then a form fills the values **before** the workflow starts:

| Parameter in `C08` | Value |
| --- | --- |
| `VIEW_ID` | the field set of the request form |
| `TYPEID` | an object type of your own for the case |
| `NRANGE` | a number range object; conFLOW draws the key from it, interval `01` |

**`NRANGE` is not convenience, it is necessary.** At start, conFLOW checks by definition, object type and key whether a case is already open. Without a key of its own, the second request would find the first and be rejected as a duplicate start — a fault that only shows up on the second request.

The values travel **with** the start, not after it. So a first background step can work with them right away and apply its rule to them.

## Checking, in this order

1. **In the SAP GUI**, on the work item: does the form appear? Are the labels in your language? Are mandatory fields marked with an asterisk?
2. **Decide without filling a mandatory field** — the decision has to be rejected, with a message naming the field.
3. **Look into `/C09/CFL_S04`:** the value is there **internally** — a date as `YYYYMMDD`, a number with a dot. That is correct; the interface converts.
4. **In `/C09/CFL_S05`:** one row per change with old value, new value, user and channel.
5. **Only then Fiori.** If something is wrong in the SAP GUI, it is not the app.

## The traps

{% hint style="warning" %}
**Texts belong in the `*t` tables**, not in `C13` or `C14` themselves. If you look for them in the main table you will not find them — and if you maintain them there, you lose them with the next language.
{% endhint %}

**Mandatory applies at start and when deciding, not when saving.** Anyone who clears a mandatory field may save that. Otherwise the screen would show it empty while the container still held the old value, and the decision would be taken on the old one — although the agent saw something else.

**A container row holds 132 characters.** conFLOW spreads longer values over several rows itself and reassembles them on reading; you notice nothing. Only if you write into the container **yourself** do you have to split it yourself.

**A display field can be written once: at start.** After that a value for it is rejected, not silently dropped. That keeps an original entry traceable.

**Check the second language.** What exists in only one language is invisible to exactly the person who maintained it.

## When Customizing is not enough

Dependent value lists, mandatory-by-context, derived proposals: that is what the **field exit** is for — a class behind the parameter `FIELD_EXIT` in `C08` that gets the finished field list once more before the interface draws it.

You only need a **user interface of your own** for something other than fields: a document preview, a simulation, an interaction that does not exist yet. The way there is in the [how-to on the Fiori UI](fiori-ui-in-work-item/README.md).

The tables, the data model and the boundaries in detail: [Technical documentation, section 14](../technical-documentation.md).
