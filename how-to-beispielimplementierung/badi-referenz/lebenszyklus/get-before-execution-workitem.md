# `get_before_execution_workitem`

> **Dieser Hook bleibt in der Referenzklasse leer.** Warum, steht unten.

| | |
|---|---|
| **Wann** | Unmittelbar bevor ein Workitem ausgeführt wird - also nach dem Doppelklick, vor dem Aufbau der Oberfläche. |
| **Rein** | IS_SWR_WIHDR  der Workitem-Kopf |
| **Raus** | nichts. |

```
WOFÜR   Vorbereitungen, die genau dann nötig sind, wenn jemand das
       Workitem tatsächlich öffnet - Sperren setzen, einen Cache
       füllen, Zähler hochsetzen.
```

**IN KEINER DER UNTERSUCHTEN PRODUKTIVIMPLEMENTIERUNGEN GEFÜLLT.**

Der Grund: er kann nichts verhindern. Es gibt keinen Rückgabeparameter und keine Möglichkeit abzubrechen - was hier passiert, passiert nebenbei. Wer eine Prüfung sucht, ist bei GET_BEFORE_DECISION_WORKITEM richtig (Button wegnehmen) oder bei GET_AFTER_EXECUTION_MOBILE (abbrechen mit Meldung).

BLEIBT LEER. Das ist die Entscheidung, nicht ein Rest.
