# Deploy, check, kill switch

*A check after every step. If it fails, the next step isn't worth doing — and the cause is still within reach.*

## Order with checks

| # | Step | Check |
| --- | --- | --- |
| 1 | Check SWFVMD1 | `SE16` → `SWFVGTP`, `TASK = TS00388601`: `SEMANTIC_OBJECT`, `ACTION`, `QUERY_PARAM00` present and dynamic |
| 2 | Create the data class, declare it a `GLOBAL FRIEND` in the workflow class | `SE24` → test method `READ` with the instance GUID: text, reasons, `Editable` are returned |
| 3 | SEGW model, generate, fill DPC_EXT | all classes active |
| 4 | Register the service | `/sap/opu/odata/sap/ZCFL_00500_V2_SRV/$metadata` shows `Decision`, `DecisionSet`, `SetDecision` |
| 5 | Service, functionally | `…/DecisionSet('<id>')?$format=json` returns the instance; a foreign GUID returns **404** |
| 6 | Build and upload the app, caches | `/sap/bc/ui5_ui5/sap/zcfl_00500_uiv2/index.html` says "No work item key was passed". **That is the success case.** |
| 7 | Semantic object, target mapping, role | `…/flp#ZCFLOrderPromiseV2-openInInbox?CFLQueryObject00=<id>` opens the instance |
| 8 | maintain `VISU` in `C08` (or BAdI `set_inbox_ui( )`) | create a **new** work item; the work item container holds the three `/C09/CFL_VISU_*` values |
| 9 | My Inbox | app in the detail area, conFLOW buttons below, "Show Details" in the footer |

`<id>` is the conFLOW instance (`/C09/CFL_S03-ID`) of an open dialog work item, as a 32-character hex string.

## Acceptance in the work item

| | Expected |
| --- | --- |
| Select a reason | status line green: *Saved hh:mm:ss* |
| Type a note | after a typing pause *Saved* again |
| Required input missing | saved **and** a yellow hint about what is missing for completion |
| Step without input (e.g. escalation) | fields grayed out, the line says why |
| Click a second work item | title, text **and** fields change, not only the buttons |
| Other user, foreign GUID | nothing to see, nothing to write |
| `/C09/CFL_S04` | reason and note are stored, `AENAM` is the agent |

## Pitfalls

| Symptom | Cause |
| --- | --- |
| Work item still shows the **text block** | work item older than the change · semantic object spelled differently · `openMode` as *Default Value* · role missing · SWFVMD1 empty |
| Detail area stays **white** | app error on startup → browser console. Cross-check: set `openMode` to `embedIntoDetails` (then without "Show Details") |
| On the second work item **only the buttons change** | `navigateBasedOnStartupParameter( )` missing in `Component.js` |
| App shows an **old** version | app index and site data, see [Customizing](customizing.md) |
| A field stays **empty**, no error | ABAP field name in SEGW ≠ component of the structure (`MOVE-CORRESPONDING`) · metadata cache not cleared |
| Long text **truncated** | `Edm.String` **with** max length generates `CHAR(n)`; leave the field empty for text without a fixed length |
| Note **cut off** after 132 characters | limit of `/C09/CFL_S04-VALUE` |
| Saving reports an error **without text** | `/IWFND/ERROR_LOG` — text, program and line are there |
| Function module fails at runtime, syntax check was green | parameter passed as `TYPE i` instead of the module's DDIC type |
| A file changed in SE80 **has no effect** | the app runs from `Component-preload.js`; change only via build and upload |

{% hint style="danger" %}
**Before rebuilding anything that sounds like work item standard** (attachments, notes, object links), click **"Show Details"** first. The My Inbox behavior described here is **observed, not guaranteed**: it comes from its source code, not from an extension interface. After a UI5 or S/4HANA upgrade, re-check `openMode`, "Show Details" and the work item switch.
{% endhint %}

## Kill switch and removal

| Goal | Action | Affects |
| --- | --- | --- |
| gone immediately, including open work items | delete the target mapping in the catalog | **all** work items, no transport |
| new work items without the app | clear `VISU` in `C08` (for the BAdI route `mc_inbox_ui = abap_false`) | only **new** work items |
| remove completely | target mapping, semantic object, BSP application, service, SEGW project, data class, parameter `VISU` or call in the BAdI | — |

{% hint style="success" %}
**The fallback is not an emergency solution but the normal case** for every work item without the three container elements. That is why it works reliably: it is in use all the time, not only when something breaks.
{% endhint %}
