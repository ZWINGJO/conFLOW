# Die Konstantenklasse

Konstantenklasse zum Referenz-Workflow 00900.

Sie ist das ERSTE Objekt, das beim Nachbau entsteht - vor der BAdI-Klasse. Grund: jeder Schlüssel aus dem Customizing taucht im Code wieder auf, und jeder davon ist ein zweistelliger Code ohne Bedeutung. `IF ls_s03-gen_stat = '02'` ist nicht lesbar und beim nächsten Customizing-Umbau nicht auffindbar.

Wer sie später anlegt, hat die Codes schon zwanzigmal getippt.

Die Klasse enthält AUSSCHLIESSLICH Konstanten - keine Logik. Damit können BAdI-Klasse, conMOBILE-Model, Query-Provider einer eigenen Fiori-App und Reports sie gemeinsam benutzen, ohne voneinander abzuhängen.

## Die ganze Klasse

```abap
CLASS zcl_cfl_const_00900 DEFINITION
  PUBLIC
  FINAL
  CREATE PUBLIC.

*----------------------------------------------------------------------*
* Konstantenklasse zum Referenz-Workflow 00900.
*
* Sie ist das ERSTE Objekt, das beim Nachbau entsteht - vor der
* BAdI-Klasse. Grund: jeder Schluessel aus dem Customizing taucht im
* Code wieder auf, und jeder davon ist ein zweistelliger Code ohne
* Bedeutung. `IF ls_s03-gen_stat = '02'` ist nicht lesbar und beim
* naechsten Customizing-Umbau nicht auffindbar.
*
* Wer sie spaeter anlegt, hat die Codes schon zwanzigmal getippt.
*
* Die Klasse enthaelt AUSSCHLIESSLICH Konstanten - keine Logik. Damit
* koennen BAdI-Klasse, conMOBILE-Model, Query-Provider einer eigenen
* Fiori-App und Reports sie gemeinsam benutzen, ohne voneinander
* abzuhaengen.
*----------------------------------------------------------------------*

  PUBLIC SECTION.

*--------------------------------------------------------------------*
* Identitaet - was diesen Workflow ausmacht
*
* MC_WF_DEFINITION muss mit dem Filterwert der BAdI-Implementierung
* uebereinstimmen und mit dem Schluessel in /C09/CFL_C06.
*
* MC_OBJECTTYPE ist der BOR-Typ, unter dem die Instanz gefuehrt wird.
* Er muss KEIN in SWO1 angelegtes Objekt sein - conFLOW behandelt
* TYPEID als freien Schluessel. Ein eigener Typ je Workflow
* (`ZCFL00900`) ist deshalb erlaubt und meistens die bessere Wahl:
* er verhindert, dass sich zwei Workflows auf demselben BOR-Typ
* gegenseitig ihre Instanzen sehen. Wer den Standardtyp nimmt
* (`BUS2012` = Einkaufsbeleg), erbt dessen Verhalten in
* EXECUTE_DEFAULT_METHOD und in der Objektanzeige - dafuer teilt er
* sich den Raum mit allem anderen, was auf BUS2012 laeuft.
*--------------------------------------------------------------------*
    CONSTANTS mc_wf_definition TYPE /c09/cfl_s01-wf_definition VALUE '00900' ##NO_TEXT.
    CONSTANTS mc_objecttype    TYPE swo_objtyp                 VALUE 'BUS2012' ##NO_TEXT.

*--------------------------------------------------------------------*
* Schritte - die Schluessel aus /C09/CFL_C01
*
* Zwei Konventionen, die conFLOW selbst voraussetzt:
*   - Hintergrundschritte beginnen mit 'B'. Das Framework wertet das
*     erste Zeichen aus (siehe GET_STATUS_MAIL_DYNAMIC weiter unten:
*     `is_data-gen_stat+0(1) = 'B'` unterscheidet Dialog von Batch).
*   - Endeschritte beginnen ueblicherweise mit 'X'. Das ist Konvention,
*     keine Pruefung - aber jede conFLOW-Installation liest es so.
*--------------------------------------------------------------------*
    CONSTANTS:
      BEGIN OF mc_stat,
        start        TYPE /c09/cfl_c01-gen_stat VALUE 'X0',   " Start
        classify     TYPE /c09/cfl_c01-gen_stat VALUE 'B1',   " Hintergrund: Beleg lesen und bewerten
        approve      TYPE /c09/cfl_c01-gen_stat VALUE '01',   " Dialog: Einkauf entscheidet
        escalate     TYPE /c09/cfl_c01-gen_stat VALUE '02',   " Dialog: Vorgesetzter entscheidet
        post         TYPE /c09/cfl_c01-gen_stat VALUE 'B2',   " Hintergrund: Rueckschreibung
        notify       TYPE /c09/cfl_c01-gen_stat VALUE 'B3',   " Hintergrund: Benachrichtigung
        end_ok       TYPE /c09/cfl_c01-gen_stat VALUE 'X1',   " Ende: freigegeben
        end_rejected TYPE /c09/cfl_c01-gen_stat VALUE 'X2',   " Ende: abgelehnt
        end_auto     TYPE /c09/cfl_c01-gen_stat VALUE 'X3',   " Ende: nichts zu tun
      END OF mc_stat.

*--------------------------------------------------------------------*
* Bearbeiterkreise - die Schluessel aus /C09/CFL_C05
*
* GEN_STAT_USER ist NICHT dasselbe wie GEN_STAT. Ein Schritt (GEN_STAT)
* kann mehrere Bearbeiterkreise haben - das ist der Weg zu parallelen
* Workitems. Und derselbe Bearbeiterkreis kann an mehreren Schritten
* haengen.
*
* GET_ACTORS bekommt den GEN_STAT_USER, nicht den GEN_STAT. Wer im
* Hook auf GEN_STAT verzweigt, hat frueher oder spaeter den Fall, in
* dem es nicht mehr passt.
*--------------------------------------------------------------------*
    CONSTANTS:
      BEGIN OF mc_gsu,
        buyer      TYPE /c09/cfl_c05-gen_stat_user VALUE '01',   " Einkaeufer
        supervisor TYPE /c09/cfl_c05-gen_stat_user VALUE '02',   " Vorgesetzter
        creator    TYPE /c09/cfl_c05-gen_stat_user VALUE '03',   " Ersteller des Belegs
        mail_info  TYPE /c09/cfl_c05-gen_stat_user VALUE 'M1',   " nur Mailempfaenger, kein Workitem
      END OF mc_gsu.

*--------------------------------------------------------------------*
* Container-Attribute - die Elementnamen in /C09/CFL_S04
*
* Faustregel fuer die Auswahl: in den Container gehoert, was den Beleg
* IDENTIFIZIERT oder in die ENTSCHEIDUNG eingeht. Alles, was nur
* angezeigt wird und im Beleg steht, wird von dort gelesen.
*
* Der Grund ist nicht Speicherplatz, sondern Wahrheit: ein
* Container-Wert ist eine Kopie und altert. Wer den Lieferantennamen
* mitspeichert, zeigt ihn ein Jahr spaeter falsch an. Wer den
* freigegebenen BETRAG mitspeichert, hat recht - denn entschieden
* wurde ueber diesen Betrag, auch wenn der Beleg inzwischen ein
* anderer ist.
*--------------------------------------------------------------------*
    CONSTANTS:
      BEGIN OF mc_prop,
        " identifiziert den Beleg
        doc_number  TYPE /c09/cfl_s04-element VALUE 'DOC_NUMBER' ##NO_TEXT,
        doc_item    TYPE /c09/cfl_s04-element VALUE 'DOC_ITEM' ##NO_TEXT,
        " geht in die Entscheidung ein
        net_value   TYPE /c09/cfl_s04-element VALUE 'NET_VALUE' ##NO_TEXT,
        currency    TYPE /c09/cfl_s04-element VALUE 'CURRENCY' ##NO_TEXT,
        vendor      TYPE /c09/cfl_s04-element VALUE 'VENDOR' ##NO_TEXT,
        " in B1 berechnet - die Bewertung, nicht die Rohdaten
        severity    TYPE /c09/cfl_s04-element VALUE 'SEVERITY' ##NO_TEXT,
        limit_hit   TYPE /c09/cfl_s04-element VALUE 'LIMIT_HIT' ##NO_TEXT,
        " vom Bearbeiter gesetzt
        decision_by TYPE /c09/cfl_s04-element VALUE 'DECISION_BY' ##NO_TEXT,
        note        TYPE /c09/cfl_s04-element VALUE 'NOTE' ##NO_TEXT,
      END OF mc_prop.

*--------------------------------------------------------------------*
* Fachliche Werte
*--------------------------------------------------------------------*
    CONSTANTS:
      BEGIN OF mc_severity,
        green  TYPE string VALUE 'GREEN' ##NO_TEXT,
        yellow TYPE string VALUE 'YELLOW' ##NO_TEXT,
        red    TYPE string VALUE 'RED' ##NO_TEXT,
      END OF mc_severity.

*--------------------------------------------------------------------*
* Die Grenze, ab der ein Mensch entscheidet.
*
* Sie steht hier als Konstante, weil sie in einem Referenzbeispiel
* nachvollziehbar sein soll. In einer echten Installation gehoert so
* ein Wert in eine Customizing-Tabelle oder nach BRFplus - sonst
* braucht jede Aenderung einen Transport.
*--------------------------------------------------------------------*
    CONSTANTS mc_limit_value TYPE p LENGTH 8 DECIMALS 2 VALUE '10000.00'.

*--------------------------------------------------------------------*
* Workitem-Prioritaet (SWW_PRIO, 1-9, 1 = hoechste)
*
* Achtung bei 1: SAP verschickt Workitems der Prioritaet 1 als
* Express-Nachricht. Im Testsystem faellt das nicht auf, im
* Produktivsystem bekommt der Bearbeiter ein Popup. 4 ist die
* hoechste Stufe, die My Inbox noch als "High" anzeigt, ohne das
* auszuloesen.
*--------------------------------------------------------------------*
    CONSTANTS:
      BEGIN OF mc_prio,
        high   TYPE sww_prio VALUE 4,
        medium TYPE sww_prio VALUE 5,
      END OF mc_prio.

*--------------------------------------------------------------------*
* Schalter
*
* Ein Referenzbeispiel sollte zeigen, wie man Darstellungsdetails
* abschaltbar haelt. Faerbt eine My-Inbox-Version die Buttons anders
* als erwartet, ist ein Schalter im Termin schneller als ein
* Transport.
*--------------------------------------------------------------------*
    CONSTANTS mc_fiori_nature TYPE abap_bool VALUE abap_true.

*--------------------------------------------------------------------*
* Derselbe Schalter fuer den zweiten Faerbeweg - HTML im ALTTEXT, den
* der Entscheidungs-Screen im Business Workplace rendert.
*
* Zwei Schalter und nicht einer, weil die beiden Oberflaechen
* unabhaengig voneinander stoeren koennen: faellt in Fiori etwas auf,
* soll das GUI davon nichts merken und umgekehrt.
*--------------------------------------------------------------------*
    CONSTANTS mc_gui_html_color TYPE abap_bool VALUE abap_true.

    CONSTANTS:
      BEGIN OF mc_gui_color,
        positive TYPE string VALUE 'green' ##NO_TEXT,
        negative TYPE string VALUE 'red' ##NO_TEXT,
      END OF mc_gui_color.

*--------------------------------------------------------------------*
* Schriftgroesse fuer denselben Knopf. Farbe allein traegt im GUI zu
* wenig - die Leiste ist grau und der Text klein. Groesse verstaerkt
* dieselbe Aussage und trifft dieselben zwei Knoepfe, macht also kein
* zweites Signal auf.
*
* Leer heisst "keine Groessenangabe", dann bleibt nur die Farbe. Das
* ist der Wert, an dem man dreht, wenn die Buttonleiste zu breit wird.
*--------------------------------------------------------------------*
    CONSTANTS mc_gui_font_size TYPE string VALUE '120%' ##NO_TEXT.

ENDCLASS.


CLASS zcl_cfl_const_00900 IMPLEMENTATION.
ENDCLASS.
```
