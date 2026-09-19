# `get_add_attachments`

| | |
|---|---|
| **When** | During mail dispatch, after the text is in place. |
| **In** | IS_DATA   the instance<br>IT_SMTP   the recipients |
| **Out** | CT_ATTACHMENT  the attachments |

**PURPOSE** Attach documents to the mail that the recipient would otherwise have to look up in the system: the archived invoice image, the files attached to the document, an SAP shortcut.

**THE SAP SHORTCUT IS THE UNDERRATED CASE**

A .SAP file in the attachment opens the right system, the right client and the right transaction directly on the recipient's side - with a double click from the mail. For approvers who rarely use the system, that is the difference between "gets done" and "sits there".

IMPORTANT: the shortcut contains a user name. By default conFLOW sends to each recipient separately, so that fits. If IT_SMTP does hold several, only the first gets a shortcut - otherwise everyone would see the IDs of the others.

**THREE SOURCES, THREE PRODUCT METHODS**

```
GET_SHORTCUT   the SAP shortcut
GET_ARCHIVE    documents from the archive (ArchiveLink)
GET_GOS        document attachments (Generic Object Services)
```

All three live in /C09/CFL_CL_HELPER_0101 and return ready-made attachment lines. Building them yourself is work without gain.

**KEEP AN EYE ON SIZE**

Attachments go into mail dispatch as SOLIX. A 20 MB PDF to ten recipients is 200 MB in the send request. Where size can be an issue, the shortcut is the better answer than the document.

## The code

```abap
    LOOP AT it_smtp INTO DATA(ls_smtp) WHERE uname IS NOT INITIAL.

      /c09/cfl_cl_helper_0101=>get_shortcut(
        EXPORTING iv_user        = CONV syuname( ls_smtp-uname )
                  iv_transaction = CONV tcode( 'SBWP' )
                  iv_parameter   = space
        CHANGING  ct_attachment  = ct_attachment ).

*--------------------------------------------------------------------*
* Only for the first recipient with a user ID. If you leave out the
* EXIT, the mail gets as many shortcuts as there are recipients - and
* everyone sees the user IDs of the others.
*--------------------------------------------------------------------*
      EXIT.

    ENDLOOP.
```
