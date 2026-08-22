# CA Vergabevorschlag

## Die Anforderung

Ein Vergabevorschlag im Einkauf durchläuft mehrere Genehmigungsstufen, bevor er zum Auftrag wird. Je nach Volumen, Warengruppe oder Lieferant sind unterschiedliche Freigeber zuständig. Der Prozess muss nachvollziehbar sein und Fristen einhalten.

## Was conFLOW hier leistet

Der Workflow bildet eine **mehrstufige Genehmigung** ab: der Vergabevorschlag geht zunächst an den Fachbereich, dann an den Einkaufsleiter, und bei Überschreitung einer Wertgrenze an die Geschäftsführung. Jede Stufe hat eigene Bearbeiter und kann eigene Entscheidungsalternativen anbieten -- neben Genehmigen und Ablehnen auch Rückfrage oder Weiterleitung.

Die Bearbeiterfindung kann dabei dynamisch erfolgen: die BAdI-Methode `GET_ACTORS` ermittelt den richtigen Freigeber zur Laufzeit, abhängig von Belegdaten wie Einkaufsorganisation, Warengruppe oder Bestellwert.

conFLOW bietet hier bis zu **sieben Entscheidungsalternativen** je Schritt (`OK`, `NOK`, `UC1` bis `UC5`). Damit lassen sich differenzierte Entscheidungen abbilden -- etwa "Genehmigen", "Ablehnen", "Genehmigen mit Auflage", "Zurück an Ersteller" oder "Weiterleiten an nächste Stufe".

### User Story

<figure><img src="../../.gitbook/assets/Folie15 (2).png" alt="User Story CA Vergabevorschlag"><figcaption><p>Fachliche User Story: Mehrstufige Genehmigung im Einkauf</p></figcaption></figure>

### Workflow

<figure><img src="../../.gitbook/assets/Folie16 (2).png" alt="Workflow CA Vergabevorschlag"><figcaption><p>Technischer Laufweg mit mehreren Genehmigungsstufen</p></figcaption></figure>
