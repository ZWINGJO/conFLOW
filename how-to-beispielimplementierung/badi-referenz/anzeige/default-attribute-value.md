# `default_attribute_value`

| | |
|---|---|
| **Wann** | Wenn das Workflow-Framework nach dem Standardattribut der Instanz fragt - beim Aufbau von Objektlisten, Anlagen und überall dort, wo ein Objekt "sich selbst benennen" soll. |
| **Raus** | RESULT  eine DATENREFERENZ auf den Wert |

**DIE EINE ZEILE, DIE IN JEDER IMPLEMENTIERUNG GLEICH IST**

Von allen 26 Hooks ist dies der einzige, der in jeder untersuchten Produktivimplementierung gefüllt war - und zwar jedesmal mit exakt derselben Zeile.

MIT c06t-OBJTEXT: LEER LASSEN

Dann liefert das Framework den Belegschlüssel selbst, aus der richtigen Instanz, VOR diesem Hook. Deshalb setzt die Zeile unten RESULT nur, wenn es noch leer ist.

**WARUM GET REFERENCE OF UND NICHT EINE ZUWEISUNG**

```abap
       RESULT ist REF TO DATA. Das Framework dereferenziert
       spaeter. Eine lokale Variable waere zu diesem Zeitpunkt
       laengst weg - deshalb die Referenz auf das Instanzdatum des
       Frameworks, das die ganze Zeit lebt.
```

**FALLE** Wer eine Referenz auf eine METHODENLOKALE Variable zurückgibt, bekommt keinen Fehler, sondern später Datensalat. Deshalb die Referenz auf MS_INSTANCES.

**ACHTUNG MS_INSTANCES**

MS_INSTANCES ist klassenweit und wird bei jedem FIND_BY_LPOR überschrieben. Mehrere Workitems in einer Sitzung liefern darüber den Schlüssel des zuletzt erzeugten Objekts - ein Grund mehr für c06t-OBJTEXT.

## Der Code

```abap
IF result IS INITIAL.
  GET REFERENCE OF /c09/cfl_cl_workflow_0101=>ms_instances-instance->ms_data-instid INTO result.
ENDIF.
```
