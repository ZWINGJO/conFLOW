# Technical documentation

This page describes conFLOW in full: the Customizing model, the runtime data, the step types, agent determination, email notification and the BAdI interface. It is meant to be read as a whole and replaces the earlier specification document.

{% hint style="info" %}
**Where else to look.** Every BAdI method on its own, with source code, is in the [reference of all 26 BAdI methods](how-to/badi-reference/README.md). For a workflow built end to end, see the [sick leave how-to](how-to/sick-leave-workflow/README.md).
{% endhint %}

---

## 1 Overview

conFLOW is a framework on top of SAP Business Workflow. An approval process is not built in the Workflow Builder but in Customizing tables: steps, agents, rules, document data, buttons and emails are settings. For genuine exceptions there is a BAdI interface. You do not need SAP Workflow skills to build one.

There is exactly **one** workflow template in the system for **all** conFLOW workflows. What an individual process does is not stored in a template of its own but in its Customizing rows. This is why there is no workflow development in the classic sense: no SWDD, and no transport dependency between a process change and the workflow definition.

```
Customizing  /C09/CFL_C*          what the process looks like
      │                           steps · transitions · agents · deadlines · mail
      ▼
conFLOW framework                 creates work items, resolves agents,
      │                           monitors deadlines, sends mail, logs
      ├──▶ BAdI class ZCL_CFL_WORKFLOW_<nnnnn>   optional: special logic, one class per workflow
      ▼
SAP Business Workflow             generic conFLOW tasks, SBWP, Fiori My Inbox
      │
      ▼
Runtime data  /C09/CFL_S*         instance · work item history · container
```

Every workflow has a five-digit number, for example `00900`. That number is the common thread: it identifies the workflow definition, it is the filter of the BAdI implementation, and it appears in every runtime row.

## 2 The Customizing model

The entry point is transaction `/C09/CONFLOW_C`, a view cluster covering all nodes.

| Node (EN logon) | Table | Content |
| --- | --- | --- |
| Workflow definition | `/C09/CFL_C06` | one row per workflow number; object label in `C06T-OBJTEXT` |
| Approval steps | `/C09/CFL_C01` | steps with status code `gen_stat`, attribute, texts, class/method, priority (`PRIO`), decision rule for several agents |
| Approval status - control | `/C09/CFL_C02` | transitions (OK / NOK / deadline), deadlines |
| Control additional status | `/C09/CFL_C09` | decision options `UC1`–`UC5` per step; button color and mandatory comment (also for OK/NOK); rules |
| User status definition | `/C09/CFL_C04` | roles, that is the agent keys `gen_stat_user` |
| User status assignment (current settings) | `/C09/CFL_C03` | who is behind a role: users, organization, PFCG role, exclusions |
| User status assignment (default - transport) | `/C09/CFL_C12` | the transportable default for that assignment |
| Assignment of user status | `/C09/CFL_C05` | which role processes which step |
| Control mail setting | `/C09/CFL_C07` | who receives which email on which decision |
| General parameters | `/C09/CFL_C08` | settings per workflow, see section 10 |
| Type linkages standard | `/C09/CFL_C10` | which event starts which workflow |

{% hint style="warning" %}
**Texts never belong in `C01`, `C06` or `C09` themselves, but always in the matching `*t` table.** If you look for the text in the main table you will not find it — and if you maintain it there, you lose it with the next language.
{% endhint %}

Every view of the maintenance dialog has a *Documentation* button: an explanation of that view appears on the right. With *Edit/extend own documentation* you can create your own versioned documentation for each workflow definition; once it exists, it appears above the standard documentation. Documents can also be uploaded through the GOS integration, at the node *General parameters* in the maintenance tree.

## 3 Starting a workflow

There are three ways. The first one is the recommended one.

**Through an event and the type linkage.** `/C09/CFL_C10` defines which event starts which workflow. An entry links object category, object type, event and the receiver type `CONFLOW` to a workflow definition. For the event to reach conFLOW at all, you also need the **event type linkage** in the SAP standard (`SWETYPV`). There you register the object type, the event and the receiver type `CONFLOW`:

