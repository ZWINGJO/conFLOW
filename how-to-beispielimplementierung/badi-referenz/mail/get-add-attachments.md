# `get_add_attachments`

| | |
|---|---|
| **Wann** | Beim Mailversand, nachdem der Text steht. |
| **Rein** | IS_DATA   die Instanz<br>IT_SMTP   die Empfänger |
| **Raus** | CT_ATTACHMENT  die Anhänge |

```
WOFÜR   Dokumente ans Mail hängen, die der Empfänger sonst erst im
       System suchen müsste: das archivierte Rechnungsbild, die
       angehängten Dateien aus dem Beleg, ein SAP-Shortcut.
```

**DER SAP-SHORTCUT IST DER UNTERSCHÄTZTE FALL**

Eine .SAP-Datei im Anhang öffnet beim Empfänger direkt das richtige System, den richtigen Mandanten, die richtige Transaktion - mit einem Doppelklick aus dem Mail heraus. Für Genehmiger, die selten im System sind, ist das der Unterschied zwischen "wird erledigt" und "liegt liegen".

WICHTIG: der Shortcut wird JE EMPFÄNGER erzeugt, weil der Benutzername darin steht. Deshalb die Schleife über

**IT_SMTP - und deshalb ist es falsch, ihn einmal zu bauen**

und allen zu schicken.

**DREI QUELLEN, DREI PRODUKT-METHODEN**

```
GET_SHORTCUT   der SAP-Shortcut
GET_ARCHIVE    Dokumente aus dem Archiv (ArchiveLink)
GET_GOS        Anlagen am Beleg (Generic Object Services)
```

Alle drei liegen in /C09/CFL_CL_HELPER_0101 und liefern fertige Anhangszeilen. Selbst bauen ist Arbeit ohne Gewinn.

**GRÖSSE IM AUGE BEHALTEN**

Anhänge gehen in den Mailversand als SOLIX. Ein 20-MB-PDF an zehn Empfänger ist 200 MB im Sendeauftrag. Wo Größe ein Thema sein kann, ist der Shortcut die bessere Antwort als das Dokument.

## Der Code

```abap
    LOOP AT it_smtp INTO DATA(ls_smtp) WHERE uname IS NOT INITIAL.

      /c09/cfl_cl_helper_0101=>get_shortcut(
        EXPORTING iv_user        = CONV syuname( ls_smtp-uname )
                  iv_transaction = CONV tcode( 'SBWP' )
                  iv_parameter   = space
        CHANGING  ct_attachment  = ct_attachment ).

*--------------------------------------------------------------------*
* Nur fuer den ersten Empfaenger mit Benutzerkennung. Wer den EXIT
* weglaesst, haengt so viele Shortcuts ans Mail, wie es Empfaenger
* gibt - und jeder sieht die Kennungen der anderen.
*--------------------------------------------------------------------*
      EXIT.

    ENDLOOP.
```
