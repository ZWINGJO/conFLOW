# `release`

> **Dieser Hook bleibt in der Referenzklasse leer.** Warum, steht unten.

| | |
|---|---|
| **Wann** | Wenn die BAdI-Instanz freigegeben wird - am Ende der Verarbeitung. |

```
WOFÜR   Aufräumen. Und zwar genau das, was man in
       GET_OBJECT_INFO oder GET_AFTER_CREATION_WORKITEM ANGELEGT
       hat.
```

**DER ZUSAMMENHANG, DEN MAN SONST ÜBERSIEHT**

Wer im SAP-GUI eine eigene Anzeige an das Workitem hängt (ein Docking-Control mit Belegdetails, das übliche Muster), legt dafür in GET_OBJECT_INFO eine Singleton-Instanz an. Die lebt dann länger als das Workitem.

Ohne das Gegenstück hier sieht der Bearbeiter beim ZWEITEN geöffneten Workitem die Daten des ERSTEN. Kein Fehler, kein Dump - nur falsche Zahlen. In beiden untersuchten Implementierungen mit Docking-Control steht deshalb hier genau eine Zeile: DEL_INSTANCE( ).

Merksatz: RELEASE ist leer, ODER es ist das Gegenstück zu etwas, das man selbst angelegt hat. Ein dritter Fall kommt nicht vor.

BLEIBT HIER LEER, weil dieses Beispiel keine eigene SAP-GUI-Anzeige mitbringt.

## Der Code

```abap
*   zcl_cfl_workflow_00900_doc=>del_instance( ).
```
