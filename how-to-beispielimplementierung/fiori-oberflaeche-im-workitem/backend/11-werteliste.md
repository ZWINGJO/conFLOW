# Die Werteliste
*Dieselbe Mechanik, kleiner — und mit demselben Paging-Zwang.*

**`ZCFL_00500_C_ACTIONVH`**

```abap
@EndUserText.label: 'Order Promise Exception - actions'
@ObjectModel.query.implementedBy: 'ABAP:ZCL_CFL_00500_VH'
@Search.searchable: false

// Die Werteliste fuers Dropdown. Bewusst dieselben Schluessel wie
// ZCL_CFL_CONST_00500=>MC_RECOMMENDATION - der vorbelegte Wert ist die
// in B1 berechnete Empfehlung, und der muss in dieser Liste vorkommen,
// sonst zeigt Fiori Elements ein leeres Feld.
define custom entity ZCFL_00500_C_ACTIONVH
{
      @EndUserText.label: 'Action'
      @UI.lineItem: [{ position: 10 }]
  key ActionKey  : abap.char(20);

      @EndUserText.label: 'Description'
      @UI.lineItem: [{ position: 20 }]
      ActionText : abap.char(40);
}
```
**`ZCL_CFL_00500_VH`**

```abap
CLASS zcl_cfl_00500_vh DEFINITION
  PUBLIC
  FINAL
  CREATE PUBLIC .

*--------------------------------------------------------------------*
* Werteliste fuer das Dropdown "Proposed action".
*
* Die Schluessel sind dieselben wie in ZCL_CFL_CONST_00500=>
* MC_RECOMMENDATION. Das ist nicht Bequemlichkeit, sondern Bedingung:
* vorbelegt wird mit der in B1 berechneten Empfehlung, und ein
* vorbelegter Wert, der nicht in der Liste steht, erscheint in Fiori
* Elements als leeres Feld.
*
* Die Beschriftungen entstehen aus PRETTY_CODE - derselbe Weg wie im
* Workitem-Text und in der Buttonleiste.
*--------------------------------------------------------------------*
  PUBLIC SECTION.

    INTERFACES if_rap_query_provider .

ENDCLASS.

CLASS zcl_cfl_00500_vh IMPLEMENTATION.

  METHOD if_rap_query_provider~select.

    DATA lt_row TYPE STANDARD TABLE OF zcfl_00500_c_actionvh WITH EMPTY KEY.

*--------------------------------------------------------------------*
* Die Liste kommt aus ZCL_CFL_00500_RULES - derselben Quelle, gegen
* die SETDECISION prueft. Stuende sie hier ein zweites Mal, muesste man
* einen neuen Eintrag an zwei Stellen nachtragen, und wer es vergisst,
* baut den unangenehmsten Fall: die Oberflaeche bietet etwas an, das
* der Server ablehnt.
*
* Seit dem 22.08.2026 sind es die ENTSCHEIDUNGSGRUENDE, nicht mehr die
* Aktionen. Die Aktionen stehen auf den conFLOW-Buttons; ein Dropdown
* mit denselben Werten daneben war eine Doppelung, die niemand
* aufloesen konnte.
*--------------------------------------------------------------------*
    LOOP AT zcl_cfl_00500_rules=>get_valid_reasons( ) INTO DATA(lv_reason).
      APPEND VALUE #(
        actionkey  = lv_reason
        actiontext = zcl_cfl_workflow_00500=>pretty_code( lv_reason ) ) TO lt_row.
    ENDLOOP.

    IF io_request->is_total_numb_of_rec_requested( ).
      io_response->set_total_number_of_records( lines( lt_row ) ).
    ENDIF.

    IF io_request->is_data_requested( ).

*--------------------------------------------------------------------*
* Paging MUSS abgeholt werden - auch bei vier Zeilen.
*
* RAP prueft nicht, ob das Ergebnis stimmt, sondern ob der Provider
* jedes angeforderte Merkmal ANGEFASST hat. Wer GET_PAGING nicht ruft,
* bekommt die ganze Anfrage abgewiesen:
*
*   RAP_RUNTIME/014  Query not fully covered by implementation:
*                    Call to method if_rap_query_request~get_paging missing
*
* Das ist die irrefuehrendste Stelle daran: der Fehler klingt nach
* einer fehlenden Funktion, gemeint ist ein fehlender Aufruf. Und er
* kommt erst zur Laufzeit - die Klasse aktiviert sauber, die Liste
* bleibt im Dropdown einfach leer.
*
* Derselbe Ablauf wie in ZCL_CFL_00500_QUERY: erst schneiden, dann
* zurueckgeben.
*--------------------------------------------------------------------*
      DATA(lo_paging) = io_request->get_paging( ).
      DATA(lv_offset) = lo_paging->get_offset( ).
      DATA(lv_page)   = lo_paging->get_page_size( ).

      IF lv_offset > 0.
        DELETE lt_row TO lv_offset.
      ENDIF.

      IF lv_page >= 0 AND lines( lt_row ) > lv_page.
        DATA(lv_from) = lv_page + 1.
        DELETE lt_row FROM lv_from.
      ENDIF.

      io_response->set_data( lt_row ).
    ENDIF.

  ENDMETHOD.

ENDCLASS.
```

{% hint style="success" %}
**Die Liste kommt aus der Regelklasse** — derselben Quelle, gegen die `setDecision` prüft. Eine neue Aktion ist damit tatsächlich eine Zeile, nicht zwei.
{% endhint %}
{% hint style="info" %}
**Der vorbelegte Wert muss in der Liste vorkommen**, sonst zeigt Fiori Elements ein leeres Feld — obwohl ein Wert gespeichert ist. Das ist der unangenehmste Ausgang: die App sieht aus, als sei nichts erfasst.
{% endhint %}
