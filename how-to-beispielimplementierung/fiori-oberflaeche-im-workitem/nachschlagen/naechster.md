# Der nächste conFLOW-Workflow — vierzehn Schritte
*Der Faden, an dem man sich entlanghangelt. Die Verweise führen zum jeweiligen Kapitel.*
| 0 | **„Show Details" drücken und schauen, was der Standard schon kann** | [Was der Standard schon kann](../grundlagen/standard.md) |
| --- | --- | --- |
| 1 | **Vertragsmatrix ausfüllen** | [Vertragsmatrix](../grundlagen/vertrag.md) |
| 2 | Containerfelder und Konstanten festlegen | [Konstantenklasse](../backend/const.md) |
| 3 | Custom Entity auf die benötigten Felder zuschneiden | [Custom Entity](../backend/entity.md) |
| 4 | `read_one( )` auf die Workflowdaten abbilden | [Query-Provider](../backend/query.md) |
| 5 | `ZCL_CFL_NNNNN_RULES`: Schritte, Aktionen, Statustexte | [Regelklasse](../backend/rules.md) |
| 6 | Action-Parameter festlegen | [Behavior Definition](../backend/bdef.md) |
| 7 | Custom Section zuschneiden | [Das Fragment](../oberflaeche/fragment.md) |
| 8 | **Quellprojekt anlegen, bauen, deployen** | [Das Quellprojekt](../oberflaeche/quellprojekt.md) |
| 9 | Service Definition und Binding publizieren | [Service und manifest](../oberflaeche/service.md) |
| 10 | Semantic Object · Target Mapping · Rolle | [Launchpad](../anbindung/launchpad.md) |
| 11 | `set_inbox_ui( )` im BAdI anbinden | [Anbindung im BAdI](../anbindung/badi.md) |
| 12 | **Decision-Exit mit den Abschlussvalidierungen** | [Die drei Schranken](../anbindung/schranken.md) |
| 13 | Testmatrix ausführen | [Testmatrix](test.md) |

{% hint style="info" %}
**Die drei teuersten**

**Schritt 0, 1 und 12** überspringt man am ehesten — und sie fehlen am teuersten.

**Ohne Schritt 0** baut man nach, was es schon gibt. Im Referenzprojekt waren das fünf Objekte, zwei RAP-Aktionen und rund 430 Zeilen JavaScript, alle wieder entfernt. Er kostet einen Klick.

**Ohne Vertragsmatrix** baut man erst und merkt später, dass die Semantik nicht trägt. **Ohne Decision-Exit** gibt es keine verbindliche fachliche Prüfung vor dem Zustandsübergang — nur Hinweise, die man wegklicken kann.

**Schritt 8 wird nicht übersprungen, sondern aufgeschoben** — „erst zum Laufen bringen, dann aufräumen". Er ist der billigste von allen, solange die App aus drei Dateien besteht, und der teuerste, sobald jemand die Frage stellt, welcher Stand eigentlich läuft.
{% endhint %}

## Was dieses Muster nicht abdeckt
| Nicht abgedeckt | Warum, und was stattdessen |
| --- | --- |
| Listen mit vielen Instanzen | Der Query-Provider liest je Instanz rund zwanzig Container-Werte einzeln. Für eine Detailsicht auf ein Workitem belanglos, für eine Liste nicht. |
| Offline-Fähigkeit | Jeder Wert kommt aus einem Live-Zugriff auf den conFLOW-Container. |
| **Anhänge in der eigenen App** | **Braucht man nicht** — siehe [Kapitel „Was der Standard schon kann"](../grundlagen/standard.md). Wer es trotzdem tut, braucht zusätzlich die Virenscan-Konfiguration und hat zwei Listen derselben Daten. |
| Mehrere Aktionen pro Schritt über die App | Die Entscheidung gehört den conFLOW-Buttons. Die App liefert Kontext und Begründung — sie konkurriert nicht um den Zustandsübergang. |
| Mandantenübergreifende Szenarien | Der Intent trägt keinen Mandanten; Target Mapping und Katalog hängen am Frontend-Server. |

{% hint style="success" %}
**Wenn du nur einen Satz aus diesem Dokument mitnimmst:** Eine eigene Oberfläche im Workitem ersetzt den *Informationsbereich* — nicht das Workitem. Alles, was das Workitem selbst ausmacht (Anhänge, Notizen, Objektlinks, Entscheidungsbuttons, Protokoll), bleibt beim Framework und ist besser dort aufgehoben.
{% endhint %}
