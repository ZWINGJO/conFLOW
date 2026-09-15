# `get_status_mail_dynamic`

> **This hook stays empty in the reference class.** The reason is below.

| | |
|---|---|
| **When** | During mail dispatch, once it is fixed WHICH agent group is notified. |
| **In** | IS_DATA - the instance |
| **In and out** | CS_CFL_C05 (the intended group) and CT_CFL_C05 (the list of groups) - here you can turn one into several or replace it |

**PURPOSE** The counterpart to GET_STATUS_DYNAMIC, but for mail dispatch: "Who gets a mail depends on what has just happened."

**THE CHECK FOR BACKGROUND STEPS IS THE TRICK**

`is_data-gen_stat+0(1) = 'B'` tells dialog from batch - that is the reason for the naming convention that background steps start with B. Without this line you send notifications about steps that nobody has seen.

STAYS EMPTY HERE, because the sample process has a fixed recipient group per step. The pattern is below.

## The code

```abap
*   " Example: on escalation, also inform the original
*   " agent
*   IF is_data-gen_stat+0(1) = 'B'.
*     RETURN.                      " background step - no mail
*   ENDIF.
*
*   IF is_data-gen_stat <> zcl_cfl_const_00900=>mc_stat-escalate.
*     RETURN.
*   ENDIF.
*
*   CLEAR ct_cfl_c05.
*   APPEND cs_cfl_c05 TO ct_cfl_c05.
*   APPEND INITIAL LINE TO ct_cfl_c05 ASSIGNING FIELD-SYMBOL(<fs_c05>).
*   <fs_c05>               = cs_cfl_c05.
*   <fs_c05>-gen_stat_user = zcl_cfl_const_00900=>mc_gsu-buyer.
```
