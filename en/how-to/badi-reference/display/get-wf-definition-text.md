# `get_wf_definition_text`

> **This hook stays empty in the reference class.** The reason is below.

| | |
|---|---|
| **When** | Building the title for the ENTIRE workflow - not for a single step. |
| **In** | IS_DATA    the instance |
| **Out** | CV_WITEXT  the title (SWW_WITEXT) |

**DO YOU NEED IT AT ALL**

Usually not. The standard text comes from /C09/CFL_C06T and can be maintained in Customizing - without a transport, with translation. That is the better way.

**CASES WHERE YOU DO**

When the title should contain document data that the Customizing text does not know. "Purchase order approval" is impossible to find in the workflow log next to twenty others, "Purchase order approval 4500001234 / Example Corp." is not.

Stays EMPTY here, because the sample process gets by with the Customizing text - and because a reference example should show that you do not fill hooks just because they exist.
