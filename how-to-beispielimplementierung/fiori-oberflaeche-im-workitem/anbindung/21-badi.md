# Die Anbindung im conFLOW-BAdI
*Drei Methoden aus der Workflow-Klasse — mehr braucht es nicht, um die App an ein Workitem zu hängen.*

## Der Hook
`get_after_creation_workitem` läuft einmal je Workitem, nach dem Anlegen und vor der ersten Anzeige. Dort werden Priorität **und** Oberfläche gesetzt.

**`set_inbox_ui( ) — hängt die App an das Workitem`**

```abap
  METHOD set_inbox_ui.
*--------------------------------------------------------------------*
* Eigene Oberflaeche fuer dieses Workitem in der Fiori My Inbox.
*
* Gesetzt werden drei Container-Elemente des Workitems; conFLOW liest
* sie ueber die dynamische Visualisierung des Tasks TS00388601 und
* baut daraus den Intent. Query-Parameter 00 ist der Schluessel, mit
* dem die App ihren Satz findet - die conFLOW-Instanz als Hex-Kette,
* weil sie so durch die URL passt.
*
* Faellt hier etwas aus, bleibt es beim Standard-Textblock. Das ist
* der Grund fuer das stille CATCH: eine Oberflaeche, die nicht kommt,
* darf kein Workitem verhindern.
*--------------------------------------------------------------------*
    DATA lv_key TYPE c LENGTH 32.

    IF zcl_cfl_const_00500=>mc_inbox_ui = abap_false.
      RETURN.
    ENDIF.

    IF iv_wi_id IS INITIAL OR iv_id IS INITIAL.
      RETURN.
    ENDIF.

    lv_key = iv_id.

    TRY.
        DATA(lo_container) = cl_swf_run_workitem_context=>get_instance(
                               im_wiid = iv_wi_id )->if_wapi_workitem_context~get_wi_container( ).

        lo_container->set( name  = zcl_cfl_const_00500=>mc_visu-semantic_object
                           value = zcl_cfl_const_00500=>mc_ui_semantic_object ).

        lo_container->set( name  = zcl_cfl_const_00500=>mc_visu-action
                           value = zcl_cfl_const_00500=>mc_ui_action ).

        lo_container->set( name  = zcl_cfl_const_00500=>mc_visu-query_obj00
                           value = lv_key ).

      CATCH cx_swf_ifs_exception.
        RETURN.
    ENDTRY.

  ENDMETHOD.
```

{% hint style="success" %}
**Das stille `CATCH` ist Absicht.** Fällt hier etwas aus, bleibt es beim Standard-Textblock. Eine Oberfläche, die nicht kommt, darf kein Workitem verhindern.
{% endhint %}

## Priorität — die Ampel als Arbeitsliste
**`set_priority( ) — zweistufig, mit Grund`**

```abap
  METHOD set_priority.
* Zwei Stufen, weil im After-Create-Hook nichts selbstverstaendlich ist:
* das Workitem existiert da noch nicht auf der Datenbank.
*
* 1. Der Workitem-Manager der laufenden Transaktion - dieselbe Schiene,
*    die update_witext( ) benutzt. Gesucht wird ueber die WI_ID, nicht
*    ueber wi_type: der Hook meint ein bestimmtes Workitem.
* 2. SWW_WI_PRIORITY_CHANGE, die FM, die SAP_WAPI_CHANGE_WORKITEM_PRIO
*    umhuellt - aber mit abgeschalteten Pruefungen. Genau die duerften
*    die WAPI blockiert haben (`INVALID_STATUS`): das Workitem ist
*    gerade erst entstanden. do_commit bleibt leer, das COMMIT gehoert
*    dem Framework.
    DATA lt_instances TYPE swwtwihndl.
    DATA ls_instances LIKE LINE OF lt_instances.
    DATA lo_flow      TYPE REF TO if_swf_run_wim_internal.

    TRY.
        DATA(lo_factory) = cl_swf_run_wim_factory=>get_instance( ).
        lt_instances = lo_factory->get_registered_workitems( ).

        LOOP AT lt_instances INTO ls_instances.
          TRY.
              lo_flow ?= ls_instances.
              IF lo_flow->m_sww_wihead-wi_id = iv_wi_id.
                lo_flow->if_swf_run_wim~change_priority( iv_prio ).
                rv_ok = abap_true.
              ENDIF.
            CATCH cx_root.
          ENDTRY.
        ENDLOOP.

      CATCH cx_root.
    ENDTRY.

    CALL FUNCTION 'SWW_WI_PRIORITY_CHANGE'
      EXPORTING
        wi_id                 = iv_wi_id
        priority              = iv_prio
        do_commit             = space
        authorization_checked = abap_true
        preconditions_checked = abap_true
      EXCEPTIONS
        no_authorization      = 1
        update_failed         = 2
        invalid_type          = 3
        invalid_status        = 4
        OTHERS                = 5.

    IF sy-subrc = 0.
      rv_ok = abap_true.
    ENDIF.

  ENDMETHOD.
```
{% hint style="danger" %}
**`SAP_WAPI_CHANGE_WORKITEM_PRIO` ist hier der falsche Weg** — nachgemessen: die Priorität blieb unverändert. Die WAPI liest `SWWWIHEAD` von der Datenbank und prüft den Workitem-Status; beides passt nicht zu einem Workitem, das gerade erst entsteht.
{% endhint %}

