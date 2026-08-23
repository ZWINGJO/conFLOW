# Der nächste conFLOW-Workflow — dreizehn Schritte
*Der Faden, an dem man sich entlanghangelt. Die Verweise führen zum jeweiligen Kapitel.*
| 0 | **„Show Details" drücken und schauen, was der Standard schon kann** | [Was der Standard schon kann](#standard) |
| --- | --- | --- |
| 1 | **Vertragsmatrix ausfüllen** | [Vertragsmatrix](#vertrag) |
| 2 | Containerfelder und Konstanten festlegen | [Konstantenklasse](#const) |
| 3 | Custom Entity auf die benötigten Felder zuschneiden | [Custom Entity](#entity) |
| 4 | `read_one( )` auf die Workflowdaten abbilden | [Query-Provider](#query) |
| 5 | `ZCL_CFL_NNNNN_RULES`: Schritte, Aktionen, Statustexte | [Regelklasse](#rules) |
| 6 | Action-Parameter festlegen | [Behavior Definition](#bdef) |
| 7 | Custom Section zuschneiden | [Das Fragment](#fragment) |
| 8 | Service Definition und Binding publizieren | [Service und manifest](#service) |
| 9 | Semantic Object · Target Mapping · Rolle | [Launchpad](#launchpad) |
| 10 | `set_inbox_ui( )` im BAdI anbinden | [Anbindung im BAdI](#badi) |
| 11 | **Decision-Exit mit den Abschlussvalidierungen** | [Die drei Schranken](#schranken) |
| 12 | Testmatrix ausführen | [Testmatrix](#test) |

{% hint style="info" %}
**Die drei teuersten**

**Schritt 0, 1 und 11** überspringt man am ehesten — und sie fehlen am teuersten. **Ohne Schritt 0** baut man nach, was es schon gibt. Im Referenzprojekt waren das fünf Objekte, zwei RAP-Aktionen und rund 430 Zeilen JavaScript, alle wieder entfernt. Er kostet einen Klick. **Ohne Vertragsmatrix** baut man erst und merkt später, dass die Semantik nicht trägt. **Ohne Decision-Exit** gibt es keine verbindliche fachliche Prüfung vor dem Zustandsübergang — nur Hinweise, die man wegklicken kann.
{% endhint %}

## Was dieses Muster nicht abdeckt
| Nicht abgedeckt | Warum, und was stattdessen |
| --- | --- |
| Listen mit vielen Instanzen | Der Query-Provider liest je Instanz rund zwanzig Container-Werte einzeln. Für eine Detailsicht auf ein Workitem belanglos, für eine Liste nicht. |
| Offline-Fähigkeit | Jeder Wert kommt aus einem Live-Zugriff auf den conFLOW-Container. |
| **Anhänge in der eigenen App** | **Braucht man nicht** — siehe [Kapitel „Was der Standard schon kann"](#standard). Wer es trotzdem tut, braucht zusätzlich die Virenscan-Konfiguration und hat zwei Listen derselben Daten. |
| Mehrere Aktionen pro Schritt über die App | Die Entscheidung gehört den conFLOW-Buttons. Die App liefert Kontext und Begründung — sie konkurriert nicht um den Zustandsübergang. |
| Mandantenübergreifende Szenarien | Der Intent trägt keinen Mandanten; Target Mapping und Katalog hängen am Frontend-Server. |

{% hint style="success" %}
**Wenn du nur einen Satz aus diesem Dokument mitnimmst:** Eine eigene Oberfläche im Workitem ersetzt den *Informationsbereich* — nicht das Workitem. Alles, was das Workitem selbst ausmacht (Anhänge, Notizen, Objektlinks, Entscheidungsbuttons, Protokoll), bleibt beim Framework und ist besser dort aufgehoben.
{% endhint %}
