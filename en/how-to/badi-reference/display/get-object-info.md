# `get_object_info`

| | |
|---|---|
| **When** | The FIORI inbox builds its "Links" tab. The path there goes through SAP_WAPI_GET_OBJECTS, not through<br>GET_NEW_PREVIEW_DESCR - which is why this hook works there and the other one does not. |
| **In** | IS_LPOR    the object to be labeled |
| **Out** | CV_RETURN  the prefix of the label |

**WHAT HAPPENS IF YOU LEAVE IT OUT**

In the Fiori inbox, "Objects and attachments" shows the class name of the framework:

```
    CON: conFLOW Worflow V.0101: 0004500001234
```

With the hook:

```
    Purchase order: 0004500001234
```

The framework appends the instance ID itself - CV_RETURN only replaces the part in front of it.

**THE TEXT BELONGS IN A TEXT SYMBOL, NOT IN THE CODE**

TEXT-001 belongs to the text pool of the class. That makes it translatable, and the same label for both user interfaces lives in exactly one place.

PITFALL WHEN TRANSPORTING: text symbols belong to the text pool, not to the code. If you move the class via abapGit or a transport and forget the text pool, the label is lost - and you only notice in the inbox of the target system.

**THIS HOOK LOOKS DEAD IN THE WHERE-USED LIST**

Where-used does not find it, because it is called through an enhancement. It runs anyway. Rule of thumb for conFLOW BAdI hooks in general: TRY IT OUT INSTEAD OF TRUSTING WHERE-USED.

## The code

```abap
cv_return = text-001.
```
