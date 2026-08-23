# `get_actors`

| | |
|---|---|
| **Wann** | Bei jeder Workitem-Erzeugung, und ein zweites Mal vor jedem Mailversand. Der wichtigste Hook des ganzen Interface. |
| **Rein** | IS_CFL_C05  der Bearbeiterkreis - GEN_STAT_USER ist der Schlüssel, auf den verzweigt wird<br>IS_CFL_S01  die Instanz (optional, aber praktisch immer da) |
| **Raus** | CT_ACTORS   die Bearbeiter |

**VORHER PRÜFEN, OB MAN IHN BRAUCHT**

/C09/CFL_C03 trägt je GEN_STAT_USER direkt OTYPE und OBJID - etwa US/MEIER, US/WF-BATCH, US/WF_INITIATOR. Erst USER_BADI = 'X' schaltet auf diesen Hook um. Für PoCs, Demos und feste Zuordnungen bleibt GET_ACTORS damit LEER, und es braucht weder Rolle noch Code.

**DAS FORMAT IST DER HÄUFIGSTE FEHLER**

Ein Eintrag in CT_ACTORS ist immer ein TYPISIERTES Org-Objekt, nie ein blanker Benutzername:

```
US<uname>      Benutzer
S<planstelle>  Planstelle
O<orgeinheit>  Organisationseinheit
AC<rolle>      Rolle
```

'MEIER' erzeugt kein Workitem und keine Fehlermeldung. Das Workitem landet bei niemandem und fällt erst auf, wenn jemand fragt, wo es geblieben ist.

**WENN NIEMAND GEFUNDEN WIRD**

Ein Workitem ohne Bearbeiter geht in den Fehlerstatus und bleibt liegen. Besser ist ein definierter Auffangbearbeiter: conFLOW kennt dafür den Eintrag 'C09_NO_USER'. Der Prozess läuft weiter und die Lücke ist sichtbar, statt still zu stehen. Siehe ganz unten in dieser Methode.

## Der Code

```abap
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
```