| Setting | Value |
| --- | --- |
| Receiver call | function module |
| Receiver function module | **`/C09/CFL_WI_CREATE_0101`** (for class-based events `/C09/CFL_WI_CREATE_IBF_0101`) |
| Event delivery | using tRFC (default) |
| Linkage activated | set |

Without that entry nothing happens when the event is raised — the conFLOW Customizing on its own does not start a workflow. The fields on the conFLOW side:

| Field | Meaning |
| --- | --- |
| `WF_DEFINITION` | the workflow definition that is started |
| `OBJCATEG` | object category, `BO` for BOR objects |
| `OBJTYPE` | object type, e.g. `BUS2012` |
| `EVENT` | event of the object type |
| `RECTYPE` | receiver type, `CONFLOW` |
| `EXECUTE_FIRST` | labelled *Step 1 auto* — the first step is confirmed automatically, the workflow starts at the second |

{% hint style="warning" %}
**`EXECUTE_FIRST` only takes effect if `GEN_TASK` is maintained in the general parameters.** Without the generic conFLOW task the framework has no task to confirm automatically — the flag is set but does nothing.
{% endhint %}

{% hint style="warning" %}
**Object type, event and receiver type must be unique.** A second entry with the same combination may start the wrong workflow. And: **only one open workflow per object and workflow definition is possible.** When creating a workflow, the framework looks for an open instance with the same object key, object type and workflow definition; if it finds one, the start is rejected with a message instead of creating a second workflow. A *different* workflow definition may run on the same object at the same time.
{% endhint %}

**Through your own event from a user exit, BAdI or enhancement.** If no standard event fits, you can raise the workflow yourself when an object is saved. The call optionally carries container values that are then available from the first step on:

```abap
DATA: ls_sweinstcou TYPE /c09/cfl_sweinstcou_st,
      ls_swhactor   TYPE swhactor,
      lt_container  TYPE swconttab,
      ls_container  LIKE LINE OF lt_container.

ls_sweinstcou-instid  = lv_document_number.
ls_sweinstcou-objtype = 'BUS2012'.
ls_sweinstcou-event   = 'CHANGED'.
ls_sweinstcou-rectype = 'CONFLOW'.

" optional - SY-UNAME is used if you leave this out
ls_swhactor-otype = 'US'.
ls_swhactor-objid = sy-uname.

ls_container-element = 'AMOUNT'.
ls_container-value   = lv_amount.
APPEND ls_container TO lt_container.

/c09/cfl_cl_workflow_0101=>start_workflow_int(
  is_sweinstcou = ls_sweinstcou
  is_creator    = ls_swhactor
  it_container  = lt_container ).
```

**Directly, without an event.** Possible through `/c09/cfl_cl_workflow_0101=>start_workflow_extern`, but not the preferred way: the start then depends on the calling code instead of on the document event.

{% hint style="info" %}
`start_workflow_extern` checks by itself whether a workflow is already open for the object. If one is, a message is returned in `ET_BAPIRET2` and no second workflow is started.
{% endhint %}

## 4 Steps and step types

Every step has a two-character status code `gen_stat`. The first character decides what kind of step it is.

| Code | Meaning |
| --- | --- |
| `01`, `02`, … | process steps, business meaning per workflow from `C01` |
| `X0` | workflow start |
| `X1` | end — approved |
| `X2` | end — withdrawn |
| `X3` | end — rejected |
| `Y1`–`Y5` | branch points: the follow-up status is determined at runtime, see below |
| `B*` | background steps, run without a user |

**Every status starting with `X` ends the workflow.** The framework checks the first character and sets the `wf_end` flag. The meaning of `X1`, `X2` and `X3` is convention — further `X*` statuses are allowed.

**`X0` is the only status hard-coded in the framework.** `C02` needs a row with `gen_stat = 'X0'` whose OK outcome points to the first real step. That step may be a background step.

