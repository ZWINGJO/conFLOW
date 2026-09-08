# Drei Schranken — jede schützt etwas anderes
*Die App besitzt nicht den Prozess. Sie liest den Kontext und hält den Arbeitszustand fest.*

| Schranke | Frage | Beispiele |
| --- | --- | --- |
| Workitem-Reservierung | Wer darf dieses Workitem bearbeiten? | Framework, nichts zu tun |
| RAP `setDecision` | Darf dieser **Arbeitszustand** gespeichert werden? | Aktionscode gültig? · Schritt editierbar? · Instanz erlaubt? |
| conFLOW Decision-Exit | Darf der **Prozess weiterlaufen**? | Notiz bei Eskalation? · fachliche Voraussetzungen noch erfüllt? · Abschluss erlaubt? |

## Wo die dritte Schranke sitzt
conFLOW ruft nach dem Klick auf einen Entscheidungs-Button zwei Hooks, die beide über `cv_subrc` abbrechen können. Der Unterschied ist entscheidend:
| Hook | Oberfläche | kann abbrechen | kann eine **Meldung** mitgeben |
| --- | --- | --- | --- |
| `get_after_execution_mobile` | conMOBILE / BSP | ja | **ja** — `cs_t100msg` |
| `get_after_execution` | SAP-GUI | ja | **nein** |
Beide bekommen `iv_altkey` — den **tatsächlich geklickten Ausgang**. Das erlaubt mehr als eine Pflichtfeldprüfung.

**`check_completion( ) — eine Prüfung, zwei Einstiege`**

```abap
  METHOD check_completion.
*--------------------------------------------------------------------*
* Zwei Aufgaben, in dieser Reihenfolge:
*
* 1. DIE AKTION AUS DEM BUTTON FESTSCHREIBEN. Der geklickte Ausgang
*    IST die Entscheidung - der Akt, den conFLOW protokolliert und nach
*    dem es verzweigt. PROPOSED_ACTION wird daraus gesetzt.
*
*    Das ist kein Ueberschreiben einer Benutzereingabe: seit dem
*    22.08.2026 bietet die App im Dropdown GRUENDE an, keine Aktionen.
*    Vorher standen dort dieselben vier Werte wie auf den Buttons -
*    zwei Bedienelemente fuer eine Entscheidung, und wer sie
*    unterschiedlich bediente, bekam seine Auswahl stillschweigend
*    ersetzt.
*
*    Nebenwirkung, die bleibt: das Auto-Save-Rennen ist damit
*    geschlossen. Selbst wenn der Speichervorgang der App den Klick
*    nicht eingeholt hat, steht hinterher die richtige Aktion im
*    Container.
*
* 2. VERBINDLICH PRUEFEN, ob der GRUND vollstaendig ist. Dieselbe
*    Bedingung wie der Hinweis waehrend der Bearbeitung
*    (ZCL_CFL_00500_RULES=>DECISION_HINT) - nur mit anderer Wirkung:
*    dort ein gelber Hinweis, hier ein Abbruch.
*--------------------------------------------------------------------*
    SELECT SINGLE * FROM /c09/cfl_s03 INTO @DATA(ls_cfl_s03) "#EC CI_ALL_FIELDS_NEEDED
      WHERE wi_id = @iv_wi_id.                                "#EC CI_NOORDER
    IF sy-subrc <> 0.
      RETURN.
    ENDIF.

*--------------------------------------------------------------------*
* Nur die Entscheidungsschritte. Hintergrundschritte haben keinen
* Bearbeiter, der etwas haette eingeben koennen.
*--------------------------------------------------------------------*
    IF zcl_cfl_00500_rules=>is_editable( ls_cfl_s03-gen_stat ) = abap_false AND
       ls_cfl_s03-gen_stat <> zcl_cfl_const_00500=>mc_stat-escalation.
      RETURN.
    ENDIF.

*--------------------------------------------------------------------*
* 1. Die Aktion steht am Button
*--------------------------------------------------------------------*
    DATA(lv_action) = zcl_cfl_00500_rules=>action_of_key( iv_altkey ).

    IF lv_action IS INITIAL.
*     Ein Ausgang ohne fachliche Aktion - etwa "nok". Nichts zu tun.
      RETURN.
    ENDIF.

    DATA(lv_note)   = get_val( iv_element = zcl_cfl_const_00500=>mc_prop-decision_note
                               iv_id      = ls_cfl_s03-id ).
    DATA(lv_reason) = get_val( iv_element = zcl_cfl_const_00500=>mc_prop-decision_reason
                               iv_id      = ls_cfl_s03-id ).

*--------------------------------------------------------------------*
* 2. Verbindliche Pruefung - BEVOR geschrieben wird.
*
* Geprueft wird der GRUND, nicht die Aktion: die kommt vom Knopf und
* ist damit per Definition gesetzt. Fehlt der Grund, oder ist er
* "Sonstiges" ohne Erlaeuterung, bleibt das Workitem offen - der
* Bearbeiter ergaenzt und klickt erneut.
*
* Wird abgebrochen, darf im Container nichts stehen, was der Bearbeiter
* nicht bestaetigt hat.
*--------------------------------------------------------------------*
    rv_error = zcl_cfl_00500_rules=>decision_hint( iv_reason = lv_reason
                                                   iv_note   = lv_note ).
    IF rv_error IS NOT INITIAL.
      RETURN.
    ENDIF.

*--------------------------------------------------------------------*
* Alles in Ordnung - die tatsaechliche Entscheidung festschreiben.
*--------------------------------------------------------------------*
    IF get_val( iv_element = zcl_cfl_const_00500=>mc_prop-proposed_action
                iv_id      = ls_cfl_s03-id ) <> lv_action.
      set_val( iv_element = zcl_cfl_const_00500=>mc_prop-proposed_action
               iv_id      = ls_cfl_s03-id
               iv_value   = lv_action ).
    ENDIF.

  ENDMETHOD.
```

