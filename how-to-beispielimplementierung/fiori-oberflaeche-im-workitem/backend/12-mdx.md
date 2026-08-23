# Die Darstellung — Metadata Extension
*Die Aufteilung in Kopfbereich und Reiter kommt vollständig aus Annotationen; die App enthält dafür keine Zeile Code.*

**`ZCFL_00500_X_EXCEPTION`**

```abap
@Metadata.layer: #CORE

// Die Darstellung in der Fiori My Inbox.
//
// Aufbau wie im Workitem-Text: Befund im Kopf, darunter Order,
// Promise, Exception, Customer Rules - in unveraenderter Reihenfolge.
// Die Entscheidung kommt als fuenfter Block ans Ende, weil sie das
// Einzige ist, was der Bearbeiter anfasst.
//
// ACHTUNG @UI.facet: In diesem System hat die Annotation den Scope
// #ELEMENT, nicht #ENTITY - sie haengt deshalb am Schluesselfeld und
// nicht vor der Entity. Steht sie vor der Entity, meldet die
// Aktivierung "used at wrong position (wrong scope)" fuer jedes
// einzelne Unterattribut, was wie ein Syntaxfehler aussieht, aber
// nur die falsche Ebene ist.
@UI.headerInfo: { typeName: 'Order Promise Exception',
                  typeNamePlural: 'Order Promise Exceptions',
                  title: { value: 'Headline' },
                  description: { value: 'RecommendationReason' } }
annotate entity ZCFL_00500_C_EXCEPTION with
{
  @UI.facet: [ { id: 'Finding',   purpose: #HEADER,   type: #FIELDGROUP_REFERENCE,
                 targetQualifier: 'Finding',   position: 10 },

               { id: 'Order',     purpose: #STANDARD, type: #FIELDGROUP_REFERENCE,
                 label: 'Order',          targetQualifier: 'Order',     position: 10 },
               { id: 'Promise',   purpose: #STANDARD, type: #FIELDGROUP_REFERENCE,
                 label: 'Promise',        targetQualifier: 'Promise',   position: 20 },
               { id: 'Exception', purpose: #STANDARD, type: #FIELDGROUP_REFERENCE,
                 label: 'Exception',      targetQualifier: 'Exception', position: 30 },
               // "Your decision" fehlt hier bewusst: diesen Bereich
               // rendert die Custom Section der App, weil ein Facet nur
               // anzeigen, aber keine Eingabe entgegennehmen kann.
               { id: 'Rules',     purpose: #STANDARD, type: #FIELDGROUP_REFERENCE,
                 label: 'Customer rules', targetQualifier: 'Rules',     position: 40 },

               // Die Einteilungen als TABELLE. Anderer Facet-Typ:
               // #LINEITEM_REFERENCE statt #FIELDGROUP_REFERENCE, und
               // targetElement statt targetQualifier - er zeigt auf die
               // Association, nicht auf eine Feldgruppe. Die Spalten
               // stehen als @UI.lineItem in ZCFL_00500_X_SCHEDLINE.
               { id: 'SchedLine', purpose: #STANDARD, type: #LINEITEM_REFERENCE,
                 label: 'Schedule lines', targetElement: '_SchedLine',  position: 50 } ]
               // KEIN Facet fuer die Dokumente.
               //
               // Eine Facet-Tabelle kann anzeigen, aber keine Knoepfe je
               // Zeile tragen, die etwas OEFFNEN. Ein DataFieldForAction
               // koennte loeschen, aber kein Dokument an den Browser
               // geben - dafuer braucht es den Controller.
               //
               // Die Liste steht deshalb in der Custom Section, gebaut
               // aus demselben Entity Set. Das Muster der Einteilungen
               // (Facet) und das der Dokumente (Custom Section) stehen
               // damit nebeneinander im selben Projekt - der Unterschied
               // ist genau die Frage, ob man in den Zeilen handeln will.
  //--------------------------------------------------------------------
  // KEIN @UI.identification mit #FOR_ACTION - und ueberhaupt kein Knopf.
  //
  // Ein Aktionsknopf von Fiori Elements oeffnet zwingend einen
  // Parameterdialog; das laesst sich nicht abschalten, und die Felder
  // stehen ja schon auf der Seite. Ein eigener Knopf in der
  // manifest.json war der Zwischenstand - auch der ist entfallen.
  //
  // Gesichert wird jetzt ohne Knopf: die Custom Section ruft die
  // RAP-Aktion, sobald eine Auswahl getroffen ist und sobald die Notiz
  // stehenbleibt (siehe ext/DecisionSection.controller.js). Der
  // Container ist der Audit Trail - ein Knopf, den man vergessen kann,
  // ist dort die schlechtere Loesung. Die Aktion selbst, ihre
  // Parameter, die Validierung und die Feature Control sind davon
  // unberuehrt.
  //--------------------------------------------------------------------

  // Das Target Mapping benennt CFLQueryObject00 auf WfId um; damit
  // Fiori Elements den Startup-Parameter als Filter anwendet, muss
  // das Feld ein selectionField sein.
  @UI.selectionField: [{ position: 10 }]
  @UI.hidden: true
  WfId;

  // Steuert die Verfuegbarkeit der Aktion - siehe Behavior-Handler.
  // Am Bildschirm hat er nichts verloren.
  @UI.hidden: true
  GenStat;

  @UI.hidden: true
  Headline;

  // Die Ampel: Fiori Elements faerbt den Wert nach SeverityCriticality
  @UI.fieldGroup: [{ qualifier: 'Finding', position: 10, criticality: 'SeverityCriticality' }]
  @UI.lineItem: [{ position: 10, criticality: 'SeverityCriticality' }]
  Severity;

  @UI.hidden: true
  SeverityCriticality;

  @UI.fieldGroup: [{ qualifier: 'Finding', position: 20 }]
  @UI.lineItem: [{ position: 20 }]
  Recommendation;

  @UI.fieldGroup: [{ qualifier: 'Finding', position: 30 }]
  RecommendationReason;

  @UI.fieldGroup: [{ qualifier: 'Order', position: 10 }]
  @UI.lineItem: [{ position: 30 }]
  SalesOrder;

  @UI.fieldGroup: [{ qualifier: 'Order', position: 20 }]
  @UI.lineItem: [{ position: 40 }]
  SalesOrderItem;

  @UI.fieldGroup: [{ qualifier: 'Order', position: 30 }]
  Customer;

  @UI.fieldGroup: [{ qualifier: 'Order', position: 40 }]
  @UI.lineItem: [{ position: 50 }]
  CustomerName;

  @UI.fieldGroup: [{ qualifier: 'Order', position: 50 }]
  Material;

  @UI.fieldGroup: [{ qualifier: 'Promise', position: 10 }]
  RequestedQuantity;

  @UI.fieldGroup: [{ qualifier: 'Promise', position: 20 }]
  RequestedDate;

  @UI.fieldGroup: [{ qualifier: 'Promise', position: 30 }]
  ConfirmedQuantity;

  @UI.fieldGroup: [{ qualifier: 'Promise', position: 40 }]
  ConfirmedDate;

  @UI.hidden: true
  Unit;

  @UI.fieldGroup: [{ qualifier: 'Exception', position: 10 }]
  BackorderQuantity;

  @UI.fieldGroup: [{ qualifier: 'Exception', position: 20 }]
  @UI.lineItem: [{ position: 60 }]
  BackorderPercent;

  @UI.fieldGroup: [{ qualifier: 'Exception', position: 30 }]
  DelayText;

  @UI.fieldGroup: [{ qualifier: 'Exception', position: 40 }]
  Reason;

  @UI.fieldGroup: [{ qualifier: 'Rules', position: 10 }]
  Segment;

  @UI.fieldGroup: [{ qualifier: 'Rules', position: 20 }]
  PartialDeliveryText;

  // Die Governance-Regel ist die Pointe der Demo - sie darf rot sein
  @UI.fieldGroup: [{ qualifier: 'Rules', position: 30, criticality: 'CancelCriticality' }]
  CancelResidualText;

  @UI.hidden: true
  CancelCriticality;

  //--------------------------------------------------------------------
  // Der Block "Your decision" zeigt, was erfasst wurde - eingegeben
  // wird ueber den Dialog der Aktion. Beide Felder sind in der
  // Behavior Definition readonly; ein Feldedit gaebe es ohne Draft
  // oder Sticky Session ohnehin nicht.
  //--------------------------------------------------------------------
  @UI.fieldGroup: [{ qualifier: 'Decision', position: 10 }]
  ProposedAction;

  @UI.hidden: true
  ProposedActionText;

  //--------------------------------------------------------------------
  // Der Grund - die Eingabe der App.
  //
  // ProposedAction darueber ist reine Anzeige: die Aktion entscheidet
  // der conFLOW-Button. Bis zum 22.08.2026 bot das Dropdown dieselben
  // Aktionen an wie die Buttons darunter; wer beides unterschiedlich
  // bediente, bekam seine Auswahl stillschweigend ersetzt.
  //--------------------------------------------------------------------
  @UI.fieldGroup: [{ qualifier: 'Decision', position: 20 }]
  DecisionReason;

  @UI.hidden: true
  DecisionReasonText;

  @UI.multiLineText: true
  @UI.fieldGroup: [{ qualifier: 'Decision', position: 30 }]
  DecisionNote;

  // Die Association selbst bekommt keine Annotation - der Facet oben
  // verweist auf sie, die Spalten stehen in der Extension der
  // Kind-Entitaet.
}
```