In the *Attribute* column (`ATTRIBUT`) of the approval steps, five fixed values control the behavior. The normal case is the empty value:

| Attribute | Effect |
| --- | --- |
| *(empty)* | standard: user decision — the agent decides in the work item |
| `BACK` | background task in the **update task** — runs the assigned method, no work item |
| `BACK_BATCH` | background task in **batch** |
| `WAIT` | wait step for parallel processing: the workflow waits until all triggered subworkflows are finished |
| `BADI` | the next step is determined dynamically through the BAdI, as with a `Y` step |

The class and method of a step are stored in `CLSNAME` and `CMPNAME`, the SO10 text for the work item text in `TDNAME`.

### Branch points: `Y` steps

A step whose code starts with `Y` is a decision point without an agent. Once conFLOW has determined the follow-up status from `C02`, it looks at that step: if its code starts with `Y` — or if it carries the attribute `BADI` — the class/method assigned in `C01` runs first, and then the BAdI method `GET_STATUS_DYNAMIC` is called. Both receive the intended status and the previous state, and both may overwrite the intended status.

{% hint style="warning" %}
**`GET_STATUS_DYNAMIC` does not run on every status change.** The framework calls the hook in exactly one place, and only under this condition: the target step starts with `Y`, or the target step carries the attribute `BADI`. If you implement the hook without setting up the step accordingly, you will wait for a call that never comes.
{% endhint %}

The price of a branch point: at that place the process flow is no longer in Customizing but in code. So check in this order whether an additional status in `C09`, a rule on a background step (section 7) or a background step returning `EV_DECISION_KEY` already does the job — with all three, the branch stays visible in `C02`.

## 5 Outcomes and decision paths

`C02` gives every step exactly three outcomes: OK, NOK and deadline expiry. Everything beyond that is in `C09`.

| Decision | Key | Follow-up status from |
| --- | --- | --- |
| OK | `0001` | `C02-gen_stat_ok` |
| NOK | `0002` | `C02-gen_stat_nok` |
| deadline expiry | — | `C02-gen_stat_frist` |
| `UC1` | `0003` | `C09-gen_stat_ok` |
| `UC2` | `0004` | `C09-gen_stat_ok` |
| `UC3` | `0005` | `C09-gen_stat_ok` |
| `UC4` | `0006` | `C09-gen_stat_ok` |
| `UC5` | `0007` | `C09-gen_stat_ok` |

That gives every step up to seven outcomes. The key of `C09` is `(wf_definition, gen_stat, gen_decision)` — **one row per step.** If the row is missing, pressing the button does nothing, without an error message.

The checkbox *no display* (`NODISPLAY`) hides a decision option. The outcome still exists and can be set from code, but the agent is not offered a button for it. That is the way to model technical outcomes nobody should pick by hand.

`C09` carries three further columns: `NATURE` colors the button green (`P`) or red (`N`), `COMMENT_REQ` makes a comment mandatory — both in SAP GUI and in Fiori My Inbox, also for OK and NOK. `BEDINGUNG` holds a rule for the outcome, see section 7.

### A loop instead of a restart

A recurring pattern: a document goes back for rework and is to be processed again.

| Table | Step | Decision | Follow-up status |
| --- | --- | --- | --- |
| `C09` | `01` | `UC4` | `B1` |
| `C02` | `B1` | OK | `01` — new work item, same instance |

The background step `B1` is deliberately empty. Its purpose is not logic but reporting: every pass leaves a row in the history. A direct `01 → 01` could not be told apart from an ordinary resubmission later; with `B1` you can count how often a document went back for rework.

## 6 Agent determination

Two keys that must not be confused:

- **`gen_stat`** — *where* the process stands, that is the step code.
- **`gen_stat_user`** — *who* is up next, that is the agent key. Several steps may share the same agent key.

`C04` defines the roles, `C05` defines which role processes which step, and `C03` holds who is actually behind a role. The options:

