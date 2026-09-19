# Eigene Fiori-Oberfläche im Workitem

*Eine kleine UI5-App im Detailbereich der My Inbox, angehängt über conFLOW, ohne den Workflow umzubauen.*

## Was am Ende dasteht

```
┌─────────────────────────────────────────┬────────────────────────┐
│  Titel des Workitems                    │ Comments │ Attachments │
│                                         │ More ∨                 │
│  Workitem-Text                          │                        │
│  (Befund, Beleg, Mengen, Termine …)     │  ← Standard der Inbox, │
│                                         │    über „Show Details" │
│  Grund      [ Auswahl            ▾ ]    │                        │
│  Notiz      [                      ]    │                        │
│  Saved 09:26:03                         │                        │
├─────────────────────────────────────────┴────────────────────────┤
│  Partial delivery │ Accept new date │ Escalate │ Show Details │ ⋯ │
└──────────────────────────────────────────────────────────────────┘
      eigene App (links)              conFLOW-Knöpfe (unten)
```

{% hint style="info" %}
**Die App entscheidet nichts.** Knöpfe, Bearbeiterfindung, Frist, Priorität und Protokoll bleiben conFLOW. Die App zeigt den Kontext und hält fest, *warum* entschieden wird: Grund und Notiz landen im conFLOW-Container. Entschieden wird weiter mit den Knöpfen darunter.
{% endhint %}

## Die Kette in einem Bild

```
conFLOW
  Parameter VISU in C08   (Sonderfall: BAdI set_inbox_ui( ))
    └─ setzt drei Container-Elemente am Workitem
                           │
Customizing SWFVMD1        ▼   (einmal je System)
  Task TS00388601  →  Intent  #ZCFLOrderPromiseV2-openInInbox?CFLQueryObject00=<Instanz>
                           │
Launchpad                  ▼   (je Workflow)
  Semantic Object · Target Mapping (openMode!) · Katalog · Rolle
                           │
App                        ▼   (je Workflow)
  UI5 freestyle  →  OData V2 (SEGW)  →  conFLOW-Container /C09/CFL_S04
```

**Welche App erscheint, entscheidet das einzelne Workitem**, nicht der Task. conFLOW hat die Visualisierung generisch verdrahtet: Der Task liest zur Laufzeit die Container-Elemente, die conFLOW gesetzt hat. Fehlen sie, zeigt die Inbox wie bisher den Textblock.

## Was angelegt wird

| Bereich | Was | Wie oft | Seite |
| --- | --- | --- | --- |
| conFLOW-Customizing | Parameter `VISU` in `C08` (Semantic Object) | je Workflow | [conFLOW](conflow.md) |
| conFLOW-BAdI | nur für Sonderfälle, z. B. eine andere App je Schritt | optional | [conFLOW](conflow.md) |
| SWFVMD1 | dynamische Visualisierung für Task `TS00388601` | **einmal je System** | [Customizing](customizing.md) |
| Launchpad | Semantic Object, Target Mapping, Katalog, Rolle | je Workflow | [Customizing](customizing.md) |
| Gateway | OData-Service registrieren | je Workflow | [Customizing](customizing.md) |
| App | SEGW-Modell, zwei ABAP-Klassen, UI5-App | je Workflow | [Die App](app.md) |

Die Reihenfolge beim Einspielen und die Proben nach jedem Schritt stehen unter [Prüfen](pruefen.md).

## Voraussetzungen

| | |
| --- | --- |
| ABAP | SAP_BASIS **7.50**, SAP_GWFND 750 — kein RAP nötig |
| UI5 | SAPUI5 **1.71** oder neuer |
| Inbox | Fiori My Inbox (BSP `CA_FIORI_INBOX`) |

{% hint style="info" %}
**Alle Namen sind Beispiele** aus einem Referenzworkflow (Nummer `00500`, eine Lieferterminabweichung im Kundenauftrag). Für den eigenen Workflow Nummer und Namen tauschen, das Muster bleibt.
{% endhint %}
