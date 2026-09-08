# Die Konstantenklasse — Elementnamen, Schrittcodes, Notaus
*Das erste Objekt, und das langweiligste. Es hält alles, was an mehr als einer Stelle vorkommt — und einen Schalter, der die ganze Erweiterung abschaltet.*

Sie trägt vier Arten von Konstanten, und die Trennung ist keine Ordnungsliebe:

| Gruppe | Warum zentral |
| --- | --- |
| `mc_stat` · `mc_gsu` | Die conFLOW-Schrittcodes. Sie stehen im Customizing (`c01`, `c03`) **und** im Code. Zwei Orte lassen sich nicht vermeiden, drei schon. |
| `mc_prop` | Die Namen der Container-Elemente. Sie erscheinen im Customizing (`c04`), in den `§{…}`-Platzhaltern der Texte (`c01t`) und in jedem `get_val( )`. Ein Tippfehler liefert hier **leer statt Fehler**. |
| `mc_ui_*` | Semantic Object und Action. Müssen zeichengleich mit dem Target Mapping im Launchpad sein — und dort steht kein Compiler daneben. |
| `mc_inbox_ui` | **Der Notaus.** Siehe unten. |

{% hint style="success" %}
**Der Notaus ist Pflicht, nicht Kür.** `mc_inbox_ui = abap_false` lässt `set_inbox_ui( )` die drei Container-Elemente nicht mehr setzen — neue Workitems fallen auf den ITF-Textblock zurück, ohne dass eine Zeile Code geändert wird. Für einen Kundentermin ist das die billigste Versicherung, die es gibt. **Er wirkt nur auf neue Workitems.** Muss es mitten in einer Demo sofort weg, ist das **Target Mapping im Katalog** der schnellere Hebel: löschen, und jedes Workitem zeigt wieder den Textblock — auch die bestehenden.
{% endhint %}

**`ZCL_CFL_CONST_00500`**

