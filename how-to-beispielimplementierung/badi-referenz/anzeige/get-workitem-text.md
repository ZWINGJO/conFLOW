# `get_workitem_text`

| | |
|---|---|
| **Wann** | Beim Öffnen des Workitems. |
| **Rein** | IS_DATA           die Instanz |
| **Raus** | CT_WORKITEM_TEXT  der Textblock, Zeile für Zeile |

**DAS FORMAT IST SAPSCRIPT-ITF, NICHT HTML**

Der Parametertyp heißt /C09/CFL_HTML_TABLE_TT, und die ausgelieferte Musterimplementierung hängt `<br>` hinein. Beides ist irreführend. Was der Workitem-Anzeiger auswertet, sind SAPscript-ZEICHENFORMATE:

```
<H>ORDER</>      richtig - fett
<b>ORDER</b>     wird stillschweigend ENTFERNT
```

ITF liest `<b>` als Zeichenformat namens 'b', kennt es nicht, und löscht die Klammern kommentarlos. Kein Fett, kein sichtbares Tag, keine Fehlermeldung - der irreführendste denkbare Ausgang, weil er wie "HTML wird nicht unterstützt" aussieht.

Geschlossen wird IMMER mit `</>`, nie mit `</H>`.

**AUFBAU, DER SICH BEWÄHRT HAT**

Befund zuerst, dann Blöcke mit fetter Überschrift. Der Bearbeiter soll nach zwei Zeilen wissen, worum es geht, und erst danach die Einzelheiten lesen.

**WAS HIER NICHT HINEINGEHÖRT**

Rechnen. Severity und Empfehlung gehören in einen HINTERGRUNDSCHRITT und von dort in den Container. Nur dann steht im Audit Trail, WAS DAS SYSTEM EMPFOHLEN HAT - und ob der Bearbeiter davon abgewichen ist. Rechnet die Anzeige selbst, ist diese Information weg, sobald das Workitem geschlossen ist.

## Der Code

```abap
    CLEAR ct_workitem_text.

    DATA(lv_id) = is_data-id.

*--------------------------------------------------------------------*
* Befund
*--------------------------------------------------------------------*
    APPEND |<H>{ get_val( iv_element = zcl_cfl_const_00900=>mc_prop-severity
                          iv_id      = lv_id ) }</> - | &&
           |{ fmt_amount( iv_element = zcl_cfl_const_00900=>mc_prop-net_value
                          iv_id      = lv_id ) } | &&
           |{ get_val( iv_element = zcl_cfl_const_00900=>mc_prop-currency
                       iv_id      = lv_id ) }|
      TO ct_workitem_text.

    APPEND space TO ct_workitem_text.

*--------------------------------------------------------------------*
* Block "Beleg"
*--------------------------------------------------------------------*
    APPEND '<H>DOCUMENT</>' TO ct_workitem_text.

    APPEND |Number   : { fmt_doc( get_val( iv_element = zcl_cfl_const_00900=>mc_prop-doc_number
                                           iv_id      = lv_id ) ) }|
      TO ct_workitem_text.

    APPEND |Item     : { get_val( iv_element = zcl_cfl_const_00900=>mc_prop-doc_item
                                  iv_id      = lv_id ) }|
      TO ct_workitem_text.

    APPEND |Vendor   : { fmt_doc( get_val( iv_element = zcl_cfl_const_00900=>mc_prop-vendor
                                           iv_id      = lv_id ) ) }|
      TO ct_workitem_text.

    APPEND space TO ct_workitem_text.

*--------------------------------------------------------------------*
* Block "Regel"
*
* Warum die Regel am Workitem steht und nicht nur ihr Ergebnis: der
* Bearbeiter soll nachvollziehen koennen, warum er gefragt wird. Ein
* Workitem, das nur "bitte entscheiden" sagt, erzeugt Rueckfragen.
*--------------------------------------------------------------------*
    APPEND '<H>RULE</>' TO ct_workitem_text.

    IF is_true( get_val( iv_element = zcl_cfl_const_00900=>mc_prop-limit_hit
                         iv_id      = lv_id ) ) = abap_true.
      APPEND |Value exceeds the approval limit of { zcl_cfl_const_00900=>mc_limit_value } - decision required.|
        TO ct_workitem_text.
    ELSE.
      APPEND 'Value within the approval limit.' TO ct_workitem_text.
    ENDIF.
```
