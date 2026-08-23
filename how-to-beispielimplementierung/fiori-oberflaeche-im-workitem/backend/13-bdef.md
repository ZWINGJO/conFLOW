# Behavior Definition und Aktionsparameter
*Feld-Editieren braucht Draft oder Sticky. Eine Entity ohne Tabelle hat beides nicht — also eine Aktion mit Parametern.*

**`ZCFL_00500_D_DECISION`**

```abap
@EndUserText.label: 'Order Promise Exception - decision'

// Parameter der Aktion SETDECISION.
//
// Eine ABSTRACT ENTITY, keine Tabelle und keine View: sie beschreibt
// nur die Form der Eingabe - ein Feld je Element, Wertehilfe inklusive.
//
// EINEN DIALOG GIBT ES NICHT MEHR. Fiori Elements wuerde aus einer
// Aktion mit Parametern einen Knopf mit Parameterdialog bauen; beides
// ist entfallen, seit die Custom Section die Felder selbst zeigt und
// bei jeder Aenderung speichert. Die Entity beschreibt seither nur
// noch die SIGNATUR der Aktion - der Controller ruft sie ueber
// bindContext( ) und fuellt die Parameter aus den eigenen Feldern.
//
// Warum ueberhaupt eine Aktion und kein Feldedit: Fiori Elements V4
// braucht zum Bearbeiten von Feldern entweder Draft oder eine Sticky
// Session. Beides hat dieser Service nicht - er liest aus einem
// Container, nicht aus einer Tabelle. Der Standardweg ist dann die
// Aktion mit Parametern. Fachlich ist sie ohnehin die richtige Form:
// eine Entscheidung ist ein Vorgang, kein Feldedit.
define abstract entity ZCFL_00500_D_DECISION
{
      @EndUserText.label: 'Reason for decision'

      // KEINE Pflichtfeld-Annotation.
      //
      // Naheliegend waere @Common.FieldControl: #MANDATORY - das
      // Vokabular "Common" gibt es in diesem System aber nicht
      // ("Annotation 'COMMON.FIELDCONTROL' unknown"). Nachgesehen in
      // den Annotationsdefinitionen: 'mandatory' existiert nur
      // innerhalb von @Consumption.filter und gilt fuer Filter, nicht
      // fuer Aktionsparameter.
      //
      // Die Pflicht wird deshalb ausschliesslich im Handler geprueft
      // (SETDECISION). Der Dialog laesst ein leeres Feld zu, die
      // Aktion lehnt ab und Fiori Elements zeigt die Meldung. Etwas
      // weniger bequem, aber verlaesslich - und die Server-Pruefung
      // haette es ohnehin gebraucht.

      // Dieselbe Werteliste wie am Feld der Entity: ZCL_CFL_00500_VH
      // liefert die Gruende aus MC_REASON. Damit kann im Dropdown
      // nichts stehen, was der Workflow nicht kennt.
      //
      // Seit dem 22.08.2026 der GRUND, nicht mehr die Aktion: die
      // Aktion entscheidet der conFLOW-Button, und zwei Bedienelemente
      // fuer dieselbe Frage waren nicht aufloesbar.
      @Consumption.valueHelpDefinition: [{ entity: { name:    'ZCFL_00500_C_ACTIONVH',
                                                     element: 'ActionKey' } }]
      DecisionReason : abap.char(20);

      @EndUserText.label: 'Note'
      @UI.multiLineText: true
      DecisionNote   : abap.char(250);
}
```
**`ZCFL_00500_C_EXCEPTION (BDEF)`**