```abap
CLASS zcl_cfl_const_00500 DEFINITION
  PUBLIC
  FINAL
  CREATE PUBLIC .

  PUBLIC SECTION.

*--------------------------------------------------------------------*
* Identitaet
*--------------------------------------------------------------------*
    CONSTANTS mc_wf_definition TYPE /c09/cfl_wf_definition VALUE '00500' ##NO_TEXT.
    CONSTANTS mc_typeid        TYPE sibftypeid             VALUE 'ZCFL00500' ##NO_TEXT.
    CONSTANTS mc_objtype       TYPE swo_objtyp             VALUE 'ZCFL00500' ##NO_TEXT.
    CONSTANTS mc_event         TYPE swo_event              VALUE 'ORDER_PROMISE_EXCEPTION' ##NO_TEXT.
    CONSTANTS mc_rectype       TYPE swetypecou-rectype     VALUE 'CONFLOW' ##NO_TEXT.

*--------------------------------------------------------------------*
* Schritte (gen_stat) - siehe customizing/WF-00500-CUSTOMIZING.md
*--------------------------------------------------------------------*
    CONSTANTS:
      BEGIN OF mc_stat,
        start           TYPE /c09/cfl_gen_stat VALUE 'X0',
        classify        TYPE /c09/cfl_gen_stat VALUE 'B1',
        decision_red    TYPE /c09/cfl_gen_stat VALUE '01',
        escalation      TYPE /c09/cfl_gen_stat VALUE '02',
        decision_yellow TYPE /c09/cfl_gen_stat VALUE '03',
        act_date        TYPE /c09/cfl_gen_stat VALUE 'B2',
        act_partial     TYPE /c09/cfl_gen_stat VALUE 'B3',
        act_cancel      TYPE /c09/cfl_gen_stat VALUE 'B4',
        customer_info   TYPE /c09/cfl_gen_stat VALUE 'B5',
        end_resolved    TYPE /c09/cfl_gen_stat VALUE 'X1',
        end_error       TYPE /c09/cfl_gen_stat VALUE 'X2',
        end_no_action   TYPE /c09/cfl_gen_stat VALUE 'X3',
      END OF mc_stat .

*--------------------------------------------------------------------*
* Bearbeiter-Keys (gen_stat_user)
*--------------------------------------------------------------------*
    CONSTANTS:
      BEGIN OF mc_gsu,
        customer_service TYPE /c09/cfl_gen_stat_user VALUE '10',
        teamlead         TYPE /c09/cfl_gen_stat_user VALUE '20',
        background       TYPE /c09/cfl_gen_stat_user VALUE 'BU',
      END OF mc_gsu .

*--------------------------------------------------------------------*
* Container-Elemente (/c09/cfl_s04) - identisch mit den
* §{...}-Platzhaltern in /c09/cfl_c01t
*
* Bewusst knapp gehalten: im Container steht nur, was den Beleg
* identifiziert und was in die Entscheidung eingeht. Alles, was sich
* aus dem Beleg jederzeit nachlesen laesst (Vertriebsbereich,
* Auftragswert) oder rein ableitbar ist (Prioritaet aus dem Segment),
* gehoert nicht hierher. Ergebnistexte - Aktionsprotokoll und
* Kundenschreiben - haengen als Notiz am Workitem, nicht im Container.
*--------------------------------------------------------------------*
    CONSTANTS:
      BEGIN OF mc_prop,
        " Order - Identifikation
        order           TYPE swfdname VALUE 'ORDER',
        item            TYPE swfdname VALUE 'ITEM',
        customer        TYPE swfdname VALUE 'CUSTOMER',
        customer_name   TYPE swfdname VALUE 'CUSTOMER_NAME',
        material        TYPE swfdname VALUE 'MATERIAL',
        " Promise - der Feldvergleich, aus dem die Ausnahme entsteht
        req_qty         TYPE swfdname VALUE 'REQ_QTY',
        req_date        TYPE swfdname VALUE 'REQ_DATE',
        conf_qty        TYPE swfdname VALUE 'CONF_QTY',
        conf_date       TYPE swfdname VALUE 'CONF_DATE',
        " Exception - das Delta und sein Grund
        bo_qty          TYPE swfdname VALUE 'BO_QTY',
        bo_pct          TYPE swfdname VALUE 'BO_PCT',
        delay_days      TYPE swfdname VALUE 'DELAY_DAYS',
        reason          TYPE swfdname VALUE 'REASON',
        " Customer Rules - die beiden, die in die Bewertung eingehen
        segment         TYPE swfdname VALUE 'SEGMENT',
        partial_allowed TYPE swfdname VALUE 'PARTIAL_ALLOWED',
        " Bewertung - wird in B1 gesetzt, nicht vom Trigger
        severity        TYPE swfdname VALUE 'SEVERITY',
        recommendation  TYPE swfdname VALUE 'RECOMMENDATION',
        recomm_reason   TYPE swfdname VALUE 'RECOMMENDATION_REASON',
        cancel_allowed  TYPE swfdname VALUE 'CANCEL_ALLOWED',

*--------------------------------------------------------------------*
* Bearbeitung. Drei Werte, und sie beantworten drei verschiedene
* Fragen - das ist der Grund, warum sie nebeneinander stehen:
*
*   PROPOSED_ACTION   WAS wird getan. Kommt vom conFLOW-BUTTON, nicht
*                     aus der App: der Klick ist die Entscheidung.
*                     Vorbelegt in B1 mit der Empfehlung.
*   DECISION_REASON   WARUM. Das ist die Eingabe der App - ein
*                     kontrolliertes Vokabular, damit man es auswerten
*                     kann und nicht nur nachlesen.
*   DECISION_NOTE     Details zum Grund, Freitext.
*
* Bis zum 22.08.2026 bot die App im Dropdown dieselben AKTIONEN an wie
* die Buttons darunter. Zwei Bedienelemente fuer dieselbe Entscheidung -
* und wer oben "Escalate" waehlte und unten "Partial delivery" klickte,
* bekam seine Auswahl stillschweigend ueberschrieben. Das Dropdown
* fragt jetzt etwas, das die Buttons nicht beantworten koennen.
*--------------------------------------------------------------------*
        proposed_action TYPE swfdname VALUE 'PROPOSED_ACTION',
        decision_reason TYPE swfdname VALUE 'DECISION_REASON',
        decision_note   TYPE swfdname VALUE 'DECISION_NOTE',
      END OF mc_prop .

*--------------------------------------------------------------------*
* Wertebereiche
*--------------------------------------------------------------------*
    CONSTANTS:
      BEGIN OF mc_severity,
        green  TYPE string VALUE 'GREEN',
        yellow TYPE string VALUE 'YELLOW',
        red    TYPE string VALUE 'RED',
      END OF mc_severity .

    CONSTANTS:
      BEGIN OF mc_recommendation,
        accept_new_date  TYPE string VALUE 'ACCEPT_NEW_DATE',
        partial_delivery TYPE string VALUE 'PARTIAL_DELIVERY',
        cancel_remaining TYPE string VALUE 'CANCEL_REMAINING',
        escalate         TYPE string VALUE 'ESCALATE',
      END OF mc_recommendation .

*--------------------------------------------------------------------*
* Die Entscheidungsgruende - das Vokabular des Dropdowns.
*
* Bewusst kontrolliert und nicht als Freitext: ein Grund, den man
* auszaehlen kann, beantwortet Fragen, die eine Notiz nicht beantwortet
* - "in wie vielen Faellen war die Bestandssituation der Ausloeser?".
* Die Notiz bleibt daneben fuer die Einzelheiten.
*
* OTHER ist die Ausnahme mit Folge: dort ist die Notiz Pflicht, sonst
* waere der Grund keiner. Geprueft wird das in
* ZCL_CFL_00500_RULES=>DECISION_HINT.
*--------------------------------------------------------------------*
    CONSTANTS:
      BEGIN OF mc_reason,
        customer_agreed      TYPE string VALUE 'CUSTOMER_AGREED',
        customer_request     TYPE string VALUE 'CUSTOMER_REQUEST',
        stock_situation      TYPE string VALUE 'STOCK_SITUATION',
        production_confirmed TYPE string VALUE 'PRODUCTION_CONFIRMED',
        internal_policy      TYPE string VALUE 'INTERNAL_POLICY',
        other                TYPE string VALUE 'OTHER',
      END OF mc_reason .

    CONSTANTS mc_segment_key_account TYPE string VALUE 'KEY_ACCOUNT' ##NO_TEXT.

*--------------------------------------------------------------------*
* Governance-Parameter
*
* Die 5 % sind ein *Vorschlag* aus dem Backorder-Management-Protokoll,
* kein Beschluss. Hier bewusst als eine Konstante, damit die Regel im
* Termin an genau einer Stelle gezeigt und geaendert werden kann.
*--------------------------------------------------------------------*
    CONSTANTS mc_cancel_tolerance_pct TYPE p LENGTH 5 DECIMALS 2 VALUE '5.00' ##NO_TEXT.

*--------------------------------------------------------------------*
* Schwellwerte der Severity-Bewertung (Abschnitt 8 des Konzepts)
*--------------------------------------------------------------------*
    CONSTANTS mc_red_pct_threshold   TYPE p LENGTH 5 DECIMALS 2 VALUE '10.00' ##NO_TEXT.
    CONSTANTS mc_red_delay_threshold TYPE i VALUE 3 ##NO_TEXT.

*--------------------------------------------------------------------*
* Fiori My Inbox - Darstellung
*
* Drei Stellschrauben, bewusst an einer Stelle: keine davon war vor dem
* ersten Blick in die Inbox bewiesen. Sieht im Termin etwas anders aus
* als gedacht, wird hier umgeschaltet und aktiviert - nicht gesucht.
*--------------------------------------------------------------------*

*--------------------------------------------------------------------*
* 1. Ampelsymbole, als UTF-8-Bytefolge statt als Literal
*
* Emoji direkt in den Quelltext zu schreiben ist die Variante, die den
* Transport, abapGit und den ADT-Roundtrip nicht zuverlaessig ueberlebt.
* Ein Hex-String schon - er ist reines ASCII. Die Ampel-Methode
* der WF-Klasse macht daraus das Zeichen.
*
* Vorgabe: die drei farbigen Kreise (U+1F534 / U+1F7E1 / U+1F7E2).
* Fallback, falls in der Inbox leere Kaestchen stehen - dann rendert die
* Schrift die Ebene ausserhalb der BMP nicht:
*   red 'E29B94' (U+26D4), yellow 'E29AA0' (U+26A0), green 'E29C85'
*   (U+2705) - das sind je 3 Bytes, also LENGTH 3 statt LENGTH 4.
*--------------------------------------------------------------------*
    CONSTANTS:
      BEGIN OF mc_ampel_utf8,
        red    TYPE x LENGTH 4 VALUE 'F09F94B4',
        yellow TYPE x LENGTH 4 VALUE 'F09F9FA1',
        green  TYPE x LENGTH 4 VALUE 'F09F9FA2',
      END OF mc_ampel_utf8 .

*--------------------------------------------------------------------*
* 1b. Geschuetztes Leerzeichen (U+00A0)
*
* Normale Leerzeichen am Zeilenanfang schluckt die Fiori-Inbox - im Lauf
* vom 20.08.2026 nachgesehen: die Einrueckung des Kontextblocks kam
* nicht an, alles stand buendig links. Der Text landet also in einem
* HTML-Kontext, der Leerraum zusammenfaltet. NBSP ueberlebt das.
*--------------------------------------------------------------------*
    CONSTANTS mc_nbsp_utf8 TYPE x LENGTH 2 VALUE 'C2A0'.

*--------------------------------------------------------------------*
* 2. Workitem-Prioritaet je Severity
*
* SAP kennt 1 (hoechste) bis 9 (niedrigste), Vorgabe ist 5. My Inbox
* faltet das auf wenige Stufen zusammen und laesst danach filtern und
* sortieren - der eigentliche Gewinn gegenueber der Farbe.
*
* !! NICHT die 1 nehmen !!
*
* Stufe 1 heisst im Workflow Builder woertlich "Highest - Express", und
* das ist keine Beschriftung, sondern Verhalten: CL_SWF_RUN_WIM_DIALOG=>
* SEND_EXPRESS_POPUP prueft `wi_prio EQ swfco_wi_express_priority` und
* ruft dann SO_EXPRESS_FLAG_SET - eine Express-Nachricht an *alle*
* Bearbeiter. Im Kundentermin ist das ein Popup, das niemand bestellt
* hat.
*
* Gewaehlt ist die 4 = "High". Die Stufen darueber (2 "Very High",
* 3 "Higher") waeren auch harmlos - die 4 ist die, die im Workflow
* Builder "High" heisst und damit sagt, was gemeint ist.
*--------------------------------------------------------------------*
    CONSTANTS:
      BEGIN OF mc_prio,
        red    TYPE sww_prio VALUE '4',
        yellow TYPE sww_prio VALUE '5',
      END OF mc_prio .

*--------------------------------------------------------------------*
* 3. Zeichenformat der Abschnittsueberschrift
*
* Der Workitem-Text ist ein SAPscript-**ITF**-Text, kein HTML - auch
* wenn der BAdI-Parameter /C09/CFL_HTML_TABLE_TT heisst und die Muster-
* implementierung dort '<br>' hineinhaengt.
*
* Damit gilt die ITF-Schreibweise:  <FORMAT>Text</>
* Geschlossen wird mit </> - NICHT mit </FORMAT>.
*
* Genau daran ist der erste Versuch gescheitert: '<b>ORDER</b>' liest
* ITF als Zeichenformat namens 'b', kennt es nicht und entfernt die
* Klammern stillschweigend. Kein Fett, kein sichtbares Tag, keine
* Fehlermeldung - der irrefuehrendste denkbare Ausgang.
*
* 'H' ist das uebliche Hervorhebungsformat. Kommt es nicht an, hat der
* Stil des Textes es nicht - dann hier ein anderes eintragen. Leer
* lassen schaltet die Hervorhebung ganz ab.
*--------------------------------------------------------------------*
    CONSTANTS mc_itf_charfmt TYPE string VALUE 'H' ##NO_TEXT.

*--------------------------------------------------------------------*
* NOTAUS fuer die eigene Oberflaeche.
*
* abap_false schaltet set_inbox_ui( ) ab: die Container-Elemente werden
* nicht mehr gesetzt, der Intent loest ins Leere, und die Inbox zeigt
* wieder den ITF-Kontextblock. Wirkt fuer NEU angelegte Workitems -
* bestehende behalten ihre Elemente.
*
* Muss es waehrend einer laufenden Demo sofort und fuer alle Workitems
* weg, ist der schnellere Weg das Target Mapping: im Launchpad Designer
* Katalog Z_CFL_00500 -> Target Mapping loeschen. Danach faellt jedes
* Workitem auf den Textblock zurueck, ohne dass Code angefasst wird.
*--------------------------------------------------------------------*
    CONSTANTS mc_inbox_ui TYPE abap_bool VALUE abap_true .

*--------------------------------------------------------------------*
* Fiori My Inbox - eigene Oberflaeche am Workitem
*
* conFLOW loest die Visualisierung des Tasks TS00388601 ueber
* SWFVMD1 auf, und zwar dynamisch: Semantic Object, Action und sechs
* freie Query-Parameter stehen dort als Container-Elemente. Gefuellt
* werden sie beim Anlegen des Workitems - deshalb steuert jeder
* Workflow seine eigene App, obwohl alle denselben Task benutzen.
*
* SWFVMD1, nicht SWFVISU: CL_SWF_UTL_URL_GENERATE liest zuerst den
* Satz aus SWFVMD1 und faellt nur ersatzweise auf SWFVISU zurueck.
* Wer in SWFVISU pflegt, waehrend SWFVMD1 einen Eintrag hat, pflegt
* ins Leere.
*--------------------------------------------------------------------*
    CONSTANTS:
      BEGIN OF mc_visu,
        semantic_object TYPE swfdname VALUE '/C09/CFL_VISU_SEMANTIC_OBJECT',
        action          TYPE swfdname VALUE '/C09/CFL_VISU_ACTION',
        query_obj00     TYPE swfdname VALUE '/C09/CFL_VISU_QUERY_OBJ00',
      END OF mc_visu .

*--------------------------------------------------------------------*
* Ziel im Launchpad. Muss als Target Mapping existieren, sonst laeuft
* die Aufloesung ins Leere und die Inbox zeigt wieder den Textblock.
*--------------------------------------------------------------------*
    CONSTANTS mc_ui_semantic_object TYPE string VALUE 'ZCFLOrderPromise' ##NO_TEXT.

*--------------------------------------------------------------------*
* Eigene Action statt einer SAP-Standard-Action. Der Launchpad
* Designer schlaegt zwar nur die Standards vor ('display', 'manage',
* ...), das Feld nimmt aber freien Text - im Lauf am 21.08.2026
* bestaetigt. Sprechender Name gewinnt.
*--------------------------------------------------------------------*
    CONSTANTS mc_ui_action          TYPE string VALUE 'openInInbox' ##NO_TEXT.

*--------------------------------------------------------------------*
* 4. Buttons in der Inbox einfaerben?
*
* /IWWRK/S_TGW_DECISION_OPTION-NATURE ist die einzige Farbe, die die
* Standard-Inbox hergibt, und sie kennt genau zwei Werte: POSITIVE fuer
* die berechnete Empfehlung, NEGATIVE fuer 'Restmenge stornieren'. Der
* Rest bleibt neutral. Damit ist die Empfehlung am Button sichtbar -
* die halbe Antwort auf '[Accept Recommendation]' aus Abschnitt 12,
* ohne eigene Oberflaeche.
*
* Der Haken: manche My-Inbox-Staende ziehen eine markierte Option nach
* vorne und schieben den Rest in ein Ueberlaufmenue. Das ist genau die
* Stelle, an der die Buttonleiste anders aussehen koennte als vorher -
* wenn ja, hier auf abap_false und alles ist wie gehabt.
*--------------------------------------------------------------------*
    CONSTANTS mc_fiori_nature TYPE abap_bool VALUE abap_true .

*--------------------------------------------------------------------*
* 5. Buttons im SAP Business Workplace einfaerben?
*
* Das SAP GUI liest ALTNATURE nicht - Punkt 4 wirkt nur in Fiori. Der
* Entscheidungs-Screen rendert stattdessen SWR_DECIALTS-ALTTEXT als
* **HTML**. Farbe entsteht dort also im Text selbst, ueber ein <span>.
*
* In einem anderen Kundenprojekt seit 08/2026 im Betrieb: dieselbe
* Technik faerbt dort "Genehmigen" gruen und "Ablehnen" rot. Zwei
* Oberflaechen, zwei Mechanismen, eine Aussage.
*
* Warum es einen eigenen Schalter braucht: das Markup darf NUR ins
* SAP GUI. Die neue Fiori-Inbox nimmt die Buttontexte aus demselben
* Hook (BEFORE_DECISION) - dort stuende sonst das <span> als
* Buttonbeschriftung. Der Guard ist GUI_IS_AVAILABLE, siehe
* ZCL_CFL_WORKFLOW_00500=>IS_SAPGUI( ).
*
* ALTTEXT ist CHAR255 und der Wrapper kostet rund 50 Zeichen - die
* c09t-Texte passen also mit Abstand. Wird es je knapp, ist die kurze
* Form <font color=green>.
*--------------------------------------------------------------------*
    CONSTANTS mc_gui_html_color TYPE abap_bool VALUE abap_true .

*--------------------------------------------------------------------*
* Dieselbe Aussage wie NATURE, nur in der Sprache des GUI-Screens:
* gruen die berechnete Empfehlung, rot die Aktion, die Kundenbedarf
* endgueltig wegwirft. Kein drittes Signal - eine Palette waere keine
* Aussage mehr.
*--------------------------------------------------------------------*
    CONSTANTS:
      BEGIN OF mc_gui_color,
        positive TYPE string VALUE 'green',
        negative TYPE string VALUE 'red',
      END OF mc_gui_color .

*--------------------------------------------------------------------*
* Schriftgroesse der eingefaerbten Knoepfe im SAP GUI.
*
* Farbe allein traegt im GUI zu wenig - die Leiste ist grau und der
* Text klein. Groesse verstaerkt dieselbe Aussage, statt eine zweite
* aufzumachen: hervorgehoben ist genau das, was auch farbig ist.
*
* 120 % laeuft in einem Kundenprojekt im Betrieb, und genau diese Form
* wird hier nachgebaut - Farbe und Groesse, kein font-weight. Wird die
* Buttonleiste zu breit und rutscht etwas ins Ueberlaufmenue, hier
* zurueckdrehen; leer heisst "keine Groessenangabe", dann bleibt nur
* die Farbe.
*--------------------------------------------------------------------*
    CONSTANTS mc_gui_font_size TYPE string VALUE '120%' ##NO_TEXT.

  PROTECTED SECTION.
  PRIVATE SECTION.
ENDCLASS.

CLASS zcl_cfl_const_00500 IMPLEMENTATION.
ENDCLASS.
```