**`get_after_execution_mobile( ) — mit Meldung`**

```abap
  METHOD /c09/cfl_if_badi_0101~get_after_execution_mobile.
*--------------------------------------------------------------------*
* DIE DRITTE SCHRANKE - die einzige Stelle, an der der Prozess
* angehalten werden kann.
*
* Der Hook bekommt IV_ALTKEY, also den tatsaechlich geklickten Ausgang,
* und kann ueber CV_SUBRC abbrechen. Damit ist er das Gegenstueck zu
* RAP SETDECISION:
*
*   SETDECISION  darf dieser ARBEITSZUSTAND gespeichert werden?
*                Halbfertig ist erlaubt - sonst kann niemand arbeiten.
*   HIER         darf der PROZESS WEITERLAUFEN?
*                Hier ist eine fehlende Begruendung verbindlich.
*
* Warum ausgerechnet dieser Hook: er ist der einzige mit CS_T100MSG.
* GET_AFTER_EXECUTION kann zwar auch abbrechen, aber ohne Meldung -
* der Bearbeiter saehe dann nur, dass nichts passiert. Beide rufen
* deshalb dieselbe Pruefung, und diese hier sagt zusaetzlich, warum.
*
* Der Name ist irrefuehrend: laut conFLOW-Dokumentation haengt er am
* conMOBILE-/BSP-Pfad. Ob die Fiori-Inbox ihn ruft, ist im Lauf zu
* pruefen - bei GET_FIORI_TASK_DEC_OP_ACT sah es genauso aus und der
* Hook lief trotzdem. Merksatz aus diesem Projekt: bei
* conFLOW-BAdI-Hooks ausprobieren statt Where-Used glauben.
*
* Faellt er in der Inbox aus, wirkt die Schranke dort nicht - dann
* bleibt es beim gelben Hinweis waehrend der Bearbeitung, und die
* Pruefung muss in einen Hintergrundschritt hinter der Entscheidung.
*--------------------------------------------------------------------*
    CONSTANTS mc_var_len TYPE i VALUE 50.

    DATA(lv_error) = check_completion( iv_wi_id  = iv_wi_id
                                       iv_altkey = iv_altkey ).

    IF lv_error IS INITIAL.
      RETURN.
    ENDIF.

*--------------------------------------------------------------------*
* Freitext als T100-Nachricht: 00/398 besteht aus '&1&2&3&4', also
* vier Variablen a 50 Zeichen. Derselbe Weg wie in ADD_LOG - eine
* eigene Nachrichtenklasse waere sauberer und ist der naechste
* Ausbauschritt, aber nicht die Sache, an der die Schranke haengt.
*--------------------------------------------------------------------*
    DATA(lv_len) = strlen( lv_error ).

    cs_t100msg-msgid = '00'.
    cs_t100msg-msgno = '398'.
    cs_t100msg-msgty = 'E'.
    cs_t100msg-msgv1 = lv_error.

    IF lv_len > mc_var_len.
      cs_t100msg-msgv2 = lv_error+mc_var_len.
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

  ENDMETHOD.
```

**`get_after_execution( ) — SAP-GUI, ohne Meldung`**

