# `get_number_actors_rel`

| | |
|---|---|
| **Wann** | Vor GET_ACTORS, einmal je Schritt. |
| **Rein** | IS_CFL_S01  die Instanz<br>IV_PROCESS  der Zusammenhang - 'MAIL' beim Mailversand |
| **Raus** | CT_ACTORS   die BEARBEITERKREISE (/C09/CFL_C05_TT), nicht die Bearbeiter. Der Unterschied zu GET_ACTORS liegt genau hier. |

```
WOFÜR   Zwei Dinge, die sonst nirgends gehen:
```

1. PARALLELE WORKITEMS. Wer aus einem Eintrag mehrere macht, bekommt mehrere Workitems auf demselben Schritt. So entstehen Genehmigungsstufen, deren ANZAHL erst zur Laufzeit feststeht - vier Unterschriften bei diesem Beleg, zwei beim nächsten.

2. DEN MERKER FÜR GET_ACTORS SETZEN. IV_PROCESS gibt es nur hier. Wer in GET_ACTORS zwischen Workitem und Mail unterscheiden will, braucht diese Zeile.

BRAUCHT MAN IHN?

Für den Normalfall nein. Ein Schritt, ein Bearbeiterkreis, beliebig viele Personen darin - das regelt GET_ACTORS allein. Dieser Hook ist für den Fall, dass die ANZAHL DER SCHRITTE variabel ist.

**WORAUF ZU ACHTEN IST**

Das Feld WF_DEFINITION_OK in den Einträgen wird bei parallelen Schritten als Durchlaufzähler benutzt. Es ist zweckentfremdet, aber die etablierte Lösung - wer die Einträge ohne diesen Index dupliziert, bekommt Workitems, die sich nicht auseinanderhalten lassen.

## Der Code

```abap
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
```
