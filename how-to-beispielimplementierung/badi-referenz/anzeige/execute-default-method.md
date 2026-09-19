# `execute_default_method`

| | |
|---|---|
| **Wann** | Doppelklick auf das Objekt im SAP-GUI-Workitem. |
| **Rein** | nichts - die Instanz steht in /C09/CFL_CL_WORKFLOW_0101=>MS_INSTANCES-INSTANCE->MS_DATA |

```
WOFÜR   "Zeig mir den Beleg". Mit c06t-OBJTEXT und c08 TEMPLATE
       öffnet das Framework den Beleg selbst (Default-Methode des
       Template-Objekttyps) - dann leer lassen, sonst öffnet er
       zweimal. Ohne beides passiert beim Doppelklick nichts, und
       der Bearbeiter muss die Belegnummer abschreiben.
```

MS_INSTANCES ist klassenweit - bei mehreren Workitems in einer Sitzung nicht unbedingt die eigene Instanz.

**MIT ANZEIGE-TRANSAKTION, NICHT MIT ÄNDERUNGS-TRANSAKTION**

ME23N, nicht ME22N. Der Bearbeiter soll den Beleg SEHEN, während er entscheidet. Wer ihn hier ändern lässt, hat einen Beleg, der sich unter dem laufenden Workflow bewegt - und einen Audit Trail, der nicht mehr stimmt.

**AND SKIP FIRST SCREEN**

Spart den Einstiegsbildschirm. Bei den Enjoy-Transaktionen (ME23N, VA03) ist der Zusatz wirkungslos bis störend - dort genügt SET PARAMETER ID.

**WENN MEHRERE BELEGTYPEN AUF DERSELBEN KLASSE LAUFEN**

Über IS_DATA-TYPEID verzweigen. Bei einem eigenen BOR-Typ je Workflow (der Normalfall) entfällt das.

## Der Code

```abap
* Nur noetig ohne c06t-OBJTEXT und c08 TEMPLATE - sonst oeffnet das
* Framework den Beleg schon, und diese Methode bleibt leer.
    DATA lv_ebeln TYPE ekko-ebeln.

    lv_ebeln = /c09/cfl_cl_workflow_0101=>ms_instances-instance->ms_data-instid.

    SET PARAMETER ID 'BES' FIELD lv_ebeln.
    CALL TRANSACTION 'ME23N'.
```
