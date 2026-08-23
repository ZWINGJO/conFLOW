# `get_after_execution`

| | |
|---|---|
| **Wann** | Nachdem der Bearbeiter im SAP-GUI entschieden hat, bevor das Workitem abgeschlossen wird. |
| **Rein** | IV_WI_ID     das Workitem<br>IV_ALTKEY    der GEKLICKTE Ausgang - der eigentliche Wert<br>IV_ALT_TEXT  dessen Text<br>IV_MSELNOTE  Vorgabe für den Notiz-Dialog |
| **Raus** | CV_SUBRC      <> 0 bricht ab<br>CS_OBJECT_ID  Referenz auf eine erfasste Notiz |

**ZWEI VERSCHIEDENE AUFGABEN, DIE HIER ZUSAMMENFALLEN**

1. NACHLAUFLOGIK - den Container fortschreiben, festhalten wer entschieden hat. Das ist der übliche Fall.

2. EINE NOTIZ ERZWINGEN - über SWU_INTERN_DECI_NOTE_POPUP. Das ist der Standardweg für "Ablehnung bitte begründen".

**DER ABBRUCH KANN NICHT SAGEN WARUM**

CV_SUBRC <> 0 hält den Prozess an, aber es gibt keinen Meldungsparameter. Der Bearbeiter klickt und es passiert nichts - die schlechteste aller Rückmeldungen. Wer eine Prüfung MIT Begründung braucht, ruft dieselbe Prüfung zusätzlich in GET_AFTER_EXECUTION_MOBILE, dem einzigen Hook mit CS_T100MSG.

**ERSTE ZEILE: AUF IV_ALTKEY PRÜFEN**

Der Hook läuft auch bei Aktionen, die keine Entscheidung sind (Weiterleiten, Zurückstellen). Dann ist IV_ALTKEY leer, und jede Logik, die einen Ausgang voraussetzt, greift ins Leere.

## Der Code

```abap
    IF iv_altkey IS INITIAL.
      RETURN.
    ENDIF.

    SELECT SINGLE * FROM /c09/cfl_s03 INTO @DATA(ls_cfl_s03) "#EC CI_ALL_FIELDS_NEEDED
      WHERE wi_id = @iv_wi_id.                                "#EC CI_NOORDER
    IF sy-subrc <> 0.
      RETURN.
    ENDIF.

*--------------------------------------------------------------------*
* Festhalten, WER entschieden hat.
*
* Das steht zwar auch im Workitem-Protokoll (/C09/CFL_S03 mit
* Benutzer und Zeit) - aber im Container ist es fuer die folgenden
* Schritte LESBAR, ohne dass sie das Protokoll auswerten muessen.
* Der naechste Schritt kann daraus zum Beispiel den Bearbeiter
* ableiten (siehe GET_ACTORS, Variante B).
*--------------------------------------------------------------------*
    set_val( iv_element = zcl_cfl_const_00900=>mc_prop-decision_by
             iv_id      = ls_cfl_s03-id
             iv_value   = CONV #( sy-uname ) ).

*--------------------------------------------------------------------*
* Bei Ablehnung eine Begruendung verlangen.
*
* Der Popup gehoert dem Workflow-Standard, nicht conFLOW. Bricht der
* Bearbeiter ihn ab (RETURNCODE 'A'), liefert die FM eine Exception -
* dann wird auch die Entscheidung nicht wirksam.
*--------------------------------------------------------------------*
    IF iv_altkey = /c09/cfl_cl_workflow_0101=>mc_decision-nok.

      CALL FUNCTION 'SWU_INTERN_DECI_NOTE_POPUP'
        EXPORTING  wi_id          = iv_wi_id
                   alt_text       = iv_alt_text
                   mselnote       = iv_mselnote
        IMPORTING  ex_object_id   = cs_object_id
        EXCEPTIONS user_cancelled = 1
                   OTHERS         = 2.

      IF sy-subrc <> 0.
        cv_subrc = sy-subrc.
        RETURN.
      ENDIF.

    ENDIF.
```