## Der Startwert — ein angezeigtes Feld muss ein gespeichertes Feld sein
**`classify_exception( ) — der Hintergrundschritt, der die Startwerte setzt`**

```abap
  METHOD classify_exception.
*--------------------------------------------------------------------*
* B1 - Klassifizierung und Weiche
*
*   RED    -> ok  -> 01   (Entscheidung mit Frist, c02)
*   YELLOW -> uc1 -> 03   (Entscheidung ohne Frist, c09)
*   GREEN  -> uc2 -> X3   (Workflow schliesst sich selbst, c09)
*   Fehler -> nok -> X2   (setzt das Framework selbst bei E/A)
*--------------------------------------------------------------------*

    DATA lv_severity       TYPE string.
    DATA lv_recommendation TYPE string.
    DATA lv_reason         TYPE string.
    DATA lv_cancel_allowed TYPE abap_bool.

    DATA(lv_id) = is_cfl_s03-id.

    DATA(lv_bo_qty)     = get_num( iv_element = zcl_cfl_const_00500=>mc_prop-bo_qty     iv_id = lv_id ).
    DATA(lv_bo_pct)     = get_num( iv_element = zcl_cfl_const_00500=>mc_prop-bo_pct     iv_id = lv_id ).
    DATA(lv_delay)      = get_num( iv_element = zcl_cfl_const_00500=>mc_prop-delay_days iv_id = lv_id ).
    DATA(lv_segment)    = get_val( iv_element = zcl_cfl_const_00500=>mc_prop-segment         iv_id = lv_id ).
    DATA(lv_partial_ok) = is_true( get_val( iv_element = zcl_cfl_const_00500=>mc_prop-partial_allowed iv_id = lv_id ) ).

*--------------------------------------------------------------------*
* 1. Cancellation Policy - die Regel steht genau hier, nicht im Trigger
*--------------------------------------------------------------------*
    IF lv_bo_pct <= zcl_cfl_const_00500=>mc_cancel_tolerance_pct.
      lv_cancel_allowed = abap_true.
    ELSE.
      lv_cancel_allowed = abap_false.
    ENDIF.

*--------------------------------------------------------------------*
* 2. Severity (Abschnitt 8 des Konzepts)
*--------------------------------------------------------------------*
    IF lv_bo_qty <= 0 AND lv_delay <= 0.
      lv_severity = zcl_cfl_const_00500=>mc_severity-green.
      lv_reason   = 'Confirmation matches request - no exception'.

    ELSEIF lv_segment    = zcl_cfl_const_00500=>mc_segment_key_account OR
           lv_bo_pct     > zcl_cfl_const_00500=>mc_red_pct_threshold   OR
           lv_delay      > zcl_cfl_const_00500=>mc_red_delay_threshold OR
           lv_partial_ok = abap_false.
      lv_severity = zcl_cfl_const_00500=>mc_severity-red.

    ELSE.
      lv_severity = zcl_cfl_const_00500=>mc_severity-yellow.
    ENDIF.

*--------------------------------------------------------------------*
* 3. Empfehlung (Abschnitt 12)
*--------------------------------------------------------------------*
    IF lv_severity = zcl_cfl_const_00500=>mc_severity-green.
      lv_recommendation = space.

    ELSEIF lv_partial_ok = abap_true.
      lv_recommendation = zcl_cfl_const_00500=>mc_recommendation-partial_delivery.
      lv_reason = 'Partial delivery allowed - confirmed quantity ships on time'.

    ELSEIF lv_delay <= zcl_cfl_const_00500=>mc_red_delay_threshold.
      lv_recommendation = zcl_cfl_const_00500=>mc_recommendation-accept_new_date.
      lv_reason = 'Delay within tolerance - accept new confirmation'.

    ELSEIF lv_cancel_allowed = abap_true.
      lv_recommendation = zcl_cfl_const_00500=>mc_recommendation-cancel_remaining.
      lv_reason = 'Residual quantity within cancellation tolerance'.

    ELSE.
      lv_recommendation = zcl_cfl_const_00500=>mc_recommendation-escalate.
      lv_reason = 'No standard resolution available - decision required'.
    ENDIF.

*--------------------------------------------------------------------*
* 4. Ergebnis in den Container - ab hier ist es Teil des Audit Trails
*--------------------------------------------------------------------*
    set_val( iv_element = zcl_cfl_const_00500=>mc_prop-severity       iv_id = lv_id iv_value = lv_severity ).
    set_val( iv_element = zcl_cfl_const_00500=>mc_prop-recommendation iv_id = lv_id iv_value = lv_recommendation ).

*--------------------------------------------------------------------*
* Die Empfehlung wird ZUSAETZLICH Startwert der Bearbeitung.
*
* Zwei Elemente mit demselben Anfangswert, und das ist Absicht:
*
*   RECOMMENDATION   was das System empfohlen hat - bleibt stehen
*   PROPOSED_ACTION  was gilt - der Bearbeiter kann es aendern
*
* Der Vergleich beider zeigt im Audit Trail, ob jemand abgewichen ist.
*
* Ohne diese Zeile blendete die App die Empfehlung nur VOR, waehrend
* der Container leer blieb - ein Feld, das aussieht wie gespeichert,
* ohne es zu sein. Wer dann den conFLOW-Button drueckt, ohne das
* Dropdown anzufassen, hinterlaesst keinen Wert. Und "leer" war
* zweideutig: nicht entschieden oder Empfehlung akzeptiert.
*
* Nebenwirkung, die zaehlt: ein Folgeschritt, der PROPOSED_ACTION
* liest, findet nie mehr einen leeren Wert - auch dann nicht, wenn der
* Bearbeiter den Genehmigen-Knopf schneller drueckt, als die App
* speichern kann.
*--------------------------------------------------------------------*
    set_val( iv_element = zcl_cfl_const_00500=>mc_prop-proposed_action iv_id = lv_id iv_value = lv_recommendation ).
    set_val( iv_element = zcl_cfl_const_00500=>mc_prop-recomm_reason  iv_id = lv_id iv_value = lv_reason ).
    set_val( iv_element = zcl_cfl_const_00500=>mc_prop-cancel_allowed iv_id = lv_id iv_value = lv_cancel_allowed ).

    DATA(lv_log) = |Severity { lv_severity }, recommendation { lv_recommendation } ({ lv_reason })|.
    add_log( EXPORTING iv_text = lv_log CHANGING ct_bapiret2 = et_bapiret2 ).

*--------------------------------------------------------------------*
* Ampel ins Workflow-Protokoll - sonst steht dort nur der neutrale
* c01t-Text und man sieht dem GREEN-Fall nicht an, dass er GREEN war
*--------------------------------------------------------------------*
    IF lv_recommendation IS INITIAL.
      update_witext( |Exception classification: { lv_severity }| ).
    ELSE.
      update_witext( |Exception classification: { lv_severity } - { lv_recommendation }| ).
    ENDIF.

*--------------------------------------------------------------------*
* 5. Weiche
*--------------------------------------------------------------------*
    CASE lv_severity.
      WHEN zcl_cfl_const_00500=>mc_severity-green.
        ev_decision_key = /c09/cfl_cl_workflow_0101=>mc_decision-uc2.
      WHEN zcl_cfl_const_00500=>mc_severity-yellow.
        ev_decision_key = /c09/cfl_cl_workflow_0101=>mc_decision-uc1.
      WHEN OTHERS.
        ev_decision_key = /c09/cfl_cl_workflow_0101=>mc_decision-ok.
    ENDCASE.

  ENDMETHOD.
```

Warum zwei Elemente mit demselben Anfangswert
`RECOMMENDATION` ist die **unveränderliche Systemempfehlung**. `PROPOSED_ACTION` ist der **aktuell geltende Arbeitswert** und wird initial mit ihr vorbelegt. Der Vergleich beider zeigt im Audit Trail, ob jemand abgewichen ist.
Ohne die zweite Zeile blendet die App die Empfehlung nur *vor*, während der Container leer bleibt — **ein Feld, das aussieht wie gespeichert, ohne es zu sein.** Wer dann den Workflow-Button drückt, ohne das Dropdown anzufassen, hinterlässt keinen Wert. Und „leer" wäre zweideutig: *nicht entschieden* oder *Empfehlung akzeptiert*.

{% hint style="info" %}
**Die Gleichheit beider Werte beweist keine Bestätigung.** Der Bestätigungsakt entsteht erst beim Klick auf den conFLOW-Button und steht mit Benutzer, Zeitpunkt und Ausgang im Workitem-Protokoll. Zusätzliche Felder wie `CONFIRMED_BY` wären genau die zweite Wahrheit, die Kapitel 2 verbietet.
{% endhint %}
