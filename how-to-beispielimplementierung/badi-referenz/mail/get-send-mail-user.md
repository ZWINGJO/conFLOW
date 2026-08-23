# `get_send_mail_user`

> **Dieser Hook bleibt in der Referenzklasse leer.** Warum, steht unten.

| | |
|---|---|
| **Wann** | Vor dem Mailversand, wenn der Empfängerkreis feststeht. |
| **Rein** | IS_DATA   die Instanz |
| **Raus** | CT_USER   die Empfänger als Benutzer<br>CT_MAIL   die Empfänger als Mailadressen |

```
WOFÜR   Empfänger ergänzen oder entfernen, die aus der
       Bearbeiterfindung nicht kommen können: eine
       Verteiler-Adresse, ein externer Ansprechpartner, ein
       Postfach.
```

**IN KEINER DER UNTERSUCHTEN PRODUKTIVIMPLEMENTIERUNGEN GEFÜLLT.**

Der Grund ist gut: der Empfängerkreis kommt aus /C09/CFL_C05 und /C09/CFL_C03, also aus dem Customizing. Wer ihn im Code ergänzt, hat einen Empfänger, den niemand findet, der die Konfiguration liest - und der bleibt, wenn die Person das Haus verlässt.

Der saubere Weg für "eine Adresse soll immer mit" ist ein eigener GEN_STAT_USER in C05, im Customizing auf die Adresse gesetzt. Dann steht er da, wo man ihn sucht.

**BLEIBT LEER.**
