# How-To: Workflow Krankmeldung

Dieses How-To zeigt Schritt fuer Schritt, wie ein conFLOW-Workflow aufgebaut wird -- am Beispiel einer Krankmeldung. Am Ende steht ein lauffaehiger Workflow, der ein Workitem in der Inbox erzeugt und den Prozess dokumentiert.

## Der Prozess

| Schritt | Typ | Wer | Was passiert |
| --- | --- | --- | --- |
| `X0` | Start | System | Workflow wird gestartet |
| `01` | Dialog | Melder | Erfasst die Krankmeldung |
| `02` | Dialog | Personalabteilung | Bearbeitet und entscheidet |
| `03` | Hintergrund | System | Nachlaufverarbeitung |
| `X1` | Ende | System | Workflow abgeschlossen (genehmigt) |
| `X2` | Ende | System | Workflow abgeschlossen (abgelehnt) |

## Rollen

| Rolle | Bearbeiter-Key (`gen_stat_user`) | Beschreibung |
| --- | --- | --- |
| Melder | `WI` | Der Workflow-Initiator -- wird automatisch ermittelt |
| Personalabteilung | `01` | Fest zugeordneter Bearbeiter im Customizing |
| Hintergrund | `BU` | Technischer User `WF-BATCH` |

## Voraussetzungen

- Zugriff auf die Transaktion `/C09/CONFLOW_C` (conFLOW-Customizing)
- Entwicklungszugriff fuer die BAdI-Implementierung (SE80 oder ADT)
- Ein Transportauftrag

## Die vier Schritte

<figure><img src="../../.gitbook/assets/Folie5.png" alt="Uebersicht Krankmeldung"><figcaption><p>Workflow Krankmeldung: Prozessueberblick</p></figcaption></figure>

| Schritt | Inhalt | Ergebnis |
| --- | --- | --- |
| [Schritt 1: Customizing anlegen](schritt-1-customizing.md) | Workflow-Definition, Schritte, Bearbeiter, Laufweg | Lauffaehiger Workflow |
| [Schritt 2: Customizing erweitern](schritt-2-customizing-erweitern.md) | Hintergrundschritte, Texte, Buttons, Mailversand | Vollstaendig konfigurierter Prozess |
| [Schritt 3: BAdI-Implementierung](schritt-3-badi-implementierung.md) | Dynamische Bearbeiterfindung, parallele Schritte, Absprung | Fachliche Logik |
| [Schritt 4: Workflow starten und testen](schritt-4-workflow-starten.md) | Trigger, Container, Test | Produktionsreifer Workflow |

{% hint style="warning" %}
**Reihenfolge beachten:** Zuerst das Customizing (Schritt 1 und 2), dann die BAdI-Implementierung (Schritt 3), zuletzt den Trigger (Schritt 4). Die BAdI-Klasse braucht die Customizing-Eintraege als Grundlage, und der Trigger setzt einen funktionierenden Workflow voraus.
{% endhint %}
