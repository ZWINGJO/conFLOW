# `get_mail_language`

> **This hook stays empty in the reference class.** The reason is below.

| | |
|---|---|
| **When** | Without wave 1: never. The hook is in the interface, but the framework does not call it (checked 2026-09-11). With wave 1: per recipient, right before dispatch. |
| **In** | IT_USER / IT_MAIL  the one recipient of this dispatch |
| **Out** | CS_DATA-WI_LANG    the language for this recipient |

**WITHOUT WAVE 1 EVERY MAIL GOES OUT IN THE SENDER'S LANGUAGE**

That is, in the logon language of whoever triggers the dispatch - dialog user or WF-BATCH -, not in the language of the instance (S01-WI_LANG). That is why the hook is not filled in any of the production implementations examined: it had no effect.

**WITH WAVE 1 WI_LANG ARRIVES WITH THE SENDER'S LANGUAGE**

Not with that of the instance - if you want that one, read it via CS_DATA-ID from /C09/CFL_S01. If the hook leaves WI_LANG as it is, nothing changes. The language is only switched if it is installed and the subject (SO10) exists in it.

**WHEN YOU NEED IT**

When recipients should be addressed in their own language, typically the one from the user master (USR01-LANGU for

**IT_USER).**

STAYS EMPTY HERE: the reference does not require wave 1.
