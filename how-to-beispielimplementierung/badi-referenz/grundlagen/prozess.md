# Der Prozess

## Ein Freigabeprozess, mehr nicht

conFLOW wird in den allermeisten Projekten für dasselbe gebaut: **etwas
muss freigegeben werden, und je nachdem wie groß es ist, von wem
anderem.** Genau das ist der Beispielprozess hier. Wer ihn versteht,
kann ihn auf Bestellung, Rechnung, Urlaubsantrag oder Vertrag umbauen,
ohne die Struktur anzufassen.

Ein Beispiel braucht trotzdem einen konkreten Beleg. Die Wahl fällt auf
den **Einkaufsbeleg** (`BUS2012`), weil ihn jeder kennt: Kopf,
Positionen, Lieferant, Nettowert. Damit kostet der Datenteil keine
Aufmerksamkeit, und die bleibt beim Workflow.

Der Prozess ist absichtlich klein. Er hat genau so viele Schritte, wie
es braucht, damit jeder Hook einen sinnvollen Platz bekommt - und
keinen mehr.

> **Was man austauschen muss, wenn es ein anderer Beleg wird:**
> `MC_OBJECTTYPE` in der Konstantenklasse, `READ_DOCUMENT( )` und
> `EXECUTE_DEFAULT_METHOD`. Sonst nichts. Die Hooks kennen den Beleg
> nicht - sie kennen nur den Container.

## Der Ablauf

```
X0  Start
 |
B1  Hintergrund: Beleg lesen, Betrag gegen das Limit prüfen
 |
 +-- unter dem Limit ----------------------------> X3  Ende, nichts zu tun
 |
01  Dialog: Einkauf entscheidet
 |
 +-- freigegeben ---> B2  Hintergrund: Rückschreibung ---> X1  Ende
 |
02  Dialog: Vorgesetzter entscheidet (Eskalation)
 |
 +-- abgelehnt ----------------------------------> X2  Ende, abgelehnt
```

## Die Pointe des Ablaufs

**Der Normalfall erzeugt keine Arbeit.** Liegt der Beleg unter der
Grenze, schließt sich der Workflow in `B1` selbst - und im Protokoll
steht, warum. Das ist der Unterschied zwischen einem Workflow und
einem Genehmigungsstau.

**Die Verzweigung steht im Customizing, nicht im Code.** `B1` setzt
nur einen Entscheidungsschlüssel; wohin der führt, steht in
`/C09/CFL_C02`. Deshalb bleibt `GET_STATUS_DYNAMIC` in diesem
Beispiel leer, obwohl der Prozess verzweigt - siehe dort.

## Die Schritte

| Schritt | Art | Was passiert |
|---|---|---|
| `X0` | Start | Instanz anlegen |
| `B1` | Hintergrund | Beleg lesen, Werte in den Container, bewerten, Weiche stellen |
| `01` | Dialog | Einkauf: freigeben, ablehnen, eskalieren |
| `02` | Dialog | Vorgesetzter: freigeben oder ablehnen |
| `B2` | Hintergrund | Rückschreibung in den Beleg |
| `B3` | Hintergrund | Benachrichtigung |
| `X1` / `X2` / `X3` | Ende | freigegeben / abgelehnt / nichts zu tun |

## Die Bearbeiterkreise

| Schlüssel | Wer | Woher |
|---|---|---|
| `01` | Einkäufer | Rolle - `ZCL_CFL_GET_ACTORS` |
| `02` | Vorgesetzter | Aufbauorganisation - `RH_GET_ACTORS` |
| `03` | Ersteller des Belegs | Container-Attribut |
| `M1` | nur Mailempfänger | Customizing, kein Workitem |

Vier Kreise, weil es vier **verschiedene Arten** der Bearbeiterfindung
gibt. Der Prozess bräuchte sie nicht alle - das Beispiel schon.
