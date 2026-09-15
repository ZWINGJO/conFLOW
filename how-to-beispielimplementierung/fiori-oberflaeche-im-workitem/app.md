# Die App: OData V2 und UI5 freestyle

*Klein gehalten: ein Entity Type, ein Function Import, eine View. Die Regeln liegen im ABAP, die App legt nichts aus.*

## Die Schichten

```
UI5 freestyle              BSP ZCFL_00500_UIV2       eine View, ein Controller
      │  OData V2
ZCL_ZCFL_00500_V2_DPC_EXT  (SEGW-generiert)          zwei Redefinitionen, entscheidet nichts
      │
ZCL_CFL_00500_V2_DATA      Lesen · Schreiben · Berechtigung   kennt kein OData
      │
      ├─ Workflow-Klasse   Container-Helfer (GLOBAL FRIENDS)
      ├─ Regelklasse       welche Gründe, welcher Schritt editierbar, was fehlt
      └─ conFLOW           /C09/CFL_S01 · S03 · S04
```

**Die Datenklasse ist der Teil, der bleibt.** Der DPC_EXT darüber ist eine dünne Hülle und kann bei einem späteren Technologiewechsel weg.

## Das SEGW-Modell

Projekt `ZCFL_00500_V2`, **ein** Entity Type `Decision`, **alle Properties `Edm.String`**. So gibt es beim Abtippen keine Precision, keine Scale und keine versteckte Typableitung.

| Property | Max Length | Inhalt |
| --- | --- | --- |
| `WfId` (Key) | 32 | conFLOW-Instanz als Hex-Kette |
| `Title` | 120 | Titel des Workitems (`SWWWIHEAD-WI_TEXT`) |
| `WorkitemText` | **leer** | der Kontextblock aus `SAP_WAPI_WORKITEM_DESCRIPTION` |
| `ReasonList` | **leer** | zulässige Gründe als `KEY=Text`, eine Zeile je Grund |
| `Reason` | 20 | Eingabe |
| `Note` | 132 | Eingabe, Länge des Container-Werts |
| `Editable` | 1 | `X` / leer, damit steuert das Backend die Felder |
| `MessageText` | 80 | warum nicht eingebbar, oder was zum Abschluss fehlt |

Entity Set `DecisionSet`. Dazu unter *Data Model* ein **Function Import** `SetDecision`:

| | |
| --- | --- |
| Return Type Kind / Type | Entity Type / `Decision` |
| Return Cardinality | `1`, damit bringt das Speichern den neuen Stand samt `MessageText` gleich mit |
| Return Entity Set | leer (bei Kardinalität 1 nicht belegbar) |
| HTTP Method | **`POST`** |
| Parameter | `WfId` (32), `Reason` (20), `Note` (132), alle `Edm.String` |

{% hint style="danger" %}
**`WorkitemText` und `ReasonList` ohne Max Length.** Mit einer Länge erzeugt SEGW `CHAR(n)`, und `MOVE-CORRESPONDING` schneidet ohne Meldung ab.
{% endhint %}

## Die zwei ABAP-Klassen

**`ZCL_ZCFL_00500_V2_DPC_EXT`** redefiniert genau zwei Methoden:

| Methode | tut |
| --- | --- |
| `DECISIONSET_GET_ENTITY` | Schlüssel auspacken → `READ` → `MOVE-CORRESPONDING`; nichts gefunden → 404 |
| `/IWBEP/IF_MGW_APPL_SRV_RUNTIME~EXECUTE_ACTION` | Parameter auspacken → `SAVE` → Rückgabe; abgelehnt → Ausnahme **mit Text** |

Kein Create, Update, Delete oder EntitySet. Was es nicht gibt, muss man nicht absichern.

**`ZCL_CFL_00500_V2_DATA`** hat zwei öffentliche Methoden, `READ` und `SAVE`. Beide prüfen vorab dasselbe:

| Prüfung | wie |
| --- | --- |
| Schlüssel gültig? | 32 Hex-Zeichen |
| Instanz gehört zu **diesem** Workflow? | `/C09/CFL_S01-WF_DEFINITION` |
| welches Workitem? | `/C09/CFL_S03` ⋈ `SWWWIHEAD` (Typ `W`), das offene zuerst, sonst das jüngste |
| **darf dieser Benutzer?** | `SWWUSERWI` (offenes Workitem im Arbeitsvorrat) **oder** `SWWWIHEAD-WI_AAGENT` (selbst ausgeführt) |

{% hint style="danger" %}
**Die Instanzprüfung ist Pflicht.** Die PFCG-Rolle erlaubt den Zugriff auf den *Service*, nicht auf bestimmte Vorgänge, und der Schlüssel steht in der URL. Beide Zugriffe laufen über den Primärschlüssel und brauchen keinen Funktionsbaustein. „Nicht gefunden“ und „nicht berechtigt“ bekommen denselben Text, damit ein Fremder nicht erfährt, welche GUIDs es gibt.
{% endhint %}

