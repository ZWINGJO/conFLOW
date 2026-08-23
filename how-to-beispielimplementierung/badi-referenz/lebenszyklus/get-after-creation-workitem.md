# `get_after_creation_workitem`

| | |
|---|---|
| **Wann** | Nach dem Anlegen JEDES Workitems - auch der Hintergrundschritte - und vor der ersten Anzeige. |
| **Rein** | IS_DATA_STEP  der Schritt (/C09/CFL_S03)<br>IS_SWR_WIHDR  der Workitem-Kopf, inklusive WI_ID |
| **Raus** | nichts. Wirkung nur über Seiteneffekte. |

```
WOFÜR   Alles, was EINMAL JE WORKITEM passieren soll: Priorität,
       Anlagen, Notizen, Container vorbereiten, eine eigene
       Oberfläche anhängen.
```

**DER HOOK LÄUFT AUCH FÜR HINTERGRUNDSCHRITTE**

Und das ist fast immer unerwünscht. Eine Priorität an einem Workitem, das kein Mensch sieht, kostet nur Laufzeit. Deshalb steht am Anfang eine Weiche auf die Dialogschritte - die Zeile sieht nach Kleinkram aus und ist keiner.

**DAS WORKITEM STEHT HIER NOCH NICHT AUF DER DATENBANK**

Der zentrale Punkt dieses Hooks, und die Ursache der häufigsten Enttäuschung: jede API, die SWWWIHEAD LIEST, läuft ins Leere. SAP_WAPI_CHANGE_WORKITEM_PRIO tut genau das - sie meldet keinen Fehler, sie wirkt nur nicht. Der richtige Weg geht über den Workitem-Manager der laufenden Transaktion, siehe SET_PRIORITY( ).

## Der Code

```abap
    IF is_data_step-gen_stat <> zcl_cfl_const_00900=>mc_stat-approve AND
       is_data_step-gen_stat <> zcl_cfl_const_00900=>mc_stat-escalate.
      RETURN.
    ENDIF.

    IF is_swr_wihdr-wi_id IS INITIAL.
      RETURN.
    ENDIF.

*--------------------------------------------------------------------*
* Die Eskalation ist immer dringend - sie ist ja gerade deshalb beim
* Vorgesetzten gelandet. Sonst entscheidet die in B1 berechnete
* Severity.
*--------------------------------------------------------------------*
    DATA lv_prio TYPE sww_prio.

    IF is_data_step-gen_stat = zcl_cfl_const_00900=>mc_stat-escalate.
      lv_prio = zcl_cfl_const_00900=>mc_prio-high.

    ELSEIF get_val( iv_element = zcl_cfl_const_00900=>mc_prop-severity
                    iv_id      = is_data_step-id ) = zcl_cfl_const_00900=>mc_severity-red.
      lv_prio = zcl_cfl_const_00900=>mc_prio-high.

    ELSE.
      lv_prio = zcl_cfl_const_00900=>mc_prio-medium.
    ENDIF.

    set_priority( iv_wi_id = is_swr_wihdr-wi_id
                  iv_prio  = lv_prio ).
```
