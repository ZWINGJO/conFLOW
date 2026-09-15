# `get_send_mail_user`

> **This hook stays empty in the reference class.** The reason is below.

| | |
|---|---|
| **When** | Before mail dispatch, once the recipient group is fixed. |
| **In** | IS_DATA   the instance |
| **Out** | CT_USER   the recipients as users<br>CT_MAIL   the recipients as mail addresses |

**PURPOSE** Add or remove recipients that agent determination cannot supply: a distribution list address, an external contact, a shared mailbox.

**NOT FILLED IN ANY OF THE PRODUCTION IMPLEMENTATIONS EXAMINED.**

For a good reason: the recipient group comes from /C09/CFL_C05 and /C09/CFL_C03, i.e. from the customizing. If you add to it in code, you have a recipient that nobody reading the configuration will find - and who stays when the person leaves the company.

The clean way for "one address should always be included" is a separate GEN_STAT_USER in C05, set to the address in the customizing. Then it is where people look for it.

**STAYS EMPTY.**
