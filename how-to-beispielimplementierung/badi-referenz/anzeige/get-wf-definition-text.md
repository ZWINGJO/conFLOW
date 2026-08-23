# `get_wf_definition_text`

> **Dieser Hook bleibt in der Referenzklasse leer.** Warum, steht unten.

| | |
|---|---|
| **Wann** | Beim Aufbau des Titels für den GESAMTEN Workflow - nicht für einen einzelnen Schritt. |
| **Rein** | IS_DATA    die Instanz |
| **Raus** | CV_WITEXT  der Titel (SWW_WITEXT) |

BRAUCHT MAN IHN?

Meistens nicht. Der Standardtext kommt aus /C09/CFL_C06T und ist im Customizing pflegbar - ohne Transport, mit Übersetzung. Das ist der bessere Weg.

**WANN DOCH**

Wenn der Titel Belegdaten enthalten soll, die der Customizing-Text nicht kennt. "Bestellfreigabe" ist im Workflow-Protokoll neben zwanzig anderen nicht auffindbar, "Bestellfreigabe 4500001234 / Mustermann GmbH" schon.

Bleibt hier LEER, weil der Beispielprozess mit dem Customizing-Text auskommt - und weil ein Referenzbeispiel zeigen soll, dass man Hooks nicht füllt, nur weil sie da sind.