```abap
// Unmanaged, weil die Daten nicht in einer Tabelle liegen, sondern im
// conFLOW-Container /c09/cfl_s04. Gespeichert wird ueber die
// Framework-API SET_ATTRIBUT_VALUE - kein direkter Schreibzugriff auf
// eine /C09/-Tabelle, das bleibt Framework-Gebiet.
//
// Ohne "strict", und die Begruendung dafuer wurde am 22.08.2026
// korrigiert. Hier stand vorher, "lock master" sei bei UNMANAGED nicht
// deklarierbar. Das ist FALSCH - es ist dort sogar der Regelfall: man
// deklariert "lock master" und implementiert dazu einen Handler
// METHODS lock FOR LOCK. Die Sperre selbst zu verwalten ist genau das,
// was die Deklaration verlangt, nicht was sie ausschliesst.
//
// Der wirkliche Grund ist ein anderer: ein LOCK-Handler braucht ein
// Sperrobjekt. Diese Custom Entity hat keine Persistenz, also gaebe es
// nichts zu sperren - man muesste eines auf die Instanz-GUID erfinden.
// Und fachlich serialisiert conFLOW bereits ueber die Reservierung des
// Workitems; eine zweite Sperrschicht daneben waere eine zweite
// Wahrheit.
//
// Also: bewusster Verzicht auf "lock master", und damit faellt auch
// "strict" weg, denn die strikte Pruefstufe verlangt fuer aendernde
// Operationen eine Sperr-Rolle.
//
// Der Preis steht in der Dokumentation als eigener Befund: es gibt
// keinen Konfliktschutz. Zwei Bearbeiter auf derselben Instanz - oder
// derselbe in zwei Tabs - ueberschreiben einander wortlos. Das ist die
// Folge dieser Entscheidung, kein davon unabhaengiger Mangel.
unmanaged implementation in class zcl_cfl_00500_behv unique;

define behavior for ZCFL_00500_C_EXCEPTION alias OrderPromiseException
{
  //--------------------------------------------------------------------
  // Weder CREATE noch UPDATE noch DELETE - nachgewiesen am 21.08.2026:
  //
  // Mit "update" allein bietet Fiori Elements KEINEN Bearbeitungsmodus
  // an; es erscheint kein "Edit". Dafuer braucht es Draft oder eine
  // Sticky Session, und Sticky setzt ein Sperrobjekt voraus - das eine
  // CUSTOM ENTITY nicht hat, weil keine Tabelle dahintersteht.
  //
  // Der Versuch blieb am 21.08.2026 als Leiche IM SYSTEM stehen,
  // waehrend das Repo schon sauber war - und er ist der Verdaechtige
  // fuer "die Aktion laeuft, speichert aber nichts": mit "update" gilt
  // die Entity als aenderbar, und Fiori Elements schickt eine Aktion
  // dann ueber den EditFlow. Der sucht einen Bearbeitungsmodus, findet
  // weder Draft noch Sticky - und die Zusage aus invokeAction wird
  // weder erfuellt noch abgelehnt. Genau das war zu sehen: kein
  // Fehler, kein Dump, kein Satz in /c09/cfl_s04.
  //
  // Dazu kommt: "update" ohne Update-Handler ist eine deklarierte,
  // aber nicht implementierte Operation. Die Pruefung des Behavior
  // Pools sagt das auch - "The operation UPDATE ... is not
  // implemented" - nur aktiviert die Klasse trotzdem.
  //
  // Geschrieben wird deshalb ausschliesslich ueber die Aktion unten.
  //--------------------------------------------------------------------

  //--------------------------------------------------------------------
  // Die Entscheidung erfassen.
  //
  // "features : instance" schaltet die dynamische Steuerung ein: der
  // Handler entscheidet je Instanz, ob der Knopf ueberhaupt angeboten
  // wird. Grundlage ist der conFLOW-Schritt - in einem Hintergrund-
  // schritt oder nach dem Abschluss gibt es nichts zu entscheiden.
  //
  // "result [1] $self" gibt die geaenderte Instanz zurueck. Damit
  // aktualisiert Fiori Elements die Objektseite ohne zweiten Roundtrip -
  // der neue Wert steht sofort im Block "Your decision".
  //--------------------------------------------------------------------
  action ( features : instance ) setDecision
         parameter ZCFL_00500_D_DECISION
         result [1] $self;

}
```

## `update;` ist ein Zeitzünder
{% hint style="danger" %}
Damit gilt die Entity als änderbar, und Fiori Elements schickt Aktionen über den **EditFlow**. Der sucht einen Bearbeitungsmodus, findet weder Draft noch Sticky — und der Aufruf bleibt **stumm stehen**: kein `.then`, kein `.catch`, keine Anfrage, kein Dump. Nichts deutet auf die Ursache hin.
{% endhint %}

## `lock master` — die Aussage, die oft falsch steht
Was stimmt und was nicht
**Falsch** ist die verbreitete Aussage, `lock master` sei bei `unmanaged` nicht deklarierbar. Es ist dort sogar der Regelfall: man deklariert es und implementiert dazu `METHODS lock FOR LOCK`.
**Richtig** ist der eigentliche Grund: ein LOCK-Handler braucht ein Sperrobjekt. Eine Custom Entity ohne Persistenz hat keines; man müsste eines auf die Instanz-GUID erfinden. Und fachlich serialisiert conFLOW bereits über die Workitem-Reservierung — eine zweite Sperrschicht wäre eine zweite Wahrheit.
Also **bewusster Verzicht**, keine technische Unmöglichkeit. Und damit fällt auch `strict` weg, denn die strikte Prüfstufe verlangt für ändernde Operationen eine Sperr-Rolle. **Der Preis ist der fehlende Konfliktschutz** — das gehört dokumentiert, nicht verschwiegen.
