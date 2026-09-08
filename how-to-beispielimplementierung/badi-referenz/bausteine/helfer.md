# Die Helfer

Elf Methoden, die kein Hook sind. Sie sind der Grund,
warum die Hooks oben lesbar bleiben: ohne sie steht in
jedem Hook dieselbe Schleife.

## `get_val`

Ein Container-Element kann MEHRERE Werte haben - deshalb liefert GET_ATTRIBUT_VALUE eine Tabelle. In neun von zehn Fällen will man den ersten und einzigen.

Diese vier Zeilen sind der Grund, warum die Hooks oben lesbar sind. Ohne sie steht in jedem Hook dieselbe Schleife.

```abap
DATA(lt_value) = /c09/cfl_cl_workflow_0101=>get_attribut_value(
                   iv_element = iv_element
                   iv_id      = iv_id ).

READ TABLE lt_value ASSIGNING FIELD-SYMBOL(<fs_value>) INDEX 1.
IF sy-subrc = 0.
  rv_val = <fs_value>-value.
ENDIF.
```

## `set_val`

Das Gegenstück. SET_ATTRIBUT_VALUE ERSETZT den Inhalt des Elements - es hängt nicht an. Wer mehrere Werte will, baut die Tabelle selbst und ruft die Framework-Methode direkt.

Das ELEMENT muss in /C09/CFL_C07 gepflegt sein. Ist es das nicht, wird der Wert kommentarlos verworfen: kein Fehler, kein Eintrag in S04, und beim Lesen kommt leer zurück. Das ist der häufigste Grund für "der Container bleibt leer".

```abap
DATA lt_value TYPE /c09/cfl_value_s04_tt.

APPEND INITIAL LINE TO lt_value ASSIGNING FIELD-SYMBOL(<fs_value>).
<fs_value>-value = iv_value.

/c09/cfl_cl_workflow_0101=>set_attribut_value(
  iv_element = iv_element
  iv_id      = iv_id
  it_value   = lt_value ).
```

## `is_true`

Im Container gibt es keine Booleans, nur Zeichenketten. Was aus SET_VAL( abap_true ) zurückkommt, ist ein 'X' - aber je nachdem, wer den Wert gesetzt hat, auch 'x', 'true' oder '1'.

Eine zentrale Auswertung ist deshalb kein Luxus: sonst prüft eine Stelle auf 'X' und die nächste auf abap_true, und bei Kleinschreibung gehen sie auseinander.

```abap
DATA(lv_upper) = to_upper( condense( iv_value ) ).

rv_yes = xsdbool( lv_upper = 'X'    OR
                  lv_upper = 'TRUE' OR
                  lv_upper = '1' ).
```

## `read_document`

Die EINZIGE Methode, die den Beleg kennt. Wer die Klasse auf einen anderen Belegtyp umbaut, ändert hier - und an keinem Hook.

**EV_FOUND STATT SY-SUBRC NACH AUSSEN**

Der Aufrufer soll nicht wissen müssen, aus wie vielen SELECTs die Methode besteht. Ein sprechendes Flag ist robuster als ein SY-SUBRC, das der nächste Befehl überschreibt.

**DIE SUMME ÜBER DIE POSITIONEN**

Für eine Freigabe zählt der Belegwert, nicht der einer Position. SELECT SUM liefert bei einem Beleg ohne Positionen SY-SUBRC 4 und einen initialen Wert - deshalb steht die Existenzprüfung auf EKKO und nicht auf der Summe.

```abap
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
```

## `classify`

Die fachliche Regel, an genau einer Stelle.

Sie steht bewusst NICHT im Hook, obwohl sie dort nur drei Zeilen wäre. Der Grund ist nicht Ästhetik: sobald die Regel an zwei Stellen steht - einmal für die Anzeige, einmal für die Entscheidung - laufen die beiden irgendwann auseinander, und dann zeigt das Workitem etwas anderes an, als der Workflow tut.

In einer echten Installation gehört diese Methode in eine EIGENE REGELKLASSE, die auch die Wertehilfe der Oberfläche bedient. Dann kommen Anzeige, Vorschlag und Prüfung nachweislich aus derselben Quelle.

