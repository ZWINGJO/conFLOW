# `get_after_execution_mobile`

| | |
|---|---|
| **Wann** | Nach der Ausführung aus conMOBILE bzw. aus dem BSP-Pfad. |
| **Rein** | IV_WI_ID   das Workitem<br>IV_ALTKEY  der geklickte Ausgang |
| **Raus** | CV_SUBRC     <> 0 bricht ab. 9 ist der übliche Wert.<br>CS_T100MSG   die Meldung dazu |

**DER EINZIGE HOOK, DER ABBRECHEN UND DABEI SAGEN KANN WARUM.**

Das macht ihn zur wichtigsten Schranke im ganzen Interface - wichtiger, als sein Name vermuten lässt.

**DER NAME IST IRREFÜHREND**

"MOBILE" legt nahe, dass er nur für conMOBILE läuft. Laut Dokumentation hängt er am conMOBILE-/BSP-Pfad; ob eine bestimmte Fiori-Inbox ihn ruft, ist im eigenen System zu PRÜFEN und nicht anzunehmen. Bei GET_FIORI_TASK_DEC_OP_ACT sah es genauso tot aus, und der Hook lief.

Fällt er in der eigenen Oberfläche aus, wirkt die Schranke dort nicht. Dann bleibt nur: dieselbe Prüfung zusätzlich in einen HINTERGRUNDSCHRITT hinter der Entscheidung legen - der läuft immer.

**FREITEXT ALS T100-MELDUNG**

CS_T100MSG will ID und Nummer, keinen String. 00/398 ist '&1&2&3&4' - vier Variablen a 50 Zeichen. Damit passen 200 Zeichen Freitext hinein. Eine eigene Nachrichtenklasse ist sauberer, aber dieser Weg braucht kein neues Objekt.

## Der Code

```abap
    CONSTANTS lc_var_len TYPE i VALUE 50.

    IF iv_altkey IS INITIAL.
      RETURN.
    ENDIF.

    SELECT SINGLE * FROM /c09/cfl_s03 INTO @DATA(ls_cfl_s03) "#EC CI_ALL_FIELDS_NEEDED
      WHERE wi_id = @iv_wi_id.                                "#EC CI_NOORDER
    IF sy-subrc <> 0.
      RETURN.
    ENDIF.

*--------------------------------------------------------------------*
* Die Pruefung: Ablehnung ohne Notiz gibt es nicht.
*
* Bewusst dieselbe fachliche Regel wie im Guard weiter oben - aber
* an anderer Stelle wirksam. Der Guard nimmt weg, was gar nicht
* erlaubt ist; diese Schranke prueft, was zur erlaubten Aktion noch
* fehlt.
*--------------------------------------------------------------------*
    DATA lv_error TYPE string.

    IF iv_altkey = /c09/cfl_cl_workflow_0101=>mc_decision-nok AND
       get_val( iv_element = zcl_cfl_const_00900=>mc_prop-note
                iv_id      = ls_cfl_s03-id ) IS INITIAL.

      lv_error = 'A rejection needs a reason. Please fill in the note.' ##NO_TEXT.

    ENDIF.

    IF lv_error IS INITIAL.
      RETURN.
    ENDIF.

    DATA(lv_len) = strlen( lv_error ).

    cs_t100msg-msgid = '00'.
    cs_t100msg-msgno = '398'.
    cs_t100msg-msgty = 'E'.
    cs_t100msg-msgv1 = lv_error.

    IF lv_len > lc_var_len.
      cs_t100msg-msgv2 = lv_error+lc_var_len.
    ENDIF.
    IF lv_len > 100.
      cs_t100msg-msgv3 = lv_error+100.
    ENDIF.
    IF lv_len > 150.
      cs_t100msg-msgv4 = lv_error+150.
    ENDIF.

*--------------------------------------------------------------------*
* 9 heisst: nicht weiterlaufen. Das Workitem bleibt offen, der
* Bearbeiter korrigiert und klickt erneut.
*--------------------------------------------------------------------*
    cv_subrc = 9.
```
