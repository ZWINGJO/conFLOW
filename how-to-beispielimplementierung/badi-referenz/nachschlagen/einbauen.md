# Einbauen

Die Klasse allein tut nichts. conFLOW findet sie über die
BAdI-Implementierung, und was sie tun soll, steht im Customizing.
Diese Reihenfolge hat sich bewährt, weil jeder Schritt den vorigen
voraussetzt.

## Die Reihenfolge

**1 · Paket und Konstantenklasse**

`ZCL_CFL_CONST_00900` zuerst. Sie hat keine Abhängigkeiten, und alles
Weitere verweist auf sie. Wer sie später anlegt, tippt die
Customizing-Schlüssel zwanzigmal als Literal.

**2 · Customizing anlegen**

| Tabelle | Was |
|---|---|
| `C06` / `C06T` | die Workflow-Definition und ihr Text |
| `C01` / `C01T` | die Schritte - je einer für `X0`, `B1`, `01`, `02`, `B2`, `B3`, `X1`, `X2`, `X3` |
| `C02` | die Übergänge: Schritt + Entscheidung → Folgeschritt |
| `C05` | die Bearbeiterkreise je Schritt |
| `C03` | wer der Kreis ist - oder `USER_BADI = 'X'` für die Findung im Code |
| `C07` | die Container-Elemente. **Fehlt ein Element hier, wird sein Wert kommentarlos verworfen** |
| `C09` / `C09T` | die Texte der Entscheidungen `UC1`–`UC5` |
| `C08` | Objekt und Subobjekt fürs Anwendungsprotokoll |
| `C10` | die Typkopplung, wenn der Start über ein Ereignis läuft |

Beim Hintergrundschritt `B1` gehören Klassenname und Methodenname in
`C01` - der Aufruf ist dynamisch.

**3 · Die Workflow-Klasse**

`ZCL_CFL_WORKFLOW_00900` anlegen, `00900` durch die eigene Nummer
ersetzen. Drei Stellen: Klassenname, Konstantenklasse, BAdI-Filter.

Das Textsymbol `TEXT-001` nicht vergessen - es beschriftet den
Objekt-Link. Es hängt am Textpool, nicht am Code, und muss beim
Transport mit.

**4 · Die BAdI-Implementierung**

Zur Definition `/C09/CFL_BADI_0101`, Filter `WF_DEFINITION = '00900'`.

> **Ohne Filter läuft die Klasse für jeden Workflow im System.** Der
> Fehler fällt im Entwicklungssystem mit einem Workflow nicht auf und
> im Produktivsystem sofort.

**5 · Die Bearbeiterfindung**

`ZCL_CFL_GET_ACTORS` und die Rollen dazu. Erst danach ist ein Test
sinnvoll - ein Workitem ohne Bearbeiter geht in den Fehlerstatus.

**6 · SLG0**

Objekt und Subobjekt aus `C08` müssen in SLG0 existieren, sonst
schreibt conFLOW die Meldungen des Hintergrundschritts nirgendwohin.

## Der erste Test

Einen Beleg **über** der Grenze nehmen. Der andere Fall schließt sich
selbst, und dann sieht man nichts.

| | erwartet |
|---|---|
| `/C09/CFL_S01` | eine Instanz mit `INSTID` = Belegnummer |
| `/C09/CFL_S04` | die Container-Werte aus `B1` |
| SLG1 | eine Zeile "Approval required - ..." |
| Business Workplace | ein Workitem bei den Benutzern der Rolle |
| Workflow-Protokoll | beim Hintergrundschritt der berechnete Text, nicht der aus dem Customizing |

Steht in `S04` nichts, ist fast immer `C07` die Ursache.