```abap
IF iv_net_value > zcl_cfl_const_00900=>mc_limit_value * 5.
  rv_severity = zcl_cfl_const_00900=>mc_severity-red.

ELSEIF iv_net_value > zcl_cfl_const_00900=>mc_limit_value.
  rv_severity = zcl_cfl_const_00900=>mc_severity-yellow.

ELSE.
  rv_severity = zcl_cfl_const_00900=>mc_severity-green.
ENDIF.
```

## `recommended_key`

Welchen Ausgang das System empfiehlt - für die grüne Markierung in beiden Oberflächen.

Die Methode gibt einen SCHLÜSSEL zurück, keinen Text. Das ist Absicht: die Buttontexte kommen aus /C09/CFL_C02T und /C09/CFL_C09T und sind übersetzt. Wer hier auf Text vergliche, hätte eine Empfehlung, die in Englisch funktioniert und in Deutsch nicht.

```abap
IF is_true( get_val( iv_element = zcl_cfl_const_00900=>mc_prop-limit_hit
                     iv_id      = iv_id ) ) = abap_true.
  CLEAR rv_key.                          " ueber dem Limit: keine Empfehlung
ELSE.
  rv_key = /c09/cfl_cl_workflow_0101=>mc_decision-ok.
ENDIF.
```

## `fmt_doc`

Führende Nullen weg. '0004500001234' wird '4500001234'.

**WARNUNG, DIE IN DER PRAXIS GELD KOSTET**

Das Ergebnis taugt NICHT als Schlüssel für einen SELECT. Wer eine so formatierte Belegnummer in eine WHERE-Bedingung setzt, findet nichts - und bekommt keine Fehlermeldung, sondern eine leere Tabelle. Eine Leseroutine, die für die Anzeige formatiert, ist keine Schlüsselquelle.

```abap
rv_out = iv_value.
SHIFT rv_out LEFT DELETING LEADING '0'.
CONDENSE rv_out.
```

## `fmt_amount`

Beträge im Format des BENUTZERS, nicht im internen Format.

WRITE ... TO ist dafür der richtige Befehl - es beachtet die Benutzereinstellung für Dezimal- und Tausendertrennzeichen. Eine Zuweisung an einen String tut das nicht.

**DER TRY IST NICHT ZIERDE**

Im Container steht eine Zeichenkette. Ob sie eine Zahl ist, weiß man nicht: das Element kann leer sein, oder ein früherer Stand hat Text hineingeschrieben. Eine Konvertierung, die scheitert, würde die ANZEIGE des Workitems abbrechen - also genau dann, wenn jemand hinsieht.

```abap
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
```

## `add_msg`

Eine Zeile fürs Anwendungsprotokoll.

**WARUM DER UMWEG ÜBER NACHRICHT 00/398**

SLG1 zeigt eine Zeile NUR an, wenn ID und NUMBER gefüllt sind. Ein reiner Freitext im Feld MESSAGE verschwindet spurlos - kein Fehler, keine Zeile, nichts. Das ist stundenlang suchbar.

00/398 ist die Standardnachricht '&1&2&3&4' - vier Variablen a 50 Zeichen. Damit passen 200 Zeichen Freitext hinein, und SLG1 zeigt sie an.

**WAS MAN STATTDESSEN TUN SOLLTE, WENN ES ERNST WIRD**

Eine eigene Nachrichtenklasse mit sprechenden Nummern. Dann sind die Meldungen übersetzbar und auswertbar. 00/398 ist der Weg, der ohne ein neues Objekt auskommt - gut für den Anfang, nicht gut für die Dauer.

**DAMIT ES ÜBERHAUPT IN SLG1 LANDET**

In /C09/CFL_C08 müssen Objekt und Subobjekt zum Workflow hinterlegt sein, UND beide müssen in SLG0 angelegt sein. Fehlt das, sammelt conFLOW die Meldungen ein und schreibt sie nirgends hin.

```abap
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
```

## `set_priority`