## `@UI.facet` hat Scope `#ELEMENT`, nicht `#ENTITY`
In diesem System gehört die Annotation **ans Schlüsselfeld**, nicht vor die Entity. Steht sie davor, meldet die Aktivierung `used at wrong position (wrong scope)` — und zwar für *jedes* Unterattribut einzeln. Das sieht nach kaputter Syntax aus und ist doch nur die falsche Ebene.
{% hint style="info" %}
**Im Zielrelease nachsehen, nicht annehmen.** Der Scope steht im Vokabular selbst. Für ein HOW-TO ist das keine allgemeine Regel, sondern eine Prüfanweisung.
{% endhint %}

## Criticality — die Ampel
Kein Bild, sondern ein berechnetes Feld plus Annotation. Werte: `0` neutral, `1` rot, `2` gelb, `3` grün, `5` info. Berechnet wird sie im Query-Provider aus derselben Severity, die auch der Workflow für seine Verzweigung nutzt.

## Kein `DataFieldForAction`
{% hint style="danger" %}
Ein Aktionsknopf von Fiori Elements öffnet **zwingend** einen Parameterdialog; das lässt sich nicht abschalten. Hier stehen die Felder aber bereits auf der Seite — ein Dialog, der dieselben zwei Felder noch einmal abfragt, wäre absurd. Deshalb trägt die Metadata Extension keinen Aktionsknopf, und die Custom Section ruft die Aktion selbst.
{% endhint %}
