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
| Warum keine **persistente** CDS View Entity? | Die fachlichen Daten liegen in keiner modellierten Anwendungstabelle, sondern werden über die Container-API des Produkts gelesen. Es gibt nichts, worüber eine View entstehen könnte. *(Die Custom Entity ist selbst ein CDS-Objekt — sie hat nur keine Datenbanksicht unter sich.)* |
| Warum `unmanaged`? | Ein `managed`-Szenario müsste die Persistenz selbst übernehmen; für diese generische Containerablage kann es das nicht. Lesen und Schreiben müssen kontrolliert über die conFLOW-APIs laufen. *Merksatz: **RAP kann nichts verwalten, was ihm nicht gehört.*** |
| Warum kein Draft? | Für *diese* Custom Entity steht kein geeigneter Draft- oder Sticky-Session-Mechanismus zur Verfügung — beide setzen eine Ablage voraus, die es hier nicht gibt. Deshalb wird die Änderung über eine parametrisierte Aktion unmittelbar gespeichert. |
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

## Der Schlüssel aus dem Intent ist nicht vertrauenswürdig
Die Anwendung bekommt ihre Workflow-GUID als **Parameter aus der URL**. Jeder angemeldete Benutzer kann diesen Parameter ändern oder den OData-Service ohne die Oberfläche aufrufen — **Feature Control sieht nur, wer durch die App geht.** Der Dienst muss die Frage deshalb selbst beantworten, und zwar bei jedem Zugriff:
| Prüfung | wo sie hingehört |
| --- | --- |
| Darf dieser Benutzer **lesen**? | Query-Provider |
| Darf dieser Benutzer **schreiben**? | jede Aktion im Behavior Pool — *nicht* nur die Oberfläche |
| Gibt es das Workitem überhaupt (noch)? | Query-Provider |
| Ist es noch offen, also nicht abgeschlossen? | Query-Provider und Aktion |
| Ist der Benutzer **aktueller Bearbeiter**? | beide — die Auskunft steht in `/c09/cfl_s03` |
| Passt die Workflow-Instanz zum Workitem? | beide |
{% hint style="danger" %}
**Im Referenzprojekt ist davon nichts implementiert** — bewusst, dokumentiert, und für einen Showcase auf Demo-Daten vertretbar. **Für einen produktiven Nachbau nicht:** eine Workflow-ID ist kein Geheimnis, sie steht in URLs, Protokollen und Mails. Die Einzelheiten stehen im [Kapitel „Launchpad"](../anbindung/launchpad.md), Abschnitt „Berechtigung — Teil zwei", mit Behebungsweg und Probe.
{% endhint %}
{% hint style="info" %}
**Warum das hier steht und nicht erst in der Testmatrix:** es ist keine Testfrage, sondern eine Architekturentscheidung. Wer es erst beim Testen liest, hat den Query-Provider und den Behavior Pool schon geschrieben.
{% endhint %}
