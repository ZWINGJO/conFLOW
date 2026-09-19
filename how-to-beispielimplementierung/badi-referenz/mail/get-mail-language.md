# `get_mail_language`

> **Dieser Hook bleibt in der Referenzklasse leer.** Warum, steht unten.

| | |
|---|---|
| **Wann** | Je Empfänger, direkt vor dem Versand. Ältere conFLOW- Stände rufen den Hook nicht - dort geht jede Mail in der Sprache des Versenders. |
| **Rein** | IT_USER / IT_MAIL  der eine Empfänger dieses Versands |
| **Raus** | CS_DATA-WI_LANG    die Sprache für diesen Empfänger |

**OHNE DIESEN HOOK GEHT JEDE MAIL IN DER SPRACHE DES VERSENDERS**

Also in der Anmeldesprache dessen, der den Versand auslöst - Dialogbenutzer oder WF-BATCH -, nicht in der Sprache der Instanz (S01-WI_LANG).

**WI_LANG KOMMT MIT DER SPRACHE DES VERSENDERS HEREIN**

Nicht mit der der Instanz - wer die will, liest sie über CS_DATA-ID aus /C09/CFL_S01. Lässt der Hook WI_LANG stehen, ändert sich nichts. Umgeschaltet wird nur, wenn die Sprache installiert ist und Betreff und Texte (SO10) in ihr existieren.

**WANN MAN IHN BRAUCHT**

Wenn Empfänger in ihrer eigenen Sprache angeschrieben werden sollen, typischerweise der aus dem Benutzerstamm (USR01-LANGU zu IT_USER).

BLEIBT HIER LEER: für den Beispielprozess reicht die Sprache des Versenders.