| Setting | Meaning |
| --- | --- |
| `WF_INITIATOR` | the creator of the workflow |
| SAP user (`US`) | a fixed user |
| Organizational unit / position | resolved through the organizational structure |
| Email address | for email notification only, no work item |
| PFCG role (`AG` with `AGR_NAME`) | all dialog users of the role; users locked by the administrator are left out |
| Exclude (`EXCLUDE`) | whatever the row resolves is removed from the same agent key — e.g. `WF_INITIATOR` for the four-eyes principle. Special value `WF_APPROVERS`: whoever has already decided at another stage of this instance |
| BAdI | `GET_ACTORS` does the resolution — the way to BRFplus, custom tables or rule sets |

Three agent keys have a fixed meaning: `BU` for background steps, `WI` for the initiator and `$$` internally for deadline steps.

{% hint style="warning" %}
**`EXCLUDE` controls agent determination, not authorization.** A decision taken through workflow administration (`SWIA`) is not blocked by it.
{% endhint %}

`C12` holds the same assignment as `C03`, but transportable. `C03` is a current setting maintained in the target system; `C12` ships the default with the transport.

### Parallel agent paths

Several agent keys on one step mean several agents. The framework puts the applicable roles into the container element `RT_NUMBER_ACTORS` as a list — **one work item per row**, all created at the same time.

What happens with the individual decisions is set on the step: in **Approval steps** (`C01`), column **Decision rule** (short label *Rule*).

<figure><img src=".gitbook/assets/c01-decision-rule.png" alt="Column Decision rule in Approval steps"><figcaption><p>Four steps, four rules</p></figcaption></figure>

| Value | Rule | What happens |
| --- | --- | --- |
| *(blank)* | **All decide** | Every agent decides their own work item. Then the process continues; one rejection results in NOK. This is the previous behaviour — existing steps are unchanged |
| `V` | **Veto** | The first rejection ends the step with NOK. The remaining work items are closed. If nobody rejects, the result is the same as with “All decide” |
| `E` | **First decision counts** | Whoever decides first decides for everyone — OK, NOK or an additional outcome `UC1`–`UC5`. The remaining work items are closed |
| `M` | **Majority decides** | Everyone decides, the most frequent decision wins. OK, NOK and `UC1`–`UC5` all count. A tie results in NOK |

The follow-up step **and** the email follow the result of the rule. With "Majority", two approvals and one rejection take the OK path, and the OK email goes out, not the rejection email.

Worth knowing:

- **Closed means obsolete.** The closed work items disappear from the inboxes a few seconds after the deciding vote. The workflow log shows them as *obsolete*.
- **Agents whose work item was closed get no separate notification.** If they should know, set up an email rule on the result of the step.
- **If a step runs again**, for instance after a query, only the decisions of the new pass count.
- **If two people decide at the same moment under "First decision counts"**, the majority of those two counts; a tie results in NOK.

{% hint style="info" %}
**Work items not closing under Veto or "First decision counts"?** Closing runs in a separate step (tRFC) after the decision is saved. Transaction `SM58` shows where it is stuck.
{% endhint %}

### Custom logic in the BAdI

For rules none of the four settings covers — a weighted vote, say, or "two out of three from different departments" — the BAdI remains. Leave the decision rule blank in that case.

To close individual work items, there is a helper:

```abap
" in the hook GET_AFTER_EXECUTION_WORKITEM, which runs after a work item is completed
/c09/cfl_cl_helper_0101=>set_workitem_obsolet( is_data_step = is_data_step ).
COMMIT WORK AND WAIT.
```

`SET_WORKITEM_OBSOLET` looks for all open dialog work items of the same top workflow and sets them to *obsolete* — except your own. If this should only happen on a particular outcome, check `IV_KEY` first.

{% hint style="warning" %}
**Mind the scope.** The helper clears the **entire workflow**. If parallel work items exist in several steps at the same time, it also hits the ones you wanted to keep. In that case select the rows yourself and restrict on `GEN_STAT` — the pattern is in the [how-to](how-to/sick-leave-workflow/step-4-start-and-test.md).
{% endhint %}

