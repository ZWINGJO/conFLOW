# Anzeige

Was der Bearbeiter sieht, bevor er entscheidet

Fünf Hooks für vier Stellen am Bildschirm. Sie werden regelmäßig verwechselt, deshalb die Landkarte vorweg:

```
GET_DESCRIPTION        die EINE Zeile in der Trefferliste
                       (CHAR100 - mehr geht nicht)
GET_WORKITEM_TEXT      der BLOCK, den man nach dem Öffnen liest
GET_OBJECT_INFO        die Beschriftung des Objekt-Links in der
                       FIORI-Inbox
GET_NEW_PREVIEW_DESCR  dieselbe Beschriftung im SAP-GUI
```

GET_WF_DEFINITION_TEXT der Titel des Gesamtworkflows

GET_OBJECT_INFO und GET_NEW_PREVIEW_DESCR beschriften DASSELBE und werden trotzdem beide gebraucht: die Fiori-Inbox füllt ihren Reiter "Links" aus SAP_WAPI_GET_OBJECTS und sieht GET_NEW_PREVIEW_DESCR nie. Wer nur einen der beiden pflegt, hat in der anderen Oberfläche den Framework-Klassennamen stehen.

## Die Hooks

- [`get_description`](get-description.md) - Die eine Zeile in der Trefferliste (CHAR100)
- [`get_workitem_text`](get-workitem-text.md) - Der Textblock im geöffneten Workitem
- [`get_object_info`](get-object-info.md) - Beschriftung des Objekt-Links in **Fiori**
- [`get_new_preview_descr`](get-new-preview-descr.md) - Dieselbe Beschriftung im **SAP-GUI**
- [`get_wf_definition_text`](get-wf-definition-text.md) - Titel des Gesamtworkflows
- [`default_attribute_value`](default-attribute-value.md) - Das Standardattribut der Instanz - eine Zeile, immer dieselbe
- [`execute_default_method`](execute-default-method.md) - Doppelklick auf das Objekt zeigt den Beleg
