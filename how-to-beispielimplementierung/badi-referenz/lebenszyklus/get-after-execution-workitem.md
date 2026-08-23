# `get_after_execution_workitem`

| | |
|---|---|
| **Wann** | Nachdem ein Workitem abgeschlossen wurde - unabhängig davon, aus welcher Oberfläche. |
| **Rein** | IS_DATA_STEP  der Schritt<br>IS_SWR_WIHDR  der Workitem-Kopf<br>IV_KEY        der Ausgang |
| **Raus** | nichts, und AUCH KEIN ABBRUCH. Was hier passiert, passiert nach der Entscheidung. |

**DER KLASSISCHE ANWENDUNGSFALL: PARALLELE WORKITEMS AUFRÄUMEN**

Wenn ein Schritt mehrere Bearbeiter parallel hat und einer ablehnt, sollen die anderen Workitems verschwinden - sonst arbeiten Leute an einem Vorgang, der schon entschieden ist.

/C09/CFL_CL_HELPER_0101=>SET_WORKITEM_OBSOLET erledigt das: es sucht alle offenen Workitems desselben Top-Workflows und setzt sie auf obsolet - das eigene ausgenommen.

**WARUM DAS COMMIT HIER STEHT**

SET_WORKITEM_OBSOLET ruft SAP_WAPI_WORKITEM_COMPLETE mit DO_COMMIT = FALSE, damit nicht je Workitem einzeln festgeschrieben wird. Das COMMIT muss also der Aufrufer machen. Ohne die Zeile bleiben die Workitems offen - ohne Fehlermeldung.

**UNTERSCHIED ZU GET_AFTER_EXECUTION**

GET_AFTER_EXECUTION läuft VOR dem Abschluss und kann ihn verhindern. Dieser hier läuft DANACH. Wer prüfen will, nimmt den anderen.

## Der Code

```abap
IF iv_key <> /c09/cfl_cl_workflow_0101=>mc_decision-nok.
  RETURN.
ENDIF.

/c09/cfl_cl_helper_0101=>set_workitem_obsolet( is_data_step = is_data_step ).

COMMIT WORK AND WAIT.
```
