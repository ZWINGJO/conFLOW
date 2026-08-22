# Technische Dokumentation

Diese Dokumentation beschreibt die technische Architektur von conFLOW: das Datenmodell, die Customizing-Tabellen, die Laufzeit-Strukturen, das BAdI-Interface und die Integrationspunkte zum SAP Business Workflow.

{% hint style="info" %}
**Die vollständige Spezifikation steht als PDF zum Download bereit.** Die Folien darunter geben einen visuellen Überblick über die wichtigsten Konzepte.
{% endhint %}

{% file src="../.gitbook/assets/Concircle_Spezifikation_conFLOW_V1.1.pdf" %}

---

## Architektur und Überblick

conFLOW setzt auf dem SAP Business Workflow auf und steuert ihn über Customizing-Tabellen (`/C09/CFL_C*`) und eine BAdI-Schnittstelle (`/C09/CFL_IF_BADI_0101`). Die Laufzeit-Daten liegen in den Tabellen `/C09/CFL_S*`.

<figure><img src="../.gitbook/assets/Folie1 (1).png" alt="Architektur"><figcaption><p>Architektur-Überblick</p></figcaption></figure>

<figure><img src="../.gitbook/assets/Folie2 (1).png" alt="Komponenten"><figcaption><p>Komponenten und ihre Zusammenhänge</p></figcaption></figure>

## Customizing-Modell

Die Workflow-Definition, ihre Schritte, Übergänge, Bearbeiter und Fristen werden vollständig in Customizing-Tabellen gepflegt. Die Transaktion `/C09/CONFLOW_C` ist der zentrale Einstieg.

<figure><img src="../.gitbook/assets/Folie3 (1).png" alt="Customizing"><figcaption><p>Das Customizing-Modell im Überblick</p></figcaption></figure>

<figure><img src="../.gitbook/assets/Folie4 (1).png" alt="Customizing Detail"><figcaption><p>Customizing-Tabellen und ihre Beziehungen</p></figcaption></figure>

## Workflow-Definition und Schritte

Jeder Workflow hat eine eindeutige Nummer (z.B. `00208`) und wird in `/C09/CFL_C06` registriert. Die Schritte stehen in `/C09/CFL_C01`, die Übergänge in `/C09/CFL_C02`. Schritte können Dialogschritte (Workitem in der Inbox), Hintergrundschritte (automatische Verarbeitung) oder Endschritte sein.

<figure><img src="../.gitbook/assets/Folie5 (1).png" alt="Workflow-Definition"><figcaption><p>Workflow-Definition und Schritttypen</p></figcaption></figure>

<figure><img src="../.gitbook/assets/Folie6 (1).png" alt="Schritte"><figcaption><p>Schritte und Übergänge</p></figcaption></figure>

## Bearbeiterfindung und Rollen

Die Zuordnung von Schritten zu Bearbeitern erfolgt über Bearbeiter-Keys (`gen_stat_user`) in `/C09/CFL_C04` und `/C09/CFL_C05`. Die tatsächliche Auflösung -- welcher User das Workitem erhält -- kann fest im Customizing stehen oder dynamisch über die BAdI-Methode `GET_ACTORS` ermittelt werden.

<figure><img src="../.gitbook/assets/Folie7 (1).png" alt="Bearbeiterfindung"><figcaption><p>Bearbeiterfindung: statisch und dynamisch</p></figcaption></figure>

## Laufzeit-Datenmodell

Die Laufzeitdaten liegen in drei Tabellen: `/C09/CFL_S01` (eine Zeile je Workflow-Instanz), `/C09/CFL_S03` (eine Zeile je Workitem, chronologisch) und `/C09/CFL_S04` (Container-Elemente als Key-Value-Paare).

<figure><img src="../.gitbook/assets/Folie8.png" alt="Laufzeitdaten"><figcaption><p>Laufzeit-Tabellen: S01, S03, S04</p></figcaption></figure>

<figure><img src="../.gitbook/assets/Folie9 (1).png" alt="Laufzeit Detail"><figcaption><p>Zusammenhang zwischen Instanz, Workitems und Container</p></figcaption></figure>

## BAdI-Interface

Das Interface `/C09/CFL_IF_BADI_0101` definiert die Hooks, über die fachliche Logik eingehängt wird. Jeder Workflow implementiert das Interface in einer eigenen Klasse `ZCL_CFL_WORKFLOW_<nnnnn>`. Die wichtigsten Hooks: `GET_ACTORS` (Bearbeiterfindung), `GET_DESCRIPTION` / `GET_WORKITEM_TEXT` (Workitem-Beschreibung), `GET_BEFORE_DECISION_WORKITEM` (Buttons steuern), `EXECUTE_DEFAULT_METHOD` (Absprung ins Belegobjekt).

<figure><img src="../.gitbook/assets/Folie10 (2).png" alt="BAdI-Interface"><figcaption><p>BAdI-Hooks und ihre Aufgaben</p></figcaption></figure>

<figure><img src="../.gitbook/assets/Folie11 (2).png" alt="BAdI Detail"><figcaption><p>Hook-Katalog im Detail</p></figcaption></figure>

## Entscheidungsalternativen

Jeder Schritt bietet bis zu sieben Ausgänge: `OK` und `NOK` (gesteuert über `/C09/CFL_C02`) sowie `UC1` bis `UC5` (gesteuert über `/C09/CFL_C09`). Die Beschriftung der Buttons wird in den zugehörigen Texttabellen gepflegt.

<figure><img src="../.gitbook/assets/Folie12.png" alt="Entscheidungsalternativen"><figcaption><p>Entscheidungsalternativen und ihre Steuerung</p></figcaption></figure>

## Hintergrundschritte

