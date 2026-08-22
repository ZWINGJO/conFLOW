# conFLOW - SAP Workflows easy built

{% hint style="success" %}
**conFLOW** bildet Genehmigungs- und Freigabe-Workflows im SAP-System ab -- ohne SAP-Workflow-Know-how, ohne Workflow Builder, ohne WF-Customizing im SPRO.

Statt eines klassischen SAP-Workflows mit Aufgaben, Regeln und Schrittgruppen definiert der Anwender den Prozess in **Customizing-Tabellen** und implementiert die fachliche Logik in **einer einzigen BAdI-Klasse**. Das Framework uebernimmt den Rest: Workitem-Erzeugung, Bearbeiterfindung, Fristen, Eskalation, Mailversand und Statusverfolgung.
{% endhint %}

<figure><img src=".gitbook/assets/conflow.png" alt="conFLOW Uebersicht"><figcaption><p>conFLOW - Workflows auf einfache Weise abbilden</p></figcaption></figure>

## Was conFLOW kann

| Merkmal | Beschreibung |
| --- | --- |
| **Customizing statt Entwicklung** | Schritte, Uebergaenge, Bearbeiter, Fristen und Mailversand werden in Tabellen gepflegt -- kein Workflow Builder, keine Aufgabendefinitionen |
| **Eine BAdI-Klasse je Workflow** | Die gesamte fachliche Logik liegt in einer Klasse mit definierten Hooks -- Bearbeiterfindung, Beschreibung, Absprung, Nachlauf |
| **Beliebig viele Workflows** | Jede Workflow-Definition hat eine eigene Nummer und eine eigene BAdI-Implementierung. Neue Prozesse benoetigen keinen neuen Transport des Frameworks |
| **Hintergrundschritte** | Automatische Verarbeitung zwischen den Entscheidungen -- Anreicherung, Bewertung, Statusaenderung, Belegbuchung |
| **Fristen und Eskalation** | Zeitgesteuerte Weiterleitung ueber Customizing, keine Deadline-Agents |
| **Parallele Genehmigung** | Mehrere Bearbeiter auf demselben Schritt -- das Ergebnis wird zusammengefuehrt |
| **SAP-GUI und Fiori** | Workitems erscheinen im Business Workplace (SBWP) und in der Fiori My Inbox |
| **conMOBILE-Integration** | Mobile Darstellung jedes Workflow-Schritts ueber die conMOBILE-Plattform |

## Ein Workflow in einem Tag

Einen Standard-SAP-Workflow produktiv zu setzen dauert typischerweise zehn Tage: Aufgabendefinitionen, Regelaufloesung, Schrittgruppen, Container-Operationen, Binding-Definitionen, Agenten-Ermittlung. Mit conFLOW reduziert sich das auf Customizing und eine ABAP-Klasse -- der erste lauffaehige Workflow steht am selben Tag.

<figure><img src=".gitbook/assets/conFLOW_DE.png" alt="conFLOW Architektur"><figcaption><p>Architektur und Einordnung im SAP-System</p></figcaption></figure>

## Einstieg

{% content-ref url="workflow-stories/beispiele-realisierter-user-stories/" %}
[Beispiele realisierter Workflows](workflow-stories/beispiele-realisierter-user-stories/)
{% endcontent-ref %}

{% content-ref url="technische-dokumentation/technische-dokumentation.md" %}
[Technische Dokumentation](technische-dokumentation/technische-dokumentation.md)
{% endcontent-ref %}

{% content-ref url="how-to-beispielimplementierung/beispiel-workflow-krankmeldung/" %}
[How-To: Beispiel Workflow Krankmeldung](how-to-beispielimplementierung/beispiel-workflow-krankmeldung/)
{% endcontent-ref %}
