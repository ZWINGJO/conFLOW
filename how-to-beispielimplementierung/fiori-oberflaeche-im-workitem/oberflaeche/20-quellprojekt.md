# Wie die App ins System kommt
*Die Oberfläche ist kein ABAP-Objekt, das man in SE80 pflegt. Sie ist ein Ordner mit Quelltext, der gebaut, hochgeladen und ersetzt wird. Solange das nicht eingerichtet ist, gibt es „die App" gar nicht — es gibt acht Dateien mit acht Altersständen.*

## Der Zustand, den man sonst bekommt
Die Beispiel-App entstand mit dem Generator in ADT und wurde danach **Datei für Datei in SE80** gepflegt. Das geht — und ist die Wurzel von drei Fehlersuchen, die alle wie Logikfehler aussahen und keine waren: ein „Decide"-Knopf, den es im Quellstand nicht mehr gab; ein Dropdown mit vier fest verdrahteten Einträgen, während der neue Controller schon lief; eine Statuszeile, die stumm blieb.
{% hint style="danger" %}
**Dieselbe Ursache in allen drei Fällen.** Fragment und Controller werden als *einzelne Dateien* ausgeliefert und getrennt gecacht — das XML sitzt im IndexedDB-View-Cache, das JS nicht. Sie können deshalb **unterschiedlich alt** sein. Neues Verhalten mit alter Oberfläche sieht aus wie ein Logikfehler und ist keiner.
{% endhint %}
{% hint style="info" %}
**Die Probe, an der die Entscheidung hängt**

*Was ginge verloren, wenn der Entwicklungsrechner heute kaputtgeht?*

**Mit Quellprojekt: nichts.** Ohne: alles, was seit dem letzten Generatorlauf in SE80 entstanden ist — und niemand kann sagen, was das war, weil es keinen Stand gibt, gegen den man vergleicht.
{% endhint %}

## Drei Orte, drei Rollen
| Ort | hält | ist |
| --- | --- | --- |
| Git-Repository | `app/**`, `package.json`, `ui5.yaml`, `ui5-deploy.yaml`, README | **die Quelle** — versioniert, nachvollziehbar, übergebbar |
| Entwicklungsrechner | Klon + `node_modules/` + `dist/` | der Arbeitsplatz — **vollständig wegwerfbar** |
| SAP-System | BSP-Anwendung `ZCFL_NNNNN_UI`, dazu die ABAP-Objekte und das Customizing | die **Laufzeit** — kein Ablageort |
{% hint style="info" %}
**`dist/` gehört nicht ins Repository.** Es ist abgeleitet. Ein Repo mit Build-Ergebnissen hat zwei Wahrheiten, und irgendwann weichen sie ab — genau der Fehler, gegen den das Quellprojekt gebaut wird.
{% endhint %}

## Was der Build ändert
| Datei für Datei | gebaut und hochgeladen |
| --- | --- |
| Controller und Fragment werden einzeln angefordert | `Component-preload.js` bündelt beide in **eine** Datei mit **einem** Hash |
| „JS neu, XML alt" ist jederzeit möglich | derselbe Zustand ist **strukturell ausgeschlossen** — nicht nur unwahrscheinlicher |
| `resources.json` fehlt — 404 bei jedem Laden | wird miterzeugt (`--include-task=generateResourcesJson`) |
| Dateileichen bleiben liegen | der Deploy **ersetzt** die Anwendung vollständig |
{% hint style="success" %}
**Das ist der eigentliche Gewinn** — nicht die Bequemlichkeit. Eine ganze Fehlerklasse verschwindet, statt beherrscht zu werden. Die Notnägel im Controller (Felder zusätzlich programmatisch schalten, die Statuszeile notfalls selbst anlegen) sind alle in dieser Zeit entstanden.
{% endhint %}

## Die zwei Dateien, die das Projekt ausmachen
**`package.json`**