A custom count goes into a **collecting step** after the parallel step — a `Y` step that all outcomes point to. There, `GET_STATUS_DYNAMIC` counts the workflow log and sets the follow-up status. The [how-to](how-to/sick-leave-workflow/step-4-start-and-test.md) shows the building block.

{% hint style="warning" %}
**The `COMMIT` is up to the caller.** The helper calls `SAP_WAPI_WORKITEM_COMPLETE` with `DO_COMMIT = FALSE` on purpose, so that it does not commit once per work item. Without that line the work items stay open — **with no error message**.
{% endhint %}

The hook in detail, including how it differs from `GET_AFTER_EXECUTION`, is described in the [BAdI reference](how-to/badi-reference/lifecycle/get-after-execution-workitem.md).

## 7 Background steps

A background step runs a static method, without a work item and without a user. Class and method are stored in `C01`. The method receives the workflow instance and determines how the process continues:

- `EV_DECISION_KEY` set → that outcome is taken (`0001` OK, `0002` NOK, `0003`–`0007` for `UC1`–`UC5`).
- `EV_DECISION_KEY` not set → `ET_BAPIRET2` decides: a message of type E or A sends the workflow down the NOK path, otherwise it continues with OK.

If a step carrying attribute `BACK` has no method assigned at all, it passes through positively without doing anything.

Instead of a static method, `CLSNAME` can also hold a class that implements the interface `/C09/CFL_IF_BACKGROUND_0101`; `CMPNAME` then stays empty. In that case the compiler checks the signature.

**Rules instead of a method.** If a `BACK_BATCH` step has no class assigned, conFLOW checks the conditions in `C09-BEDINGUNG` of the outcomes `UC1`–`UC5`, in that order. The first match wins; without a match the step continues with OK, on an error with NOK. The fields come from the template (`TEMPLATE` in `C08`). Decimal numbers go in quotes with a point: `GESAMTWERT_RW <= '1000.20'`.

Messages from a background step go to the application log (SLG1). **This happens only if an `OBJECT` is maintained in the general parameters** — without that entry the step runs, but nothing is logged.

The other way round: if a step is *not* marked as a background step and no class/method is maintained either, conFLOW generates a standard decision task. If you want to show a screen of your own instead, assign a class/method here as well; its `EV_DECISION_KEY` then controls the outcome.

## 8 Deadlines and escalation

Deadlines are maintained in `C02`, in three fields: `FRIST_STUNDEN` the value, `FRIST_MSEHI` the unit and `GEN_STAT_FRIST` the status to switch to on expiry. No deadline agents, no workflow Customizing in SPRO — one table row.

{% hint style="warning" %}
**The field name `FRIST_STUNDEN` is misleading.** The unit is freely selectable and stored in `FRIST_MSEHI`; in the maintenance view the column is therefore simply labelled *number*. If you read the field name and assume hours, you will calculate wrongly.
{% endhint %}

Which factory calendar applies to the calculation can be set through the BAdI method `GET_FACTORY_CALENDAR`.

## 9 Email notification

conFLOW sends HTML emails, controlled through `C07`. The key has four parts — **one row per approval step, decision and recipient role**:

| Field | Meaning |
| --- | --- |
| `GEN_STAT` | at which step the email is sent |
| `GEN_DECISION` | which decision triggers the email |
| `GEN_STAT_USER` | who receives the email |
| `SUBJECT` | SO10 text for the subject line |
| `OBJID_HEADER` / `OBJID_ITEM` / `OBJID_FOOTER` | the three HTML templates the email is built from |
| `TDNAME` | SO10 text for the content |

{% hint style="warning" %}
**The three templates are web objects from `SMW0`, not SO10 texts.** The subject and the content are SO10 texts, the templates are not — if you look for them in SO10 you will not find them.
{% endhint %}

