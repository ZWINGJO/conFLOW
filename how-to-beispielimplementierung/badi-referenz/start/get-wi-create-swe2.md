# `get_wi_create_swe2`

| | |
|---|---|
| **Wann** | Beim ereignisgesteuerten Start über SWE2, bevor conFLOW die Instanz anlegt. Nur auf diesem Weg - wer den Workflow über START_WORKFLOW_INT( ) aus eigenem Code startet, läuft hier nicht durch. |
| **Rein** | IT_EVENT_CONTAINER_TAB  der Ereignis-Container<br>CS_CFL_C10              die Typkoppelung, die getroffen hat |
| **Raus** | CS_SENDER  Objekttyp und Instanz-Schlüssel der neuen Instanz |

```
WOFÜR   Das ist die VETO-Stelle. CS_SENDER leeren heißt: kein
       Workflow. Damit filtert man Ereignisse, die zwar geworfen
       werden, aber fachlich keinen Prozess auslösen sollen -
       Belegart, Werk, Betrag unter Bagatellgrenze.
```

**WARUM HIER UND NICHT IN B1**

Ein Workflow, der startet und sich im ersten Hintergrundschritt selbst beendet, hinterlässt eine Instanz, einen Protokolleintrag und eine Zeile in jeder Auswertung. Bei zehn Belegen ist das egal, bei zehntausend nicht mehr. Was hier abgewiesen wird, hat nie existiert.

Der Preis: es gibt auch keine Spur davon, dass geprüft wurde. Wer nachweisen muss, WARUM ein Beleg keinen Workflow bekam, filtert besser in B1 und beendet dort mit einem Protokolleintrag.

## Der Code

```abap
    IF cs_sender-typeid <> zcl_cfl_const_00900=>mc_objecttype.
      RETURN.
    ENDIF.

    read_document( EXPORTING iv_instid    = CONV #( cs_sender-instid )
                   IMPORTING ev_net_value = DATA(lv_net_value)
                             ev_found     = DATA(lv_found) ).

*--------------------------------------------------------------------*
* Beleg nicht lesbar oder Bagatellbetrag: kein Workflow.
*
* Der erste Fall ist der wichtigere. Ein Ereignis kann zu einem Beleg
* kommen, den es (noch) nicht gibt - etwa weil das Ereignis vor dem
* COMMIT geworfen wurde. Ohne diese Pruefung entstehen Instanzen zu
* Belegnummern, die niemand findet.
*--------------------------------------------------------------------*
    IF lv_found = abap_false.
      CLEAR cs_sender.
      RETURN.
    ENDIF.

    IF lv_net_value IS INITIAL.
      CLEAR cs_sender.
    ENDIF.
```
