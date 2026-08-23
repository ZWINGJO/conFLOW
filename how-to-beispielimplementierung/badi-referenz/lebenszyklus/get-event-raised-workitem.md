# `get_event_raised_workitem`

> **Dieser Hook bleibt in der Referenzklasse leer.** Warum, steht unten.

| | |
|---|---|
| **Wann** | Wenn ein Ereignis auf ein LAUFENDES Workitem trifft - nicht beim Start des Workflows, sondern während er läuft. |
| **Rein** | IM_EVENT_NAME - welches Ereignis |
| **Rein und raus** | CM_WORKITEM_CONTEXT - der Kontext des betroffenen Workitems |

```
WOFÜR   Reaktion auf Änderungen am Beleg, WÄHREND der Workflow
       offen ist. Der Beleg wird storniert, während jemand über
       ihn entscheidet - dann soll das Workitem verschwinden statt
       weiter im Eingang zu liegen.
```

**IN KEINER DER UNTERSUCHTEN PRODUKTIVIMPLEMENTIERUNGEN GEFÜLLT.**

Nicht, weil er nutzlos wäre, sondern weil der Fall selten ist und die Alternative näher liegt: die Änderung wird beim nächsten Schritt geprüft, statt sofort zu wirken.

Wenn man ihn braucht, merkt man es daran: es gibt eine Anforderung der Form "wenn X passiert, WÄHREND der Workflow läuft, dann ...". Ohne dieses "während" ist es ein normaler Hintergrundschritt.

**BLEIBT LEER.**