Hintergrundschritte führen eine statische Methode automatisch aus -- ohne Workitem in einer Inbox. Die Methode erhält die aktuelle Workflow-Instanz und liefert einen Decision Key zurück, der den nächsten Schritt bestimmt. Damit lassen sich Weichen, Anreicherungen und automatische Buchungen im Prozess abbilden.

<figure><img src="../.gitbook/assets/Folie13 (1).png" alt="Hintergrundschritte"><figcaption><p>Hintergrundschritte: automatische Verarbeitung im Prozess</p></figcaption></figure>

## Fristen und Eskalation

Fristen werden im Customizing (`/C09/CFL_C02`) hinterlegt: Wert, Einheit und Folgestatus bei Fristablauf. Keine Deadline-Agents, kein WF-Customizing im SPRO -- alles in einer Tabellenzeile.

<figure><img src="../.gitbook/assets/Folie14 (2).png" alt="Fristen"><figcaption><p>Fristen und Eskalation über Customizing</p></figcaption></figure>

## Mailversand

conFLOW versendet E-Mails je Schritt, gesteuert über das Customizing. Die Inhalte kommen aus SO10-Texten, in denen Platzhalter dynamisch ersetzt werden. Die BAdI-Methode `GET_DATASOURCE_MAIL` liefert die Ersetzungswerte.

<figure><img src="../.gitbook/assets/Folie15 (1).png" alt="Mailversand"><figcaption><p>Mailversand mit SO10-Texten und Platzhaltern</p></figcaption></figure>

## Container und Datenübergabe

Der conFLOW-Container (`/C09/CFL_S04`) speichert beliebige Attribut-Wert-Paare je Workflow-Instanz. Geschrieben und gelesen wird über die Framework-Methoden `SET_ATTRIBUT_VALUE` und `GET_ATTRIBUT_VALUE`. Der Container ist auch der natürliche Audit Trail -- jeder Wert, der dort steht, ist später nachvollziehbar.

<figure><img src="../.gitbook/assets/Folie16.png" alt="Container"><figcaption><p>Container: Datenübergabe zwischen Schritten</p></figcaption></figure>

## Workflow starten

Es gibt mehrere Wege, einen conFLOW-Workflow zu starten: über einen Anwendungs-BAdI (z.B. nach dem Sichern eines Belegs), über die SAP-Statusverwaltung (Statuswechsel löst ein BOR-Ereignis aus) oder über einen Report. Entscheidend ist die **Typkoppelung** in `/C09/CFL_C10`, die Ereignis und Workflow-Definition verknüpft.

<figure><img src="../.gitbook/assets/Folie17 (1).png" alt="Workflow starten"><figcaption><p>Trigger-Varianten für den Workflow-Start</p></figcaption></figure>

## Integration SAP-GUI und Fiori

Workitems erscheinen sowohl im SAP Business Workplace (SBWP, Transaktion `SBWP`) als auch in der SAP Fiori My Inbox. conFLOW steuert in beiden Oberflächen die Buttonbeschriftung, den Kontextblock und den Absprung ins Belegobjekt über dieselben BAdI-Hooks.

<figure><img src="../.gitbook/assets/Folie18 (1).png" alt="SAP-GUI Integration"><figcaption><p>Integration in SAP-GUI und Fiori My Inbox</p></figcaption></figure>

## conMOBILE-Integration

Die Integration mit conMOBILE ermöglicht die mobile Darstellung jedes Workflow-Schritts. Die conMOBILE-App liest denselben conFLOW-Container und verwendet dieselben Decision-Keys -- eine Datenquelle, ein Entscheidungsmodell.

<figure><img src="../.gitbook/assets/Folie19 (1).png" alt="conMOBILE Integration"><figcaption><p>Mobile Darstellung über conMOBILE</p></figcaption></figure>

## Parallele Workflows und Synchronisierung

conFLOW unterstützt parallele Bearbeiterwege: mehrere Bearbeiter erhalten gleichzeitig ein Workitem, der Workflow wartet auf alle Entscheidungen, bevor er weitergeht. Die Konfiguration erfolgt über die Zuordnung mehrerer Bearbeiter-Keys zu einem Genehmigungsschritt.

<figure><img src="../.gitbook/assets/Folie20.png" alt="Parallele Workflows"><figcaption><p>Parallele Bearbeiterwege</p></figcaption></figure>

## Statusverwaltung und Audit Trail

Der Workflow-Status wird in `/C09/CFL_S01` (aktueller Schritt) und `/C09/CFL_S03` (Historie aller Workitems) geführt. Zusammen mit dem Container (`S04`) ergibt sich ein vollständiger Audit Trail ohne eigene Z-Tabelle: wer hat wann welchen Schritt mit welchem Ergebnis bearbeitet, und welche Daten lagen zum Zeitpunkt der Entscheidung vor.

<figure><img src="../.gitbook/assets/Folie21.png" alt="Statusverwaltung"><figcaption><p>Statusverwaltung und Audit Trail</p></figcaption></figure>

<figure><img src="../.gitbook/assets/Folie22.png" alt="Audit Trail Detail"><figcaption><p>Nachvollziehbarkeit über S01, S03 und S04</p></figcaption></figure>

## Erweiterte Szenarien

<figure><img src="../.gitbook/assets/Folie23.png" alt="Erweiterte Szenarien"><figcaption><p>Erweiterte Szenarien und Ausbaustufen</p></figcaption></figure>

<figure><img src="../.gitbook/assets/Folie24 (1).png" alt="Erweiterungen"><figcaption><p>Erweiterungsmöglichkeiten</p></figcaption></figure>

<figure><img src="../.gitbook/assets/Folie25 (1).png" alt="Zusammenfassung"><figcaption><p>Zusammenfassung der technischen Architektur</p></figcaption></figure>
