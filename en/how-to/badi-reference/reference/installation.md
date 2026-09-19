# Installation

The class alone does nothing. conFLOW finds it through the BAdI
implementation, and what it should do is defined in Customizing. This
order has proven itself, because each step requires the previous one.

## The order

**1 · Package and constants class**

`ZCL_CFL_CONST_00900` first. It has no dependencies, and everything
else refers to it. If you create it later, you type the Customizing
keys as literals twenty times.

**2 · Create the Customizing**

| Table | What |
|---|---|
| `C06` / `C06T` | the workflow definition and its text |
| `C01` / `C01T` | the steps - one each for `X0`, `B1`, `01`, `02`, `B2`, `B3`, `X1`, `X2`, `X3`; if needed priority (`PRIO`) and decision rule (`DECI_RULE`) |
| `C02` | the transitions: step + decision → next step |
| `C05` | the agent groups per step |
| `C03` | who the group is - users, PFCG role (`AGR_NAME`), exclusion (`EXCLUDE`) - or `USER_BADI = 'X'` for determination in code |
| `C07` | email notification: who receives which mail at which step |
| `C09` / `C09T` | the decisions `UC1`–`UC5` with colour, mandatory comment and rule; the texts in `C09T` |
| `C08` | object and subobject for the application log, if needed `TEMPLATE` and `VISU` |
| `C10` | the type linkage, if the start runs via an event |

For the background step `B1`, class name and method name go into
`C01` - the call is dynamic.

**3 · The workflow class**

Create `ZCL_CFL_WORKFLOW_00900`, replacing `00900` with your own
number. Three places: class name, constants class, BAdI filter.

Don't forget the text symbol `TEXT-001` - it labels the object link. It
belongs to the text pool, not to the code, and must be transported too.

**4 · The BAdI implementation**

For the definition `/C09/CFL_BADI_0101`, filter `WF_DEFINITION = '00900'`.

> **Without a filter the class runs for every workflow in the system.**
> The mistake goes unnoticed in a development system with one workflow
> and shows up immediately in production.

**5 · Agent determination**

For a PFCG role, `C03` with `AGR_NAME` is enough; `ZCL_CFL_GET_ACTORS`
only for special cases. Only then does a test make
sense - a work item without an agent goes into error status.

**6 · SLG0**

Object and subobject from `C08` must exist in SLG0, otherwise conFLOW
writes the messages of the background step nowhere.

## The first test

Use a document **above** the limit. The other case closes itself, and
then you see nothing.

| | expected |
|---|---|
| `/C09/CFL_S01` | an instance with `INSTID` = document number |
| `/C09/CFL_S04` | the container values from `B1` |
| SLG1 | a line "Approval required - ..." |
| Business Workplace | a work item for the users of the role |
| Workflow log | for the background step, the calculated text, not the one from Customizing |

If `S04` stays empty, check first whether `B1` ran at all (workflow log, SLG1).
`SET_ATTRIBUT_VALUE` writes only for an instance that exists in `/C09/CFL_S01`.
