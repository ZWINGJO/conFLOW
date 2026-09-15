# `get_status_dynamic`

> **This hook stays empty in the reference class.** The reason is below.

| | |
|---|---|
| **When** | On every status change, after conFLOW has determined the next status from /C09/CFL_C02 - and before it uses it. |
| **In and out** | CS_DATA - the instance WITH the intended next status. Overwriting CS_DATA-GEN_STAT here overrides the customizing. CS_DATA_OLD holds the state before. |

**PURPOSE** Branches whose target is only known at runtime: approval levels by amount, skipping a level when it does not apply for business reasons, returning to the point from which the item was forwarded.

**THE RELATIONSHIP TO /C09/CFL_C02**

C02 says "after step 01 with outcome OK comes step B2". This hook says "unless ...". If you use it, you have a process flow that is NO LONGER VISIBLE in the customizing - that is the price, and it is high.

**HENCE THE ORDER OF QUESTIONS**

1. Does an additional status in C02 do the job?

2. Does a background step that branches via EV_DECISION_KEY do the job? (That is the clean way - the branch stays visible in C02.)

3. Only then this hook.

The sample process gets by with 2 - see BACKGROUND_CLASSIFY( ) at the very bottom, which does exactly that.

**WHAT TO KEEP IN MIND**

The hook runs on EVERY status change, including those that are none of your business. Without a switch at the start you rewrite cases you never meant to touch.

**STAYS EMPTY HERE - on purpose, see above. The pattern is**

commented out below, so you have it when you need it.

## The code

```abap
*   " Example: an amount limit decides whether to escalate
*   IF cs_data-gen_stat <> zcl_cfl_const_00900=>mc_stat-approve.
*     RETURN.
*   ENDIF.
*
*   IF is_true( get_val( iv_element = zcl_cfl_const_00900=>mc_prop-limit_hit
*                        iv_id      = cs_data-id ) ) = abap_true.
*     cs_data-gen_stat = zcl_cfl_const_00900=>mc_stat-escalate.
*   ELSE.
*     cs_data-gen_stat = zcl_cfl_const_00900=>mc_stat-post.
*   ENDIF.
```
