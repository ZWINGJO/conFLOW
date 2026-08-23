# `default_attribute_value`

| | |
|---|---|
| **Wann** | Wenn das Workflow-Framework nach dem Standardattribut der Instanz fragt - beim Aufbau von Objektlisten, Anlagen und überall dort, wo ein Objekt "sich selbst benennen" soll. |
| **Raus** | RESULT  eine DATENREFERENZ auf den Wert |

**DIE EINE ZEILE, DIE IN JEDER IMPLEMENTIERUNG GLEICH IST**

Von allen 26 Hooks ist dies der einzige, der in jeder untersuchten Produktivimplementierung gefüllt war - und zwar jedesmal mit exakt derselben Zeile. Wer sie übernimmt, hat sie richtig.

**WARUM GET REFERENCE OF UND NICHT EINE ZUWEISUNG**

```abap
       RESULT ist REF TO DATA. Das Framework dereferenziert
       spaeter. Eine lokale Variable waere zu diesem Zeitpunkt
       laengst weg - deshalb die Referenz auf das Instanzdatum des
       Frameworks, das die ganze Zeit lebt.
```

**FALLE** Wer eine Referenz auf eine METHODENLOKALE Variable zurückgibt, bekommt keinen Fehler, sondern später Datensalat. Immer auf MS_INSTANCES-INSTANCE->MS_DATA referenzieren.

## Der Code

```abap
GET REFERENCE OF /c09/cfl_cl_workflow_0101=>ms_instances-instance->ms_data-instid INTO result.
```
