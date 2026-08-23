# `get_before_decision_workitem`

| | |
|---|---|
| **Wann** | Wenn der Workitem-Exit die Entscheidungsalternativen aufbaut - also bevor die Knöpfe gezeichnet werden. |
| **Rein und raus** | CM_WORKITEM_CONTEXT - der Workitem-Kontext. Aus ihm holt man Kopf und Alternativen, in ihn schreibt man die geänderten zurück. |

**ZWEI DINGE GEHEN HIER, UND NUR HIER**

1. EINEN BUTTON WEGNEHMEN (der "Guard") Wenn eine Aktion fachlich nicht erlaubt ist, verschwindet sie - statt hinterher abgelehnt zu werden. Das ist die freundlichere Bauart: der Bearbeiter sieht nur, was er darf.

   Der Hook kann KEINE Fehlermeldung erzwingen. "Darf nicht" heißt hier Button weg, nicht Fehler danach.

**2. EINEN BUTTON EINFÄRBEN**

   SWR_DECIALTS hat das Feld ALTNATURE, und der Task-Gateway kopiert es unverändert nach NATURE. Damit färbt die Fiori-Inbox.

   Es gibt GENAU ZWEI Werte - POSITIVE und NEGATIVE. Das ist keine Palette, sondern eine Aussage, und sie wird sparsam vergeben:

   ```
   POSITIVE   die berechnete Empfehlung
   NEGATIVE   die eine Aktion, die etwas endgültig wegwirft
   neutral    alles andere
   ```

   Empfiehlt das System ausnahmsweise selbst die harte Aktion, gewinnt die Empfehlung. Zwei Signale auf demselben Button wären keins.

**WARUM DIE FARBE TROTZDEM ZWEIMAL GESETZT WIRD**

Hier UND in GET_FIORI_TASK_DEC_OP_ACT. Die beiden Hooks arbeiten auf verschiedenen Tabellen: dieser auf den Alternativen des Workitem-Exits (SAP-GUI), jener auf den Optionen des Task-Gateways (Fiori). Wer nur einen pflegt, hat die Farbe in einer der beiden Oberflächen nicht.

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
```
