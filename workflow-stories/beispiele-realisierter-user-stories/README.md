# Beispiele realisierter Workflows

Die folgenden Beispiele zeigen, welche Prozesse sich mit conFLOW abbilden lassen. Gemeinsam ist ihnen: der Workflow wurde in Customizing-Tabellen definiert und in einer einzigen BAdI-Klasse implementiert -- ohne Workflow Builder und ohne SAP-Workflow-Entwicklung.

Die Beispiele stammen aus verschiedenen SAP-Modulen und decken unterschiedliche Muster ab: einfache Zwei-Schritt-Genehmigungen, mehrstufige Freigaben mit Eskalation, Hintergrundschritte mit automatischer Verarbeitung und parallele Bearbeiterwege.

{% hint style="info" %}
**Jedes Beispiel zeigt zwei Sichten:** die fachliche User Story (was soll der Prozess leisten?) und den technischen Workflow (wie sieht der Laufweg in conFLOW aus?). Die Kombination aus beiden macht die Staerke von conFLOW sichtbar: ein fachlich verstaendlicher Prozess, der technisch einfach bleibt.
{% endhint %}

<figure><img src="../../.gitbook/assets/Folie2.png" alt="Uebersicht Workflow Stories"><figcaption><p>Uebersicht der realisierten Workflow-Szenarien</p></figcaption></figure>

| Modul | Workflow | Muster |
| --- | --- | --- |
| **HR** | [Krankmeldung](hr-krankmeldung.md) | Meldung durch Mitarbeiter, Bearbeitung durch Personalabteilung |
| **HR** | [Onboarding](hr-onboarding.md) | Mehrstufiger Einarbeitungsprozess mit mehreren Beteiligten |
| **SD** | [Faktura Anforderung](sd-faktura-anforderung.md) | Freigabe-Workflow im Vertrieb |
| **SD** | [Business Partner Synchronisierung](sd-business-partner-synchronisierung.md) | Automatisierter Abgleich mit Hintergrundschritten |
| **BC** | [Alert Monitor](bc-alert-monitor.md) | Systemueberwachung mit Eskalation |
| **BC** | [Stammdaten Verteilung](bc-stammdaten-verteilung.md) | Verteilung und Freigabe von Stammdaten ueber Systeme hinweg |
| **CA** | [Vergabevorschlag](ca-vergabevorschlag.md) | Mehrstufige Genehmigung im Einkauf |
