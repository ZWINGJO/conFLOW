# Der Query-Provider
*`if_rap_query_provider~select` ist die einzige Methode. Sie muss vier Dinge tun — und **jedes Weglassen ist ein Laufzeitfehler, kein Aktivierungsfehler**.*

| Pflicht | warum |
| --- | --- |
| Filter auswerten | sonst liefert er alles statt einer Instanz |
| Zähler bedienen | sonst fehlt die Trefferzahl |
| **Paging abholen** | sonst wird die ganze Anfrage abgewiesen |
| Daten setzen | — |

**`ZCL_CFL_00500_QUERY`**

```abap
CLASS zcl_cfl_00500_query DEFINITION
  PUBLIC
  FINAL
  CREATE PUBLIC .

*--------------------------------------------------------------------*
* Query-Provider der Custom Entity ZCFL_00500_C_EXCEPTION.
*
* Liest den conFLOW-Container /c09/cfl_s04 ueber dieselben Helfer, die
* auch den Workitem-Text fuellen (ZCL_CFL_WORKFLOW_00500=>GET_VAL /
* GET_NUM / GET_DATE). Das ist Absicht: Text und App koennen so gar
* nicht auseinanderlaufen. Wer die Klartexte hier neu rechnet, baut
* genau den Fehler ein, der im Kundentermin auffaellt - im Text steht
* "not allowed", in der App steht " ".
*
* Aufgerufen wird der Provider aus My Inbox mit genau einem Schluessel;
* ohne Filter liefert er alle laufenden 00500-Instanzen, damit die App
* auch als eigenstaendige Kachel brauchbar ist.
*--------------------------------------------------------------------*
  PUBLIC SECTION.

    INTERFACES if_rap_query_provider .

    TYPES ty_ids TYPE STANDARD TABLE OF guid_16 WITH EMPTY KEY .

    " Auch der Behavior-Handler liest hierueber - eine Leseroutine fuer
    " Anzeige und Nachlesen nach dem Speichern.
    CLASS-METHODS read_one
      IMPORTING !iv_id        TYPE guid_16
      RETURNING VALUE(rs_row) TYPE zcfl_00500_c_exception .

*--------------------------------------------------------------------*
* Den Instanzschluessel aus dem OData-Filter holen.
*
* Oeffentlich, weil JEDER Provider dieses Services ihn braucht - die
* Hauptentitaet ebenso wie die Tabelle der Einteilungen. Zwei
* Implementierungen derselben Filterauswertung waeren genau die Art
* Duplikat, die beim naechsten Parameternamen auseinanderlaeuft.
*--------------------------------------------------------------------*
    CLASS-METHODS ids_from_filter
      IMPORTING !io_request  TYPE REF TO if_rap_query_request
      RETURNING VALUE(rt_id) TYPE ty_ids .

  PRIVATE SECTION.

    TYPES ty_rows TYPE STANDARD TABLE OF zcfl_00500_c_exception WITH EMPTY KEY.

ENDCLASS.

CLASS zcl_cfl_00500_query IMPLEMENTATION.

  METHOD if_rap_query_provider~select.

    DATA lt_row TYPE ty_rows.

    DATA(lt_id) = ids_from_filter( io_request ).

*--------------------------------------------------------------------*
* OHNE FILTER GIBT ES NICHTS.
*
* Hier stand bis zum 22.08.2026 ein Zweig, der ohne Filter alle
* Instanzen der WF-Definition las (SELECT auf /c09/cfl_s01, begrenzt
* auf 100). Gedacht war er fuer eine eigenstaendige Kachel, die es nie
* gab - und er kostete an drei Stellen:
*
*   1. Aufwand   100 Instanzen x ~20 Container-Werte einzeln = bis zu
*                2.000 Zugriffe fuer eine Liste, die niemand braucht
*   2. Bindung   ein SELECT auf eine conFLOW-Tabelle mehr, also eine
*                Abhaengigkeit mehr von deren Produktstand
*   3. Sicht     wer die URL kannte, konnte OHNE eine einzige GUID zu
*                kennen jeden laufenden Vorgang lesen
*
* Diese App ist eine Detailsicht auf EIN Workitem. Der Schluessel kommt
* aus dem Intent, Fiori Elements setzt ihn als Filter. Kommt kein
* Filter an, ist etwas falsch - und die richtige Antwort darauf ist
* nichts, nicht alles.
*
* Bewusst KEINE Ausnahme: ein Query-Provider, der wirft, reisst im
* Zweifel die Object Page mit. Leer ist hier die sichere Antwort.
*
* Und bewusst KEIN eigener Zweig mit vorzeitigem RETURN: der erste
* Versuch hatte einen, und er war falsch. Wer hier aussteigt, hat
* GET_PAGING nicht gerufen - und RAP weist die Anfrage dann komplett
* ab (RAP_RUNTIME/014), statt eine leere Liste zu liefern. Genau die
* Falle, die weiter unten dokumentiert ist, im eigenen Code.
*
* Es braucht den Zweig auch nicht: ist LT_ID leer, laeuft die Schleife
* nicht, LT_ROW bleibt leer, und die Antwortlogik darunter tut das
* Richtige - inklusive GET_PAGING.
*--------------------------------------------------------------------*
    LOOP AT lt_id INTO DATA(lv_id).
      DATA(ls_row) = read_one( lv_id ).
      IF ls_row-wfid IS NOT INITIAL.
        APPEND ls_row TO lt_row.
      ENDIF.
    ENDLOOP.

    IF io_request->is_total_numb_of_rec_requested( ).
      io_response->set_total_number_of_records( lines( lt_row ) ).
    ENDIF.

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

  METHOD ids_from_filter.
*--------------------------------------------------------------------*
* Der Schluessel kommt als 32-stellige Hex-Kette herein - so steht er
* in CFLQueryObject00 und so vertraegt ihn die URL. Die Zuweisung
* char(32) -> guid_16 ist die eingebaute C-nach-X-Konvertierung von
* ABAP: die Zeichen werden als Hex-Ziffern gelesen, nicht als Text.
* Sieht nach einem Fehler aus, ist aber genau richtig.
*--------------------------------------------------------------------*

    TRY.
        DATA(lt_range) = io_request->get_filter( )->get_as_ranges( ).
      CATCH cx_rap_query_filter_no_range.
        RETURN.
    ENDTRY.

*--------------------------------------------------------------------*
* Zwei Namen, ein Schluessel: WFID ist der Key, CFLQUERYOBJECT00 der
* Name, unter dem conFLOW ihn in die URL schreibt. Beide werden
* akzeptiert - je nachdem, ob der Aufruf aus der Inbox kommt oder
* jemand die Liste direkt oeffnet.
*--------------------------------------------------------------------*
    LOOP AT lt_range INTO DATA(ls_range) WHERE name = 'WFID'
                                            OR name = 'CFLQUERYOBJECT00'.
      LOOP AT ls_range-range INTO DATA(ls_sel) WHERE option = 'EQ' AND sign = 'I'.
        DATA lv_hex TYPE c LENGTH 32.
        DATA lv_id  TYPE guid_16.

        lv_hex = ls_sel-low.
        TRANSLATE lv_hex TO UPPER CASE.

        IF lv_hex CN '0123456789ABCDEF'.
          CONTINUE.
        ENDIF.

        lv_id = lv_hex.
        APPEND lv_id TO rt_id.
      ENDLOOP.
    ENDLOOP.

  ENDMETHOD.

  METHOD read_one.

    DATA(lv_severity) = zcl_cfl_workflow_00500=>get_val(
                          iv_element = zcl_cfl_const_00500=>mc_prop-severity
                          iv_id      = iv_id ).

*--------------------------------------------------------------------*
* Kein SEVERITY im Container heisst: B1 ist noch nicht gelaufen oder
* die Instanz gehoert zu einem anderen Workflow. Dann keine Zeile -
* eine halb gefuellte Object Page ist schlimmer als gar keine.
*--------------------------------------------------------------------*
    IF lv_severity IS INITIAL.
      RETURN.
    ENDIF.

    rs_row-wfid     = iv_id.
    rs_row-severity = lv_severity.

*--------------------------------------------------------------------*
* Der Schritt, in dem die Instanz gerade steht. Gesucht wird das
* offene DIALOG-Workitem: eine Instanz hat im Lauf mehrere s03-Saetze
* (jeder Hintergrundschritt ist einer), aber hoechstens eines davon
* wartet auf einen Menschen. Genau dessen gen_stat steuert, was die
* App anbietet.
*--------------------------------------------------------------------*
    SELECT SINGLE s~gen_stat, s~wi_id
      FROM /c09/cfl_s03 AS s
           INNER JOIN swwwihead AS h ON h~wi_id = s~wi_id
      WHERE s~id      = @iv_id
        AND h~wi_type = 'W'
        AND h~wi_stat = 'READY'
      INTO (@rs_row-genstat, @rs_row-workitemid).        "#EC CI_BUFFJOIN

    rs_row-severitycriticality = SWITCH #( lv_severity
      WHEN zcl_cfl_const_00500=>mc_severity-red    THEN 1
      WHEN zcl_cfl_const_00500=>mc_severity-yellow THEN 2
      ELSE 3 ).

    rs_row-recommendation = zcl_cfl_workflow_00500=>pretty_code(
      zcl_cfl_workflow_00500=>get_val( iv_element = zcl_cfl_const_00500=>mc_prop-recommendation
                                       iv_id      = iv_id ) ).

    rs_row-recommendationreason = zcl_cfl_workflow_00500=>get_val(
      iv_element = zcl_cfl_const_00500=>mc_prop-recomm_reason
      iv_id      = iv_id ).

*--------------------------------------------------------------------*
* ORDER
*--------------------------------------------------------------------*
    rs_row-salesorder = zcl_cfl_workflow_00500=>fmt_doc(
      zcl_cfl_workflow_00500=>get_val( iv_element = zcl_cfl_const_00500=>mc_prop-order
                                       iv_id      = iv_id ) ).

    rs_row-salesorderitem = zcl_cfl_workflow_00500=>fmt_doc(
      zcl_cfl_workflow_00500=>get_val( iv_element = zcl_cfl_const_00500=>mc_prop-item
                                       iv_id      = iv_id ) ).

    rs_row-customer = zcl_cfl_workflow_00500=>fmt_doc(
      zcl_cfl_workflow_00500=>get_val( iv_element = zcl_cfl_const_00500=>mc_prop-customer
                                       iv_id      = iv_id ) ).

    rs_row-customername = zcl_cfl_workflow_00500=>get_val(
      iv_element = zcl_cfl_const_00500=>mc_prop-customer_name
      iv_id      = iv_id ).

    rs_row-material = zcl_cfl_workflow_00500=>get_val(
      iv_element = zcl_cfl_const_00500=>mc_prop-material
      iv_id      = iv_id ).

*--------------------------------------------------------------------*
* PROMISE
*--------------------------------------------------------------------*
    rs_row-requestedquantity = zcl_cfl_workflow_00500=>get_num(
      iv_element = zcl_cfl_const_00500=>mc_prop-req_qty  iv_id = iv_id ).
    rs_row-requesteddate     = zcl_cfl_workflow_00500=>get_date(
      iv_element = zcl_cfl_const_00500=>mc_prop-req_date iv_id = iv_id ).

    rs_row-confirmedquantity = zcl_cfl_workflow_00500=>get_num(
      iv_element = zcl_cfl_const_00500=>mc_prop-conf_qty  iv_id = iv_id ).
    rs_row-confirmeddate     = zcl_cfl_workflow_00500=>get_date(
      iv_element = zcl_cfl_const_00500=>mc_prop-conf_date iv_id = iv_id ).

    rs_row-unit = 'PC'.

*--------------------------------------------------------------------*
* EXCEPTION
*--------------------------------------------------------------------*
    rs_row-backorderquantity = zcl_cfl_workflow_00500=>get_num(
      iv_element = zcl_cfl_const_00500=>mc_prop-bo_qty iv_id = iv_id ).
    rs_row-backorderpercent  = zcl_cfl_workflow_00500=>get_num(
      iv_element = zcl_cfl_const_00500=>mc_prop-bo_pct iv_id = iv_id ).

    DATA(lv_delay) = zcl_cfl_workflow_00500=>get_num(
      iv_element = zcl_cfl_const_00500=>mc_prop-delay_days iv_id = iv_id ).

    " "1 days" ist der Satz, an dem im Termin jemand haengenbleibt
    rs_row-delaytext = COND #( WHEN lv_delay = 1 THEN |1 day|
                               ELSE |{ lv_delay DECIMALS = 0 } days| ).

    rs_row-reason = zcl_cfl_workflow_00500=>get_val(
      iv_element = zcl_cfl_const_00500=>mc_prop-reason iv_id = iv_id ).

*--------------------------------------------------------------------*
* CUSTOMER RULES - Klartext, wortgleich mit dem Workitem-Text
*--------------------------------------------------------------------*
    rs_row-segment = zcl_cfl_workflow_00500=>pretty_code(
      zcl_cfl_workflow_00500=>get_val( iv_element = zcl_cfl_const_00500=>mc_prop-segment
                                       iv_id      = iv_id ) ).

    DATA(lv_partial_ok) = zcl_cfl_workflow_00500=>is_true(
      zcl_cfl_workflow_00500=>get_val( iv_element = zcl_cfl_const_00500=>mc_prop-partial_allowed
                                       iv_id      = iv_id ) ).

    rs_row-partialdeliverytext = COND #( WHEN lv_partial_ok = abap_true
                                         THEN 'allowed' ELSE 'not allowed' ).

    DATA(lv_cancel_ok) = zcl_cfl_workflow_00500=>is_true(
      zcl_cfl_workflow_00500=>get_val( iv_element = zcl_cfl_const_00500=>mc_prop-cancel_allowed
                                       iv_id      = iv_id ) ).

    rs_row-cancelresidualtext = COND #( WHEN lv_cancel_ok = abap_true
                                        THEN 'allowed'
                                        ELSE 'not allowed - decision required' ).

    " Die Governance-Regel ist die Pointe der Demo - sie darf rot sein
    rs_row-cancelcriticality = COND #( WHEN lv_cancel_ok = abap_true THEN 3 ELSE 1 ).

*--------------------------------------------------------------------*
* Kopfzeile - dieselbe Zeile wie oben im Workitem-Text, ohne Ampel.
* Die Ampel malt Fiori Elements aus SEVERITYCRITICALITY selbst; ein
* zweites Emoji im Titel waere doppelt.
*--------------------------------------------------------------------*
    rs_row-headline = COND #(
      WHEN rs_row-recommendation IS INITIAL
      THEN |{ lv_severity }: no exception|
      ELSE |{ lv_severity }: { rs_row-recommendation } recommended| ).

*--------------------------------------------------------------------*
* Bearbeitung - unveraendert aus dem Container, OHNE Ersatzwert.
*
* Hier stand ein Fallback auf RECOMMENDATION, falls PROPOSED_ACTION
* leer ist. Er ist am 22.08.2026 entfallen, und das war ein echter
* Fehler, kein Schoenheitsfehler: die Oberflaeche zeigte damit eine
* ausgewaehlte Aktion an, die im Container nicht stand. Ein Feld darf
* nie einen Zustand suggerieren, den das Backend nicht besitzt - der
* Bearbeiter haette den conFLOW-Button gedrueckt in der Annahme, seine
* Auswahl sei erfasst.
*
* Stattdessen setzt B1 die berechnete Empfehlung gleich als Startwert
* von PROPOSED_ACTION. Damit ist jeder angezeigte Wert ein
* persistierter Wert, und der Vergleich mit RECOMMENDATION bleibt
* moeglich: die bleibt unveraendert stehen und zeigt, wovon der
* Bearbeiter abgewichen ist.
*
* Nebeneffekt, der den Audit Trail erst eindeutig macht: "leer" hiess
* vorher zweierlei - nicht entschieden ODER Empfehlung akzeptiert.
*--------------------------------------------------------------------*
    rs_row-proposedaction = zcl_cfl_workflow_00500=>get_val(
      iv_element = zcl_cfl_const_00500=>mc_prop-proposed_action
      iv_id      = iv_id ).

    rs_row-proposedactiontext = zcl_cfl_workflow_00500=>pretty_code( rs_row-proposedaction ).

    rs_row-decisionreason = zcl_cfl_workflow_00500=>get_val(
      iv_element = zcl_cfl_const_00500=>mc_prop-decision_reason
      iv_id      = iv_id ).

    rs_row-decisionreasontext = zcl_cfl_workflow_00500=>pretty_code( rs_row-decisionreason ).

    rs_row-decisionnote = zcl_cfl_workflow_00500=>get_val(
      iv_element = zcl_cfl_const_00500=>mc_prop-decision_note
      iv_id      = iv_id ).

*--------------------------------------------------------------------*
* Der Bearbeitungszustand als FELDER, nicht als Regel im Browser.
*
* Dieselbe Quelle wie die Feature Control und wie die Abweisung in
* SETDECISION: ZCL_CFL_00500_RULES. Der Controller liest das Ergebnis
* und legt GEN_STAT nicht mehr selbst aus.
*--------------------------------------------------------------------*
    rs_row-isdecisioneditable = zcl_cfl_00500_rules=>is_editable( rs_row-genstat ).

    rs_row-decisionstatustext = zcl_cfl_00500_rules=>status_text( rs_row-genstat ).

    rs_row-decisionstatuscriticality = zcl_cfl_00500_rules=>status_criticality( rs_row-genstat ).

    rs_row-decisionhint = zcl_cfl_00500_rules=>decision_hint(
                            iv_reason = rs_row-decisionreason
                            iv_note   = rs_row-decisionnote ).

  ENDMETHOD.

ENDCLASS.
```

