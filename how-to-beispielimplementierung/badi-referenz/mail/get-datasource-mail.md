# `get_datasource_mail`

| | |
|---|---|
| **Wann** | Vor jedem Mailversand, und außerdem beim Aufbau des Workitem-Textes (IV_WORKITEM_DESC = 'X'). |
| **Rein** | IS_DATA            die Instanz<br>IS_TEXT            der Textbaustein, der gerade gefüllt wird<br>IV_WORKITEM_DESC   'X' = es geht um den Workitem-Text, nicht um ein Mail |
| **Raus** | CT_APPLICATION_INPUT  die DATENSTRUKTUREN für die Platzhalter &STRUKTUR-FELD&<br>CT_SO10_TEXT          ganze Textblöcke für benannte Platzhalter wie &NOTE& und &WF_PROT& |

**DAS PRINZIP: MAN LIEFERT DATEN, NICHT TEXT**

/C09/CFL_CL_HELPER_0101=>ADD_DATASOURCE_MAIL nimmt eine beliebige Struktur entgegen und macht daraus Platzhalter - einen je Feld, benannt nach TABELLE-FELD. Wer EKKO übergibt, kann im Textbaustein &EKKO-LIFNR& schreiben.

Welche Felder im Mail landen, entscheidet damit der TEXTBAUSTEIN, also der Kunde. Ohne Transport, ohne Entwickler.

**DIE FALLE, DIE JEDEN EINMAL TRIFFT**

ADD_DATASOURCE_MAIL arbeitet über RTTI und braucht einen DDIC-HEADER - es liest den Tabellennamen, um daraus die Platzhalternamen zu bauen. Eine LOKALE Struktur (TYPES BEGIN OF ...) hat keinen. Sie wird kommentarlos ignoriert: keine Platzhalter, keine Fehlermeldung, ein Mail mit Lücken.

Wer berechnete oder formatierte Werte ins Mail bringen will - Betrag im Benutzerformat, Belegnummer ohne führende Nullen, ein zusammengesetzter Text - braucht dafür eine EIGENE DDIC-STRUKTUR im Data Dictionary. Das ist kein Umweg, das ist die Bauart.

**DIE ZWEI BENANNTEN TEXTBLÖCKE**

```
&NOTE&     die Notizen, die Bearbeiter unterwegs erfasst
           haben - der Gesprächsverlauf
&WF_PROT&  das Workflow-Protokoll als HTML-Tabelle: wer hat
           wann was entschieden
```

Beides liefert der Produkt-Helper fertig. Selbst bauen lohnt nicht.

**WAS DAS FRAMEWORK SCHON LIEFERT - VOR DIESEM HOOK**

/C09/CFL_C06T (VTEXT, OBJTEXT, mit Sprach-Rückfall) und mit c08 TEMPLATE die Belegzeile als &/C09/CFL_S_TPL_`<typ>`-`<feld>`&, Beträge nach Währung formatiert. Hier nur, was darin fehlt.

**GRENZE: NUR DIE ERSTEN FÜNF STRUKTUREN**

Der Mailversand liest aus CT_APPLICATION_INPUT nur fünf Einträge - die des Frameworks zählen mit.

## Der Code

```abap
*--------------------------------------------------------------------*
* 1. Die Belegdaten - hier als ganze DDIC-Struktur.
*
* Bewusst OHNE Vorauswahl: EKKO komplett zu uebergeben kostet einen
* der fuenf Plaetze, und der Kunde kann jedes Feld im Textbaustein
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
```
