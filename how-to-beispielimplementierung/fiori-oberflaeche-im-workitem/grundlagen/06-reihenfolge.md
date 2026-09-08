# Reihenfolge der Objektanlage
*RAP prüft Abhängigkeiten beim Aktivieren. Die Reihenfolge ist nicht beliebig.*
| # | Objekt | Anmerkung |
| --- | --- | --- |
| 1 | `ZCL_CFL_NNNNN_CONST` | Elementnamen, Schrittcodes, **Notaus-Schalter** |
| 2 | `ZCFL_NNNNN_C_<Sache>` | Custom Entity — Schlüssel muss durch URL und Intent passen |
| 3 | `ZCL_CFL_NNNNN_QUERY` | Filter · Zähler · **Paging** · Daten |
| 4 | `ZCFL_NNNNN_C_<Sache>VH` | Werteliste, falls Auswahlfelder gebraucht werden |
| 5 | `ZCL_CFL_NNNNN_VH` | dasselbe Pflichtmuster wie 3 |
| 6 | `ZCFL_NNNNN_D_<Aktion>` | Abstract Entity — Parameter der Aktion |
| 7 | `ZCFL_NNNNN_C_<Sache>` (BDEF) | **unmanaged**, kein `update`, eine Aktion |
| 8 | `ZCL_CFL_NNNNN_BEHV` | global + Local Types — **gemeinsam mit 7 aktivieren** |
| 8b | `ZCL_CFL_NNNNN_RULES` | die Prozessregeln — von Query *und* Handler gerufen |
| 9 | `ZCFL_NNNNN_X_<Sache>` | Metadata Extension — **Annotation-Scope im Zielrelease prüfen** |
| 10 | `ZCFL_NNNNN_SD / _SB` | Service Definition + Binding, publizieren |
| 11 | `ZCFL_NNNNN_UI` | BSP **einmal** über den ADT-Generator anlegen — danach nur noch aus dem [Quellprojekt](../oberflaeche/20-quellprojekt.md) deployen |
| 12 | Semantic Object · Katalog · Target Mapping · Rolle | `/UI2/SEMOBJ`, `/UI2/FLPD_CUST`, PFCG |
| 13 | `set_inbox_ui( )` im BAdI-Hook | **zuletzt** — sonst zeigen Workitems auf eine App, die es nicht gibt |
{% hint style="success" %}
**Ab Schritt 10 ist testbar**, lange bevor eine Oberfläche existiert: `GET <service>/<EntitySet>?sap-client=100` im Browser. **Erst den Service beweisen, dann die App bauen** — die wichtigste Gewohnheit aus diesem Projekt.
{% endhint %}