## Die teuerste Falle: `get_paging( )` ist Pflicht
RAP prüft **nicht**, ob das Ergebnis stimmt, sondern ob der Provider jedes angeforderte Merkmal *angefasst* hat. Fehlt der Aufruf, weist SADL die komplette Anfrage ab:
```
RAP_RUNTIME/004   Error occurred during execution of query provider
SADL_DIAGNOSTIC/006 The requested feature is not implemented
RAP_RUNTIME/014   Query not fully covered by implementation:
                  Call to method if_rap_query_request~get_paging missing
```
Drei Gründe, warum das teuer ist:
<ol>
  - Der Text klingt nach einer fehlenden **Funktion**, gemeint ist ein fehlender **Aufruf**.
  - Die Klasse aktiviert sauber. Der Fehler erscheint erst zur Laufzeit.
  - In der Anwendung sieht man nur ein leeres Feld — kein Dump, keine Meldung, nichts in ST22.
</ol>
{% hint style="danger" %}
**Und in JEDEM Provider.** Im Referenzprojekt war er in der Hauptentität vorhanden und in der Wertehilfe vergessen — die App lief, nur das Dropdown blieb leer.
{% endhint %}
{% hint style="info" %}
**10 Sekunden**

Entity Set direkt im Browser aufrufen: `GET <service>/<EntitySet>?sap-client=100`. Kommen Zeilen, liegt es nicht am Backend. Kommt `RAP_RUNTIME/014`, steht die Ursache im Klartext da.
{% endhint %}

## Kein ungefilterter Listenmodus
{% hint style="info" %}
**Hier stand einmal ein Zweig, der ohne Filter alle Instanzen las.** Er ist entfernt — er kostete an drei Stellen: Laufzeit (bis zu 2.000 Einzelzugriffe), eine zusätzliche Abhängigkeit von einer Framework-Tabelle, und er öffnete den Zugriff auf jeden laufenden Vorgang, ohne dass man eine GUID kennen musste.
{% endhint %}
{% hint style="success" %}
**Wichtig beim Entfernen:** keinen Sonderzweig mit vorzeitigem `RETURN` bauen. Der überspränge `get_paging( )`. Ist die ID-Liste leer, läuft die Schleife einfach nicht — und die normale Antwortlogik tut das Richtige.
{% endhint %}