```json
{
  "name": "zcfl00500inbox",
  "version": "0.0.1",
  "private": true,
  "description": "Order Promise Exception - eigene Oberfläche im conFLOW-Workitem (WF 00500)",
  "scripts": {
    "build": "ui5 build --clean-dest --include-task=generateResourcesJson",
    "deploy": "fiori deploy --config ui5-deploy.yaml",
    "deploy-test": "fiori deploy --config ui5-deploy.yaml --testMode true",
    "undeploy": "fiori undeploy --config ui5-deploy.yaml"
  },
  "devDependencies": {
    "@ui5/cli": "^4",
    "@sap/ux-ui5-tooling": "^1"
  },
  "ui5": {
    "dependencies": [
      "@sap/ux-ui5-tooling"
    ]
  }
}
```
**`ui5.yaml`**

```ini
# Quellprojekt der Inbox-App — Bauen.
#
# WICHTIG: paths.webapp zeigt auf "app". Damit bleiben die Dateien dort
# liegen, wo sie seit August liegen - die Dokumentationsgeneratoren
# (tools/refresh_doc_sources.py, tools/build_howto_doc.py) lesen genau
# diese Pfade. Ein Verschieben nach webapp/ wäre konventioneller und
# hätte sechs Generatorzeilen gekostet, ohne etwas zu verbessern.
specVersion: "4.0"
metadata:
  name: zcfl00500inbox
type: application
resources:
  configuration:
    paths:
      webapp: app
framework:
  name: SAPUI5
  version: "1.136.0"
  libraries:
    - name: sap.m
    - name: sap.ui.core
    - name: sap.fe.templates
    - name: sap.uxap
    - name: themelib_sap_horizon

# KEIN LOKALER VORSCHAU-SERVER - das ist eine Entscheidung, kein Versäumnis.
#
# Die Anwendung läuft ausschließlich im Zielsystem, eingebettet im
# Detailbereich eines conFLOW-Workitems. Lokal fehlt ihr alles, was sie
# ausmacht: ein Workitem, der Intent-Parameter mit der Instanz-ID, der
# Container mit den Zahlen und die Knöpfe des Task-Gateways in der
# Fußleiste. Man sähe eine leere Objektseite - und würde daraus falsche
# Schlüsse ziehen.
#
# Der Arbeitsablauf ist deshalb: lokal ändern, lokal bauen, hochladen,
# im Zielsystem im Workitem prüfen.
#
# Nebeneffekt, der zählt: ohne Vorschau-Server braucht dieser Rechner
# weder eine Verbindung zum System noch Zugangsdaten in einer
# Konfigurationsdatei. Verbindung braucht nur der Upload.

# HINWEIS ZUM BUILD-PROTOKOLL
#
# Der Lauf meldet vier Zeilen "error lbt:bundle:Resolver ... missing module
# sap/ui/integration/...". Sie kommen aus SAPs EIGENEN Bibliotheken:
# sap.ui.fl verweist optional auf sap.ui.integration, das hier nicht
# deklariert ist. Mit dieser App hat das nichts zu tun, und der Build ist
# erfolgreich - "Build succeeded".
#
# Nicht versuchen, sie durch Nachdeklarieren wegzubekommen. Genau das wurde
# probiert: sap.ui.integration aufgenommen, danach waren es MEHR Zeilen
# (sap/ui/export, von sap.m gefordert). Optionale Abhängigkeiten des
# Frameworks nachzuziehen ist eine Kette ohne Ende.
```
{% hint style="info" %}
**`paths.webapp` zeigt hier auf `app`**, nicht auf das konventionelle `webapp`. Grund: die Dokumentationsgeneratoren lesen genau diese Pfade. Beim Neuanlegen darf man konventionell bleiben — beim Nachrüsten eines bestehenden Ordners ist das Verschieben teurer als die Zeile.
{% endhint %}