The texts and templates contain placeholders of the form `&STRUCTURE-FIELD&` that are replaced while the email is built. If a `TEMPLATE` is maintained in the general parameters, its fields are available without code, e.g. `&/C09/CFL_S_TPL_BUS2012-GESAMTWERT_RW&`; amounts and quantities are formatted according to currency and unit. Anything else, such as the log (`&WF_PROT&`) or notes (`&NOTE&`), comes from the BAdI via `GET_DATASOURCE_MAIL`.

**Language per recipient:** the BAdI method `GET_MAIL_LANGUAGE` sets the language for each recipient. It only switches if the language is installed and the subject and texts exist in it; otherwise the email goes out in the original language.

**Dynamic recipients:** if the agent key in `C07` starts with `Y`, the framework calls the BAdI method `GET_STATUS_MAIL_DYNAMIC`. That method may not only change the recipient but return a whole table of recipients — the way to model distribution lists that are only known at runtime. If the method returns nothing, the maintained recipient stands.

`Y` means the same thing in both places: *ask the BAdI*. In the step code it leads to `GET_STATUS_DYNAMIC`, in the recipient key of the email notification to `GET_STATUS_MAIL_DYNAMIC`.

## 10 General parameters and inheritance

`C08` holds settings per workflow definition. The key is the workflow number plus the parameter name `PARAM`, the value is stored in `VALUE`. The permitted names are fixed values of a domain — a new parameter is a new fixed value and not a table change.

| Parameter | Meaning |
| --- | --- |
| `OBJECT` / `SUBOBJECT` | object and subobject of the application log for background steps |
| `WF_DEF` | inheritance: which workflow definition this workflow inherits its Customizing from |
| `GEN_TASK` | the generic conFLOW task, required to confirm the first step automatically |
| `LICENSE` | license information |
| `REPPR` | substitute profile |
| `TCLASS` | classification of tasks for the substitution rules |
| `TEMPLATE` | template class: reads the document and supplies the fields for rules, work item title and email. Shipped for `BUS2012`, `BUS2032`, `BUS2105`, `BUS2081`, `BKPF`, `LFA1`, `KNA1`, `BUS1006`; your own via inheritance or append |
| `RULE_CURR` | rule currency |
| `RATE_TYPE` | exchange rate type for rules |
| `VISU` | semantic object of the Fiori app that opens the work item in My Inbox (action fixed to `openInInbox`). Empty = standard display. Existing work items are updated by report `/C09/CFL_MIGRATE_VISU` |

### What `WF_DEF` inherits — and what it does not

A workflow with `WF_DEF` maintained takes over the Customizing of the parent definition. Inherited are `C01`, `C02`, `C03`, `C04`, `C05`, `C07` and `C09` together with their text tables.

{% hint style="warning" %}
**The general parameters themselves (`C08`) and the definition text (`C06T`) are not inherited.** An inheriting workflow therefore has the steps and outcomes of its parent, but not its application log object, `TEMPLATE`, `VISU` or `GEN_TASK` — these values are maintained in each definition. If you rely on it, you get a workflow that runs but, without its own `OBJECT`, logs nothing and, without its own `TEMPLATE`, evaluates no rules.
{% endhint %}

Your own rows win: inherited rows are appended at the end, even when your own definition already has the same key. An access therefore always hits your own row first.

## 11 Subworkflows

In *Assignment of user status* you can start a subworkflow per decision: `WF_DEFINITION_OK` (labelled *Definition OK*) on decision OK, `WF_DEFINITION_NOK` (*Definition NOK*) on NOK. The subworkflow is a workflow definition of its own with an instance of its own.

The key of `C05` includes the sort field `SORTF`. **Several rows per approval step** are therefore possible — each with its own role and its own subworkflow.

{% hint style="warning" %}
**If the main workflow is to wait for the subworkflow, it needs a wait step.** Without a step carrying attribute `WAIT`, the main workflow continues while the subworkflow is still open. The wait step moves on only once no triggered workflow is open any more.
{% endhint %}

## 12 The runtime data model

Four tables, linked by the instance `id`:

