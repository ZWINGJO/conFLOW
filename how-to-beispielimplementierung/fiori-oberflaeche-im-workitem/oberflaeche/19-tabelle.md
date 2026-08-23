# Eine Tabelle in der Object Page
*Zwei neue Objekte und eine Association — Fiori Elements macht den Rest.*

## Die Regel, die vorher zu klären ist
Woher kommen die Daten?
Bis hierher kam **alles** aus dem Framework-Container. Eine Tabelle mit Belegdaten — Einteilungen, Partner, Positionen — steht dort nicht, und sie gehört auch nicht hinein: eine Liste in einem Key-Value-Speicher abzulegen hieße, sie zu erfinden.
**Die Regel:** Was in die *Entscheidung* eingeht, steht im Container — dort hat es der Trigger abgelegt und der Klassifizierungsschritt bewertet. Was nur *angezeigt* wird und im Beleg steht, wird von dort gelesen. Das ist keine zweite Wahrheit, sondern dieselbe in ihrer Originalform.
Der Weg dorthin führt trotzdem über den Container: die Workflow-Instanz kennt Beleg und Position, und daraus wird der Schlüssel für den SD-Zugriff. **Nicht die App kennt den Auftrag, sondern die Instanz.**

## Die Kind-Entität
**`ZCFL_00500_C_SCHEDLINE`**

```abap
@EndUserText.label: 'Order Promise Exception - schedule lines'
@ObjectModel.query.implementedBy: 'ABAP:ZCL_CFL_00500_SCHED'
@Metadata.allowExtensions: true

// Die Einteilungen der Auftragsposition - die Tabelle, aus der die
// Ausnahme ueberhaupt entsteht.
//
// ERSTE STELLE, an der die App aus SD liest statt aus dem Container.
// Die Regel dahinter:
//
//   Was in die ENTSCHEIDUNG eingeht, steht im Container (Menge,
//   Termin, Rueckstand, Severity) - dort hat es der Trigger abgelegt
//   und B1 bewertet.
//
//   Was nur ANGEZEIGT wird und im Beleg steht, wird von dort gelesen.
//   Es waere keine zweite Wahrheit, sondern dieselbe in ihrer
//   Originalform - und eine Einteilungsliste im Key-Value-Container
//   abzulegen hiesse, sie zu erfinden.
//
// Der Schluessel ist die conFLOW-Instanz plus die Einteilungsnummer.
// WfId muss dabei sein: die Association der Hauptentitaet laeuft
// darueber, und Fiori Elements filtert die Tabelle danach.
define custom entity ZCFL_00500_C_SCHEDLINE
{
  key WfId              : abap.char(32);
  key ScheduleLine      : abap.numc(4);

      @EndUserText.label: 'Delivery date'
      DeliveryDate      : abap.dats;

      @EndUserText.label: 'Requested'
      @Semantics.quantity.unitOfMeasure: 'Unit'
      RequestedQuantity : abap.dec(15,3);

      @EndUserText.label: 'Confirmed'
      @Semantics.quantity.unitOfMeasure: 'Unit'
      ConfirmedQuantity : abap.dec(15,3);

      // Was auf dieser Einteilung offen bleibt. Berechnet, nicht aus
      // VBEP - die Tabelle fuehrt kein Delta.
      @EndUserText.label: 'Open'
      @Semantics.quantity.unitOfMeasure: 'Unit'
      OpenQuantity      : abap.dec(15,3);

      @Semantics.unitOfMeasure: true
      Unit              : abap.unit(3);

      // 1 = rot, 2 = gelb, 3 = gruen. Faerbt die Zeile: voll
      // bestaetigt ist gruen, teilbestaetigt gelb, nichts bestaetigt
      // rot. Damit sieht man der Tabelle die Ausnahme an, ohne zu
      // rechnen.
      Criticality       : abap.int1;
}
```
{% hint style="info" %}
**Der Schlüssel muss `WfId` enthalten.** Die Association läuft darüber, und Fiori Elements filtert die Tabelle danach — beim Aufklappen schickt es einen **eigenen Request** an das Entity Set der Kind-Entität.
{% endhint %}

## Der Provider
**`ZCL_CFL_00500_SCHED`**