## Vom Bestand ins Quellprojekt
*Der häufigere Fall ist nicht der Neubau, sondern das Nachrüsten: im System steht eine Anwendung, die der Generator angelegt und die Hand gepflegt hat. **Dann ist das System die Wahrheit** — nicht der Ordner, in dem man die Dateien zu haben glaubt.*
| # | Schritt | Womit |
| --- | --- | --- |
| 1 | **Dateien aus dem System holen** — vollständig, nicht die, die man für aktuell hält | Report `/UI5/UI5_REPOSITORY_LOAD`: BSP-Name eintragen, **Download**, Zielverzeichnis und Codepage bestätigen |
| 2 | Gegen den vermuteten Stand vergleichen | ein `diff` über beide Ordner. **Jede Abweichung ist eine SE80-Änderung, die nie im Projekt ankam** |
| 3 | Projekt darum bauen | der Download wird `app/`; daneben `package.json`, `ui5.yaml`, `ui5-deploy.yaml`, `.gitignore`, README |
| 4 | Erster Commit — **vor** der ersten eigenen Änderung | damit der Systemstand als Nullpunkt in der Historie steht |
| 5 | `npm run build`, dann `deploy-test` | der Trockenlauf sagt, was der echte Deploy täte |
| 6 | Deployen und im Workitem prüfen | ab hier **nie wieder SE80** |
{% hint style="danger" %}
**Schritt 1 ist nicht optional.** Wer die lokalen Kopien für den Wahrheitsstand hält und sie deployt, überschreibt jede Änderung, die seit dem letzten Abgleich nur im System steht — **und merkt es nicht**, weil der Deploy ohne Rückfrage ersetzt. Ein Download kostet zwei Minuten.
{% endhint %}
{% hint style="info" %}
**Der Report kann auch hochladen** — dieselbe Zielablage, nur mit Dialog statt Kommandozeile. Das ist der Weg für einen Rechner ohne Node. Er braucht SAP GUI auf demselben Rechner: Verzeichnis und Codepage holt er sich von dort.
{% endhint %}
{% hint style="success" %}
**SAP sagt an dieser Stelle selbst, was hochgehört.** Wählt man beim Upload einen Ordner mit `webapp/` und ohne `manifest.json` in der Wurzel, warnt der Report: *„If you want to deploy an app, please run a build, and then upload the `dist` folder with the build results"* (SAP-Hinweis 3225159). **Hochgeladen wird das Build-Ergebnis, nicht der Quellordner** — genau die Unterscheidung, um die es in diesem Kapitel geht.
{% endhint %}

## Kein lokaler Vorschau-Server — eine Entscheidung, kein Versäumnis
Die übliche Fiori-Tools-Einrichtung bringt einen Proxy und `npm start` mit. Beides wurde hier wieder ausgebaut. Die Anwendung läuft ausschließlich **eingebettet im Detailbereich eines Workitems**; lokal fehlen ihr das Workitem, der Intent-Parameter mit der Instanz-ID, der Container mit den Zahlen und die Knöpfe des Task-Gateways. Man sähe eine leere Objektseite — und zöge daraus falsche Schlüsse.
{% hint style="success" %}
**Der Nebeneffekt ist der bessere Grund.** Ohne Vorschau-Server braucht der Entwicklungsrechner **weder eine Verbindung zum System noch Zugangsdaten in einer Datei**. Verbindung braucht nur der Upload — und der fragt danach.
{% endhint %}
Der Arbeitsablauf ist deshalb: **lokal ändern → lokal bauen → hochladen → im Zielsystem im Workitem prüfen**. Das ist ein Schritt mehr als „F5" und der einzige, der zeigt, was der Bearbeiter sieht.

