# Die Landkarte

Alle 26 Hooks des Interface, in der Reihenfolge, in der sie im
Ablauf vorkommen. Die Spalte **Code** sagt, ob die Referenzklasse
sie füllt - sechs bleiben absichtlich leer.

## Start

| Hook | Code | Wozu |
|---|---|---|
| [`get_wi_create_swe2`](start/get-wi-create-swe2.md) | ja | Startet der Workflow überhaupt? Die Veto-Stelle |
| [`get_actors`](start/get-actors.md) | ja | Wer bekommt das Workitem |
| [`get_number_actors_rel`](start/get-number-actors-rel.md) | ja | Parallele Schritte, und der Merker für den Mailversand |

## Anzeige

| Hook | Code | Wozu |
|---|---|---|
| [`get_description`](anzeige/get-description.md) | ja | Die eine Zeile in der Trefferliste (CHAR100) |
| [`get_workitem_text`](anzeige/get-workitem-text.md) | ja | Der Textblock im geöffneten Workitem |
| [`get_object_info`](anzeige/get-object-info.md) | ja | Beschriftung des Objekt-Links in **Fiori** |
| [`get_new_preview_descr`](anzeige/get-new-preview-descr.md) | ja | Dieselbe Beschriftung im **SAP-GUI** |
| [`get_wf_definition_text`](anzeige/get-wf-definition-text.md) | leer | Titel des Gesamtworkflows |
| [`default_attribute_value`](anzeige/default-attribute-value.md) | ja | Das Standardattribut der Instanz - eine Zeile, immer dieselbe |
| [`execute_default_method`](anzeige/execute-default-method.md) | ja | Doppelklick auf das Objekt zeigt den Beleg |

## Workitem-Lebenszyklus

| Hook | Code | Wozu |
|---|---|---|
| [`get_after_creation_workitem`](lebenszyklus/get-after-creation-workitem.md) | ja | Priorität, Anlagen, eigene Oberfläche anhängen |
| [`get_before_execution_workitem`](lebenszyklus/get-before-execution-workitem.md) | leer | Vorbereitung beim Öffnen - kann nichts verhindern |
| [`get_before_decision_workitem`](lebenszyklus/get-before-decision-workitem.md) | ja | Buttons wegnehmen und einfärben |
| [`get_fiori_task_dec_op_act`](lebenszyklus/get-fiori-task-dec-op-act.md) | ja | Dasselbe für die Fiori-Inbox - andere Tabelle |
| [`get_after_execution`](lebenszyklus/get-after-execution.md) | ja | Nachlauf nach der Entscheidung, Abbruch ohne Meldung |
| [`get_after_execution_mobile`](lebenszyklus/get-after-execution-mobile.md) | ja | Abbruch **mit** Meldung - die einzige echte Schranke |
| [`get_after_execution_workitem`](lebenszyklus/get-after-execution-workitem.md) | ja | Nach dem Abschluss: parallele Workitems aufräumen |
| [`get_event_raised_workitem`](lebenszyklus/get-event-raised-workitem.md) | leer | Ereignis auf ein laufendes Workitem |

## Die Weiche

| Hook | Code | Wozu |
|---|---|---|
| [`get_status_dynamic`](weiche/get-status-dynamic.md) | leer | Folgestatus zur Laufzeit statt aus dem Customizing |

## Mail

| Hook | Code | Wozu |
|---|---|---|
| [`get_send_mail_user`](mail/get-send-mail-user.md) | leer | Empfänger ergänzen oder entfernen |
| [`get_mail_language`](mail/get-mail-language.md) | leer | Sprache des Mails |
| [`get_status_mail_dynamic`](mail/get-status-mail-dynamic.md) | leer | Welcher Bearbeiterkreis angeschrieben wird |
| [`get_datasource_mail`](mail/get-datasource-mail.md) | ja | Die Daten für den Mailtext und den Workitem-Text |
| [`get_add_attachments`](mail/get-add-attachments.md) | ja | Anhänge - Archiv, Belegdateien, SAP-Shortcut |

## Rahmen

| Hook | Code | Wozu |
|---|---|---|
| [`get_factory_calendar`](rahmen/get-factory-calendar.md) | leer | Fabrikkalender für die Fristenrechnung |
| [`release`](rahmen/release.md) | leer | Aufräumen, was man selbst angelegt hat |

## Die sechs leeren

Sie sind nicht vergessen. Bei jedem steht im Kapitel, wofür er
gedacht ist und woran man merkt, dass der eigene Fall dazugehört:

- `get_before_execution_workitem` - kann nichts verhindern
- `get_event_raised_workitem` - braucht ein "während der Workflow läuft"
- `get_status_dynamic` - macht die Verzweigung im Customizing unsichtbar
- `get_send_mail_user` - der Empfängerkreis gehört ins Customizing
- `get_mail_language` - der Standard tut schon das Richtige
- `get_factory_calendar` - erst nötig, wenn Fristen in Tagen laufen

Dazu `get_wf_definition_text` und `release`, die in diesem Beispiel
leer bleiben, aber häufiger gebraucht werden als die sechs oben.
