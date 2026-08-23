# Die ganze Klasse

Zum Kopieren. Alles, was in den Kapiteln davor erklärt wurde,
steht hier am Stück - mit denselben Kommentaren, weil beides aus
derselben Datei kommt.

> Vor dem Aktivieren `00900` durch die eigene Workflow-Nummer
> ersetzen, und `TEXT-001` im Textpool pflegen.

```abap
CLASS zcl_cfl_workflow_00900 DEFINITION
  PUBLIC
  FINAL
  CREATE PUBLIC.

*----------------------------------------------------------------------*
* REFERENZ-IMPLEMENTIERUNG /C09/CFL_IF_BADI_0101
*
* Alle 26 Hooks des conFLOW-BAdI, jeder mit demselben Kopfblock:
* wann er laeuft, was er bekommt, was er aendern darf, und ob man ihn
* ueberhaupt braucht.
*
* Sechs Hooks sind hier ABSICHTLICH LEER. Das ist kein Rest, den noch
* jemand ausfuellen muss - es ist die Aussage, dass ein normaler
* Workflow sie nicht braucht. Der Kopfblock sagt jeweils, wofuer sie
* da waeren und woran man merkt, dass der eigene Fall dazugehoert.
*
* Der Beispielprozess ist eine Bestellfreigabe:
*
*   X0  Start
*   B1  Hintergrund: Beleg lesen, Betrag gegen das Limit pruefen
*   01  Dialog:      Einkauf entscheidet
*   02  Dialog:      Vorgesetzter entscheidet (Eskalation)
*   B2  Hintergrund: Rueckschreibung in den Beleg
*   B3  Hintergrund: Benachrichtigung
*   X1  Ende freigegeben / X2 abgelehnt / X3 nichts zu tun
*
* Bewusst auf einem SAP-Standardbeleg (Einkaufsbeleg, BUS2012), damit
* jeder Leser den Datenteil kennt und sich auf den Workflowteil
* konzentrieren kann.
*
* NACHBAUEN
*   1. Klasse kopieren, 00900 durch die eigene WF-Nummer ersetzen
*      (drei Stellen: Klassenname, Konstantenklasse, BAdI-Filter)
*   2. ZCL_CFL_CONST_00900 mitkopieren und an das eigene Customizing
*      anpassen - sie ist die einzige Stelle mit Schluesseln
*   3. Hooks, die man nicht braucht, LEER LASSEN. Nicht loeschen:
*      das Interface verlangt sie, und ein leerer Rumpf mit Kopfblock
*      ist die Dokumentation, dass die Entscheidung getroffen wurde
*
* Der Filter der BAdI-Implementierung ist WF_DEFINITION = '00900'.
* Ohne diesen Filter laeuft die Klasse fuer JEDEN Workflow im System.
*----------------------------------------------------------------------*

  PUBLIC SECTION.

    INTERFACES /c09/cfl_if_badi_0101.
    INTERFACES if_badi_interface.

*--------------------------------------------------------------------*
* HINTERGRUNDSCHRITT
*
* Kein BAdI-Hook, sondern der zweite Vertrag, den conFLOW kennt: eine
* Methode, die in /C09/CFL_C01 als Hintergrund-Task eingetragen wird
* (ATTRIBUT = 'BACK_BATCH', Klasse und Methodenname im Customizing).
*
* Die Signatur ist FEST vorgegeben und muss exakt so aussehen:
*
*     IMPORTING is_cfl_s03      TYPE /c09/cfl_s03
*     EXPORTING et_bapiret2     TYPE bapiret2_t
*               ev_decision_key TYPE swr_decikey
*
* Weicht sie ab, findet conFLOW die Methode zur Laufzeit nicht - und
* zwar ohne Syntaxfehler, weil der Aufruf dynamisch ist. Der Workflow
* bleibt dann im Hintergrundschritt stehen.
*
* EV_DECISION_KEY ist der Ausgang. Er wird wie bei einem
* Dialogschritt ausgewertet, nur dass ihn hier der Code setzt statt
* eines Menschen. Bleibt er leer, laeuft der Workflow nicht weiter.
*
* Warum das Beispiel diese Methode mitbringt, obwohl sie nicht zum
* BAdI gehoert: die BAdI-Hooks ENTSCHEIDEN nichts und TUN nichts, sie
* stellen dar und lenken. Die Arbeit passiert hier. Ein Beispiel ohne
* Hintergrundschritt zeigt einen Workflow ohne Inhalt.
*--------------------------------------------------------------------*
    CLASS-METHODS background_classify
      IMPORTING is_cfl_s03      TYPE /c09/cfl_s03
      EXPORTING et_bapiret2     TYPE bapiret2_t
                ev_decision_key TYPE swr_decikey.

  PRIVATE SECTION.

*--------------------------------------------------------------------*
* Merker fuer GET_ACTORS.
*
* GET_NUMBER_ACTORS_REL laeuft VOR GET_ACTORS und bekommt als
* einziger Hook mit, in welchem Zusammenhang die Bearbeiterfindung
* gerade stattfindet (IV_PROCESS = 'MAIL' beim Mailversand). Wer in
* GET_ACTORS zwischen "Workitem anlegen" und "Mail verschicken"
* unterscheiden will, muss sich den Wert hier merken - die Signatur
* von GET_ACTORS gibt ihn nicht her.
*--------------------------------------------------------------------*
    DATA mv_process TYPE char10.

*--------------------------------------------------------------------*
* Alle Helfer sind KLASSENMETHODEN.
*
* Nicht aus Stilgruenden: der Hintergrundschritt BACKGROUND_CLASSIFY
* ist selbst eine Klassenmethode (so verlangt es conFLOW), und eine
* Klassenmethode kann keine Instanzmethode rufen. Waeren die Helfer
* Instanzmethoden, muesste der Hintergrundschritt sich eine Instanz
* der BAdI-Klasse bauen - was funktioniert, aber niemand erwartet.
*--------------------------------------------------------------------*

*--------------------------------------------------------------------*
* Container-Zugriff
*
* /C09/CFL_CL_WORKFLOW_0101=>GET_ATTRIBUT_VALUE liefert eine TABELLE -
* ein Container-Element kann mehrere Werte haben. In den allermeisten
* Faellen will man den ersten und einzigen. GET_VAL kapselt das, damit
* der Aufrufer nicht jedesmal eine Schleife schreibt.
*
* Diese drei Helfer sind der Grund, warum die Hooks unten so kurz
* sind. Wer sie weglaesst, schreibt dieselben vier Zeilen zwanzigmal.
*--------------------------------------------------------------------*
    CLASS-METHODS get_val
      IMPORTING iv_element    TYPE /c09/cfl_s04-element
                iv_id         TYPE /c09/cfl_s01-id
      RETURNING VALUE(rv_val) TYPE string.

    CLASS-METHODS set_val
      IMPORTING iv_element TYPE /c09/cfl_s04-element
                iv_id      TYPE /c09/cfl_s01-id
                iv_value   TYPE string.

    CLASS-METHODS is_true
      IMPORTING iv_value      TYPE string
      RETURNING VALUE(rv_yes) TYPE abap_bool.

*--------------------------------------------------------------------*
* Belegzugriff und Bewertung
*
* Getrennt gehalten, weil sie die einzigen Methoden sind, die den
* Beleg kennen. Wer die Klasse auf einen anderen Belegtyp umbaut,
* aendert genau hier - und nichts an den Hooks.
*--------------------------------------------------------------------*
    CLASS-METHODS read_document
      IMPORTING iv_instid        TYPE /c09/cfl_s01-instid
      EXPORTING ev_net_value     TYPE ekpo-netwr
                ev_currency      TYPE ekko-waers
                ev_vendor        TYPE ekko-lifnr
                ev_created_by    TYPE ekko-ernam
                ev_found         TYPE abap_bool.

    CLASS-METHODS classify
      IMPORTING iv_net_value       TYPE ekpo-netwr
      RETURNING VALUE(rv_severity) TYPE string.

*--------------------------------------------------------------------*
* Anzeige-Formatierung
*
* Belegnummern stehen in der Datenbank mit fuehrenden Nullen
* (0004500001234). Am Workitem sieht das nach technischem Ausrutscher
* aus. Mengen und Betraege wiederum haben in der Datenbank das interne
* Format - ein Bearbeiter erwartet sie in SEINER Darstellung.
*
* WICHTIG: eine Leseroutine, die fuer die ANZEIGE formatiert, taugt
* NICHT als Schluesselquelle fuer einen SELECT. Wer FMT_DOC( ) auf
* eine Belegnummer anwendet und damit sucht, findet nichts - und
* bekommt keinen Fehler, sondern eine leere Tabelle.
*--------------------------------------------------------------------*
    CLASS-METHODS fmt_doc
      IMPORTING iv_value      TYPE string
      RETURNING VALUE(rv_out) TYPE string.

    CLASS-METHODS fmt_amount
      IMPORTING iv_element    TYPE /c09/cfl_s04-element
                iv_id         TYPE /c09/cfl_s01-id
      RETURNING VALUE(rv_out) TYPE string.

*--------------------------------------------------------------------*
* Protokoll
*
* Ein Hintergrundschritt, der etwas tut, muss sagen koennen was.
* conFLOW nimmt dafuer die BAPIRET2-Tabelle entgegen, die jede
* Hintergrund-Methode zurueckgibt, und schreibt sie ins
* Anwendungsprotokoll (SLG1) - vorausgesetzt, in /C09/CFL_C08 ist ein
* Objekt/Subobjekt hinterlegt und dieses ist in SLG0 angelegt.
*
* FALLE: SLG1 zeigt eine Zeile nur an, wenn ID und NUMBER gefuellt
* sind. Ein reiner Freitext in MESSAGE verschwindet spurlos - kein
* Fehler, keine Zeile. Deshalb geht der Text hier ueber die
* Sammelnachricht 00/398 ('&1&2&3&4'), vier Variablen a 50 Zeichen.
* Eine eigene Nachrichtenklasse ist sauberer und der naechste
* Ausbauschritt.
*--------------------------------------------------------------------*
    CLASS-METHODS add_msg
      IMPORTING iv_type      TYPE bapiret2-type DEFAULT 'S'
                iv_text      TYPE string
      CHANGING  ct_bapiret2  TYPE bapiret2_t.

*--------------------------------------------------------------------*
* Workitem-Prioritaet setzen.
*
* Steht hier als eigene Methode, weil der naheliegende Weg nicht
* funktioniert - siehe den Kommentar in der Implementierung. Der
* Aufwand lohnt sich: die Prioritaet ist der einzige Weg, nach dem
* Befund zu FILTERN, statt ihn nur zu sehen.
*--------------------------------------------------------------------*
    CLASS-METHODS set_priority
      IMPORTING iv_wi_id TYPE sww_wiid
                iv_prio  TYPE sww_prio.

*--------------------------------------------------------------------*
* Der Entscheidungsschluessel, den das System empfiehlt.
*
* Wird an ZWEI Stellen gebraucht (GET_BEFORE_DECISION_WORKITEM fuer
* das SAP-GUI, GET_FIORI_TASK_DEC_OP_ACT fuer die Fiori-Inbox), und
* die beiden Tabellen sind nicht dieselbe. Eine gemeinsame Methode
* verhindert, dass die zwei Oberflaechen verschiedene Buttons
* hervorheben.
*--------------------------------------------------------------------*
    CLASS-METHODS recommended_key
      IMPORTING iv_id         TYPE /c09/cfl_s01-id
      RETURNING VALUE(rv_key) TYPE swr_decikey.

*--------------------------------------------------------------------*
* Den Text des laufenden HINTERGRUND-Workitems nachziehen.
*
* Ohne das steht im Workflow-Protokoll bei jedem Hintergrundschritt
* der Customizing-Text - also bei allen fuenf derselbe. Mit dem
* Nachzug steht dort, was der Schritt herausgefunden hat. Das ist
* der Unterschied zwischen einem Protokoll und einer Liste.
*--------------------------------------------------------------------*
    CLASS-METHODS update_witext
      IMPORTING iv_text TYPE string.

ENDCLASS.


CLASS zcl_cfl_workflow_00900 IMPLEMENTATION.

*======================================================================*
*
*   GRUPPE 1 - START
*   Wer bekommt das Workitem, und entsteht ueberhaupt eines?
*
*======================================================================*

  METHOD /c09/cfl_if_badi_0101~get_wi_create_swe2.
*--------------------------------------------------------------------*
* WANN   Beim ereignisgesteuerten Start ueber SWE2, bevor conFLOW die
*        Instanz anlegt. Nur auf diesem Weg - wer den Workflow ueber
*        START_WORKFLOW_INT( ) aus eigenem Code startet, laeuft hier
*        nicht durch.
*
* REIN   IT_EVENT_CONTAINER_TAB  der Ereignis-Container
*        CS_CFL_C10              die Typkoppelung, die getroffen hat
*
* RAUS   CS_SENDER  Objekttyp und Instanz-Schluessel der neuen Instanz
*
* WOFUER   Das ist die VETO-Stelle. CS_SENDER leeren heisst: kein
*        Workflow. Damit filtert man Ereignisse, die zwar geworfen
*        werden, aber fachlich keinen Prozess ausloesen sollen -
*        Belegart, Werk, Betrag unter Bagatellgrenze.
*
* WARUM HIER UND NICHT IN B1
*        Ein Workflow, der startet und sich im ersten
*        Hintergrundschritt selbst beendet, hinterlaesst eine Instanz,
*        einen Protokolleintrag und eine Zeile in jeder Auswertung.
*        Bei zehn Belegen ist das egal, bei zehntausend nicht mehr.
*        Was hier abgewiesen wird, hat nie existiert.
*
*        Der Preis: es gibt auch keine Spur davon, dass geprueft
*        wurde. Wer nachweisen muss, WARUM ein Beleg keinen Workflow
*        bekam, filtert besser in B1 und beendet dort mit einem
*        Protokolleintrag.
*--------------------------------------------------------------------*

    IF cs_sender-typeid <> zcl_cfl_const_00900=>mc_objecttype.
      RETURN.
    ENDIF.

    read_document( EXPORTING iv_instid    = CONV #( cs_sender-instid )
                   IMPORTING ev_net_value = DATA(lv_net_value)
                             ev_found     = DATA(lv_found) ).

*--------------------------------------------------------------------*
* Beleg nicht lesbar oder Bagatellbetrag: kein Workflow.
*
* Der erste Fall ist der wichtigere. Ein Ereignis kann zu einem Beleg
* kommen, den es (noch) nicht gibt - etwa weil das Ereignis vor dem
* COMMIT geworfen wurde. Ohne diese Pruefung entstehen Instanzen zu
* Belegnummern, die niemand findet.
*--------------------------------------------------------------------*
    IF lv_found = abap_false.
      CLEAR cs_sender.
      RETURN.
    ENDIF.

    IF lv_net_value IS INITIAL.
      CLEAR cs_sender.
    ENDIF.

  ENDMETHOD.


  METHOD /c09/cfl_if_badi_0101~get_actors.
*--------------------------------------------------------------------*
* WANN   Bei jeder Workitem-Erzeugung, und ein zweites Mal vor jedem
*        Mailversand. Der wichtigste Hook des ganzen Interface.
*
* REIN   IS_CFL_C05  der Bearbeiterkreis - GEN_STAT_USER ist der
*                    Schluessel, auf den verzweigt wird
*        IS_CFL_S01  die Instanz (optional, aber praktisch immer da)
*
* RAUS   CT_ACTORS   die Bearbeiter
*
* VORHER PRUEFEN, OB MAN IHN BRAUCHT
*        /C09/CFL_C03 traegt je GEN_STAT_USER direkt OTYPE und OBJID -
*        etwa US/MEIER, US/WF-BATCH, US/WF_INITIATOR. Erst
*        USER_BADI = 'X' schaltet auf diesen Hook um. Fuer PoCs,
*        Demos und feste Zuordnungen bleibt GET_ACTORS damit LEER,
*        und es braucht weder Rolle noch Code.
*
* DAS FORMAT IST DER HAEUFIGSTE FEHLER
*        Ein Eintrag in CT_ACTORS ist immer ein TYPISIERTES
*        Org-Objekt, nie ein blanker Benutzername:
*
*          US<uname>      Benutzer
*          S<planstelle>  Planstelle
*          O<orgeinheit>  Organisationseinheit
*          AC<rolle>      Rolle
*
*        'MEIER' erzeugt kein Workitem und keine Fehlermeldung. Das
*        Workitem landet bei niemandem und faellt erst auf, wenn
*        jemand fragt, wo es geblieben ist.
*
* WENN NIEMAND GEFUNDEN WIRD
*        Ein Workitem ohne Bearbeiter geht in den Fehlerstatus und
*        bleibt liegen. Besser ist ein definierter Auffangbearbeiter:
*        conFLOW kennt dafuer den Eintrag 'C09_NO_USER'. Der Prozess
*        laeuft weiter und die Luecke ist sichtbar, statt still zu
*        stehen. Siehe ganz unten in dieser Methode.
*--------------------------------------------------------------------*

    CASE is_cfl_c05-gen_stat_user.

*--------------------------------------------------------------------*
* VARIANTE A - Rolle
*
* Der haeufigste Fall. Die Rolle pflegt der Kunde selbst, der Code
* bleibt unveraendert, wenn Personen wechseln.
*
* Der Aufruf gehoert NICHT hierher, sondern in eine zentrale Klasse
* ZCL_CFL_GET_ACTORS. Grund: dieselbe Rolle wird von mehreren
* Workflows gebraucht, und TWP_GET_ROLE_USER_ASSIGNMENT mit
* Sortieren und Entdoppeln ist jedesmal derselbe Block. Die Klasse
* steht als Muster im Kapitel zu diesem Hook.
*--------------------------------------------------------------------*
      WHEN zcl_cfl_const_00900=>mc_gsu-buyer.
        ct_actors = zcl_cfl_get_actors=>buyer( ).

*--------------------------------------------------------------------*
* VARIANTE B - Bearbeiter aus dem Container
*
* Wenn ein frueherer Schritt festgelegt hat, wer den naechsten
* bekommt: der Ersteller des Belegs, der Vertreter, der in Schritt 01
* Ausgewaehlte.
*
* Achtung auf das Praefix - GET_VAL liefert den blanken Benutzernamen,
* 'US' muss davor.
*--------------------------------------------------------------------*
      WHEN zcl_cfl_const_00900=>mc_gsu-creator.
        DATA(lv_uname) = get_val( iv_element = zcl_cfl_const_00900=>mc_prop-decision_by
                                  iv_id      = is_cfl_s01-id ).
        IF lv_uname IS NOT INITIAL.
          APPEND |US{ lv_uname }| TO ct_actors.
        ENDIF.

*--------------------------------------------------------------------*
* VARIANTE C - Organisationsstruktur
*
* Der Vorgesetzte, der Kostenstellenverantwortliche, die naechste
* Genehmigungsstufe. RH_GET_ACTORS wertet eine SWO1-Rolle (AC...)
* gegen die Aufbauorganisation aus.
*
* DIE FALLE: das Ergebnis sind meistens PLANSTELLEN oder PERSONEN
* (OTYPE 'S' bzw. 'P'), keine Benutzer. Ein Workitem an eine Person
* ohne Benutzerzuordnung erreicht niemanden. Deshalb der Umweg ueber
* PTRV_CONVERT_PERNR_TO_USERID. Wer den weglaesst, hat einen
* Workflow, der im Testsystem laeuft (dort ist alles zugeordnet) und
* im Produktivsystem stehenbleibt.
*
* Die Rollennummer ist mandantenabhaengiges Customizing und gehoert
* deshalb NICHT als Literal in den Code - hier steht sie nur, damit
* das Beispiel vollstaendig ist.
*--------------------------------------------------------------------*
      WHEN zcl_cfl_const_00900=>mc_gsu-supervisor.

        DATA lt_container TYPE TABLE OF swcont.
        DATA lt_swhactor  TYPE TABLE OF swhactor.

        APPEND INITIAL LINE TO lt_container ASSIGNING FIELD-SYMBOL(<fs_cont>).
        <fs_cont>-element = 'OBJECT'.
        <fs_cont>-value   = 'US'.

        CALL FUNCTION 'RH_GET_ACTORS'
          EXPORTING  act_object      = CONV rhobjects-object( 'AC00000168' )
                     search_date     = sy-datum
          TABLES     actor_container = lt_container
                     actor_tab       = lt_swhactor
          EXCEPTIONS OTHERS          = 1.
        IF sy-subrc <> 0.
          CLEAR lt_swhactor.
        ENDIF.

*--------------------------------------------------------------------*
* LV_USER_ID muss VORHER deklariert werden. Eine Inline-Deklaration
* DATA(...) ist an einem klassischen CALL FUNCTION ... IMPORTING
* nicht erlaubt - der Compiler weist es ab. Betrifft alle Aufrufe
* dieser Bauart, und es sind in einem conFLOW-BAdI viele.
*--------------------------------------------------------------------*
        DATA lv_user_id TYPE usr02-bname.

        LOOP AT lt_swhactor ASSIGNING FIELD-SYMBOL(<fs_actor>).

          IF <fs_actor>-otype = 'P'.
            CLEAR lv_user_id.
            CALL FUNCTION 'PTRV_CONVERT_PERNR_TO_USERID'
              EXPORTING  personnel_number  = CONV p0001-pernr( <fs_actor>-objid )
              IMPORTING  user_id           = lv_user_id
              EXCEPTIONS user_id_not_found = 1
                         OTHERS            = 2.
            IF sy-subrc = 0.
              APPEND |US{ lv_user_id }| TO ct_actors.
            ENDIF.
          ELSE.
            APPEND |{ <fs_actor>-otype }{ <fs_actor>-objid }| TO ct_actors.
          ENDIF.

        ENDLOOP.

      WHEN OTHERS.
    ENDCASE.

*--------------------------------------------------------------------*
* NACHLAUF 1 - Mailversand anders behandeln als Workitem-Erzeugung
*
* Derselbe Hook laeuft fuer beides. Beim Mailversand will man oft
* einen anderen Kreis: nicht jeder, der entscheiden DARF, will auch
* eine Mail. MV_PROCESS kommt aus GET_NUMBER_ACTORS_REL, der vorher
* laeuft.
*
* Der Merker wird danach geleert - sonst wirkt er beim naechsten
* Aufruf nach, und der ist keine Mail mehr.
*--------------------------------------------------------------------*
    IF mv_process = 'MAIL'.
      CLEAR mv_process.
      RETURN.
    ENDIF.

*--------------------------------------------------------------------*
* NACHLAUF 2 - Auffangbearbeiter
*
* Nur fuer Workitems, nicht fuer Mails: eine Mail an niemanden ist
* harmlos, ein Workitem an niemanden bleibt liegen.
*--------------------------------------------------------------------*
    IF ct_actors IS INITIAL.
      APPEND 'C09_NO_USER' TO ct_actors.
    ENDIF.

  ENDMETHOD.


  METHOD /c09/cfl_if_badi_0101~get_number_actors_rel.
*--------------------------------------------------------------------*
* WANN   Vor GET_ACTORS, einmal je Schritt.
*
* REIN   IS_CFL_S01  die Instanz
*        IV_PROCESS  der Zusammenhang - 'MAIL' beim Mailversand
*
* RAUS   CT_ACTORS   die BEARBEITERKREISE (/C09/CFL_C05_TT), nicht
*                    die Bearbeiter. Der Unterschied zu GET_ACTORS
*                    liegt genau hier.
*
* WOFUER   Zwei Dinge, die sonst nirgends gehen:
*
*        1. PARALLELE WORKITEMS. Wer aus einem Eintrag mehrere macht,
*           bekommt mehrere Workitems auf demselben Schritt. So
*           entstehen Genehmigungsstufen, deren ANZAHL erst zur
*           Laufzeit feststeht - vier Unterschriften bei diesem
*           Beleg, zwei beim naechsten.
*
*        2. DEN MERKER FUER GET_ACTORS SETZEN. IV_PROCESS gibt es nur
*           hier. Wer in GET_ACTORS zwischen Workitem und Mail
*           unterscheiden will, braucht diese Zeile.
*
* BRAUCHT MAN IHN?
*        Fuer den Normalfall nein. Ein Schritt, ein Bearbeiterkreis,
*        beliebig viele Personen darin - das regelt GET_ACTORS
*        allein. Dieser Hook ist fuer den Fall, dass die ANZAHL DER
*        SCHRITTE variabel ist.
*
* WORAUF ZU ACHTEN IST
*        Das Feld WF_DEFINITION_OK in den Eintraegen wird bei
*        parallelen Schritten als Durchlaufzaehler benutzt. Es ist
*        zweckentfremdet, aber die etablierte Loesung - wer die
*        Eintraege ohne diesen Index dupliziert, bekommt Workitems,
*        die sich nicht auseinanderhalten lassen.
*--------------------------------------------------------------------*

    mv_process = iv_process.

*--------------------------------------------------------------------*
* Beim Mailversand den Info-Empfaenger nur mitnehmen, wenn er im
* Customizing aktiviert ist. So kann der Kunde eine Benachrichtigung
* an- und abschalten, ohne den Code anzufassen.
*--------------------------------------------------------------------*
    IF iv_process = 'MAIL'.

      SELECT SINGLE objid FROM /c09/cfl_c03 INTO @DATA(lv_objid)
        WHERE wf_definition = @is_cfl_s01-wf_definition
          AND gen_stat_user = @zcl_cfl_const_00900=>mc_gsu-mail_info
          AND objid         = @abap_true.

      IF sy-subrc <> 0.
        DELETE ct_actors WHERE gen_stat_user = zcl_cfl_const_00900=>mc_gsu-mail_info.
      ENDIF.

    ENDIF.

  ENDMETHOD.


*======================================================================*
*
*   GRUPPE 2 - ANZEIGE
*   Was der Bearbeiter sieht, bevor er entscheidet
*
*   Fuenf Hooks fuer vier Stellen am Bildschirm. Sie werden regelmaessig
*   verwechselt, deshalb die Landkarte vorweg:
*
*     GET_DESCRIPTION        die EINE Zeile in der Trefferliste
*                            (CHAR100 - mehr geht nicht)
*     GET_WORKITEM_TEXT      der BLOCK, den man nach dem Oeffnen liest
*     GET_OBJECT_INFO        die Beschriftung des Objekt-Links in der
*                            FIORI-Inbox
*     GET_NEW_PREVIEW_DESCR  dieselbe Beschriftung im SAP-GUI
*     GET_WF_DEFINITION_TEXT der Titel des Gesamtworkflows
*
*   GET_OBJECT_INFO und GET_NEW_PREVIEW_DESCR beschriften DASSELBE und
*   werden trotzdem beide gebraucht: die Fiori-Inbox fuellt ihren
*   Reiter "Links" aus SAP_WAPI_GET_OBJECTS und sieht
*   GET_NEW_PREVIEW_DESCR nie. Wer nur einen der beiden pflegt, hat
*   in der anderen Oberflaeche den Framework-Klassennamen stehen.
*
*======================================================================*

  METHOD /c09/cfl_if_badi_0101~get_description.
*--------------------------------------------------------------------*
* WANN   Bei jeder Anzeige der Instanz - Trefferliste, Kopfzeile,
*        Workflow-Protokoll.
*
* REIN   IS_DATA         die Instanz
* RAUS   CV_DESCRIPTION  die Zeile. CHAR100, HART.
*
* ZWEI DINGE, DIE MAN WISSEN MUSS
*
* 1. DER TECHNISCHE PRAEFIX
*    Das Framework liefert CV_DESCRIPTION bereits gefuellt. Gebaut
*    wird der Text in /C09/CFL_CL_WORKFLOW_0101 so:
*
*        CONCATENATE ms_data-wf_definition '|' ms_data-gen_stat '-'
*                    ms_cfl_c01t-vtext INTO ev_description
*                    SEPARATED BY space.
*
*    Ausgerechnet ergibt das
*
*        00900 | 01 - <c01t-vtext>
*        |<--- 13 --->|
*
*    also 5 (WF_DEFINITION) + 1 + 1 + 1 + 2 (GEN_STAT) + 1 + 1 + 1.
*    Das erklaert die Zeile `cv_description = cv_description+13`, die
*    ohne diesen Absatz wie ein Zufallswert aussieht.
*
*    ACHTUNG BEIM VERGLEICH MIT AELTEREM CODE: in der Praxis findet
*    man haeufig `+12`. Das ist nicht falsch, nur ungenau - es laesst
*    ein fuehrendes Leerzeichen stehen, das beim Anzeigen niemandem
*    auffaellt. Wer 13 nimmt, spart sich das CONDENSE.
*
*    NICHT UNGEPRUEFT ABSCHNEIDEN: ist der Text kuerzer als der
*    Praefix, laeuft der Offset ins Leere und der Kurzdump kommt zur
*    Anzeigezeit - also genau dann, wenn jemand zuschaut.
*
* 2. DIE PLATZHALTER
*    Der Text hinter dem Praefix kommt aus /C09/CFL_C01T und kann
*    Platzhalter der Form §{name} enthalten. Der Kunde pflegt sie im
*    Customizing, dieser Hook ersetzt sie. Damit aendert sich die
*    Zeile ohne Transport.
*
* WAS VORNE STEHEN SOLL
*    Die erste Bildschirmspalte ist die teuerste. Dort gehoert der
*    BEFUND hin, nicht die Belegnummer - die steht ohnehin im Text
*    dahinter. Ein Symbol ganz vorne (Ampel) macht aus der Liste eine
*    Arbeitsliste, die man ueberfliegen kann.
*--------------------------------------------------------------------*

    CONSTANTS lc_prefix_len TYPE i VALUE 13.
    CONSTANTS lc_max_len    TYPE i VALUE 100.

    DATA lv_text TYPE string.

    IF strlen( cv_description ) > lc_prefix_len.
      lv_text = cv_description+lc_prefix_len.
    ELSE.
      lv_text = cv_description.
    ENDIF.

    REPLACE ALL OCCURRENCES OF '§{doc}' IN lv_text
      WITH fmt_doc( get_val( iv_element = zcl_cfl_const_00900=>mc_prop-doc_number
                             iv_id      = is_data-id ) ).

    REPLACE ALL OCCURRENCES OF '§{value}' IN lv_text
      WITH fmt_amount( iv_element = zcl_cfl_const_00900=>mc_prop-net_value
                       iv_id      = is_data-id ).

    REPLACE ALL OCCURRENCES OF '§{severity}' IN lv_text
      WITH get_val( iv_element = zcl_cfl_const_00900=>mc_prop-severity
                    iv_id      = is_data-id ).

*--------------------------------------------------------------------*
* Auf CHAR100 kuerzen - und zwar SELBST.
*
* Wer es dem Zuweisungsoperator ueberlaesst, bekommt eine Zeile, die
* mitten im Wort endet. Drei Punkte sagen dem Leser, dass da noch
* etwas war.
*--------------------------------------------------------------------*
    CONDENSE lv_text.

    IF strlen( lv_text ) > lc_max_len.
      lv_text = |{ lv_text(97) }...|.
    ENDIF.

    cv_description = lv_text.

  ENDMETHOD.


  METHOD /c09/cfl_if_badi_0101~get_workitem_text.
*--------------------------------------------------------------------*
* WANN   Beim Oeffnen des Workitems.
*
* REIN   IS_DATA           die Instanz
* RAUS   CT_WORKITEM_TEXT  der Textblock, Zeile fuer Zeile
*
* DAS FORMAT IST SAPSCRIPT-ITF, NICHT HTML
*
*        Der Parametertyp heisst /C09/CFL_HTML_TABLE_TT, und die
*        ausgelieferte Musterimplementierung haengt <br> hinein.
*        Beides ist irrefuehrend. Was der Workitem-Anzeiger auswertet,
*        sind SAPscript-ZEICHENFORMATE:
*
*            <H>ORDER</>      richtig - fett
*            <b>ORDER</b>     wird stillschweigend ENTFERNT
*
*        ITF liest <b> als Zeichenformat namens 'b', kennt es nicht,
*        und loescht die Klammern kommentarlos. Kein Fett, kein
*        sichtbares Tag, keine Fehlermeldung - der irrefuehrendste
*        denkbare Ausgang, weil er wie "HTML wird nicht unterstuetzt"
*        aussieht.
*
*        Geschlossen wird IMMER mit </>, nie mit </H>.
*
* AUFBAU, DER SICH BEWAEHRT HAT
*        Befund zuerst, dann Bloecke mit fetter Ueberschrift. Der
*        Bearbeiter soll nach zwei Zeilen wissen, worum es geht, und
*        erst danach die Einzelheiten lesen.
*
* WAS HIER NICHT HINEINGEHOERT
*        Rechnen. Severity und Empfehlung gehoeren in einen
*        HINTERGRUNDSCHRITT und von dort in den Container. Nur dann
*        steht im Audit Trail, WAS DAS SYSTEM EMPFOHLEN HAT - und ob
*        der Bearbeiter davon abgewichen ist. Rechnet die Anzeige
*        selbst, ist diese Information weg, sobald das Workitem
*        geschlossen ist.
*--------------------------------------------------------------------*

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

  ENDMETHOD.


  METHOD /c09/cfl_if_badi_0101~get_object_info.
*--------------------------------------------------------------------*
* WANN   Wenn die FIORI-Inbox ihren Reiter "Links" aufbaut. Der Weg
*        dorthin fuehrt ueber SAP_WAPI_GET_OBJECTS, nicht ueber
*        GET_NEW_PREVIEW_DESCR - deshalb wirkt dieser Hook dort und
*        der andere nicht.
*
* REIN   IS_LPOR    das Objekt, das beschriftet werden soll
* RAUS   CV_RETURN  der Praefix der Beschriftung
*
* WAS PASSIERT, WENN MAN IHN WEGLAESST
*        In der Fiori-Inbox steht unter "Objects and attachments" der
*        Klassenname des Frameworks:
*
*            CON: conFLOW Worflow V.0101: 0004500001234
*
*        Mit dem Hook:
*
*            Purchase order: 0004500001234
*
*        Die Instanz-ID haengt das Framework selbst an - CV_RETURN
*        ersetzt nur den Teil davor.
*
* DER TEXT GEHOERT IN EIN TEXTSYMBOL, NICHT IN DEN CODE
*        TEXT-001 haengt am Textpool der Klasse. Damit ist er
*        uebersetzbar, und dieselbe Beschriftung steht fuer beide
*        Oberflaechen an genau einer Stelle.
*
*        FALLE BEIM TRANSPORT: Textsymbole haengen am Textpool, nicht
*        am Code. Wer die Klasse ueber abapGit oder einen Transport
*        umzieht und den Textpool vergisst, hat die Beschriftung
*        verloren - und merkt es erst in der Inbox des Zielsystems.
*
* DIESER HOOK SIEHT IM AUFRUFNACHWEIS TOT AUS
*        Where-Used findet ihn nicht, weil er ueber eine Enhancement
*        gerufen wird. Er laeuft trotzdem. Merksatz fuer conFLOW-BAdI-
*        Hooks allgemein: AUSPROBIEREN STATT WHERE-USED GLAUBEN.
*--------------------------------------------------------------------*

    cv_return = text-001.

  ENDMETHOD.


  METHOD /c09/cfl_if_badi_0101~get_new_preview_descr.
*--------------------------------------------------------------------*
* WANN   Wenn das SAP-GUI die Objektliste des Workitems aufbaut
*        ("Objekte und Anlagen").
*
* REIN   IV_WI_ID            das Workitem
* RAUS   CT_PREVIEW_OBJECTS  die Objektliste, aenderbar
*
* WOFUER   Dasselbe wie GET_OBJECT_INFO, nur fuer die andere
*        Oberflaeche. Beide pflegen, sonst ist eine von beiden haesslich.
*
* DER CHECK AUF DEN OBJTYP IST NICHT OPTIONAL
*        In der Liste stehen mehrere Eintraege: das conFLOW-
*        Instanzobjekt, Notizen (SOFM), Anlagen. Wer ohne CHECK durch
*        die Tabelle laeuft, beschriftet alles gleich - auch die
*        Notizen, die dann ihren eigenen Namen verlieren.
*
* AUCH LOESCHEN IST ERLAUBT
*        Die Tabelle ist CHANGING. Ein DELETE nimmt einen Eintrag aus
*        der Anzeige - der uebliche Fall ist eine technische Notiz,
*        die den Bearbeiter nichts angeht. Beispiel steht auskommentiert
*        darunter, weil es einen Schluessel braucht, den es nur im
*        eigenen Projekt gibt.
*--------------------------------------------------------------------*

    LOOP AT ct_preview_objects ASSIGNING FIELD-SYMBOL(<fs_object>).
      CHECK <fs_object>-objtype = '/C09/CFL_CL_WORKFLOW_0101'.
      <fs_object>-descript = text-001.
    ENDLOOP.

*   DELETE ct_preview_objects
*     WHERE objtype    = 'SOFM'
*       AND def_attrib = zcl_cfl_const_00900=>mc_def_attrib_intern.

  ENDMETHOD.


  METHOD /c09/cfl_if_badi_0101~get_wf_definition_text.
*--------------------------------------------------------------------*
* WANN   Beim Aufbau des Titels fuer den GESAMTEN Workflow - nicht
*        fuer einen einzelnen Schritt.
*
* REIN   IS_DATA    die Instanz
* RAUS   CV_WITEXT  der Titel (SWW_WITEXT)
*
* BRAUCHT MAN IHN?
*        Meistens nicht. Der Standardtext kommt aus /C09/CFL_C06T und
*        ist im Customizing pflegbar - ohne Transport, mit
*        Uebersetzung. Das ist der bessere Weg.
*
* WANN DOCH
*        Wenn der Titel Belegdaten enthalten soll, die der
*        Customizing-Text nicht kennt. "Bestellfreigabe" ist im
*        Workflow-Protokoll neben zwanzig anderen nicht auffindbar,
*        "Bestellfreigabe 4500001234 / Mustermann GmbH" schon.
*
*        Bleibt hier LEER, weil der Beispielprozess mit dem
*        Customizing-Text auskommt - und weil ein Referenzbeispiel
*        zeigen soll, dass man Hooks nicht fuellt, nur weil sie da sind.
*--------------------------------------------------------------------*
  ENDMETHOD.


  METHOD /c09/cfl_if_badi_0101~default_attribute_value.
*--------------------------------------------------------------------*
* WANN   Wenn das Workflow-Framework nach dem Standardattribut der
*        Instanz fragt - beim Aufbau von Objektlisten, Anlagen und
*        ueberall dort, wo ein Objekt "sich selbst benennen" soll.
*
* RAUS   RESULT  eine DATENREFERENZ auf den Wert
*
* DIE EINE ZEILE, DIE IN JEDER IMPLEMENTIERUNG GLEICH IST
*
*        Von allen 26 Hooks ist dies der einzige, der in jeder
*        untersuchten Produktivimplementierung gefuellt war - und
*        zwar jedesmal mit exakt derselben Zeile. Wer sie
*        uebernimmt, hat sie richtig.
*
* WARUM GET REFERENCE OF UND NICHT EINE ZUWEISUNG
*        RESULT ist REF TO DATA. Das Framework dereferenziert
*        spaeter. Eine lokale Variable waere zu diesem Zeitpunkt
*        laengst weg - deshalb die Referenz auf das Instanzdatum des
*        Frameworks, das die ganze Zeit lebt.
*
* FALLE  Wer eine Referenz auf eine METHODENLOKALE Variable
*        zurueckgibt, bekommt keinen Fehler, sondern spaeter
*        Datensalat. Immer auf MS_INSTANCES-INSTANCE->MS_DATA
*        referenzieren.
*--------------------------------------------------------------------*

    GET REFERENCE OF /c09/cfl_cl_workflow_0101=>ms_instances-instance->ms_data-instid INTO result.

  ENDMETHOD.


  METHOD /c09/cfl_if_badi_0101~execute_default_method.
*--------------------------------------------------------------------*
* WANN   Doppelklick auf das Objekt im SAP-GUI-Workitem.
*
* REIN   nichts - die Instanz steht in
*        /C09/CFL_CL_WORKFLOW_0101=>MS_INSTANCES-INSTANCE->MS_DATA
*
* WOFUER   "Zeig mir den Beleg". Ohne diesen Hook passiert beim
*        Doppelklick nichts, und der Bearbeiter muss die Belegnummer
*        abschreiben und die Transaktion selbst aufrufen.
*
* MIT ANZEIGE-TRANSAKTION, NICHT MIT AENDERUNGS-TRANSAKTION
*        ME23N, nicht ME22N. Der Bearbeiter soll den Beleg SEHEN,
*        waehrend er entscheidet. Wer ihn hier aendern laesst, hat
*        einen Beleg, der sich unter dem laufenden Workflow bewegt -
*        und einen Audit Trail, der nicht mehr stimmt.
*
* AND SKIP FIRST SCREEN
*        Spart den Einstiegsbildschirm. Bei den Enjoy-Transaktionen
*        (ME23N, VA03) ist der Zusatz wirkungslos bis stoerend - dort
*        genuegt SET PARAMETER ID.
*
* WENN MEHRERE BELEGTYPEN AUF DERSELBEN KLASSE LAUFEN
*        Ueber IS_DATA-TYPEID verzweigen. Bei einem eigenen BOR-Typ
*        je Workflow (der Normalfall) entfaellt das.
*--------------------------------------------------------------------*

    DATA lv_ebeln TYPE ekko-ebeln.

    lv_ebeln = /c09/cfl_cl_workflow_0101=>ms_instances-instance->ms_data-instid.

    SET PARAMETER ID 'BES' FIELD lv_ebeln.
    CALL TRANSACTION 'ME23N'.

  ENDMETHOD.


*======================================================================*
*
*   GRUPPE 3 - DER WORKITEM-LEBENSZYKLUS
*   Anlegen, oeffnen, entscheiden, abschliessen
*
*   Hier sitzen die drei Stellen, an denen man den Prozess ANHALTEN
*   kann. Sie koennen unterschiedlich viel, und das ist der wichtigste
*   Unterschied in dieser ganzen Gruppe:
*
*     GET_BEFORE_DECISION_WORKITEM
*         kann Buttons ENTFERNEN. Keine Meldung, kein Abbruch.
*         "Darf nicht" heisst hier: der Knopf ist weg.
*
*     GET_AFTER_EXECUTION
*         kann ABBRECHEN (CV_SUBRC), aber OHNE Meldung. Der Bearbeiter
*         sieht nur, dass nichts passiert - unbrauchbar allein.
*
*     GET_AFTER_EXECUTION_MOBILE
*         kann ABBRECHEN UND SAGEN WARUM (CS_T100MSG). Der einzige
*         Hook mit beidem.
*
*   Merksatz: was der Bearbeiter NICHT DARF, nimmt man ihm vorher weg.
*   Was er FALSCH GEMACHT hat, sagt man ihm nachher.
*
*======================================================================*

  METHOD /c09/cfl_if_badi_0101~get_after_creation_workitem.
*--------------------------------------------------------------------*
* WANN   Nach dem Anlegen JEDES Workitems - auch der
*        Hintergrundschritte - und vor der ersten Anzeige.
*
* REIN   IS_DATA_STEP  der Schritt (/C09/CFL_S03)
*        IS_SWR_WIHDR  der Workitem-Kopf, inklusive WI_ID
*
* RAUS   nichts. Wirkung nur ueber Seiteneffekte.
*
* WOFUER   Alles, was EINMAL JE WORKITEM passieren soll: Prioritaet,
*        Anlagen, Notizen, Container vorbereiten, eine eigene
*        Oberflaeche anhaengen.
*
* DER HOOK LAEUFT AUCH FUER HINTERGRUNDSCHRITTE
*        Und das ist fast immer unerwuenscht. Eine Prioritaet an
*        einem Workitem, das kein Mensch sieht, kostet nur Laufzeit.
*        Deshalb steht am Anfang eine Weiche auf die Dialogschritte -
*        die Zeile sieht nach Kleinkram aus und ist keiner.
*
* DAS WORKITEM STEHT HIER NOCH NICHT AUF DER DATENBANK
*        Der zentrale Punkt dieses Hooks, und die Ursache der
*        haeufigsten Enttaeuschung: jede API, die SWWWIHEAD LIEST,
*        laeuft ins Leere. SAP_WAPI_CHANGE_WORKITEM_PRIO tut genau
*        das - sie meldet keinen Fehler, sie wirkt nur nicht. Der
*        richtige Weg geht ueber den Workitem-Manager der laufenden
*        Transaktion, siehe SET_PRIORITY( ).
*--------------------------------------------------------------------*

    IF is_data_step-gen_stat <> zcl_cfl_const_00900=>mc_stat-approve AND
       is_data_step-gen_stat <> zcl_cfl_const_00900=>mc_stat-escalate.
      RETURN.
    ENDIF.

    IF is_swr_wihdr-wi_id IS INITIAL.
      RETURN.
    ENDIF.

*--------------------------------------------------------------------*
* Die Eskalation ist immer dringend - sie ist ja gerade deshalb beim
* Vorgesetzten gelandet. Sonst entscheidet die in B1 berechnete
* Severity.
*--------------------------------------------------------------------*
    DATA lv_prio TYPE sww_prio.

    IF is_data_step-gen_stat = zcl_cfl_const_00900=>mc_stat-escalate.
      lv_prio = zcl_cfl_const_00900=>mc_prio-high.

    ELSEIF get_val( iv_element = zcl_cfl_const_00900=>mc_prop-severity
                    iv_id      = is_data_step-id ) = zcl_cfl_const_00900=>mc_severity-red.
      lv_prio = zcl_cfl_const_00900=>mc_prio-high.

    ELSE.
      lv_prio = zcl_cfl_const_00900=>mc_prio-medium.
    ENDIF.

    set_priority( iv_wi_id = is_swr_wihdr-wi_id
                  iv_prio  = lv_prio ).

  ENDMETHOD.


  METHOD /c09/cfl_if_badi_0101~get_before_execution_workitem.
*--------------------------------------------------------------------*
* WANN   Unmittelbar bevor ein Workitem ausgefuehrt wird - also nach
*        dem Doppelklick, vor dem Aufbau der Oberflaeche.
*
* REIN   IS_SWR_WIHDR  der Workitem-Kopf
* RAUS   nichts.
*
* WOFUER   Vorbereitungen, die genau dann noetig sind, wenn jemand das
*        Workitem tatsaechlich oeffnet - Sperren setzen, einen Cache
*        fuellen, Zaehler hochsetzen.
*
* IN KEINER DER UNTERSUCHTEN PRODUKTIVIMPLEMENTIERUNGEN GEFUELLT.
*
*        Der Grund: er kann nichts verhindern. Es gibt keinen
*        Rueckgabeparameter und keine Moeglichkeit abzubrechen -
*        was hier passiert, passiert nebenbei. Wer eine Pruefung
*        sucht, ist bei GET_BEFORE_DECISION_WORKITEM richtig (Button
*        wegnehmen) oder bei GET_AFTER_EXECUTION_MOBILE (abbrechen
*        mit Meldung).
*
*        BLEIBT LEER. Das ist die Entscheidung, nicht ein Rest.
*--------------------------------------------------------------------*
  ENDMETHOD.


  METHOD /c09/cfl_if_badi_0101~get_before_decision_workitem.
*--------------------------------------------------------------------*
* WANN   Wenn der Workitem-Exit die Entscheidungsalternativen
*        aufbaut - also bevor die Knoepfe gezeichnet werden.
*
* REIN/RAUS  CM_WORKITEM_CONTEXT - der Workitem-Kontext. Aus ihm holt
*            man Kopf und Alternativen, in ihn schreibt man die
*            geaenderten zurueck.
*
* ZWEI DINGE GEHEN HIER, UND NUR HIER
*
* 1. EINEN BUTTON WEGNEHMEN (der "Guard")
*    Wenn eine Aktion fachlich nicht erlaubt ist, verschwindet sie -
*    statt hinterher abgelehnt zu werden. Das ist die freundlichere
*    Bauart: der Bearbeiter sieht nur, was er darf.
*
*    Der Hook kann KEINE Fehlermeldung erzwingen. "Darf nicht" heisst
*    hier Button weg, nicht Fehler danach.
*
* 2. EINEN BUTTON EINFAERBEN
*    SWR_DECIALTS hat das Feld ALTNATURE, und der Task-Gateway kopiert
*    es unveraendert nach NATURE. Damit faerbt die Fiori-Inbox.
*
*    Es gibt GENAU ZWEI Werte - POSITIVE und NEGATIVE. Das ist keine
*    Palette, sondern eine Aussage, und sie wird sparsam vergeben:
*
*        POSITIVE   die berechnete Empfehlung
*        NEGATIVE   die eine Aktion, die etwas endgueltig wegwirft
*        neutral    alles andere
*
*    Empfiehlt das System ausnahmsweise selbst die harte Aktion,
*    gewinnt die Empfehlung. Zwei Signale auf demselben Button
*    waeren keins.
*
* WARUM DIE FARBE TROTZDEM ZWEIMAL GESETZT WIRD
*        Hier UND in GET_FIORI_TASK_DEC_OP_ACT. Die beiden Hooks
*        arbeiten auf verschiedenen Tabellen: dieser auf den
*        Alternativen des Workitem-Exits (SAP-GUI), jener auf den
*        Optionen des Task-Gateways (Fiori). Wer nur einen pflegt,
*        hat die Farbe in einer der beiden Oberflaechen nicht.
*
* DER EINSTIEG UEBER DIE WI_ID IST PFLICHT
*        Der Hook bekommt die conFLOW-Instanz NICHT mit. Der einzige
*        Weg dorthin fuehrt ueber den Workitem-Kopf und /C09/CFL_S03.
*--------------------------------------------------------------------*

    CONSTANTS lc_positive TYPE swr_nature VALUE 'POSITIVE' ##NO_TEXT.
    CONSTANTS lc_negative TYPE swr_nature VALUE 'NEGATIVE' ##NO_TEXT.

    DATA lt_decialts TYPE if_wapi_workitem_context=>swrtdecialts.

    DATA(ls_wihdr) = cm_workitem_context->get_header( ).

    SELECT SINGLE * FROM /c09/cfl_s03 INTO @DATA(ls_cfl_s03) "#EC CI_ALL_FIELDS_NEEDED
      WHERE wi_id = @ls_wihdr-wi_id.                          "#EC CI_NOORDER
    IF sy-subrc <> 0.
      RETURN.
    ENDIF.

    cm_workitem_context->get_decision_alts( IMPORTING et_decialts = lt_decialts ).

*--------------------------------------------------------------------*
* GUARD - ablehnen darf nur, wer eskalieren kann.
*
* Im Beispiel: der Einkaeufer auf Schritt 01 soll einen Beleg ueber
* dem Limit nicht allein ablehnen koennen. Der Vorgesetzte auf
* Schritt 02 darf.
*--------------------------------------------------------------------*
    IF ls_cfl_s03-gen_stat = zcl_cfl_const_00900=>mc_stat-approve AND
       is_true( get_val( iv_element = zcl_cfl_const_00900=>mc_prop-limit_hit
                         iv_id      = ls_cfl_s03-id ) ) = abap_true.

      DELETE lt_decialts WHERE altkey = /c09/cfl_cl_workflow_0101=>mc_decision-nok.

    ENDIF.

*--------------------------------------------------------------------*
* FARBE
*--------------------------------------------------------------------*
    IF zcl_cfl_const_00900=>mc_fiori_nature = abap_true.

      DATA(lv_recommended) = recommended_key( ls_cfl_s03-id ).

      LOOP AT lt_decialts ASSIGNING FIELD-SYMBOL(<fs_alt>).
        IF lv_recommended IS NOT INITIAL AND <fs_alt>-altkey = lv_recommended.
          <fs_alt>-altnature = lc_positive.
        ELSEIF <fs_alt>-altkey = /c09/cfl_cl_workflow_0101=>mc_decision-nok.
          <fs_alt>-altnature = lc_negative.
        ENDIF.
      ENDLOOP.

    ENDIF.

    cm_workitem_context->set_decision_alts( it_decialts = lt_decialts ).

  ENDMETHOD.


  METHOD /c09/cfl_if_badi_0101~get_fiori_task_dec_op_act.
*--------------------------------------------------------------------*
* WANN   Wenn die Fiori-Inbox ihre Entscheidungsoptionen holt. Der
*        Weg fuehrt ueber den Task-Gateway-Handler, nicht ueber den
*        Workitem-Exit - deshalb reicht GET_BEFORE_DECISION_WORKITEM
*        allein nicht.
*
* REIN   IV_INSTANCE_ID  die WI_ID
* RAUS   CT_DEC_OPT      die Optionen der Inbox
*
* DIESER HOOK SIEHT IM AUFRUFNACHWEIS TOT AUS - UND LAEUFT
*        /C09/CL_TGW_RFC_HANDLER ist keine eigene Klasse, sondern
*        eine conFLOW-ENHANCEMENT auf den Task-Gateway-Handler.
*        Where-Used findet dadurch nichts. Der Hook laeuft trotzdem,
*        und er ist dieselbe Stelle, an der auch die Buttontexte aus
*        /C09/CFL_C09T gesetzt werden.
*
* GEMATCHT WIRD UEBER DEN SCHLUESSEL, NIE UEBER DEN TEXT
*        Naheliegend waere ein Vergleich auf DECISION_TEXT. Er kann
*        nicht funktionieren: zu diesem Zeitpunkt steht dort schon
*        der UEBERSETZTE Text aus /C09/CFL_C09T, nicht mehr der
*        Rohschluessel. DECISION_KEY dagegen ist NUMC4 mit derselben
*        Nummerierung wie SWR_DECIKEY (0001 = OK, 0002 = NOK,
*        0003 = UC1 ...) und damit stabil.
*
* DECISION_TEXT NICHT ANFASSEN
*        Das Framework prueft direkt NACH diesem Aufruf, ob der Text
*        noch ein '-' enthaelt, und ueberspringt sonst die
*        C09T-Uebersetzung. Wer hier am Text schreibt, hat hinterher
*        UNBESCHRIFTETE Buttons - und sucht den Fehler an der
*        falschen Stelle.
*--------------------------------------------------------------------*

    CONSTANTS lc_positive TYPE /iwwrk/wf_decision_nature VALUE 'POSITIVE' ##NO_TEXT.
    CONSTANTS lc_negative TYPE /iwwrk/wf_decision_nature VALUE 'NEGATIVE' ##NO_TEXT.

    IF zcl_cfl_const_00900=>mc_fiori_nature = abap_false.
      RETURN.
    ENDIF.

    SELECT SINGLE * FROM /c09/cfl_s03 INTO @DATA(ls_cfl_s03) "#EC CI_ALL_FIELDS_NEEDED
      WHERE wi_id = @iv_instance_id.                          "#EC CI_NOORDER
    IF sy-subrc <> 0.
      RETURN.
    ENDIF.

    DATA(lv_recommended) = recommended_key( ls_cfl_s03-id ).

    LOOP AT ct_dec_opt ASSIGNING FIELD-SYMBOL(<fs_opt>).
      IF lv_recommended IS NOT INITIAL AND <fs_opt>-decision_key = lv_recommended.
        <fs_opt>-nature = lc_positive.
      ELSEIF <fs_opt>-decision_key = /c09/cfl_cl_workflow_0101=>mc_decision-nok.
        <fs_opt>-nature = lc_negative.
      ENDIF.
    ENDLOOP.

  ENDMETHOD.


  METHOD /c09/cfl_if_badi_0101~get_after_execution.
*--------------------------------------------------------------------*
* WANN   Nachdem der Bearbeiter im SAP-GUI entschieden hat, bevor das
*        Workitem abgeschlossen wird.
*
* REIN   IV_WI_ID     das Workitem
*        IV_ALTKEY    der GEKLICKTE Ausgang - der eigentliche Wert
*        IV_ALT_TEXT  dessen Text
*        IV_MSELNOTE  Vorgabe fuer den Notiz-Dialog
*
* RAUS   CV_SUBRC      <> 0 bricht ab
*        CS_OBJECT_ID  Referenz auf eine erfasste Notiz
*
* ZWEI VERSCHIEDENE AUFGABEN, DIE HIER ZUSAMMENFALLEN
*
* 1. NACHLAUFLOGIK - den Container fortschreiben, festhalten wer
*    entschieden hat. Das ist der uebliche Fall.
*
* 2. EINE NOTIZ ERZWINGEN - ueber SWU_INTERN_DECI_NOTE_POPUP. Das
*    ist der Standardweg fuer "Ablehnung bitte begruenden".
*
* DER ABBRUCH KANN NICHT SAGEN WARUM
*        CV_SUBRC <> 0 haelt den Prozess an, aber es gibt keinen
*        Meldungsparameter. Der Bearbeiter klickt und es passiert
*        nichts - die schlechteste aller Rueckmeldungen. Wer eine
*        Pruefung MIT Begruendung braucht, ruft dieselbe Pruefung
*        zusaetzlich in GET_AFTER_EXECUTION_MOBILE, dem einzigen
*        Hook mit CS_T100MSG.
*
* ERSTE ZEILE: AUF IV_ALTKEY PRUEFEN
*        Der Hook laeuft auch bei Aktionen, die keine Entscheidung
*        sind (Weiterleiten, Zurueckstellen). Dann ist IV_ALTKEY
*        leer, und jede Logik, die einen Ausgang voraussetzt, greift
*        ins Leere.
*--------------------------------------------------------------------*

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

  ENDMETHOD.


  METHOD /c09/cfl_if_badi_0101~get_after_execution_mobile.
*--------------------------------------------------------------------*
* WANN   Nach der Ausfuehrung aus conMOBILE bzw. aus dem BSP-Pfad.
*
* REIN   IV_WI_ID   das Workitem
*        IV_ALTKEY  der geklickte Ausgang
*
* RAUS   CV_SUBRC     <> 0 bricht ab. 9 ist der uebliche Wert.
*        CS_T100MSG   die Meldung dazu
*
* DER EINZIGE HOOK, DER ABBRECHEN UND DABEI SAGEN KANN WARUM.
*
*        Das macht ihn zur wichtigsten Schranke im ganzen Interface -
*        wichtiger, als sein Name vermuten laesst.
*
* DER NAME IST IRREFUEHREND
*        "MOBILE" legt nahe, dass er nur fuer conMOBILE laeuft. Laut
*        Dokumentation haengt er am conMOBILE-/BSP-Pfad; ob eine
*        bestimmte Fiori-Inbox ihn ruft, ist im eigenen System zu
*        PRUEFEN und nicht anzunehmen. Bei GET_FIORI_TASK_DEC_OP_ACT
*        sah es genauso tot aus, und der Hook lief.
*
*        Faellt er in der eigenen Oberflaeche aus, wirkt die Schranke
*        dort nicht. Dann bleibt nur: dieselbe Pruefung zusaetzlich
*        in einen HINTERGRUNDSCHRITT hinter der Entscheidung legen -
*        der laeuft immer.
*
* FREITEXT ALS T100-MELDUNG
*        CS_T100MSG will ID und Nummer, keinen String. 00/398 ist
*        '&1&2&3&4' - vier Variablen a 50 Zeichen. Damit passen 200
*        Zeichen Freitext hinein. Eine eigene Nachrichtenklasse ist
*        sauberer, aber dieser Weg braucht kein neues Objekt.
*--------------------------------------------------------------------*

    CONSTANTS lc_var_len TYPE i VALUE 50.

    IF iv_altkey IS INITIAL.
      RETURN.
    ENDIF.

    SELECT SINGLE * FROM /c09/cfl_s03 INTO @DATA(ls_cfl_s03) "#EC CI_ALL_FIELDS_NEEDED
      WHERE wi_id = @iv_wi_id.                                "#EC CI_NOORDER
    IF sy-subrc <> 0.
      RETURN.
    ENDIF.

*--------------------------------------------------------------------*
* Die Pruefung: Ablehnung ohne Notiz gibt es nicht.
*
* Bewusst dieselbe fachliche Regel wie im Guard weiter oben - aber
* an anderer Stelle wirksam. Der Guard nimmt weg, was gar nicht
* erlaubt ist; diese Schranke prueft, was zur erlaubten Aktion noch
* fehlt.
*--------------------------------------------------------------------*
    DATA lv_error TYPE string.

    IF iv_altkey = /c09/cfl_cl_workflow_0101=>mc_decision-nok AND
       get_val( iv_element = zcl_cfl_const_00900=>mc_prop-note
                iv_id      = ls_cfl_s03-id ) IS INITIAL.

      lv_error = 'A rejection needs a reason. Please fill in the note.' ##NO_TEXT.

    ENDIF.

    IF lv_error IS INITIAL.
      RETURN.
    ENDIF.

    DATA(lv_len) = strlen( lv_error ).

    cs_t100msg-msgid = '00'.
    cs_t100msg-msgno = '398'.
    cs_t100msg-msgty = 'E'.
    cs_t100msg-msgv1 = lv_error.

    IF lv_len > lc_var_len.
      cs_t100msg-msgv2 = lv_error+lc_var_len.
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


  METHOD /c09/cfl_if_badi_0101~get_after_execution_workitem.
*--------------------------------------------------------------------*
* WANN   Nachdem ein Workitem abgeschlossen wurde - unabhaengig
*        davon, aus welcher Oberflaeche.
*
* REIN   IS_DATA_STEP  der Schritt
*        IS_SWR_WIHDR  der Workitem-Kopf
*        IV_KEY        der Ausgang
*
* RAUS   nichts, und AUCH KEIN ABBRUCH. Was hier passiert, passiert
*        nach der Entscheidung.
*
* DER KLASSISCHE ANWENDUNGSFALL: PARALLELE WORKITEMS AUFRAEUMEN
*
*        Wenn ein Schritt mehrere Bearbeiter parallel hat und einer
*        ablehnt, sollen die anderen Workitems verschwinden - sonst
*        arbeiten Leute an einem Vorgang, der schon entschieden ist.
*
*        /C09/CFL_CL_HELPER_0101=>SET_WORKITEM_OBSOLET erledigt das:
*        es sucht alle offenen Workitems desselben Top-Workflows und
*        setzt sie auf obsolet - das eigene ausgenommen.
*
* WARUM DAS COMMIT HIER STEHT
*        SET_WORKITEM_OBSOLET ruft SAP_WAPI_WORKITEM_COMPLETE mit
*        DO_COMMIT = FALSE, damit nicht je Workitem einzeln
*        festgeschrieben wird. Das COMMIT muss also der Aufrufer
*        machen. Ohne die Zeile bleiben die Workitems offen - ohne
*        Fehlermeldung.
*
* UNTERSCHIED ZU GET_AFTER_EXECUTION
*        GET_AFTER_EXECUTION laeuft VOR dem Abschluss und kann ihn
*        verhindern. Dieser hier laeuft DANACH. Wer pruefen will,
*        nimmt den anderen.
*--------------------------------------------------------------------*

    IF iv_key <> /c09/cfl_cl_workflow_0101=>mc_decision-nok.
      RETURN.
    ENDIF.

    /c09/cfl_cl_helper_0101=>set_workitem_obsolet( is_data_step = is_data_step ).

    COMMIT WORK AND WAIT.

  ENDMETHOD.


  METHOD /c09/cfl_if_badi_0101~get_event_raised_workitem.
*--------------------------------------------------------------------*
* WANN   Wenn ein Ereignis auf ein LAUFENDES Workitem trifft - nicht
*        beim Start des Workflows, sondern waehrend er laeuft.
*
* REIN   IM_EVENT_NAME - welches Ereignis
* REIN/RAUS  CM_WORKITEM_CONTEXT - der Kontext des betroffenen Workitems
*
* WOFUER   Reaktion auf Aenderungen am Beleg, WAEHREND der Workflow
*        offen ist. Der Beleg wird storniert, waehrend jemand ueber
*        ihn entscheidet - dann soll das Workitem verschwinden statt
*        weiter im Eingang zu liegen.
*
* IN KEINER DER UNTERSUCHTEN PRODUKTIVIMPLEMENTIERUNGEN GEFUELLT.
*
*        Nicht, weil er nutzlos waere, sondern weil der Fall selten
*        ist und die Alternative naeher liegt: die Aenderung wird
*        beim naechsten Schritt geprueft, statt sofort zu wirken.
*
*        Wenn man ihn braucht, merkt man es daran: es gibt eine
*        Anforderung der Form "wenn X passiert, WAEHREND der Workflow
*        laeuft, dann ...". Ohne dieses "waehrend" ist es ein
*        normaler Hintergrundschritt.
*
*        BLEIBT LEER.
*--------------------------------------------------------------------*
  ENDMETHOD.


*======================================================================*
*
*   GRUPPE 4 - DIE WEICHE
*   Wohin es als naechstes geht, wenn das Customizing es nicht weiss
*
*======================================================================*

  METHOD /c09/cfl_if_badi_0101~get_status_dynamic.
*--------------------------------------------------------------------*
* WANN   Bei jedem Statuswechsel, nachdem conFLOW den Folgestatus aus
*        /C09/CFL_C02 ermittelt hat - und bevor er ihn benutzt.
*
* REIN/RAUS  CS_DATA - die Instanz MIT dem vorgesehenen Folgestatus.
*            Wer CS_DATA-GEN_STAT hier ueberschreibt, ueberschreibt
*            das Customizing. CS_DATA_OLD haelt den Stand davor.
*
* WOFUER   Verzweigungen, deren Ziel erst zur Laufzeit feststeht:
*        Genehmigungsstufen nach Betrag, ueberspringen einer Stufe,
*        wenn sie fachlich entfaellt, Rueckkehr an die Stelle, an der
*        weitergeleitet wurde.
*
* DAS VERHAELTNIS ZU /C09/CFL_C02
*        C02 sagt "nach Schritt 01 mit Ausgang OK kommt Schritt B2".
*        Dieser Hook sagt "ausser wenn ...". Wer ihn benutzt, hat
*        eine Wegfuehrung, die man im Customizing NICHT MEHR SIEHT -
*        das ist der Preis, und er ist hoch.
*
*        DESHALB DIE REIHENFOLGE DER FRAGEN:
*        1. Geht es mit einem zusaetzlichen Status in C02?
*        2. Geht es mit einem Hintergrundschritt, der ueber
*           EV_DECISION_KEY verzweigt? (Das ist der saubere Weg -
*           die Verzweigung bleibt in C02 sichtbar.)
*        3. Erst dann dieser Hook.
*
*        Der Beispielprozess kommt mit 2 aus - siehe
*        BACKGROUND_CLASSIFY( ) ganz unten, die genau das tut.
*
* WORAN MAN DENKEN MUSS
*        Der Hook laeuft bei JEDEM Statuswechsel, auch bei denen, die
*        einen nichts angehen. Ohne eine Weiche am Anfang schreibt
*        man Faelle um, die man nie gemeint hat.
*
*        BLEIBT HIER LEER - mit Absicht, siehe oben. Das Muster steht
*        auskommentiert darunter, damit man es hat, wenn man es
*        braucht.
*--------------------------------------------------------------------*

*   " Beispiel: Betragsgrenze entscheidet, ob eskaliert wird
*   IF cs_data-gen_stat <> zcl_cfl_const_00900=>mc_stat-approve.
*     RETURN.
*   ENDIF.
*
*   IF is_true( get_val( iv_element = zcl_cfl_const_00900=>mc_prop-limit_hit
*                        iv_id      = cs_data-id ) ) = abap_true.
*     cs_data-gen_stat = zcl_cfl_const_00900=>mc_stat-escalate.
*   ELSE.
*     cs_data-gen_stat = zcl_cfl_const_00900=>mc_stat-post.
*   ENDIF.

  ENDMETHOD.


*======================================================================*
*
*   GRUPPE 5 - MAIL
*   Wer bekommt eine Benachrichtigung, in welcher Sprache, mit welchem
*   Inhalt
*
*   conFLOW verschickt Mails aus einem SO10-Textbaustein. Darin stehen
*   Platzhalter der Form &STRUKTUR-FELD&. GET_DATASOURCE_MAIL fuellt
*   sie, indem es ganze STRUKTUREN uebergibt - nicht einzelne Werte.
*
*   Das ist der Kern und die Ursache der meisten Missverstaendnisse:
*   man liefert Daten, nicht Text. Welche davon im Mail landen,
*   entscheidet der Textbaustein - also der Kunde, ohne Transport.
*
*======================================================================*

  METHOD /c09/cfl_if_badi_0101~get_send_mail_user.
*--------------------------------------------------------------------*
* WANN   Vor dem Mailversand, wenn der Empfaengerkreis feststeht.
*
* REIN   IS_DATA   die Instanz
* RAUS   CT_USER   die Empfaenger als Benutzer
*        CT_MAIL   die Empfaenger als Mailadressen
*
* WOFUER   Empfaenger ergaenzen oder entfernen, die aus der
*        Bearbeiterfindung nicht kommen koennen: eine
*        Verteiler-Adresse, ein externer Ansprechpartner, ein
*        Postfach.
*
* IN KEINER DER UNTERSUCHTEN PRODUKTIVIMPLEMENTIERUNGEN GEFUELLT.
*
*        Der Grund ist gut: der Empfaengerkreis kommt aus
*        /C09/CFL_C05 und /C09/CFL_C03, also aus dem Customizing.
*        Wer ihn im Code ergaenzt, hat einen Empfaenger, den niemand
*        findet, der die Konfiguration liest - und der bleibt, wenn
*        die Person das Haus verlaesst.
*
*        Der saubere Weg fuer "eine Adresse soll immer mit" ist ein
*        eigener GEN_STAT_USER in C05, im Customizing auf die Adresse
*        gesetzt. Dann steht er da, wo man ihn sucht.
*
*        BLEIBT LEER.
*--------------------------------------------------------------------*
  ENDMETHOD.


  METHOD /c09/cfl_if_badi_0101~get_mail_language.
*--------------------------------------------------------------------*
* WANN   Nachdem die Empfaenger feststehen, vor dem Aufbau des Textes.
*
* REIN   IT_USER / IT_MAIL  die Empfaenger
* RAUS   CS_DATA            die Instanz - relevant ist WI_LANG
*
* WOFUER   Die Sprache des Mails festlegen. Standard ist die Sprache
*        der Instanz; hier kann man sie am Empfaenger ausrichten.
*
* IN KEINER DER UNTERSUCHTEN PRODUKTIVIMPLEMENTIERUNGEN GEFUELLT.
*
*        Nicht, weil Mehrsprachigkeit selten waere, sondern weil der
*        Standard schon das Richtige tut: er nimmt die Sprache aus
*        der Instanz, und die SO10-Textbausteine gibt es je Sprache.
*
* WANN MAN IHN BRAUCHT
*        Wenn ein Mail an MEHRERE Empfaenger mit VERSCHIEDENEN
*        Sprachen geht. Dann hilft dieser Hook allerdings auch nur
*        halb - er setzt EINE Sprache fuer den ganzen Versand. Fuer
*        echte Mehrsprachigkeit braucht es mehrere Versandvorgaenge,
*        also mehrere Eintraege in C05.
*
*        BLEIBT LEER.
*--------------------------------------------------------------------*
  ENDMETHOD.


  METHOD /c09/cfl_if_badi_0101~get_status_mail_dynamic.
*--------------------------------------------------------------------*
* WANN   Beim Mailversand, wenn feststeht, WELCHER Bearbeiterkreis
*        angeschrieben wird.
*
* REIN   IS_DATA - die Instanz
* REIN/RAUS  CS_CFL_C05 (der vorgesehene Kreis) und CT_CFL_C05 (die
*            Liste der Kreise) - hier kann man aus einem mehrere
*            machen oder ihn austauschen
*
* WOFUER   Das Gegenstueck zu GET_STATUS_DYNAMIC, aber fuer den
*        Mailversand: "Wer eine Mail bekommt, haengt davon ab, was
*        gerade passiert ist."
*
* DIE ABFRAGE AUF HINTERGRUNDSCHRITTE IST DER TRICK
*        `is_data-gen_stat+0(1) = 'B'` unterscheidet Dialog von
*        Batch - das ist der Grund fuer die Namenskonvention, dass
*        Hintergrundschritte mit B beginnen. Ohne diese Zeile
*        verschickt man Benachrichtigungen zu Schritten, die niemand
*        gesehen hat.
*
*        BLEIBT HIER LEER, weil der Beispielprozess einen festen
*        Empfaengerkreis je Schritt hat. Das Muster steht darunter.
*--------------------------------------------------------------------*

*   " Beispiel: bei Eskalation zusaetzlich den urspruenglichen
*   " Bearbeiter informieren
*   IF is_data-gen_stat+0(1) = 'B'.
*     RETURN.                      " Hintergrundschritt - keine Mail
*   ENDIF.
*
*   IF is_data-gen_stat <> zcl_cfl_const_00900=>mc_stat-escalate.
*     RETURN.
*   ENDIF.
*
*   CLEAR ct_cfl_c05.
*   APPEND cs_cfl_c05 TO ct_cfl_c05.
*   APPEND INITIAL LINE TO ct_cfl_c05 ASSIGNING FIELD-SYMBOL(<fs_c05>).
*   <fs_c05>               = cs_cfl_c05.
*   <fs_c05>-gen_stat_user = zcl_cfl_const_00900=>mc_gsu-buyer.

  ENDMETHOD.


  METHOD /c09/cfl_if_badi_0101~get_datasource_mail.
*--------------------------------------------------------------------*
* WANN   Vor jedem Mailversand, und ausserdem beim Aufbau des
*        Workitem-Textes (IV_WORKITEM_DESC = 'X').
*
* REIN   IS_DATA            die Instanz
*        IS_TEXT            der Textbaustein, der gerade gefuellt wird
*        IV_WORKITEM_DESC   'X' = es geht um den Workitem-Text, nicht
*                           um ein Mail
*
* RAUS   CT_APPLICATION_INPUT  die DATENSTRUKTUREN fuer die
*                              Platzhalter &STRUKTUR-FELD&
*        CT_SO10_TEXT          ganze Textbloecke fuer benannte
*                              Platzhalter wie &NOTE& und &WF_PROT&
*
* DAS PRINZIP: MAN LIEFERT DATEN, NICHT TEXT
*
*        /C09/CFL_CL_HELPER_0101=>ADD_DATASOURCE_MAIL nimmt eine
*        beliebige Struktur entgegen und macht daraus Platzhalter -
*        einen je Feld, benannt nach TABELLE-FELD. Wer EKKO uebergibt,
*        kann im Textbaustein &EKKO-LIFNR& schreiben.
*
*        Welche Felder im Mail landen, entscheidet damit der
*        TEXTBAUSTEIN, also der Kunde. Ohne Transport, ohne
*        Entwickler.
*
* DIE FALLE, DIE JEDEN EINMAL TRIFFT
*
*        ADD_DATASOURCE_MAIL arbeitet ueber RTTI und braucht einen
*        DDIC-HEADER - es liest den Tabellennamen, um daraus die
*        Platzhalternamen zu bauen. Eine LOKALE Struktur (TYPES
*        BEGIN OF ...) hat keinen. Sie wird kommentarlos ignoriert:
*        keine Platzhalter, keine Fehlermeldung, ein Mail mit
*        Luecken.
*
*        Wer berechnete oder formatierte Werte ins Mail bringen will
*        - Betrag im Benutzerformat, Belegnummer ohne fuehrende
*        Nullen, ein zusammengesetzter Text - braucht dafuer eine
*        EIGENE DDIC-STRUKTUR im Data Dictionary. Das ist kein
*        Umweg, das ist die Bauart.
*
* DIE ZWEI BENANNTEN TEXTBLOECKE
*
*        &NOTE&     die Notizen, die Bearbeiter unterwegs erfasst
*                   haben - der Gespraechsverlauf
*        &WF_PROT&  das Workflow-Protokoll als HTML-Tabelle: wer hat
*                   wann was entschieden
*
*        Beides liefert der Produkt-Helper fertig. Selbst bauen
*        lohnt nicht.
*--------------------------------------------------------------------*

*--------------------------------------------------------------------*
* 1. Der Kopftext des Workflows aus dem Customizing.
*
* Damit steht im Mail dieselbe Bezeichnung wie ueberall sonst - und
* sie ist uebersetzt, weil C06T sprachabhaengig ist.
*--------------------------------------------------------------------*
    SELECT SINGLE * FROM /c09/cfl_c06t INTO @DATA(ls_c06t)  "#EC CI_ALL_FIELDS_NEEDED
      WHERE wf_definition = @is_data-wf_definition
        AND lang          = @sy-langu.
    IF sy-subrc = 0.
      /c09/cfl_cl_helper_0101=>add_datasource_mail(
        EXPORTING is_datastruc         = ls_c06t
        CHANGING  ct_application_input = ct_application_input ).
    ENDIF.

*--------------------------------------------------------------------*
* 2. Die Belegdaten - hier als ganze DDIC-Strukturen.
*
* Bewusst OHNE Vorauswahl: es kostet nichts, EKKO und EKPO komplett
* zu uebergeben, und der Kunde kann jedes Feld im Textbaustein
* verwenden, ohne dass jemand den Code anfasst.
*--------------------------------------------------------------------*
    DATA lv_ebeln TYPE ekko-ebeln.

    lv_ebeln = is_data-instid.

    SELECT SINGLE * FROM ekko INTO @DATA(ls_ekko)            "#EC CI_ALL_FIELDS_NEEDED
      WHERE ebeln = @lv_ebeln.
    IF sy-subrc = 0.
      /c09/cfl_cl_helper_0101=>add_datasource_mail(
        EXPORTING is_datastruc         = ls_ekko
        CHANGING  ct_application_input = ct_application_input ).
    ENDIF.

*--------------------------------------------------------------------*
* 3. Die Notizen der bisherigen Bearbeiter.
*
* Der Einstieg geht ueber das Workitem, nicht ueber die Instanz -
* deshalb erst /C09/CFL_S03 lesen. GET_PROT_WORKITEM_MAIL sammelt
* dann alle Notizen des GESAMTEN Workflows ein, nicht nur die des
* einen Schritts.
*--------------------------------------------------------------------*
    SELECT SINGLE * FROM /c09/cfl_s03 INTO @DATA(ls_cfl_s03) "#EC CI_ALL_FIELDS_NEEDED
      WHERE id = @is_data-id.                                 "#EC CI_NOORDER
    IF sy-subrc = 0.

      APPEND INITIAL LINE TO ct_so10_text ASSIGNING FIELD-SYMBOL(<fs_so10>).
      <fs_so10>-tdname = '&NOTE&'.
      <fs_so10>-tlines = /c09/cfl_cl_helper_0101=>get_prot_workitem_mail(
                           iv_workitem = ls_cfl_s03-wi_id
                           iv_rfcdest  = space ).

*--------------------------------------------------------------------*
* Die Ueberschrift nur setzen, wenn es ueberhaupt Notizen gibt -
* sonst steht "Notizen:" ueber einem leeren Block.
*--------------------------------------------------------------------*
      IF <fs_so10>-tlines IS NOT INITIAL.
        INSERT INITIAL LINE INTO <fs_so10>-tlines ASSIGNING FIELD-SYMBOL(<fs_line>) INDEX 1.
        <fs_line>-tdline = '<b><u>Notes:</u></b><br><br>' ##NO_TEXT.
      ENDIF.

    ENDIF.

*--------------------------------------------------------------------*
* 4. Das Workflow-Protokoll als HTML-Tabelle.
*
* Anders als der Workitem-Text ist das MAIL echtes HTML - hier sind
* <b> und <table> richtig. Die Verwechslungsgefahr mit dem ITF-Format
* des Workitem-Textes ist real: beide Wege sehen im Code gleich aus.
*--------------------------------------------------------------------*
    APPEND INITIAL LINE TO ct_so10_text ASSIGNING <fs_so10>.
    <fs_so10>-tdname = '&WF_PROT&'.

    /c09/cfl_cl_helper_0101=>get_wf_prot(
      EXPORTING is_cfl_s01   = is_data
      IMPORTING et_prot_html = <fs_so10>-tlines ).

  ENDMETHOD.


  METHOD /c09/cfl_if_badi_0101~get_add_attachments.
*--------------------------------------------------------------------*
* WANN   Beim Mailversand, nachdem der Text steht.
*
* REIN   IS_DATA   die Instanz
*        IT_SMTP   die Empfaenger
* RAUS   CT_ATTACHMENT  die Anhaenge
*
* WOFUER   Dokumente ans Mail haengen, die der Empfaenger sonst erst im
*        System suchen muesste: das archivierte Rechnungsbild, die
*        angehaengten Dateien aus dem Beleg, ein SAP-Shortcut.
*
* DER SAP-SHORTCUT IST DER UNTERSCHAETZTE FALL
*        Eine .SAP-Datei im Anhang oeffnet beim Empfaenger direkt das
*        richtige System, den richtigen Mandanten, die richtige
*        Transaktion - mit einem Doppelklick aus dem Mail heraus.
*        Fuer Genehmiger, die selten im System sind, ist das der
*        Unterschied zwischen "wird erledigt" und "liegt liegen".
*
*        WICHTIG: der Shortcut wird JE EMPFAENGER erzeugt, weil der
*        Benutzername darin steht. Deshalb die Schleife ueber
*        IT_SMTP - und deshalb ist es falsch, ihn einmal zu bauen
*        und allen zu schicken.
*
* DREI QUELLEN, DREI PRODUKT-METHODEN
*        GET_SHORTCUT   der SAP-Shortcut
*        GET_ARCHIVE    Dokumente aus dem Archiv (ArchiveLink)
*        GET_GOS        Anlagen am Beleg (Generic Object Services)
*
*        Alle drei liegen in /C09/CFL_CL_HELPER_0101 und liefern
*        fertige Anhangszeilen. Selbst bauen ist Arbeit ohne Gewinn.
*
* GROESSE IM AUGE BEHALTEN
*        Anhaenge gehen in den Mailversand als SOLIX. Ein 20-MB-PDF
*        an zehn Empfaenger ist 200 MB im Sendeauftrag. Wo Groesse
*        ein Thema sein kann, ist der Shortcut die bessere Antwort
*        als das Dokument.
*--------------------------------------------------------------------*

    LOOP AT it_smtp INTO DATA(ls_smtp) WHERE uname IS NOT INITIAL.

      /c09/cfl_cl_helper_0101=>get_shortcut(
        EXPORTING iv_user        = CONV syuname( ls_smtp-uname )
                  iv_transaction = CONV tcode( 'SBWP' )
                  iv_parameter   = space
        CHANGING  ct_attachment  = ct_attachment ).

*--------------------------------------------------------------------*
* Nur fuer den ersten Empfaenger mit Benutzerkennung. Wer den EXIT
* weglaesst, haengt so viele Shortcuts ans Mail, wie es Empfaenger
* gibt - und jeder sieht die Kennungen der anderen.
*--------------------------------------------------------------------*
      EXIT.

    ENDLOOP.

  ENDMETHOD.


*======================================================================*
*
*   GRUPPE 6 - RAHMEN
*   Fristen und Aufraeumen
*
*======================================================================*

  METHOD /c09/cfl_if_badi_0101~get_factory_calendar.
*--------------------------------------------------------------------*
* WANN   Wenn conFLOW eine Frist ausrechnet - also beim Anlegen eines
*        Workitems mit Fristangabe in /C09/CFL_C02.
*
* REIN   IS_CFL_S01           die Instanz
* RAUS   CV_FACTORY_CALENDAR  der Fabrikkalender (SCAL-FCALID)
*
* WOFUER   Damit "zwei Tage Frist" nicht am Freitagnachmittag ablaeuft,
*        weil Samstag und Sonntag mitgezaehlt wurden.
*
* IN KEINER DER UNTERSUCHTEN PRODUKTIVIMPLEMENTIERUNGEN GEFUELLT.
*
*        Das heisst NICHT, dass Fristen ohne Kalender richtig sind -
*        es heisst, dass die untersuchten Prozesse Fristen in
*        Stunden rechnen oder gar keine haben.
*
* WANN MAN IHN BRAUCHT
*        Sobald eine Frist in TAGEN laeuft und die Eskalation eine
*        Wirkung hat, die jemanden aergert. Ein Workitem, das ueber
*        Ostern eskaliert, weil vier Feiertage mitgezaehlt wurden,
*        ist der klassische erste Produktivfehler.
*
*        Der Kalender kommt dann meistens aus dem Werk oder der
*        Buchungskreis-Zuordnung des Belegs - also aus den
*        Belegdaten, nicht aus einer Konstante.
*
*        BLEIBT LEER.
*--------------------------------------------------------------------*
  ENDMETHOD.


  METHOD /c09/cfl_if_badi_0101~release.
*--------------------------------------------------------------------*
* WANN   Wenn die BAdI-Instanz freigegeben wird - am Ende der
*        Verarbeitung.
*
* WOFUER   Aufraeumen. Und zwar genau das, was man in
*        GET_OBJECT_INFO oder GET_AFTER_CREATION_WORKITEM ANGELEGT
*        hat.
*
* DER ZUSAMMENHANG, DEN MAN SONST UEBERSIEHT
*        Wer im SAP-GUI eine eigene Anzeige an das Workitem haengt
*        (ein Docking-Control mit Belegdetails, das uebliche Muster),
*        legt dafuer in GET_OBJECT_INFO eine Singleton-Instanz an.
*        Die lebt dann laenger als das Workitem.
*
*        Ohne das Gegenstueck hier sieht der Bearbeiter beim ZWEITEN
*        geoeffneten Workitem die Daten des ERSTEN. Kein Fehler, kein
*        Dump - nur falsche Zahlen. In beiden untersuchten
*        Implementierungen mit Docking-Control steht deshalb hier
*        genau eine Zeile: DEL_INSTANCE( ).
*
*        Merksatz: RELEASE ist leer, ODER es ist das Gegenstueck zu
*        etwas, das man selbst angelegt hat. Ein dritter Fall kommt
*        nicht vor.
*
*        BLEIBT HIER LEER, weil dieses Beispiel keine eigene
*        SAP-GUI-Anzeige mitbringt.
*--------------------------------------------------------------------*

*   zcl_cfl_workflow_00900_doc=>del_instance( ).

  ENDMETHOD.


*======================================================================*
*
*   DER HINTERGRUNDSCHRITT
*
*   Kein BAdI-Hook - der zweite Vertrag, den conFLOW kennt. Eingetragen
*   wird er in /C09/CFL_C01 auf dem Schritt B1, mit Klassenname und
*   Methodenname. Der Aufruf ist DYNAMISCH: eine falsche Signatur
*   faellt nicht beim Aktivieren auf, sondern zur Laufzeit - und dann
*   bleibt der Workflow stehen.
*
*======================================================================*

  METHOD background_classify.
*--------------------------------------------------------------------*
* Der Schritt, in dem die Arbeit passiert:
*
*   1. Beleg lesen
*   2. Werte in den Container schreiben
*   3. Bewerten
*   4. ueber EV_DECISION_KEY sagen, wie es weitergeht
*
* WARUM DIE WERTE IN DEN CONTAINER GEHEN UND NICHT NUR GELESEN WERDEN
*
*     Der Container ist die Entscheidungsgrundlage, und er ist
*     eingefroren. Wenn der Bearbeiter morgen entscheidet und der
*     Beleg heute Nacht geaendert wurde, hat er trotzdem die Zahlen
*     vor sich, ueber die er entscheidet - und im Audit Trail steht
*     hinterher, welche das waren.
*
*     Wer stattdessen im Workitem-Text frisch aus EKKO liest, hat
*     eine Anzeige, die sich unter dem Bearbeiter bewegt, und
*     hinterher keine Moeglichkeit mehr zu sagen, was er gesehen hat.
*
* WARUM DIE BEWERTUNG HIER STEHT UND NICHT IN DER ANZEIGE
*
*     Aus demselben Grund. SEVERITY und die Empfehlung sind
*     BERECHNETE Werte - wenn sie im Container stehen, sieht man
*     hinterher, was das System empfohlen hat und ob der Bearbeiter
*     davon abgewichen ist. Rechnet die Anzeige, ist diese
*     Information weg, sobald das Workitem zu ist.
*
* DER RUECKGABEWERT IST DIE WEICHE
*
*     EV_DECISION_KEY wird genauso ausgewertet wie die Entscheidung
*     eines Menschen - nur setzt sie hier der Code. In /C09/CFL_C02
*     steht dann:
*
*         B1 + OK   -> 01   (Entscheidung noetig)
*         B1 + UC1  -> X3   (nichts zu tun, Workflow endet)
*
*     Damit bleibt die Verzweigung im CUSTOMIZING SICHTBAR. Das ist
*     der Grund, warum dieser Weg dem Hook GET_STATUS_DYNAMIC
*     vorzuziehen ist: dort waere dieselbe Weiche unsichtbar.
*
*     BLEIBT EV_DECISION_KEY LEER, laeuft der Workflow nicht weiter.
*     Das ist der haeufigste Grund fuer "der Workflow haengt im
*     Hintergrundschritt".
*
* FEHLER GEHEN IN ET_BAPIRET2, NICHT IN EINE EXCEPTION
*
*     conFLOW schreibt die Tabelle ins Anwendungsprotokoll und wertet
*     sie aus. Eine ungefangene Exception dagegen reisst den
*     Workflow-Schritt in den Fehlerstatus, und der Grund steht dann
*     nur im Dump.
*--------------------------------------------------------------------*

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

  ENDMETHOD.


*======================================================================*
*
*   PRIVATE HELFER
*
*======================================================================*

  METHOD get_val.
*--------------------------------------------------------------------*
* Ein Container-Element kann MEHRERE Werte haben - deshalb liefert
* GET_ATTRIBUT_VALUE eine Tabelle. In neun von zehn Faellen will man
* den ersten und einzigen.
*
* Diese vier Zeilen sind der Grund, warum die Hooks oben lesbar sind.
* Ohne sie steht in jedem Hook dieselbe Schleife.
*--------------------------------------------------------------------*

    DATA(lt_value) = /c09/cfl_cl_workflow_0101=>get_attribut_value(
                       iv_element = iv_element
                       iv_id      = iv_id ).

    READ TABLE lt_value ASSIGNING FIELD-SYMBOL(<fs_value>) INDEX 1.
    IF sy-subrc = 0.
      rv_val = <fs_value>-value.
    ENDIF.

  ENDMETHOD.


  METHOD set_val.
*--------------------------------------------------------------------*
* Das Gegenstueck. SET_ATTRIBUT_VALUE ERSETZT den Inhalt des
* Elements - es haengt nicht an. Wer mehrere Werte will, baut die
* Tabelle selbst und ruft die Framework-Methode direkt.
*
* Das ELEMENT muss in /C09/CFL_C07 gepflegt sein. Ist es das nicht,
* wird der Wert kommentarlos verworfen: kein Fehler, kein Eintrag in
* S04, und beim Lesen kommt leer zurueck. Das ist der haeufigste
* Grund fuer "der Container bleibt leer".
*--------------------------------------------------------------------*

    DATA lt_value TYPE /c09/cfl_value_s04_tt.

    APPEND INITIAL LINE TO lt_value ASSIGNING FIELD-SYMBOL(<fs_value>).
    <fs_value>-value = iv_value.

    /c09/cfl_cl_workflow_0101=>set_attribut_value(
      iv_element = iv_element
      iv_id      = iv_id
      it_value   = lt_value ).

  ENDMETHOD.


  METHOD is_true.
*--------------------------------------------------------------------*
* Im Container gibt es keine Booleans, nur Zeichenketten. Was aus
* SET_VAL( abap_true ) zurueckkommt, ist ein 'X' - aber je nachdem,
* wer den Wert gesetzt hat, auch 'x', 'true' oder '1'.
*
* Eine zentrale Auswertung ist deshalb kein Luxus: sonst prueft eine
* Stelle auf 'X' und die naechste auf abap_true, und bei
* Kleinschreibung gehen sie auseinander.
*--------------------------------------------------------------------*

    DATA(lv_upper) = to_upper( condense( iv_value ) ).

    rv_yes = xsdbool( lv_upper = 'X'    OR
                      lv_upper = 'TRUE' OR
                      lv_upper = '1' ).

  ENDMETHOD.


  METHOD read_document.
*--------------------------------------------------------------------*
* Die EINZIGE Methode, die den Beleg kennt. Wer die Klasse auf einen
* anderen Belegtyp umbaut, aendert hier - und an keinem Hook.
*
* EV_FOUND STATT SY-SUBRC NACH AUSSEN
*     Der Aufrufer soll nicht wissen muessen, aus wie vielen SELECTs
*     die Methode besteht. Ein sprechendes Flag ist robuster als ein
*     SY-SUBRC, das der naechste Befehl ueberschreibt.
*
* DIE SUMME UEBER DIE POSITIONEN
*     Fuer eine Freigabe zaehlt der Belegwert, nicht der einer
*     Position. SELECT SUM liefert bei einem Beleg ohne Positionen
*     SY-SUBRC 4 und einen initialen Wert - deshalb steht die
*     Existenzpruefung auf EKKO und nicht auf der Summe.
*--------------------------------------------------------------------*

    CLEAR: ev_net_value, ev_currency, ev_vendor, ev_created_by.
    ev_found = abap_false.

    DATA lv_ebeln TYPE ekko-ebeln.
    lv_ebeln = iv_instid.

    IF lv_ebeln IS INITIAL.
      RETURN.
    ENDIF.

    SELECT SINGLE waers, lifnr, ernam
      FROM ekko
      INTO ( @ev_currency, @ev_vendor, @ev_created_by )
      WHERE ebeln = @lv_ebeln.

    IF sy-subrc <> 0.
      RETURN.
    ENDIF.

    ev_found = abap_true.

    SELECT SUM( netwr )
      FROM ekpo
      INTO @ev_net_value
      WHERE ebeln = @lv_ebeln
        AND loekz = @space.                  " geloeschte Positionen nicht mitzaehlen

  ENDMETHOD.


  METHOD classify.
*--------------------------------------------------------------------*
* Die fachliche Regel, an genau einer Stelle.
*
* Sie steht bewusst NICHT im Hook, obwohl sie dort nur drei Zeilen
* waere. Der Grund ist nicht Aesthetik: sobald die Regel an zwei
* Stellen steht - einmal fuer die Anzeige, einmal fuer die
* Entscheidung - laufen die beiden irgendwann auseinander, und dann
* zeigt das Workitem etwas anderes an, als der Workflow tut.
*
* In einer echten Installation gehoert diese Methode in eine EIGENE
* REGELKLASSE, die auch die Wertehilfe der Oberflaeche bedient. Dann
* kommen Anzeige, Vorschlag und Pruefung nachweislich aus derselben
* Quelle.
*--------------------------------------------------------------------*

    IF iv_net_value > zcl_cfl_const_00900=>mc_limit_value * 5.
      rv_severity = zcl_cfl_const_00900=>mc_severity-red.

    ELSEIF iv_net_value > zcl_cfl_const_00900=>mc_limit_value.
      rv_severity = zcl_cfl_const_00900=>mc_severity-yellow.

    ELSE.
      rv_severity = zcl_cfl_const_00900=>mc_severity-green.
    ENDIF.

  ENDMETHOD.


  METHOD recommended_key.
*--------------------------------------------------------------------*
* Welchen Ausgang das System empfiehlt - fuer die gruene Markierung
* in beiden Oberflaechen.
*
* Die Methode gibt einen SCHLUESSEL zurueck, keinen Text. Das ist
* Absicht: die Buttontexte kommen aus /C09/CFL_C02T und /C09/CFL_C09T
* und sind uebersetzt. Wer hier auf Text vergliche, haette eine
* Empfehlung, die in Englisch funktioniert und in Deutsch nicht.
*--------------------------------------------------------------------*

    IF is_true( get_val( iv_element = zcl_cfl_const_00900=>mc_prop-limit_hit
                         iv_id      = iv_id ) ) = abap_true.
      CLEAR rv_key.                          " ueber dem Limit: keine Empfehlung
    ELSE.
      rv_key = /c09/cfl_cl_workflow_0101=>mc_decision-ok.
    ENDIF.

  ENDMETHOD.


  METHOD fmt_doc.
*--------------------------------------------------------------------*
* Fuehrende Nullen weg. '0004500001234' wird '4500001234'.
*
* WARNUNG, DIE IN DER PRAXIS GELD KOSTET
*     Das Ergebnis taugt NICHT als Schluessel fuer einen SELECT. Wer
*     eine so formatierte Belegnummer in eine WHERE-Bedingung setzt,
*     findet nichts - und bekommt keine Fehlermeldung, sondern eine
*     leere Tabelle. Eine Leseroutine, die fuer die Anzeige
*     formatiert, ist keine Schluesselquelle.
*--------------------------------------------------------------------*

    rv_out = iv_value.
    SHIFT rv_out LEFT DELETING LEADING '0'.
    CONDENSE rv_out.

  ENDMETHOD.


  METHOD fmt_amount.
*--------------------------------------------------------------------*
* Betraege im Format des BENUTZERS, nicht im internen Format.
*
* WRITE ... TO ist dafuer der richtige Befehl - es beachtet die
* Benutzereinstellung fuer Dezimal- und Tausendertrennzeichen. Eine
* Zuweisung an einen String tut das nicht.
*
* DER TRY IST NICHT ZIERDE
*     Im Container steht eine Zeichenkette. Ob sie eine Zahl ist,
*     weiss man nicht: das Element kann leer sein, oder ein frueherer
*     Stand hat Text hineingeschrieben. Eine Konvertierung, die
*     scheitert, wuerde die ANZEIGE des Workitems abbrechen - also
*     genau dann, wenn jemand hinsieht.
*--------------------------------------------------------------------*

    DATA lv_amount TYPE p LENGTH 13 DECIMALS 2.
    DATA lv_out(30) TYPE c.

    DATA(lv_raw) = condense( get_val( iv_element = iv_element
                                      iv_id      = iv_id ) ).

    IF lv_raw IS INITIAL.
      RETURN.
    ENDIF.

    TRY.
        lv_amount = lv_raw.
      CATCH cx_sy_conversion_error.
        rv_out = lv_raw.                     " nicht konvertierbar: roh anzeigen
        RETURN.
    ENDTRY.

    WRITE lv_amount TO lv_out LEFT-JUSTIFIED.
    rv_out = lv_out.
    CONDENSE rv_out.

  ENDMETHOD.


  METHOD add_msg.
*--------------------------------------------------------------------*
* Eine Zeile fuers Anwendungsprotokoll.
*
* WARUM DER UMWEG UEBER NACHRICHT 00/398
*     SLG1 zeigt eine Zeile NUR an, wenn ID und NUMBER gefuellt sind.
*     Ein reiner Freitext im Feld MESSAGE verschwindet spurlos - kein
*     Fehler, keine Zeile, nichts. Das ist stundenlang suchbar.
*
*     00/398 ist die Standardnachricht '&1&2&3&4' - vier Variablen a
*     50 Zeichen. Damit passen 200 Zeichen Freitext hinein, und SLG1
*     zeigt sie an.
*
* WAS MAN STATTDESSEN TUN SOLLTE, WENN ES ERNST WIRD
*     Eine eigene Nachrichtenklasse mit sprechenden Nummern. Dann
*     sind die Meldungen uebersetzbar und auswertbar. 00/398 ist der
*     Weg, der ohne ein neues Objekt auskommt - gut fuer den Anfang,
*     nicht gut fuer die Dauer.
*
* DAMIT ES UEBERHAUPT IN SLG1 LANDET
*     In /C09/CFL_C08 muessen Objekt und Subobjekt zum Workflow
*     hinterlegt sein, UND beide muessen in SLG0 angelegt sein. Fehlt
*     das, sammelt conFLOW die Meldungen ein und schreibt sie
*     nirgends hin.
*--------------------------------------------------------------------*

    CONSTANTS lc_var_len TYPE i VALUE 50.

    DATA ls_return TYPE bapiret2.

    DATA(lv_text) = iv_text.
    DATA(lv_len)  = strlen( lv_text ).

    ls_return-type       = iv_type.
    ls_return-id         = '00'.
    ls_return-number     = '398'.
    ls_return-message    = lv_text.
    ls_return-message_v1 = lv_text.

    IF lv_len > lc_var_len.
      ls_return-message_v2 = lv_text+lc_var_len.
    ENDIF.
    IF lv_len > 100.
      ls_return-message_v3 = lv_text+100.
    ENDIF.
    IF lv_len > 150.
      ls_return-message_v4 = lv_text+150.
    ENDIF.

    APPEND ls_return TO ct_bapiret2.

  ENDMETHOD.


  METHOD set_priority.
*--------------------------------------------------------------------*
* Prioritaet setzen - zweistufig, und beide Stufen werden gebraucht.
*
* DER NAHELIEGENDE WEG FUNKTIONIERT NICHT
*     SAP_WAPI_CHANGE_WORKITEM_PRIO liest SWWWIHEAD von der
*     Datenbank. Im After-Create-Hook steht das Workitem dort noch
*     nicht. Der Aufruf laeuft still ins Leere - kein Fehler, die
*     Prioritaet bleibt auf dem Vorgabewert. Nachgemessen.
*
* STUFE 1 - der Workitem-Manager der laufenden Transaktion
*     CL_SWF_RUN_WIM_FACTORY kennt die Workitems, die GERADE
*     entstehen. Gesucht wird ueber die WI_ID, nicht ueber den Typ:
*     der Hook meint ein bestimmtes Workitem, nicht irgendeins.
*
* STUFE 2 - SWW_WI_PRIORITY_CHANGE mit abgeschalteten Pruefungen
*     Dieselbe Funktion, die unter der WAPI liegt - aber
*     AUTHORIZATION_CHECKED und PRECONDITIONS_CHECKED auf 'X'.
*     Genau diese Pruefungen sind die Blockade, denn das Workitem
*     hat noch keinen Status, den sie akzeptieren wuerden.
*
*     DO_COMMIT BLEIBT LEER. Das COMMIT gehoert dem Framework - wer
*     hier selbst festschreibt, schneidet die laufende Transaktion
*     mitten durch.
*
* DIE TRY-BLOECKE SIND ABSICHT
*     Der Hook laeuft mitten im Anlegen eines Workitems. Eine
*     ungefangene Ausnahme wegen einer PRIORITAET waere ein
*     ausgesprochen teurer Preis fuer ein Darstellungsdetail.
*--------------------------------------------------------------------*

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
              ENDIF.
            CATCH cx_root.
          ENDTRY.
        ENDLOOP.

      CATCH cx_root.
    ENDTRY.

    CALL FUNCTION 'SWW_WI_PRIORITY_CHANGE'
      EXPORTING  wi_id                 = iv_wi_id
                 priority              = iv_prio
                 do_commit             = space
                 authorization_checked = abap_true
                 preconditions_checked = abap_true
      EXCEPTIONS no_authorization      = 1
                 update_failed         = 2
                 invalid_type          = 3
                 invalid_status        = 4
                 OTHERS                = 5.

    IF sy-subrc <> 0.
      /c09/cfl_cl_workflow_0101=>ignore_subrc( ).
    ENDIF.

  ENDMETHOD.


  METHOD update_witext.
*--------------------------------------------------------------------*
* Den Text des laufenden Hintergrund-Workitems nachziehen.
*
* Das Framework setzt den Text aus dem Customizing, BEVOR die
* Hintergrund-Methode laeuft. Wer ein ERGEBNIS im Protokoll sehen
* will, muss ihn danach selbst aendern.
*
* Der Unterschied im Workflow-Protokoll:
*
*     ohne:  "Klassifizierung"          (fuenfmal derselbe Text)
*     mit:   "Approval required - 12.500,00 EUR exceeds ..."
*
* GESUCHT WIRD UEBER WI_TYPE = 'B'
*     Anders als bei SET_PRIORITY gibt es hier keine WI_ID - die
*     Hintergrund-Methode kennt ihr eigenes Workitem nicht. 'B' ist
*     der Hintergrund-Workitem-Typ, und waehrend eines
*     Hintergrundschritts ist genau eines davon registriert.
*--------------------------------------------------------------------*

    DATA lt_instances TYPE swwtwihndl.
    DATA ls_instances LIKE LINE OF lt_instances.
    DATA lo_flow      TYPE REF TO if_swf_run_wim_internal.
    DATA lv_witext    TYPE sww_witext.

    lv_witext = iv_text.

    TRY.
        DATA(lo_factory) = cl_swf_run_wim_factory=>get_instance( ).
        lt_instances = lo_factory->get_registered_workitems( ).

        LOOP AT lt_instances INTO ls_instances.
          TRY.
              lo_flow ?= ls_instances.
              IF lo_flow->m_sww_wihead-wi_type = 'B'.
                lo_flow->if_swf_run_wim~change_witext( lv_witext ).
              ENDIF.
            CATCH cx_root.
          ENDTRY.
        ENDLOOP.

      CATCH cx_root.
    ENDTRY.

  ENDMETHOD.

ENDCLASS.
```
