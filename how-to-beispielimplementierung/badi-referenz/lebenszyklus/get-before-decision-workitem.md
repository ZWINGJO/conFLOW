# `get_before_decision_workitem`

| | |
|---|---|
| **Wann** | Wenn der Workitem-Exit die Entscheidungsalternativen aufbaut - also bevor die Knöpfe gezeichnet werden. |
| **Rein und raus** | CM_WORKITEM_CONTEXT - der Workitem-Kontext. Aus ihm holt man Kopf und Alternativen, in ihn schreibt man die geänderten zurück. |

**DREI DINGE GEHEN HIER, UND NUR HIER**

1. EINEN BUTTON WEGNEHMEN (der "Guard") Wenn eine Aktion fachlich nicht erlaubt ist, verschwindet sie - statt hinterher abgelehnt zu werden. Das ist die freundlichere Bauart: der Bearbeiter sieht nur, was er darf.

   Der Hook kann KEINE Fehlermeldung erzwingen. "Darf nicht" heißt hier Button weg, nicht Fehler danach.

**2. EINEN BUTTON EINFÄRBEN - SAP GUI, ÜBER HTML IM ALTTEXT**

   Der Entscheidungs-Screen im Business Workplace rendert ALTTEXT als HTML. Farbe entsteht dort also IM TEXT:

   ```
       <span style="color:green;font-size:120%">Genehmigen</span>
   ```

   Farbe und Größe, kein font-weight - beides trifft dieselben Knöpfe, ein drittes Merkmal macht daraus kein drittes Signal. ALTTEXT ist CHAR255, der Wrapper kostet rund 50 Zeichen; die c09t-Texte passen mit Abstand.

   NUR MIT GUI-GUARD. Die Fiori-Inbox nimmt ihre Buttontexte aus genau diesem Hook - ohne Guard stünde das `<span>` dort als Beschriftung auf dem Knopf. Also GUI_IS_AVAILABLE fragen und den HTML-Weg nur bei einer echten GUI gehen.

**3. DENSELBEN BUTTON EINFÄRBEN - FIORI, ÜBER ALTNATURE**

   SWR_DECIALTS hat das Feld ALTNATURE. Es wirkt über die Workflow-DEFINITION und reicht ALLEIN NICHT: die Farbe, die die Fiori-Inbox tatsächlich zeichnet, kommt aus NATURE in GET_FIORI_TASK_DEC_OP_ACT. Hier gesetzt schadet es nicht und bleibt als zweite Schiene stehen - aber wer nur diese Zeile schreibt, sieht in Fiori keine Farbe und sucht sie im falschen Hook.

   Es gibt GENAU ZWEI Werte - POSITIVE und NEGATIVE. Das ist keine Palette, sondern eine Aussage, und sie wird sparsam vergeben:

   ```
   POSITIVE   die berechnete Empfehlung
   NEGATIVE   die eine Aktion, die etwas endgültig wegwirft
   neutral    alles andere
   ```

   Empfiehlt das System ausnahmsweise selbst die harte Aktion, gewinnt die Empfehlung. Zwei Signale auf demselben Button wären keins.

**WARUM DIE ENTSCHEIDUNG TROTZDEM NUR EINMAL FÄLLT**

Zwei Oberflächen, zwei MECHANISMEN - das GUI färbt über HTML im Text, Fiori über NATURE aus dem Task-Gateway, und die beiden Hooks arbeiten auf verschiedenen Tabellen. Wer nur einen Weg pflegt, hat die Farbe in einer der beiden Oberflächen nicht.

Die Frage "welcher Button ist positiv, welcher negativ" beantwortet man deshalb EINMAL - unten im Loop über LV_ROLE - und bedient daraus beide Wege. Schreibt man die Regel zweimal hin, laufen sie beim nächsten zusätzlichen Ausgang auseinander, und niemand merkt es, weil kaum jemand beide Oberflächen nebeneinander aufmacht.

Im Lauf bestätigt in zwei Systemen.

**DER EINSTIEG ÜBER DIE WI_ID IST PFLICHT**

Der Hook bekommt die conFLOW-Instanz NICHT mit. Der einzige Weg dorthin führt über den Workitem-Kopf und /C09/CFL_S03.

## Der Code

```abap
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
* FARBE - eine Aussage, zwei Oberflaechen, zwei Mechanismen
*
* Der HTML-Weg wird nur beschritten, wenn wirklich eine GUI dranhaengt.
*--------------------------------------------------------------------*
    DATA(lv_recommended) = recommended_key( ls_cfl_s03-id ).

    DATA(lv_gui_color) = xsdbool( zcl_cfl_const_00900=>mc_gui_html_color = abap_true AND
                                  is_sapgui( )                           = abap_true ).

    LOOP AT lt_decialts ASSIGNING FIELD-SYMBOL(<fs_alt>).

*     Die Rolle des Knopfes - EINMAL bestimmt, danach zweimal bedient.
      DATA(lv_role) = COND char1(
        WHEN lv_recommended IS NOT INITIAL AND <fs_alt>-altkey = lv_recommended
          THEN 'P'
        WHEN <fs_alt>-altkey = /c09/cfl_cl_workflow_0101=>mc_decision-nok
          THEN 'N' ).

      IF lv_role IS INITIAL.
        CONTINUE.
      ENDIF.

      IF zcl_cfl_const_00900=>mc_fiori_nature = abap_true.
        <fs_alt>-altnature = COND swr_nature( WHEN lv_role = 'P' THEN lc_positive
                                              ELSE lc_negative ).
      ENDIF.

      IF lv_gui_color = abap_true.
        <fs_alt>-alttext = gui_colour(
          iv_text  = <fs_alt>-alttext
          iv_color = COND string( WHEN lv_role = 'P' THEN zcl_cfl_const_00900=>mc_gui_color-positive
                                  ELSE zcl_cfl_const_00900=>mc_gui_color-negative ) ).
      ENDIF.

    ENDLOOP.

    cm_workitem_context->set_decision_alts( it_decialts = lt_decialts ).
```
