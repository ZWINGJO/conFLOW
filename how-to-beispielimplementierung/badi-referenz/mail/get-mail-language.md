# `get_mail_language`

> **Dieser Hook bleibt in der Referenzklasse leer.** Warum, steht unten.

| | |
|---|---|
| **Wann** | Nachdem die Empfänger feststehen, vor dem Aufbau des Textes. |
| **Rein** | IT_USER / IT_MAIL  die Empfänger |
| **Raus** | CS_DATA            die Instanz - relevant ist WI_LANG |

```
WOFÜR   Die Sprache des Mails festlegen. Standard ist die Sprache
       der Instanz; hier kann man sie am Empfänger ausrichten.
```

**IN KEINER DER UNTERSUCHTEN PRODUKTIVIMPLEMENTIERUNGEN GEFÜLLT.**

Nicht, weil Mehrsprachigkeit selten wäre, sondern weil der Standard schon das Richtige tut: er nimmt die Sprache aus der Instanz, und die SO10-Textbausteine gibt es je Sprache.

**WANN MAN IHN BRAUCHT**

Wenn ein Mail an MEHRERE Empfänger mit VERSCHIEDENEN Sprachen geht. Dann hilft dieser Hook allerdings auch nur halb - er setzt EINE Sprache für den ganzen Versand. Für echte Mehrsprachigkeit braucht es mehrere Versandvorgänge, also mehrere Einträge in C05.

**BLEIBT LEER.**
