# The process

## An approval process, nothing more

In the vast majority of projects conFLOW is built for the same thing:
**something has to be approved, and depending on how large it is, by
someone else.** That is exactly the example process here. If you
understand it, you can rebuild it for purchase orders, invoices, leave
requests or contracts without touching the structure.

An example still needs a concrete document. The choice is the
**purchasing document** (`BUS2012`), because everybody knows it: header,
items, supplier, net value. That way the data part takes no attention,
and the focus stays on the workflow.

The process is intentionally small. It has exactly as many steps as it
takes for every hook to find a meaningful place - and no more.

> **What to replace when it becomes a different document:**
> `MC_OBJECTTYPE` in the constants class, `READ_DOCUMENT( )` and
> `EXECUTE_DEFAULT_METHOD`. Nothing else. The hooks don't know the
> document - they only know the container.

## The flow

```
X0  Start
 |
B1  Background: read document, check amount against the limit
 |
 +-- below the limit ----------------------------> X3  End, nothing to do
 |
01  Dialog: purchasing decides
 |
 +-- approved -------> B2  Background: write back ---> X1  End
 |
02  Dialog: supervisor decides (escalation)
 |
 +-- rejected -----------------------------------> X2  End, rejected
```

## The point of the flow

**The normal case creates no work.** If the document is below the
limit, the workflow closes itself in `B1` - and the log says why. That
is the difference between a workflow and an approval backlog.

**The branch lives in Customizing, not in code.** `B1` only sets a
decision key; where it leads is defined in `/C09/CFL_C02`. That is why
`GET_STATUS_DYNAMIC` stays empty in this example, even though the
process branches - see there.

## The steps

| Step | Kind | What happens |
|---|---|---|
| `X0` | Start | create the instance |
| `B1` | Background | read the document, values into the container, evaluate, set the switch |
| `01` | Dialog | purchasing: approve, reject, escalate |
| `02` | Dialog | supervisor: approve or reject |
| `B2` | Background | write back to the document |
| `B3` | Background | notification |
| `X1` / `X2` / `X3` | End | approved / rejected / nothing to do |

## The agent groups

| Key | Who | From where |
|---|---|---|
| `01` | buyer | role - `ZCL_CFL_GET_ACTORS` |
| `02` | supervisor | organizational structure - `RH_GET_ACTORS` |
| `03` | creator of the document | container attribute |
| `M1` | email recipients only | Customizing, no work item |

Four groups, because there are four **different kinds** of agent
determination. The process wouldn't need all of them - the example does.
