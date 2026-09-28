# Your own Fiori UI instead of the delivered app

*When an app of your own is needed, what conFLOW offers for it — and what you should not rebuild.*

## First the question: do you need one?

**Not for input fields.** A drop-down, a date, an amount, a note, a small table: a field set in Customizing does that, and the delivered app shows it in Fiori My Inbox — see [Setting up input fields on the work item](../set-up-input-fields.md).

A UI of your own pays off when the agent needs something that is **not a field**:

- a preview of the document, a drawing, a file
- a simulation — what happens if I decide this way
- an interaction that does not exist yet: a map, a timeline, a comparison of two documents

## What conFLOW offers for it

### 1 · The navigation, with no code at all

The parameter `VISU` in `C08` names the semantic object. When the work item is created, the framework sets three container elements: semantic object, action and the case ID as a 32-character hex string — that way it fits through the URL.

**This works for any app with a target mapping**, including yours. The path is the same as for the delivered app; only the URL and the component ID in the target mapping are yours.

### 2 · A different app per step

If the whole workflow is not to open the same UI but every step a different one, you set the three container elements yourself — in the BAdI hook [`get_after_creation_workitem`](../badi-reference/lifecycle/get-after-creation-workitem.md), which runs once per work item, after creation and before the first display.

{% hint style="success" %}
**Catch errors there silently.** A user interface that fails to appear must not block a work item — without the container elements the inbox simply shows its text block, and that is a usable state.
{% endhint %}

### 3 · The data

Your app does not read and write the tables itself but goes through the product:

| For what | What |
| --- | --- |
| read, check and write the fields of a field set | `/C09/CFL_CL_FIELDS_0101` — `GET_FIELDS`, `CHECK_VALUES`, `SAVE` |
| read a long value spread over several rows | `/C09/CFL_CL_FIELDS_0101=>READ_VALUE` |
| free container values | `GET_ATTRIBUT_VALUE` / `SET_ATTRIBUT_VALUE` of the workflow class |
| the work item text as SAP GUI and inbox show it | `SAP_WAPI_WORKITEM_DESCRIPTION` |

**Always write through `SET_ATTRIBUT_VALUE`** — only then does the change log in `/C09/CFL_S05` come about, with user, time and channel. Write past it and you lose the audit trail, silently.

## The setup is the same

Service, semantic object, target mapping, catalog, authorisations, `openMode`, kill switch: it is all in [Setting up input fields on the work item, step 4](../set-up-input-fields.md). For your app only two values in the target mapping change — the URL of your BSP application and the component ID.

## What you should not rebuild

| | Why not |
| --- | --- |
| The decision buttons | they come from the task gateway, including colour and mandatory comment from `C09` |
| The decision itself | decisions are taken with the inbox buttons; your app only records **why** |
| The work item text | `SAP_WAPI_WORKITEM_DESCRIPTION` returns exactly the block SAP GUI and inbox show. Change it in Customizing and your app follows with no change |
| The mandatory check | it sits in the core and applies in both interfaces and through the workflow API |
| The log | `S03`, `S04`, `S05` come about by themselves |

**An app that decides is the most expensive mistake of this design.** conFLOW stays the one place where agent determination, deadlines, substitution and the log come together. Your app shows and records — nothing more.

## Prerequisites

| | |
| --- | --- |
| ABAP | SAP_BASIS **7.50**, SAP_GWFND — no RAP needed |
| UI5 | SAPUI5 **1.71** or newer |
| Inbox | Fiori My Inbox (BSP `CA_FIORI_INBOX`) |

Without the Gateway component everything else keeps running: SAP GUI, email, conMOBILE. Only the Fiori UI is unavailable.