```abap
CLASS zcl_cfl_00500_sched DEFINITION
  PUBLIC
  FINAL
  CREATE PUBLIC .

*--------------------------------------------------------------------*
* Query-Provider der Einteilungen.
*
* Der einzige Provider dieses Services, der aus SD liest statt aus dem
* conFLOW-Container. Die Begruendung steht in der Custom Entity: was in
* die Entscheidung eingeht, kommt aus dem Container; was nur angezeigt
* wird und im Beleg steht, wird von dort gelesen.
*
* Der Weg dorthin fuehrt trotzdem ueber den Container: die
* conFLOW-Instanz kennt ORDER und ITEM, und daraus wird der Schluessel
* fuer VBEP. Ohne Instanz gibt es keine Einteilungen - eine Tabelle
* ohne Bezug zum Workitem waere sinnlos.
*
* Diese Klasse braucht KEINE Freundschaft zu ZCL_CFL_WORKFLOW_00500.
* Sie liest ueber die oeffentlichen Routinen des Hauptproviders -
* IDS_FROM_FILTER fuer den Schluessel, READ_ONE fuer Beleg und
* Position. Damit bleibt der Kreis der Klassen, die in die Interna der
* Workflow-Klasse sehen duerfen, so klein wie er ist.
*--------------------------------------------------------------------*
  PUBLIC SECTION.

    INTERFACES if_rap_query_provider .

  PRIVATE SECTION.

    TYPES ty_rows TYPE STANDARD TABLE OF zcfl_00500_c_schedline WITH EMPTY KEY .

    CLASS-METHODS rows_of
      IMPORTING !iv_id         TYPE guid_16
      RETURNING VALUE(rt_row)  TYPE ty_rows .

ENDCLASS.

CLASS zcl_cfl_00500_sched IMPLEMENTATION.

  METHOD if_rap_query_provider~select.

    DATA lt_row TYPE ty_rows.

*--------------------------------------------------------------------*
* Ohne Instanzschluessel gibt es nichts - dieselbe Regel wie in
* ZCL_CFL_00500_QUERY. Die Filterauswertung steht dort und wird von
* hier gerufen, damit es sie nur einmal gibt.
*--------------------------------------------------------------------*
    LOOP AT zcl_cfl_00500_query=>ids_from_filter( io_request ) INTO DATA(lv_id).
      APPEND LINES OF rows_of( lv_id ) TO lt_row.
    ENDLOOP.

    IF io_request->is_total_numb_of_rec_requested( ).
      io_response->set_total_number_of_records( lines( lt_row ) ).
    ENDIF.

*--------------------------------------------------------------------*
* GET_PAGING ist Pflicht - auch hier. RAP prueft nicht das Ergebnis,
* sondern ob jedes angeforderte Merkmal angefasst wurde; fehlt der
* Aufruf, weist SADL die ganze Anfrage ab (RAP_RUNTIME/014) und die
* Tabelle bleibt leer, ohne dass etwas im Protokoll steht.
*--------------------------------------------------------------------*
    IF io_request->is_data_requested( ).
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

  METHOD rows_of.

    DATA lv_vbeln TYPE vbeln_va.
    DATA lv_posnr TYPE posnr_va.

*--------------------------------------------------------------------*
* Beleg und Position stehen im Container - der Trigger hat sie beim
* Start hineingeschrieben. Das ist der Uebergang vom Workflow in den
* Beleg, und er gehoert genau hierher: nicht die App kennt den
* Auftrag, sondern die Workflow-Instanz.
*
* Gelesen wird ueber READ_ONE des Hauptproviders, NICHT ueber GET_VAL
* der Workflow-Klasse. Letzteres ist privat und nur den drei Klassen
* zugaenglich, die in deren GLOBAL FRIENDS stehen - diese hier gehoert
* nicht dazu, und sie dort einzutragen waere der falsche Weg: die
* Freundschaft oeffnet ALLES Private, gebraucht wuerden zwei Felder.
*
* READ_ONE liest mehr, als hier gebraucht wird. Das ist der Preis, und
* er ist klein: die Tabelle wird einmal je Aufklappen geholt, nicht je
* Zeile.
*--------------------------------------------------------------------*
    DATA(ls_head) = zcl_cfl_00500_query=>read_one( iv_id ).

    IF ls_head-wfid IS INITIAL.
      RETURN.
    ENDIF.

*--------------------------------------------------------------------*
* ALPHA = IN ist hier keine Kosmetik, sondern Bedingung.
*
* READ_ONE liefert die Belegnummer fuer die ANZEIGE - durch FMT_DOC,
* also ohne fuehrende Nullen: "14669" statt "0000014669". Auf der
* Datenbank steht sie mit. Ohne die Rueckkonvertierung faende der
* SELECT nichts, und die Tabelle bliebe leer - ohne Fehlermeldung,
* ohne Dump, ohne Hinweis worauf.
*
* Genau die Sorte Fehler, die man erst im Kundentermin bemerkt: alles
* laeuft, nur die Liste ist leer.
*--------------------------------------------------------------------*
    lv_vbeln = |{ ls_head-salesorder     ALPHA = IN }|.
    lv_posnr = |{ ls_head-salesorderitem ALPHA = IN }|.

    IF lv_vbeln IS INITIAL OR lv_posnr IS INITIAL.
      RETURN.
    ENDIF.

*--------------------------------------------------------------------*
* Die Einteilungen der Position, chronologisch.
*
* WMENG ist die angeforderte, BMENG die bestaetigte Menge. Beide
* stehen in VBEP; das Delta fuehrt die Tabelle nicht - es wird hier
* gerechnet, damit die Spalte "Open" nicht jeder Leser selbst bilden
* muss.
*--------------------------------------------------------------------*
    SELECT vbeln, posnr, etenr, edatu, wmeng, bmeng, vrkme
      FROM vbep
      WHERE vbeln = @lv_vbeln
        AND posnr = @lv_posnr
      ORDER BY etenr
      INTO TABLE @DATA(lt_vbep).

    LOOP AT lt_vbep INTO DATA(ls_vbep).

      DATA(lv_open) = ls_vbep-wmeng - ls_vbep-bmeng.

*--------------------------------------------------------------------*
* Die Ampel je Zeile - damit man der Tabelle die Ausnahme ansieht,
* ohne zu rechnen:
*
*   gruen   voll bestaetigt
*   gelb    teilweise bestaetigt
*   rot     nichts bestaetigt
*--------------------------------------------------------------------*
      DATA(lv_crit) = COND int1(
        WHEN lv_open <= 0            THEN 3
        WHEN ls_vbep-bmeng > 0       THEN 2
        ELSE                              1 ).

      APPEND VALUE #(
        wfid              = iv_id
        scheduleline      = ls_vbep-etenr
        deliverydate      = ls_vbep-edatu
        requestedquantity = ls_vbep-wmeng
        confirmedquantity = ls_vbep-bmeng
        openquantity      = COND #( WHEN lv_open > 0 THEN lv_open ELSE 0 )
        unit              = ls_vbep-vrkme
        criticality       = lv_crit ) TO rt_row.

    ENDLOOP.

  ENDMETHOD.

ENDCLASS.
```
{% hint style="success" %}
**Die Filterauswertung wird nicht kopiert.** `ids_from_filter( )` stand privat im Hauptprovider und ist jetzt öffentlich — beide Provider brauchen dieselbe Logik. Zwei Implementierungen derselben Filterauswertung laufen beim nächsten Parameternamen auseinander.
{% endhint %}
{% hint style="danger" %}
**Und wieder `get_paging( )`.** In *jedem* Provider, auch in diesem. Fehlt der Aufruf, bleibt die Tabelle leer, ohne dass etwas im Protokoll steht.
{% endhint %}

