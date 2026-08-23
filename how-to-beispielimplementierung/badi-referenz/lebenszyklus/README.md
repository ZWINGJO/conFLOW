# Workitem-Lebenszyklus

Anlegen, öffnen, entscheiden, abschließen

Hier sitzen die drei Stellen, an denen man den Prozess ANHALTEN kann. Sie können unterschiedlich viel, und das ist der wichtigste Unterschied in dieser ganzen Gruppe:

**GET_BEFORE_DECISION_WORKITEM**

kann Buttons ENTFERNEN. Keine Meldung, kein Abbruch. "Darf nicht" heißt hier: der Knopf ist weg.

**GET_AFTER_EXECUTION**

kann ABBRECHEN (CV_SUBRC), aber OHNE Meldung. Der Bearbeiter sieht nur, dass nichts passiert - unbrauchbar allein.

**GET_AFTER_EXECUTION_MOBILE**

```
      kann ABBRECHEN UND SAGEN WARUM (CS_T100MSG). Der einzige
      Hook mit beidem.
```

Merksatz: was der Bearbeiter NICHT DARF, nimmt man ihm vorher weg. Was er FALSCH GEMACHT hat, sagt man ihm nachher.

## Die Hooks

- [`get_after_creation_workitem`](get-after-creation-workitem.md) - Priorität, Anlagen, eigene Oberfläche anhängen
- [`get_before_execution_workitem`](get-before-execution-workitem.md) - Vorbereitung beim Öffnen - kann nichts verhindern
- [`get_before_decision_workitem`](get-before-decision-workitem.md) - Buttons wegnehmen und einfärben
- [`get_fiori_task_dec_op_act`](get-fiori-task-dec-op-act.md) - Dasselbe für die Fiori-Inbox - andere Tabelle
- [`get_after_execution`](get-after-execution.md) - Nachlauf nach der Entscheidung, Abbruch ohne Meldung
- [`get_after_execution_mobile`](get-after-execution-mobile.md) - Abbruch **mit** Meldung - die einzige echte Schranke
- [`get_after_execution_workitem`](get-after-execution-workitem.md) - Nach dem Abschluss: parallele Workitems aufräumen
- [`get_event_raised_workitem`](get-event-raised-workitem.md) - Ereignis auf ein laufendes Workitem
