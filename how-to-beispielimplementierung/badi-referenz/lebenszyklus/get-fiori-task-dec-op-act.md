# `get_fiori_task_dec_op_act`

| | |
|---|---|
| **Wann** | Wenn die Fiori-Inbox ihre Entscheidungsoptionen holt. Der Weg führt über den Task-Gateway-Handler, nicht über den Workitem-Exit - deshalb reicht GET_BEFORE_DECISION_WORKITEM allein nicht. |
| **Rein** | IV_INSTANCE_ID  die WI_ID |
| **Raus** | CT_DEC_OPT      die Optionen der Inbox |

**DIESER HOOK SIEHT IM AUFRUFNACHWEIS TOT AUS - UND LÄUFT**

/C09/CL_TGW_RFC_HANDLER ist keine eigene Klasse, sondern eine conFLOW-ENHANCEMENT auf den Task-Gateway-Handler. Where-Used findet dadurch nichts. Der Hook läuft trotzdem, und er ist dieselbe Stelle, an der auch die Buttontexte aus /C09/CFL_C09T gesetzt werden.

**GEMATCHT WIRD ÜBER DEN SCHLÜSSEL, NIE ÜBER DEN TEXT**

Naheliegend wäre ein Vergleich auf DECISION_TEXT. Er kann nicht funktionieren: zu diesem Zeitpunkt steht dort schon der ÜBERSETZTE Text aus /C09/CFL_C09T, nicht mehr der Rohschlüssel. DECISION_KEY dagegen ist NUMC4 mit derselben Nummerierung wie SWR_DECIKEY (0001 = OK, 0002 = NOK, 0003 = UC1 ...) und damit stabil.

**DECISION_TEXT NICHT ANFASSEN**

Das Framework prüft direkt NACH diesem Aufruf, ob der Text noch ein '-' enthält, und überspringt sonst die C09T-Übersetzung. Wer hier am Text schreibt, hat hinterher UNBESCHRIFTETE Buttons - und sucht den Fehler an der falschen Stelle.

## Der Code

```abap
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
```