## Association und Facet
```
" in der Hauptentitaet
_SchedLine : association [0..*] to ZCFL_00500_C_SCHEDLINE
               on $projection.WfId = _SchedLine.WfId;

" in der Metadata Extension - anderer Facet-Typ!
{ id: 'SchedLine', purpose: #STANDARD, type: #LINEITEM_REFERENCE,
  label: 'Schedule lines', targetElement: '_SchedLine', position: 50 }
```
|  | Feldgruppe | Tabelle |
| --- | --- | --- |
| Facet-Typ | `#FIELDGROUP_REFERENCE` | `#LINEITEM_REFERENCE` |
| Ziel | `targetQualifier` | `targetElement` — die Association |
| Spalten | `@UI.fieldGroup` | `@UI.lineItem` in der Extension der **Kind**-Entität |
**`ZCFL_00500_X_SCHEDLINE`**

```abap
@Metadata.layer: #CORE

// Die Spalten der Einteilungstabelle.
//
// @UI.lineItem legt Reihenfolge und Beschriftung fest; ohne diese
// Annotationen rendert Fiori Elements eine leere Tabelle - die Zeilen
// sind da, aber keine Spalte ist sichtbar.
//
// Die Ampel haengt an CRITICALITY und faerbt die bestaetigte Menge:
// gruen voll, gelb teilweise, rot nichts bestaetigt. Damit sieht man
// der Tabelle die Ausnahme an, ohne die Differenz zu bilden.
annotate entity ZCFL_00500_C_SCHEDLINE with
{
  @UI.hidden: true
  WfId;

  @UI.lineItem: [{ position: 10, label: 'No.' }]
  ScheduleLine;

  @UI.lineItem: [{ position: 20, label: 'Delivery date' }]
  DeliveryDate;

  @UI.lineItem: [{ position: 30, label: 'Requested' }]
  RequestedQuantity;

  @UI.lineItem: [{ position: 40, label: 'Confirmed', criticality: 'Criticality' }]
  ConfirmedQuantity;

  @UI.lineItem: [{ position: 50, label: 'Open' }]
  OpenQuantity;

  @UI.hidden: true
  Unit;

  @UI.hidden: true
  Criticality;
}
```
{% hint style="info" %}
**Ohne `@UI.lineItem` bleibt die Tabelle leer** — die Zeilen sind da, aber keine Spalte ist sichtbar. Und die Kind-Entität muss in der Service Definition **exponiert** sein, sonst kann die Association nicht aufgelöst werden.
{% endhint %}
{% hint style="success" %}
**Criticality macht die Tabelle lesbar.** Die bestätigte Menge wird grün bei voll, gelb bei teilweise, rot bei nicht bestätigt — damit sieht man der Liste die Ausnahme an, ohne die Differenz zu bilden.
{% endhint %}
