# `get_new_preview_descr`

| | |
|---|---|
| **Wann** | Wenn das SAP-GUI die Objektliste des Workitems aufbaut ("Objekte und Anlagen"). |
| **Rein** | IV_WI_ID            das Workitem |
| **Raus** | CT_PREVIEW_OBJECTS  die Objektliste, änderbar |

```
WOFÜR   Dasselbe wie GET_OBJECT_INFO, nur für die andere
       Oberfläche. Beide pflegen, sonst ist eine von beiden hässlich.
```

**DER CHECK AUF DEN OBJTYP IST NICHT OPTIONAL**

In der Liste stehen mehrere Einträge: das conFLOW- Instanzobjekt, Notizen (SOFM), Anlagen. Wer ohne CHECK durch die Tabelle läuft, beschriftet alles gleich - auch die Notizen, die dann ihren eigenen Namen verlieren.

**AUCH LÖSCHEN IST ERLAUBT**

Die Tabelle ist CHANGING. Ein DELETE nimmt einen Eintrag aus der Anzeige - der übliche Fall ist eine technische Notiz, die den Bearbeiter nichts angeht. Beispiel steht auskommentiert darunter, weil es einen Schlüssel braucht, den es nur im eigenen Projekt gibt.

## Der Code

```abap
    LOOP AT ct_preview_objects ASSIGNING FIELD-SYMBOL(<fs_object>).
      CHECK <fs_object>-objtype = '/C09/CFL_CL_WORKFLOW_0101'.
      <fs_object>-descript = text-001.
    ENDLOOP.

*   DELETE ct_preview_objects
*     WHERE objtype    = 'SOFM'
*       AND def_attrib = zcl_cfl_const_00900=>mc_def_attrib_intern.
```