## Bauen und hochladen
```
npm install
npm run build          # -> dist/ inkl. Component-preload.js und resources.json
npm run deploy-test    # Trockenlauf gegen das System, schreibt nichts
npm run deploy         # ersetzt die BSP-Anwendung
```
{% hint style="info" %}
**Voraussetzung ist Node** — im Referenzprojekt Version 20. Sonst nichts: kein SAP GUI, keine ADT-Installation, keine Zugangsdaten auf der Platte. **Der erste Deploy legt die BSP-Anwendung nicht an, sondern ersetzt die bestehende** — sie stammt aus Schritt 11 der [Objekt-Reihenfolge](../grundlagen/06-reihenfolge.md). Anlegen kann `fiori deploy` sie auch, dann fragt es nach Paket und Transport.
{% endhint %}
Das Ziel steht in `ui5-deploy.yaml` — System, Mandant, BSP-Name, Paket und **Transport**:
```
builder:
  customTasks:
    - name: deploy-to-abap
      afterTask: generateCachebusterInfo
      configuration:
        target:
          url: https://<host>
          client: "<mandant>"
        app:
          name: ZCFL_NNNNN_UI
          package: ZCFL_NNNNN
          transport: <Workbench-Auftrag>
          description: <Kurztext der BSP-Anwendung>
```
{% hint style="danger" %}
**Der Deploy ersetzt die Anwendung vollständig.** Was nicht im Projekt liegt, verschwindet im System. Das ist der Gewinn und das Risiko in einem — deshalb **immer erst `deploy-test`**, und deshalb gehört nichts in die BSP-Anwendung, was nicht auch im Repository steht.
{% endhint %}
{% hint style="info" %}
**Zugangsdaten gehören nie ins Repository**, auch nicht in ein privates. `fiori deploy` liest sie aus `.env`, falls vorhanden, und fragt sonst danach — also `.env` in die `.gitignore`, zusammen mit `node_modules/` und `dist/`.
{% endhint %}
{% hint style="success" %}
**Nach dem ersten Deploy einmal `/UI5/APP_INDEX_CALCULATE`** und im Browser **Clear site data**. Sonst läuft der alte Stand weiter — siehe [Kapitel „Launchpad"](../anbindung/22-launchpad.md), Abschnitt „Der Cache".
{% endhint %}
{% hint style="info" %}
**Was hier belegt ist — und was nicht**

**Der Build ist nachgemessen:** `Component-preload.js` enthält genau die fünf App-Module und keinen Framework-Code, `resources.json` entsteht mit, der 404 beim Laden ist weg.

**Der Upload ist es nicht.** Im Referenzprojekt kam die Anwendung über den Generator und SE80 ins System; das Quellprojekt entstand danach. Beide Wege oben — `fiori deploy` und `/UI5/UI5_REPOSITORY_LOAD` — sind SAP-Standard, in *diesem* Projekt aber noch nicht gefahren. Wer nachbaut, sollte das wissen und mit `deploy-test` anfangen.

Das steht hier, weil [Regel 14](../nachschlagen/26-regeln.md) es verlangt: eine Begründung, die niemand am Bildschirm geprüft hat, ist keine Begründung. Das gilt auch für die eigene.
{% endhint %}

## Der Wegweiser im System
Ein `README.md` im `app/`-Ordner wird **mitdeployt**. Wer die BSP-Anwendung in SE80 aufmacht, findet dort die Repo-URL und den Satz, der sonst Stunden kostet.
{% hint style="danger" %}
**Eine in SE80 geänderte Einzeldatei wirkt nach dem Build nicht mehr.** UI5 lädt das Bündel; die Einzeldatei wird gar nicht erst angefordert — **ohne Fehlermeldung**. Das ist derselbe Anblick wie früher der Cache und hat eine andere Ursache.
{% endhint %}
{% hint style="info" %}
**Der Notausgang** gehört in dieses README: `Component-preload.js` in der BSP-Anwendung löschen. Dann lädt UI5 wieder die Einzeldateien, und eine Reparatur direkt im System wirkt — mit dem Wissen, dass der nächste Deploy sie überschreibt.
{% endhint %}

## Wem das Quellprojekt gehört
**Dem, dem die App gehört.** Läuft die Anwendung im System des Kunden, ist sie sein Werk — also gehört das Repository dorthin, nicht auf den Rechner des Beraters. Sonst ist der Quelltext beim Ende des Projekts weg und im System steht ein Gebäude ohne Bauplan.
{% hint style="info" %}
**Zwei Zeilen fehlen in fast jeder Übergabeliste.** Das conFLOW-Customizing `C01`–`C10` reist im **Customizing**-Auftrag, nicht im Workbench-Transport. Und `SWFVMD1` ist weder App noch Launchpad, sondern das Scharnier zwischen Workitem und App — fehlt es, zeigt das Workitem kommentarlos wieder den Textblock.
{% endhint %}
{% hint style="danger" %}
**Wenn es kein Quellprojekt geben kann** — weil im Zielsystem direkt gepflegt wird —, dann gilt das Cache-Kapitel in voller Härte, und die Notnägel im Controller sind dann keine Übervorsicht, sondern notwendig. Das ist eine tragbare Entscheidung. Sie sollte nur getroffen und nicht erlitten werden.
{% endhint %}
