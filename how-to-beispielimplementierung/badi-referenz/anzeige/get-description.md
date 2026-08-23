# `get_description`

| | |
|---|---|
| **Wann** | Bei jeder Anzeige der Instanz - Trefferliste, Kopfzeile, Workflow-Protokoll. |
| **Rein** | IS_DATA         die Instanz |
| **Raus** | CV_DESCRIPTION  die Zeile. CHAR100, HART. |

**ZWEI DINGE, DIE MAN WISSEN MUSS**

**1. DER TECHNISCHE PRÄFIX**

   Das Framework liefert CV_DESCRIPTION bereits gefüllt. Gebaut wird der Text in /C09/CFL_CL_WORKFLOW_0101 so:

   ```abap
       CONCATENATE ms_data-wf_definition '|' ms_data-gen_stat '-'
                   ms_cfl_c01t-vtext INTO ev_description
                   SEPARATED BY space.
   ```

   Ausgerechnet ergibt das

   ```
       00900 | 01 - <c01t-vtext>
       |<--- 13 --->|
   ```

   also 5 (WF_DEFINITION) + 1 + 1 + 1 + 2 (GEN_STAT) + 1 + 1 + 1. Das erklärt die Zeile `cv_description = cv_description+13`, die ohne diesen Absatz wie ein Zufallswert aussieht.

   ACHTUNG BEIM VERGLEICH MIT ÄLTEREM CODE: in der Praxis findet man häufig `+12`. Das ist nicht falsch, nur ungenau - es lässt ein führendes Leerzeichen stehen, das beim Anzeigen niemandem auffällt. Wer 13 nimmt, spart sich das CONDENSE.

   NICHT UNGEPRÜFT ABSCHNEIDEN: ist der Text kürzer als der Präfix, läuft der Offset ins Leere und der Kurzdump kommt zur Anzeigezeit - also genau dann, wenn jemand zuschaut.

**2. DIE PLATZHALTER**

   Der Text hinter dem Präfix kommt aus /C09/CFL_C01T und kann Platzhalter der Form §{name} enthalten. Der Kunde pflegt sie im Customizing, dieser Hook ersetzt sie. Damit ändert sich die Zeile ohne Transport.

**WAS VORNE STEHEN SOLL**

Die erste Bildschirmspalte ist die teuerste. Dort gehört der BEFUND hin, nicht die Belegnummer - die steht ohnehin im Text dahinter. Ein Symbol ganz vorne (Ampel) macht aus der Liste eine Arbeitsliste, die man überfliegen kann.

## Der Code

```abap
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
```
