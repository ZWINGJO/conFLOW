# Pitfalls

Collected from this reference and from production implementations. They
have one thing in common: **none of them produces an error message.**
That is why they cost time.

## While building

**1 · Forgetting `US`**

An entry in `CT_ACTORS` is always a typed org object -
`US<uname>`, `S<position>`, `AC<role>`. A bare user name creates no
work item and no error. It simply ends up with nobody.

**2 · Inline `DATA()` in a classic `CALL FUNCTION`**

An inline declaration is not allowed in a `CALL FUNCTION ... IMPORTING`.
Such calls are common in a conFLOW BAdI. At least this error shows up
on activation.

**3 · Writing to the container for an unknown instance**

`SET_ATTRIBUT_VALUE` looks up the instance ID in `/C09/CFL_S01` first.
If it isn't there - a wrong ID, or an ID from a different field - the
value is discarded: no error, no row in `S04`, and reading returns
empty. The element itself does not have to be declared anywhere.

**4 · The BAdI implementation without a filter**

Then the class runs for every workflow in the system.

## In the display

**5 · `<b>` instead of `<H>`**

The work item text is SAPscript ITF, not HTML - even though the
parameter type is called `/C09/CFL_HTML_TABLE_TT`. `<b>` is silently
removed: no bold, no visible tag, no message. Correct is `<H>Text</>`,
closed with `</>`.

**6 · Maintaining only one of the two object link hooks**

`GET_OBJECT_INFO` takes effect in Fiori, `GET_NEW_PREVIEW_DESCR` in
SAP GUI. Leave one out and the other UI shows the framework class name.

**7 · Cutting off the prefix without checking**

`CV_DESCRIPTION` arrives with 13 technical characters in front. If the
text is shorter, the offset runs into nothing - and the short dump
comes at display time.

**8 · Touching `DECISION_TEXT`**

In `GET_FIORI_TASK_DEC_OP_ACT` the framework checks right after the
call whether the text still contains a `-`, and otherwise skips the
`C09T` translation. Result: unlabeled buttons.

**9 · Recognizing buttons by their text**

At the time of the hook, the translated text is already there.
`DECISION_KEY` is the stable key.

## At runtime

**10 · `SAP_WAPI_CHANGE_WORKITEM_PRIO` in the after-create hook**

The WAPI reads `SWWWIHEAD` from the database. The work item isn't there
yet at this point. The call silently does nothing.

**11 · The missing `COMMIT` after `SET_WORKITEM_OBSOLET`**

The method calls `SAP_WAPI_WORKITEM_COMPLETE` with
`DO_COMMIT = FALSE`. Without the caller's `COMMIT` the work items stay
open.

**12 · `EV_DECISION_KEY` not set**

The most common reason for "the workflow hangs in the background step".

**13 · A local structure passed to `ADD_DATASOURCE_MAIL`**

The method works via RTTI and needs a DDIC header to build the
placeholder names. A local structure has none and is silently
ignored - the email then has gaps.

**14 · Formatted document numbers as search keys**

A read routine that formats for display is no good as a key source.
Without leading zeros the `SELECT` finds nothing - and reports no
error, but returns an empty table.

**15 · Believing where-used**

`GET_OBJECT_INFO` and `GET_FIORI_TASK_DEC_OP_ACT` look dead in the
where-used list, because they are called via enhancements. They run
anyway. For conFLOW BAdI hooks: **try it instead of trusting
where-used.**

**16 · Priority 1**

SAP sends work items with priority 1 as express messages. In the test
system nobody notices; in production the agent gets a popup. 4 is the
highest level without this effect.
