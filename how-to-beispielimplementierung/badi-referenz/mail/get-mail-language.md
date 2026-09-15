# `get_mail_language`

> **Dieser Hook bleibt in der Referenzklasse leer.** Warum, steht unten.

| | |
|---|---|
| **Wann** | Ohne Welle 1: nie. Der Hook steht im Interface, das Framework ruft ihn nicht (geprüft 2026-09-11). Mit Welle 1: je Empfänger, direkt vor dem Versand. |
| **Rein** | IT_USER / IT_MAIL  der eine Empfänger dieses Versands |
| **Raus** | CS_DATA-WI_LANG    die Sprache für diesen Empfänger |

**OHNE WELLE 1 GEHT JEDE MAIL IN DER SPRACHE DES VERSENDERS**

Also in der Anmeldesprache dessen, der den Versand auslöst - Dialogbenutzer oder WF-BATCH -, nicht in der Sprache der Instanz (S01-WI_LANG). Deshalb ist der Hook in keiner der untersuchten Produktivimplementierungen gefüllt: er hatte keine Wirkung.

**MIT WELLE 1 KOMMT WI_LANG MIT DER SPRACHE DES VERSENDERS HEREIN**

Nicht mit der der Instanz - wer die will, liest sie über CS_DATA-ID aus /C09/CFL_S01. Lässt der Hook WI_LANG stehen, ändert sich nichts. Umgeschaltet wird nur, wenn die Sprache installiert ist und der Betreff (SO10) in ihr existiert.

**WANN MAN IHN BRAUCHT**

Wenn Empfänger in ihrer eigenen Sprache angeschrieben werden sollen, typischerweise der aus dem Benutzerstamm (USR01-LANGU zu IT_USER).

BLEIBT HIER LEER: die Referenz setzt Welle 1 nicht voraus.
