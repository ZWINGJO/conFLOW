# Der Hintergrundschritt

Kein BAdI-Hook - der zweite Vertrag, den conFLOW kennt. Eingetragen wird er in /C09/CFL_C01 auf dem Schritt B1, mit Klassenname und Methodenname. Der Aufruf ist DYNAMISCH: eine falsche Signatur fällt nicht beim Aktivieren auf, sondern zur Laufzeit - und dann bleibt der Workflow stehen.

---

Der Schritt, in dem die Arbeit passiert:

1. Beleg lesen 2. Werte in den Container schreiben 3. Bewerten 4. über EV_DECISION_KEY sagen, wie es weitergeht

**WARUM DIE WERTE IN DEN CONTAINER GEHEN UND NICHT NUR GELESEN WERDEN**

Der Container ist die Entscheidungsgrundlage, und er ist eingefroren. Wenn der Bearbeiter morgen entscheidet und der Beleg heute Nacht geändert wurde, hat er trotzdem die Zahlen vor sich, über die er entscheidet - und im Audit Trail steht hinterher, welche das waren.

Wer stattdessen im Workitem-Text frisch aus EKKO liest, hat eine Anzeige, die sich unter dem Bearbeiter bewegt, und hinterher keine Möglichkeit mehr zu sagen, was er gesehen hat.

**WARUM DIE BEWERTUNG HIER STEHT UND NICHT IN DER ANZEIGE**

Aus demselben Grund. SEVERITY und die Empfehlung sind BERECHNETE Werte - wenn sie im Container stehen, sieht man hinterher, was das System empfohlen hat und ob der Bearbeiter davon abgewichen ist. Rechnet die Anzeige, ist diese Information weg, sobald das Workitem zu ist.

**DER RÜCKGABEWERT IST DIE WEICHE**

EV_DECISION_KEY wird genauso ausgewertet wie die Entscheidung eines Menschen - nur setzt sie hier der Code. In /C09/CFL_C02 steht dann:

```
B1 + OK   -> 01   (Entscheidung nötig)
B1 + UC1  -> X3   (nichts zu tun, Workflow endet)
```

Damit bleibt die Verzweigung im CUSTOMIZING SICHTBAR. Das ist der Grund, warum dieser Weg dem Hook GET_STATUS_DYNAMIC vorzuziehen ist: dort wäre dieselbe Weiche unsichtbar.

BLEIBT EV_DECISION_KEY LEER, läuft der Workflow nicht weiter. Das ist der häufigste Grund für "der Workflow hängt im Hintergrundschritt".

**FEHLER GEHEN IN ET_BAPIRET2, NICHT IN EINE EXCEPTION**

conFLOW schreibt die Tabelle ins Anwendungsprotokoll und wertet sie aus. Eine ungefangene Exception dagegen reißt den Workflow-Schritt in den Fehlerstatus, und der Grund steht dann nur im Dump.

## Der Code

```abap
    CLEAR: et_bapiret2, ev_decision_key.

*--------------------------------------------------------------------*
* Die Instanz - nur sie kennt den Beleg. IS_CFL_S03 ist der SCHRITT
* und hat die INSTID nicht.
*--------------------------------------------------------------------*
    SELECT SINGLE * FROM /c09/cfl_s01 INTO @DATA(ls_cfl_s01) "#EC CI_ALL_FIELDS_NEEDED
      WHERE id = @is_cfl_s03-id.
    IF sy-subrc <> 0.
      add_msg( EXPORTING iv_type     = 'E'
                         iv_text     = |Workflow instance { is_cfl_s03-id } not found|
               CHANGING  ct_bapiret2 = et_bapiret2 ).
      RETURN.
    ENDIF.

*--------------------------------------------------------------------*
* 1. Beleg lesen
*--------------------------------------------------------------------*
    read_document( EXPORTING iv_instid     = ls_cfl_s01-instid
                   IMPORTING ev_net_value  = DATA(lv_net_value)
                             ev_currency   = DATA(lv_currency)
                             ev_vendor     = DATA(lv_vendor)
                             ev_created_by = DATA(lv_created_by)
                             ev_found      = DATA(lv_found) ).

    IF lv_found = abap_false.
      add_msg( EXPORTING iv_type     = 'E'
                         iv_text     = |Purchase order { ls_cfl_s01-instid } not found|
               CHANGING  ct_bapiret2 = et_bapiret2 ).
      RETURN.
    ENDIF.

*--------------------------------------------------------------------*
* 2. In den Container
*
* CONV #( ) auf jedem Wert: SET_VAL erwartet einen String, die
* Belegfelder sind es nicht. Ohne die Konvertierung meldet der
* Compiler nichts - er konvertiert selbst, aber bei gepackten Zahlen
* nicht so, wie man denkt. Explizit ist hier besser.
*--------------------------------------------------------------------*
    set_val( iv_element = zcl_cfl_const_00900=>mc_prop-doc_number
             iv_id      = ls_cfl_s01-id
             iv_value   = CONV #( ls_cfl_s01-instid ) ).

    set_val( iv_element = zcl_cfl_const_00900=>mc_prop-net_value
             iv_id      = ls_cfl_s01-id
             iv_value   = |{ lv_net_value }| ).

    set_val( iv_element = zcl_cfl_const_00900=>mc_prop-currency
             iv_id      = ls_cfl_s01-id
             iv_value   = CONV #( lv_currency ) ).

    set_val( iv_element = zcl_cfl_const_00900=>mc_prop-vendor
             iv_id      = ls_cfl_s01-id
             iv_value   = CONV #( lv_vendor ) ).

    set_val( iv_element = zcl_cfl_const_00900=>mc_prop-decision_by
             iv_id      = ls_cfl_s01-id
             iv_value   = CONV #( lv_created_by ) ).

*--------------------------------------------------------------------*
* 3. Bewerten
*--------------------------------------------------------------------*
    DATA(lv_severity) = classify( lv_net_value ).

    set_val( iv_element = zcl_cfl_const_00900=>mc_prop-severity
             iv_id      = ls_cfl_s01-id
             iv_value   = lv_severity ).

    DATA(lv_limit_hit) = COND string(
      WHEN lv_net_value > zcl_cfl_const_00900=>mc_limit_value
      THEN CONV string( abap_true )
      ELSE space ).

    set_val( iv_element = zcl_cfl_const_00900=>mc_prop-limit_hit
             iv_id      = ls_cfl_s01-id
             iv_value   = lv_limit_hit ).

*--------------------------------------------------------------------*
* 4. Die Weiche - und das Protokoll dazu
*
* Beide Zweige protokollieren. Gerade der Zweig, in dem NICHTS
* passiert, braucht die Zeile: sonst steht im Protokoll ein
* Workflow, der sich ohne erkennbaren Grund selbst beendet hat.
*--------------------------------------------------------------------*
    IF lv_limit_hit IS INITIAL.

      DATA(lv_text) = |No approval required - { lv_net_value } { lv_currency } | &&
                      |is within the limit of { zcl_cfl_const_00900=>mc_limit_value }|.

      ev_decision_key = /c09/cfl_cl_workflow_0101=>mc_decision-uc1.

    ELSE.

      lv_text = |Approval required - { lv_net_value } { lv_currency } | &&
                |exceeds the limit of { zcl_cfl_const_00900=>mc_limit_value } | &&
                |({ lv_severity })|.

      ev_decision_key = /c09/cfl_cl_workflow_0101=>mc_decision-ok.

    ENDIF.

    add_msg( EXPORTING iv_text     = lv_text
             CHANGING  ct_bapiret2 = et_bapiret2 ).

    update_witext( lv_text ).
```
