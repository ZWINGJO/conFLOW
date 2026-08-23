# Randbedingungen — vor jeder Bewertung lesen
*Vier der fünf üblichen Rückfragen an eine RAP-Anwendung sind hier durch Vorgaben beantwortet, die nicht zur Disposition stehen.*

## Die Daten liegen in keiner eigenen Tabelle
Der fachliche Zustand eines conFLOW-Workflows steht im **Framework-Container** `/c09/cfl_s04` — einer generischen Name-Wert-Tabelle des Produkts, Schlüssel ist die Workflow-Instanz-GUID.
| Datenart | Wo | Zugriff |
| --- | --- | --- |
| fachliche Containerdaten | `/c09/cfl_s04` | ausschließlich `get_attribut_value` / `set_attribut_value` — im ganzen Projekt **kein einziger SELECT** |
| technische Metadaten | `/c09/cfl_s03` + `SWWWIHEAD` | ein lesender `SELECT` — eine geeignete API ist nicht bekannt |
| schreibend auf `/C09/` | — | nirgends |

## Was daraus folgt
| Übliche Rückfrage | Antwort |
| --- | --- |
| Warum kein CDS-View? | Es gibt keine Tabelle. |
| Warum `unmanaged`? | RAP kann nichts verwalten, was ihm nicht gehört. |
| Warum kein Draft? | Draft setzt Persistenz voraus, die die Custom Entity nicht hat. |
| Warum eine Aktion statt `update`? | Feld-Editieren in Fiori Elements V4 braucht Draft oder Sticky. Ohne beides ist die parametrisierte Aktion der Standardweg. |
| Warum kein Speichern-Knopf? | Der Entscheiden-Knopf liegt außerhalb der App. |

## Die Buttons gehören nicht der App
Entschieden wird über die conFLOW-Buttons in der Fußleiste. Die laufen über den **Task-Gateway** in die Framework-Logik und **nicht** durch diese Anwendung. Die App kann sie weder abfangen noch um einen ungespeicherten Bildschirmzustand bitten.
{% hint style="info" %}
**Daraus folgt die wichtigste Entwurfsentscheidung: es gibt keinen Speichern-Knopf.** Gesichert wird, sobald eine Auswahl getroffen ist und sobald die Notiz stehenbleibt. Ein Knopf, den man vergessen kann, während der eigentliche Entscheiden-Knopf woanders sitzt, wäre eine Falle.
{% endhint %}

## Der Container ist nicht allein der Audit Trail
|  | enthält |
| --- | --- |
| Container `/c09/cfl_s04` | die einzige fachliche Zustandsquelle: Systemempfehlung, Arbeitswert, Notiz, Kontext |
| Workitem-Protokoll `/c09/cfl_s03` | den Bestätigungsakt: wer, wann, welchen Workflow-Ausgang |
Zusammen ergeben sie den Audit Trail. **Eine zusätzliche Z-Persistenz wäre eine zweite Wahrheit** — der schwerste denkbare Fehler in diesem Umfeld, weil sie genau dann auseinanderläuft, wenn es darauf ankommt.
