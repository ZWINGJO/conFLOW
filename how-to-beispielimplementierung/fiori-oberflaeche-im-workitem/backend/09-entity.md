# Die Custom Entity
*Eine CDS-Hülle **ohne** `select from` — sie beschreibt nur die Form der Daten, geliefert werden sie von einer ABAP-Klasse.*

**`ZCFL_00500_C_EXCEPTION`**

```abap
@EndUserText.label: 'Order Promise Exception - Inbox'
@ObjectModel.query.implementedBy: 'ABAP:ZCL_CFL_00500_QUERY'
@Metadata.allowExtensions: true

// Der Datenschnitt. Wie er aussieht, steht in der Metadata Extension
// ZCFL_00500_X_EXCEPTION - UI-Annotationen sind auf einer Custom
// Entity nicht zugelassen ("wrong scope"), und getrennt ist es
// ohnehin sauberer: hier die Felder, dort die Darstellung.
//
// Die Reihenfolge folgt dem Workitem-Text: Befund, dann Order,
// Promise, Exception, Customer Rules.
define root custom entity ZCFL_00500_C_EXCEPTION
{
      // conFLOW-Instanz /c09/cfl_s01-id als Hex-Kette. Kommt ueber
      // CFLQueryObject00 aus get_after_creation_workitem herein.
  key WfId                 : abap.char(32);

      //----------------------------------------------------------------
      // Der conFLOW-Schritt, in dem die Instanz gerade steht - gelesen
      // aus dem offenen Dialog-Workitem. Nicht Deko: an ihm haengt die
      // Feature Control. Ob die Aktion SETDECISION angeboten wird und
      // welche Felder eingabebereit sind, entscheidet der Schritt -
      // nicht die App.
      //----------------------------------------------------------------
      GenStat              : abap.char(2);

      // Das offene Dialog-Workitem. Zur Zeit von niemandem gelesen -
      // die Anhangsverwaltung, fuer die es gedacht war, ist am
      // 23.08.2026 zurueckgebaut worden (Kapitel 11.4.21).
      //
      // Es bleibt trotzdem stehen: es kostet nichts, weil es aus
      // demselben SELECT kommt wie GEN_STAT, und es ist die einzige
      // Bruecke von der Workflow-Instanz zum Workitem - an dem
      // Anhaenge, Notizen und das Kundenschreiben aus B5 haengen.
      WorkitemId           : abap.numc(12);

      //------------------------------------------------------------------
      // Befund - steht im Kopf der Object Page
      //------------------------------------------------------------------
      Headline             : abap.char(120);

      @EndUserText.label: 'Severity'
      Severity             : abap.char(10);

      // 1 = rot, 2 = gelb, 3 = gruen. Fiori Elements malt daraus die Ampel.
      SeverityCriticality  : abap.int1;

      @EndUserText.label: 'Recommended action'
      Recommendation       : abap.char(40);

      @EndUserText.label: 'Reason'
      RecommendationReason : abap.char(200);

      //------------------------------------------------------------------
      // ORDER
      //------------------------------------------------------------------
      @EndUserText.label: 'Sales order'
      SalesOrder           : abap.char(10);

      @EndUserText.label: 'Item'
      SalesOrderItem       : abap.char(6);

      @EndUserText.label: 'Customer'
      Customer             : abap.char(10);

      @EndUserText.label: 'Customer name'
      CustomerName         : abap.char(80);

      @EndUserText.label: 'Material'
      Material             : abap.char(40);

      //------------------------------------------------------------------
      // PROMISE
      //------------------------------------------------------------------
      @EndUserText.label: 'Requested quantity'
      @Semantics.quantity.unitOfMeasure: 'Unit'
      RequestedQuantity    : abap.dec(15,3);

      @EndUserText.label: 'Requested date'
      RequestedDate        : abap.dats;

      @EndUserText.label: 'Confirmed quantity'
      @Semantics.quantity.unitOfMeasure: 'Unit'
      ConfirmedQuantity    : abap.dec(15,3);

      @EndUserText.label: 'Confirmed date'
      ConfirmedDate        : abap.dats;

      @Semantics.unitOfMeasure: true
      Unit                 : abap.unit(3);

      //------------------------------------------------------------------
      // EXCEPTION
      //------------------------------------------------------------------
      @EndUserText.label: 'Backorder quantity'
      @Semantics.quantity.unitOfMeasure: 'Unit'
      BackorderQuantity    : abap.dec(15,3);

      @EndUserText.label: 'Backorder share'
      BackorderPercent     : abap.dec(5,2);

      @EndUserText.label: 'Delay'
      DelayText            : abap.char(20);

      @EndUserText.label: 'Reason'
      Reason               : abap.char(60);

      //------------------------------------------------------------------
      // CUSTOMER RULES
      //------------------------------------------------------------------
      @EndUserText.label: 'Segment'
      Segment              : abap.char(40);

      // Klartext statt 'X' / ' ' - genau wie im Workitem-Text. Die
      // Criticality faerbt "not allowed - decision required" rot.
      @EndUserText.label: 'Partial delivery'
      PartialDeliveryText  : abap.char(40);

      @EndUserText.label: 'Cancel residual'
      CancelResidualText   : abap.char(40);

      CancelCriticality    : abap.int1;

      //------------------------------------------------------------------
      // Bearbeitung - das Einzige, was die App zurueckschreibt
      //------------------------------------------------------------------
      // Nur noch ANZEIGE. Gesetzt wird das Feld in B1 (Startwert =
      // Empfehlung) und beim Abschluss aus dem geklickten Button -
      // nicht mehr von der App. Deshalb auch keine Wertehilfe.
      @EndUserText.label: 'Action'
      @ObjectModel.text.element: ['ProposedActionText']
      ProposedAction       : abap.char(20);

      @Semantics.text: true
      ProposedActionText   : abap.char(40);

      //----------------------------------------------------------------
      // Der Entscheidungsgrund - die Eingabe der App.
      //
      // Die AKTION kommt vom conFLOW-Button (PROPOSED_ACTION), der
      // GRUND von hier. Zwei Fragen, zwei Felder: bis zum 22.08.2026
      // bot das Dropdown dieselben Aktionen an wie die Buttons
      // darunter, und wer beides unterschiedlich bediente, bekam seine
      // Auswahl stillschweigend ueberschrieben.
      //----------------------------------------------------------------
      @EndUserText.label: 'Reason for decision'
      @Consumption.valueHelpDefinition: [{ entity: { name: 'ZCFL_00500_C_ACTIONVH',
                                                     element: 'ActionKey' } }]
      @ObjectModel.text.element: ['DecisionReasonText']
      DecisionReason       : abap.char(20);

      @Semantics.text: true
      DecisionReasonText   : abap.char(40);

      @EndUserText.label: 'Note'
      DecisionNote         : abap.char(250);

      //------------------------------------------------------------------
      // Bearbeitungszustand - damit der Controller GEN_STAT nicht auslegt
      //
      // Die vier Felder sind aus ZCL_CFL_00500_RULES gefuellt. Vorher
      // stand dieselbe Regel ein zweites Mal im JavaScript
      // (EDITABLE_STEPS, STEP_ESCALATION) - eine Kopie, die beim
      // naechsten Schritt auseinandergelaufen waere.
      //
      // Der Controller bindet jetzt Felder und legt nichts mehr aus.
      //------------------------------------------------------------------
      // abap.char(1), nicht abap_boolean: eine Custom Entity kennt nur
      // die eingebauten abap.*-Typen. Der Query-Provider fuellt 'X' /
      // ' ', der Controller prueft darauf.
      @EndUserText.label: 'Decision editable'
      IsDecisionEditable   : abap.char(1);

      @EndUserText.label: 'Decision status'
      DecisionStatusText   : abap.char(80);

      // Keine @UI-Annotation - auf einer Custom Entity sind sie nicht
      // zugelassen ("wrong scope"). Verborgen wird das Feld in der
      // Metadata Extension, genau wie SeverityCriticality.
      DecisionStatusCriticality : abap.int1;

      // Was zum Abschluss noch fehlt - leer, wenn nichts fehlt.
      //
      // Kommt mit der Antwort der Aktion zurueck ("result [1] $self"),
      // der Controller zeigt ihn ohne zweiten Roundtrip. Bewusst ein
      // HINWEIS und keine Ablehnung: gespeichert wird trotzdem, weil
      // ein halbfertiger Arbeitszustand legitim ist.
      @EndUserText.label: 'Missing for completion'
      DecisionHint         : abap.char(80);

      //------------------------------------------------------------------
      // Die Einteilungen als Tabelle.
      //
      // Eine Association auf eine zweite Custom Entity - Fiori Elements
      // macht daraus eine Tabelle im Facet, ohne dass die App etwas
      // dafuer tun muesste. Aufgeloest wird sie ueber die WfId: beim
      // Aufklappen schickt Fiori Elements einen eigenen Request an
      // /ScheduleLine mit Filter auf dieses Feld, und der Provider dort
      // wertet ihn mit derselben Routine aus wie dieser hier.
      //------------------------------------------------------------------
      _SchedLine : association [0..*] to ZCFL_00500_C_SCHEDLINE
                     on $projection.WfId = _SchedLine.WfId;

}
```

## Der Schlüssel
Die conFLOW-Instanz-GUID als **32-stellige Hex-Kette**, nicht als `RAW(16)`. Grund: sie muss durch die URL passen und im Intent-Parameter stehen. Die Umrechnung ist die eingebaute C-nach-X-Konvertierung — mit Prüfung gegen Müll aus der URL:
```
DATA lv_hex TYPE c LENGTH 32.
lv_hex = iv_key.
TRANSLATE lv_hex TO UPPER CASE.
IF lv_hex CN '0123456789ABCDEF'.
  RETURN.                " kein Dump, keine Zeile
ENDIF.
rv_id = lv_hex.           " char(32) -> guid_16
```

## Zwei Fallen in der Typwahl
{% hint style="danger" %}
**`abap_boolean` geht nicht.** Eine Custom Entity kennt nur die eingebauten `abap.*`-Typen. Für ein Flag also `abap.char(1)`, gefüllt mit `'X'` / `' '`.
{% endhint %}
{% hint style="danger" %}
**`@UI`-Annotationen sind nicht zugelassen** („used at wrong position — wrong scope"). Alles Darstellende gehört in die Metadata Extension. Auch `@UI.hidden` — deshalb tragen die Criticality-Felder hier keine Annotation.
{% endhint %}
