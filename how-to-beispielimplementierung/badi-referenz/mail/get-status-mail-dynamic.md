# `get_status_mail_dynamic`

> **Dieser Hook bleibt in der Referenzklasse leer.** Warum, steht unten.

| | |
|---|---|
| **Wann** | Beim Mailversand, wenn feststeht, WELCHER Bearbeiterkreis angeschrieben wird. |
| **Rein** | IS_DATA - die Instanz |
| **Rein und raus** | CS_CFL_C05 (der vorgesehene Kreis) und CT_CFL_C05 (die Liste der Kreise) - hier kann man aus einem mehrere machen oder ihn austauschen |

```
WOFÜR   Das Gegenstück zu GET_STATUS_DYNAMIC, aber für den
       Mailversand: "Wer eine Mail bekommt, hängt davon ab, was
       gerade passiert ist."
```

**DIE ABFRAGE AUF HINTERGRUNDSCHRITTE IST DER TRICK**

`is_data-gen_stat+0(1) = 'B'` unterscheidet Dialog von Batch - das ist der Grund für die Namenskonvention, dass Hintergrundschritte mit B beginnen. Ohne diese Zeile verschickt man Benachrichtigungen zu Schritten, die niemand gesehen hat.

BLEIBT HIER LEER, weil der Beispielprozess einen festen Empfängerkreis je Schritt hat. Das Muster steht darunter.

## Der Code

```abap
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
```