| Table | One row per | Important fields |
| --- | --- | --- |
| `/C09/CFL_S01` | workflow instance | `id`, `wf_definition`, `instid` (object key), `gen_stat` (current step), `wf_end` |
| `/C09/CFL_S03` | work item, chronologically | `id`, `wi_id`, `gen_stat`, `gen_stat_user`, creator and time |
| `/C09/CFL_S04` | container element | `id`, `element`, `tab_index`, `value` |
| `/C09/CFL_S05` | change to a container value | `id`, `element`, `tstmp`, `wert_alt`, `wert_neu`, `aenam`, `kanal` — only on an actual change |

The key of `S04` includes `TAB_INDEX` — one element can therefore carry **several values**, not just one.

This gives you a complete audit trail without a custom table: who processed which step when and with which outcome, and which data was available at the time of the decision.

{% hint style="info" %}
**The entry point when tracing a problem** is almost always the same path: `wi_id` → `/C09/CFL_S03` → `id` → `/C09/CFL_S01`.
{% endhint %}

## 13 The container and passing data

The container `/C09/CFL_S04` stores any attribute-value pairs per instance. It is written and read with the framework methods `SET_ATTRIBUT_VALUE` and `GET_ATTRIBUT_VALUE`. Values can be passed in at start (see section 3) or created in any step.

The container is also the natural source for placeholders in work item texts and emails — and the reason the audit trail comes for free: whatever is stored there can be traced later.

## 14 The BAdI interface

The interface `/C09/CFL_IF_BADI_0101` defines the places where special logic plugs in. If a workflow needs it, its own class `ZCL_CFL_WORKFLOW_<nnnnn>` implements this interface; the filter of the implementation is the workflow number. Hooks you do not need stay empty.

The standard cases need none of these hooks: agents (`C03`), title and placeholders (`C01T`, `C06T-OBJTEXT`), button color and mandatory comment (`C09`) and document data for email and rules (`TEMPLATE`) are settings. The hooks are for whatever goes beyond that. The most important ones:

| Hook | Purpose |
| --- | --- |
| `GET_ACTORS` | agent determination |
| `GET_DESCRIPTION` / `GET_WORKITEM_TEXT` | work item title and text |
| `GET_BEFORE_DECISION_WORKITEM` | control the buttons |
| `EXECUTE_DEFAULT_METHOD` | navigation to the business object |
| `GET_STATUS_DYNAMIC` | determine the follow-up status in code (only on `Y` steps, see section 4) |
| `GET_STATUS_MAIL_DYNAMIC` | determine email recipients in code |
| `GET_DATASOURCE_MAIL` | values for the placeholders in the mail text |
| `GET_FACTORY_CALENDAR` | factory calendar for deadline calculation |

{% hint style="info" %}
**All 26 methods, each with its purpose, signature, source code and a note on when it is better left empty,** are in the [BAdI reference](how-to/badi-reference/README.md).
{% endhint %}

## 15 User interfaces

Work items appear in the SAP Business Workplace (`SBWP`) and in SAP Fiori My Inbox. conFLOW controls label, color and mandatory comment of the buttons (`C09`/`C09T`), the title and navigation to the business object in both from the same Customizing. Which Fiori app a step opens in My Inbox is set by the parameter `VISU`. The BAdI remains for deviations.

On top of that, every step can be displayed on a mobile device: a conMOBILE app reads the same container and uses the same decision keys. One data source, one decision model — no matter where the decision is made.

If you need input fields of your own on the work item in Fiori My Inbox, the [how-to on the Fiori UI](how-to/fiori-ui-in-work-item/README.md) shows the way that was actually built.

## 16 Transactions and the admin console

| Transaction | Purpose |
| --- | --- |
| `/C09/CONFLOW_C` | conFLOW Customizing — the view cluster across all nodes |
| `/C09/CFL_ADMIN_CON` | admin console: running and completed workflows at a glance |
| `/C09/CFL_START_WF_TE` | start a workflow for test purposes |

{% hint style="warning" %}
**`/C09/CONFLOW_C` starts in display mode.** The transaction calls the view cluster with the display flag set. To maintain, switch to change mode after entering.
{% endhint %}