```abap
  METHOD /c09/cfl_if_badi_0101~get_after_execution.
*--------------------------------------------------------------------*
* Dieselbe Schranke fuer den SAP-GUI-Pfad.
*
* Der Hook hat KEIN CS_T100MSG - er kann abbrechen, aber nicht sagen
* warum. Der Bearbeiter sieht also nur, dass das Workitem offen
* bleibt. Das ist unschoen und trotzdem richtig: lieber ein
* unerklaerter Abbruch als eine Eskalation ohne Begruendung, die
* niemand mehr nachvollziehen kann.
*
* Der Grund steht im Kontextblock am Workitem, den der Bearbeiter vor
* sich hat - dort erscheint derselbe Satz als Hinweis.
*
* Gerufen wird dieselbe Methode wie in GET_AFTER_EXECUTION_MOBILE:
* eine Pruefung, zwei Einstiege. conFLOW ruft je nach Oberflaeche den
* einen oder den anderen Hook.
*--------------------------------------------------------------------*
    IF check_completion( iv_wi_id  = iv_wi_id
                         iv_altkey = iv_altkey ) IS NOT INITIAL.
      cv_subrc = 9.
    ENDIF.

  ENDMETHOD.
```

## Zwei Aufgaben, in dieser Reihenfolge
{% hint style="success" %}
**1 · Der Button gewinnt.** Der Bearbeiter kann im Dropdown eine Aktion wählen und danach einen anderen Button drücken — oder gar nichts wählen. Entschieden hat er mit dem **Button**; das ist der Akt, den conFLOW protokolliert und nach dem es verzweigt. Also wird `PROPOSED_ACTION` daraus gesetzt. **Damit ist das Auto-Save-Rennen geschlossen**: selbst wenn der Speichervorgang der App den Klick nicht mehr eingeholt hat, steht hinterher im Container, was tatsächlich entschieden wurde.
{% endhint %}
{% hint style="info" %}
**2 · Verbindlich prüfen — mit derselben Bedingung wie der Hinweis.** `decision_hint( )` wird hier ein zweites Mal gerufen, nur mit anderer Wirkung: während der Bearbeitung ein gelber Hinweis, beim Abschluss ein Abbruch. **Eine Bedingung, zwei Wirkungen** — nicht zwei Prüfungen, die auseinanderlaufen können.
{% endhint %}
{% hint style="danger" %}
**Reihenfolge beachten:** erst prüfen, dann schreiben. Wird abgebrochen, darf im Container nichts stehen, was der Bearbeiter nicht bestätigt hat.
{% endhint %}

## Die Unsicherheit, die bleibt
{% hint style="info" %}
**Der Name `get_after_execution_mobile` ist irreführend.** Laut conFLOW-Dokumentation hängt er am conMOBILE-/BSP-Pfad. Ob die **Fiori-Inbox** ihn ruft, muss im Lauf geprüft werden — bei `get_fiori_task_dec_op_act` sah es genauso aus, und der Hook lief trotzdem. **Merksatz: bei conFLOW-BAdI-Hooks ausprobieren statt Where-Used glauben.** Fällt er in der Inbox aus, bleibt es dort beim gelben Hinweis, und die Prüfung muss in einen Hintergrundschritt hinter der Entscheidung.
{% endhint %}

{% hint style="info" %}
**Die Regel**

Bietet das Framework einen Punkt, an dem der Zustandsübergang **abgelehnt** werden kann, ist das die richtige Stelle für die abschließende fachliche Prüfung — nicht die Oberfläche und nicht der OData-Service. **Die beiden schützen den Zugriff, das Framework schützt den Prozess.**
{% endhint %}

## Der Notaus
Zwei Ebenen, beide binnen Sekunden wirksam:
```
" Ebene 1 - im Code, am Anfang von set_inbox_ui( )
IF zcl_cfl_NNNNN_const=>mc_inbox_ui = abap_false.
  RETURN.
ENDIF.
```
**Neue** Workitems bekommen dann keine Container-Elemente und fallen auf den Textblock zurück. Bestehende behalten ihre App.
{% hint style="info" %}
**Ebene 2 — das Target Mapping löschen.** Wirkt **sofort und für alle** Workitems, auch bestehende, ohne Codeänderung und ohne Transport. Der schnellere Hebel, wenn es mitten in einer Demo weg muss.
{% endhint %}
{% hint style="success" %}
**Der Rückfall ist keine Notlösung, sondern der Normalfall** für alle Workitems ohne Container-Elemente. Deshalb funktioniert er zuverlässig — er wird ständig benutzt, nicht nur im Fehlerfall.
{% endhint %}