Priorität setzen - zweistufig, und beide Stufen werden gebraucht.

**DER NAHELIEGENDE WEG FUNKTIONIERT NICHT**

SAP_WAPI_CHANGE_WORKITEM_PRIO liest SWWWIHEAD von der Datenbank. Im After-Create-Hook steht das Workitem dort noch nicht. Der Aufruf läuft still ins Leere - kein Fehler, die Priorität bleibt auf dem Vorgabewert. Nachgemessen.

**STUFE 1 - der Workitem-Manager der laufenden Transaktion**

CL_SWF_RUN_WIM_FACTORY kennt die Workitems, die GERADE entstehen. Gesucht wird über die WI_ID, nicht über den Typ: der Hook meint ein bestimmtes Workitem, nicht irgendeins.

**STUFE 2 - SWW_WI_PRIORITY_CHANGE mit abgeschalteten Prüfungen**

Dieselbe Funktion, die unter der WAPI liegt - aber AUTHORIZATION_CHECKED und PRECONDITIONS_CHECKED auf 'X'. Genau diese Prüfungen sind die Blockade, denn das Workitem hat noch keinen Status, den sie akzeptieren würden.

DO_COMMIT BLEIBT LEER. Das COMMIT gehört dem Framework - wer hier selbst festschreibt, schneidet die laufende Transaktion mitten durch.

**DIE TRY-BLÖCKE SIND ABSICHT**

Der Hook läuft mitten im Anlegen eines Workitems. Eine ungefangene Ausnahme wegen einer PRIORITÄT wäre ein ausgesprochen teurer Preis für ein Darstellungsdetail.

```abap
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
```

## `update_witext`

Den Text des laufenden Hintergrund-Workitems nachziehen.

Das Framework setzt den Text aus dem Customizing, BEVOR die Hintergrund-Methode läuft. Wer ein ERGEBNIS im Protokoll sehen will, muss ihn danach selbst ändern.

Der Unterschied im Workflow-Protokoll:

```
ohne:  "Klassifizierung"          (fünfmal derselbe Text)
mit:   "Approval required - 12.500,00 EUR exceeds ..."
```

**GESUCHT WIRD ÜBER WI_TYPE = 'B'**

Anders als bei SET_PRIORITY gibt es hier keine WI_ID - die Hintergrund-Methode kennt ihr eigenes Workitem nicht. 'B' ist der Hintergrund-Workitem-Typ, und während eines Hintergrundschritts ist genau eines davon registriert.

```abap
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
```

## `is_sapgui`

GUI_IS_AVAILABLE ist der Standardweg: im OData-/RFC-Kontext der Fiori-Inbox gibt es kein Frontend, also kommt ' ' zurück. Genau diese Unterscheidung braucht der HTML-Weg zum Einfärben.

```abap
DATA lv_return TYPE c LENGTH 1.

CALL FUNCTION 'GUI_IS_AVAILABLE'
  IMPORTING
    return = lv_return.

rv_gui = xsdbool( lv_return = abap_true ).
```

## `gui_colour`

Erzeugt genau die Form, die im Betrieb läuft:

```
    <span style="color:green;font-size:120%">Genehmigen</span>
```

Die Größe steht in ZCL_CFL_CONST_00900=>MC_GUI_FONT_SIZE und darf leer sein; dann bleibt nur die Farbe.

DER SCHUTZ GEGEN DEN ZWEITEN ANLAUF ist billig und ehrlich: der Framework-Exit baut ALTTEXT zwar unmittelbar davor frisch aus c09t, aber ein doppelt gewickelter Text wäre ein Fehler, den man am Bildschirm nicht sieht.

```abap
IF iv_text CS '<span'.
  rv_text = iv_text.
  RETURN.
ENDIF.

DATA(lv_size) = COND string(
  WHEN zcl_cfl_const_00900=>mc_gui_font_size IS INITIAL THEN ``
  ELSE |;font-size:{ zcl_cfl_const_00900=>mc_gui_font_size }| ).

rv_text = |<span style="color:{ iv_color }{ lv_size }">{ iv_text }</span>|.
```