### The admin console

The console answers the questions that come up in daily operations: what is running, where is it stuck, and who would have to act.

**You can restrict** by workflow definition, instance and object type, by creation date and time of the work item, by agent, step code and work item status, and by work item text. Two switches decide whether running, completed or both kinds of workflows are shown. You can switch between a **header view** per workflow instance and an **item view** per work item.

**The list** shows work item number, text and status, creation and change date, the agent in clear text, who forwarded it, the decision taken, the details of the email sent, priority and how long the item has been lying around. A traffic light rates open work items against a threshold in days (three by default): green below, yellow from 80 %, red from the threshold; a separate icon marks orphaned work items without an agent. A threshold of 0 switches the rating off.

**From the list** you can display and execute the work item, show the actual agents, open the email that was sent, forward a work item and cancel a workflow. Cancellations are logged.

**Instead of the list** you can display an **evaluation**: per workflow definition and step, the number of items in total, open and completed, the longest and average waiting time of the open ones, the longest and average throughput time of the completed ones, and the distribution of decisions including the rejection rate, with the same traffic light per row. This is the fastest way to answer which step a process really gets stuck at.

## 17 Objects in the system

The enhancement interface:

| Object | Name |
| --- | --- |
| Enhancement spot | `/C09/CFL_ENHANCEMENT_0101` |
| BAdI definition | `/C09/CFL_BADI_0101` — multiple use, no fallback class |
| Interface | `/C09/CFL_IF_BADI_0101` |
| Filter | `WF_DEFINITION` — the workflow number |

{% hint style="warning" %}
**`/C09/CFL_CL_BADI_0101` is the sample implementation that ships with conFLOW, not the BAdI definition and not your implementation.** It is registered in the spot as the sample class and shows fragments from real projects. As a template for your own implementation, use the [reference of all 26 BAdI methods](how-to/badi-reference/README.md). Your own logic belongs in a class of your own, `ZCL_CFL_WORKFLOW_<nnnnn>`, filtered on your workflow number.
{% endhint %}

The classes you meet most often when tracing errors and when extending:

| Area | Classes |
| --- | --- |
| Process | `/C09/CFL_CL_WORKFLOW_0101` (start, control, cancel), `/C09/CFL_CL_WORKFLOW_EXIT_0101` (work item exit), `/C09/CFL_CL_PRIORITY_0101` (priority per step) |
| Agents and rules | `/C09/CFL_CL_ACTORS_0101`, `/C09/CFL_CL_RULE_0101`, `/C09/CFL_CL_DECIKEY_0101`, `/C09/CFL_CL_PARALLEL_0101` (decision rule for several agents) |
| Buttons and UI | `/C09/CFL_CL_BUTTONS_0101` (color, mandatory comment), `/C09/CFL_CL_VISU_0101` (Fiori app per workflow) |
| Mail and texts | `/C09/CFL_CL_MAIL_0101`, `/C09/CFL_CL_MAIL_LANG_0101`, `/C09/CFL_CL_TEXTPARSER_0101`, `/C09/CFL_CL_OBJTEXT_0101`, `/C09/CFL_CL_FORMAT_0101` (numbers, quantities, dates) |
| Templates | `/C09/CFL_CL_TPL_BASE_0101` (base) and per object type `/C09/CFL_CL_TPL_<type>_0101`, e.g. `…_TPL_BUS2012_0101`; `/C09/CFL_CL_TPL_MAIL_0101` (template as mail data source) |
| Extension | `/C09/CFL_IF_BADI_0101`, `/C09/CFL_IF_BACKGROUND_0101`, `/C09/CFL_IF_TEMPLATE_0101` |

For common business objects, conFLOW ships ready-made **templates** that can be used directly through the parameter `TEMPLATE`: purchase order, purchase requisition, sales order, incoming invoice, business partner, FI document header, customer and vendor. Your own fields are added via inheritance or append.

For long-term storage there is an **archiving object** of its own with a write and a delete program, used to move completed workflows out of the runtime tables.