`SAVE` schreibt **nur bei Änderung**, sonst setzt jede Schreibpause Änderer und Zeitpunkt neu. Es speichert auch Unvollständiges und meldet über `MessageText`, was zum Abschluss fehlt.

## Die UI5-App

Freestyle, gebaut gegen **SAPUI5 1.71**, nur `sap.m`, `sap.ui.core` und `sap.ui.layout`.

| Datei | Inhalt |
| --- | --- |
| `manifest.json` | Datenquelle `/sap/opu/odata/sap/ZCFL_00500_V2_SRV/`, OData 2.0, eine Root-View, kein Router |
| `Component.js` | Schlüssel aus den Startparametern, Workitem-Wechsel |
| `view/Detail.view.xml` | Panel mit Workitem-Text, Auswahl *Grund*, Textfeld *Notiz*, Statuszeile |
| `controller/Detail.controller.js` | lesen, automatisch speichern, Statuszeile |
| `i18n/i18n.properties` | Texte |

### Der Schlüssel und der Workitem-Wechsel

Die Inbox reicht den Schlüssel als Startparameter `CFLQueryObject00` durch. Unter `embedIntoDetailsNestedRouter` wird die App beim Klick auf das nächste Workitem **wiederverwendet**. Die Inbox ruft dann zwei Methoden an der Component:

```js
wfIdOf: function (oParams) {                 // Parameter kommen als Array
    var v = oParams && (oParams.CFLQueryObject00 || oParams.WfId);
    return Array.isArray(v) ? v[0] : v;
},

navigateBasedOnStartupParameter: function (oParams) {   // Pflicht
    this.setWfId(wfIdOf(oParams));           // -> EventBus -> Controller lädt neu
},

refreshForStartupParameter: function (oParams) {        // optional
    this.setWfId(wfIdOf(oParams));
}
```

{% hint style="danger" %}
**Fehlt `navigateBasedOnStartupParameter`, wechseln beim zweiten Workitem nur die Knöpfe**, der Inhalt bleibt stehen. Die Knöpfe kommen vom Task-Gateway, der Inhalt von der App. Beide Methoden sind undokumentiert und aus dem Quelltext der Inbox gelesen, also nach Upgrades prüfen. Den Wechsel über den **EventBus** melden: `JSONModel#setProperty` löst kein `propertyChange` aus.
{% endhint %}

### Lesen und Speichern

- **Einmal lesen** (`DecisionSet('<id>')`) und die Werte in ein **JSON-Modell** legen. Die Felder hängen am JSON-Modell, nicht an der OData-Bindung. Sonst stünden sie als offene Änderungen im V2-Modell.
- **Kein Speichern-Knopf.** Die conFLOW-Knöpfe liegen außerhalb der App, einen ungespeicherten Stand könnte beim Entscheiden niemand einsammeln. Gesichert wird deshalb bei Auswahl des Grunds und bei der Notiz nach **800 ms** Schreibpause (`liveChange`) sowie beim Verlassen des Felds (`change`).
- Gespeichert wird über `callFunction("/SetDecision", { method: "POST", … })`.
- **Nach dem Speichern nicht neu lesen.** Sonst schreibt der gelesene Wert in das Feld, in dem der Cursor noch steht.
- **Statuszeile** statt Dialog: grün *Saved hh:mm*, gelb *gespeichert, aber zum Abschluss fehlt …*, rot mit dem Text des Backends. Ein Merker verhindert doppelte Aufrufe.
- Beide Felder hängen an `Editable`. Welcher Schritt eingeben darf, steht in der Regelklasse und nicht im JavaScript.

## Bauen und hochladen

Das Quellprojekt liegt im Git, im System liegt nur das Build-Ergebnis.

```
npm install
npm run build            # ui5 build → dist/ mit Component-preload.js
cd dist && zip -r ../app.zip .
```

Hochladen ohne Systemverbindung am Entwicklungsrechner: `SE38` → **`/UI5/UI5_REPOSITORY_LOAD`**, Modus *Upload*, Anwendung `ZCFL_00500_UIV2`, Codepage `UTF-8`, Verzeichnis = das **ausgepackte `dist/`**. Danach die Caches, siehe [Customizing](customizing.md).

{% hint style="info" %}
**Nicht in SE80 an Einzeldateien ändern.** Die App läuft aus `Component-preload.js`. Eine geänderte Einzeldatei wird gar nicht angefordert, und es kommt keine Meldung. Änderungen gehen immer über Quellprojekt, Build und Upload.
{% endhint %}
