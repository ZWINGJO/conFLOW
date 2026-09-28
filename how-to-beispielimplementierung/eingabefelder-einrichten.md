# Eingabefelder am Workitem einrichten

*Ein Formular am Genehmigungsschritt. Vier Schritte im Customizing, kein Coding — im SAP Business Workplace sofort, in der Fiori My Inbox nach einer einmaligen Einrichtung.*

## Was am Ende dasteht

```
┌──────────────────────────────────────────────────────────────┐
│  ① Freigabe Vorgesetzter Schulungsantrag 166                 │
│                                                              │
│  TRAINING REQUEST                                            │
│  Kurs: Fiori Inbox · Beginn: 10.10.2027 · Kosten: 1.714,00 € │
│                                                              │
│  Ihre Eingaben                                               │
│    Kurs*        [ Fiori Inbox                             ]  │
│                   Pflicht bei jeder Entscheidung             │
│    Beginn*      [ 10.10.2027                           📅 ]  │
│    Kosten*      [         1714,00 ] EUR                      │
│    Währung*     [ EUR – Euro                            ▾ ]  │
│    Begründung   [                                        ]  │
│                                              Gespeichert 09:26│
├──────────────────────────────────────────────────────────────┤
│                        ✔ Genehmigen  │  ✖ Ablehnen           │
└──────────────────────────────────────────────────────────────┘
```

{% hint style="info" %}
**Die Felder stehen in keiner Oberfläche.** SAP GUI und Fiori bauen dasselbe Formular aus dem Customizing. Was Sie hier einstellen, erscheint in beiden — und die Werte landen im conFLOW-Container, also dort, wo auch die Platzhalter für Workitem-Texte und Mails herkommen.
{% endhint %}

## Die vier Schritte

### 1 · Feldgruppe anlegen

Pflegebaum `/C09/CONFLOW_C`, unter der Workflow-Definition der Knoten **Feldgruppen** (`/C09/CFL_C13`).

| Feld | Was hinein gehört |
| --- | --- |
| `VIEW_ID` | ein Name, eindeutig **innerhalb dieser Definition** — etwa `Z_ANTRAG_START` oder `Z_ANTRAG_01` |
| Überschrift | je Sprache, im Textknoten (`C13T`); sie steht über dem Formular |

Eine Gruppe ist ein **Formular**, kein Schritt. Sie darf an mehreren Schritten hängen: die Werte gehören dem Vorgang, nicht dem Schritt.

### 2 · Felder eintragen

Unterknoten **Felder der Feldgruppe** (`/C09/CFL_C14`), je Feld eine Zeile.

| Feld | Was hinein gehört |
| --- | --- |
| Reihenfolge | in Zehnerschritten, dann lässt sich später etwas dazwischen schieben |
| Element | der Name im Container, bis 32 Zeichen — er taucht als Platzhalter `&CFL-<ELEMENT>&` wieder auf |
| Datenelement | **die einzige echte Entscheidung**, siehe unten |
| Modus | Eingabe oder Anzeige |
| Pflicht | erzwungen beim Start und beim Entscheiden |
| Bezeichnung | nur nötig, wenn die aus dem Datenelement nicht passt (Textknoten `C14T`) |

**Das Datenelement bestimmt das Bedienelement.** Sie müssen nichts auswählen, was das Feld „sein soll" — das steht im DDIC:

| Was Sie wollen | Datenelement | Was erscheint |
| --- | --- | --- |
| kurzer Text | ein `CHAR`-Element, z. B. `TEXT132` | Eingabefeld, Länge aus dem DDIC |
| langer Text | ein `STRING`-Element | mehrzeiliges Feld, beliebig lang |
| Ja/Nein | `XFELD` | Ankreuzfeld |
| feste Auswahl | eigenes Element mit **Domänen-Festwerten** | Auswahlliste, übersetzt; ab 20 Einträgen mit Suche |
| Auswahl aus Customizing | Element mit **Prüftabelle**, z. B. `WERKS_D`, `EKGRP` | Auswahlliste bei kleinen Tabellen, sonst Eingabe mit Prüfung |
| Datum | `DATUM` | Kalender |
| Uhrzeit | `UZEIT` | Uhrzeitfeld |
| Betrag | `WRBTR` | Betragsfeld mit Währung |
| Menge | ein `QUAN`-Element | Zahlenfeld mit Einheit |
| eine kleine Tabelle | eine eigene **DDIC-Struktur** | Tabelle mit einer Spalte je Feld |

{% hint style="success" %}
**Ein eigenes Datenelement lohnt sich.** Für eine Auswahlliste legen Sie eine Domäne mit Festwerten an und übersetzen die Texte dort — dann ist die Liste in jeder Anmeldesprache richtig, die F4-Hilfe kommt gratis, und die Prüfung beim Speichern benutzt **dieselbe** Liste, die die Oberfläche zeigt.
{% endhint %}

### 3 · Die Gruppe an den Schritt hängen

Im Knoten **Genehmigungsschritte** (`/C09/CFL_C01`) trägt das Feld `VIEW_ID` ein, welche Gruppe ein Schritt zeigt.

**Ein Schritt ohne Eintrag zeigt kein Formular und läuft unverändert.** Sie können ein Feature also an einem Schritt einschalten und an allen anderen nichts ändern.

Dasselbe Element darf in mehreren Gruppen stehen — im einen Schritt als Eingabe, im nächsten als Anzeige. So wird aus dem Antrag des einen die Entscheidungsgrundlage des anderen, ohne die Angabe zu kopieren.

### 4 · Fiori einschalten

**Im SAP GUI erscheint der Bereich ohne jede Einrichtung** — Schritt 4 betrifft nur die Fiori My Inbox. Dort liefert conFLOW die App mit; einzurichten ist sie **einmal je System**, danach ist jeder weitere Workflow eine Zeile Customizing.

#### Wie es zusammenhängt

```
My Inbox, Workitem angeklickt
     │
     │  Task TS00388601 ist auf dynamische Visualisierung gestellt (SWFVMD1)
     │  und liest drei Container-Elemente des Workitems
     ▼
  conFLOW hat sie beim Anlegen gesetzt — aus C08-VISU
     │     /C09/CFL_CL_VISU_0101, im Workitem-Exit vor dem BAdI
     ▼
  Intent:  <SemanticObject>-openInInbox?CFLQueryObject00=<Vorgang>
     │
     │  Target Mapping im Launchpad löst ihn auf
     ▼
  App  c09.cfl.inbox   (BSP /C09/CFL_INBOX)
     │  OData V2
     ▼
  /C09/CFL_INBOX_SRV → Datenschicht → Feldkern → Container /C09/CFL_S04
```

**Welche App erscheint, entscheidet das einzelne Workitem**, nicht der Task. Deshalb können in derselben Inbox Workflows mit Formular und ohne nebeneinander liegen. Und deshalb bekommen **nur neue Workitems** die Weiterleitung: sie wird beim Anlegen in den Container geschrieben.

#### Einmal je System

| # | Was | Transaktion |
| --- | --- | --- |
| 1 | Modell registrieren: `/C09/CFL_INBOX_MDL`, Model Provider Class `/C09/CFL_CL_INBOX_MPC` | `/IWBEP/REG_MODEL` |
| 2 | Service registrieren: `/C09/CFL_INBOX_SRV`, Data Provider Class `/C09/CFL_CL_INBOX_DPC_EXT`, Modell zuordnen | `/IWBEP/REG_SERVICE` |
| 3 | Service aktivieren und `$metadata` aufrufen — kommt das Modell zurück, steht die Hälfte | `/IWFND/MAINT_SERVICE` |
| 4 | Target Mapping anlegen (**keine Kachel**) | `/UI2/FLPD_CUST` |
| 5 | Den Katalog in die Rolle der Bearbeiter | `PFCG` |

{% hint style="danger" %}
**Schritt 1 bis 3 gehen nicht mit dem Transport.** Klassen und App kommen mit, die Registrierung ist eine Systemeinstellung und muss in jedem System nachgeholt werden. Fehlt sie, meldet die App *Service nicht gefunden* — und niemand weiß warum.
{% endhint %}

#### Die Namen, und was womit übereinstimmen muss

| Wert | Wo er steht | Woher er kommt |
| --- | --- | --- |
| **Semantic Object** | Target Mapping **und** `C08-VISU` | **frei gewählt** — beide Stellen müssen exakt gleich lauten. Kein `/` erlaubt, einen Namensraum also als Präfix schreiben |
| **Action** `openInInbox` | Target Mapping | **fest im Produkt** (`/C09/CFL_CL_VISU_0101`) — nicht ändern |
| **ID** `c09.cfl.inbox` | Target Mapping | die **Component-ID** der App, nicht der BSP-Name |
| **URL** `/sap/bc/ui5_ui5/c09/cfl_inbox` | Target Mapping | die BSP-Anwendung `/C09/CFL_INBOX` |
| **Parameter** `openMode` | Target Mapping, als **Value**, nicht als Default Value, Pflicht angehakt | Wert `embedIntoDetailsNestedRouter`. Fehlt er, bleibt der Detailbereich **weiß** |
| *Allow additional parameters* | Target Mapping | angehakt — die App liest die Vorgangs-ID selbst aus der URL |

{% hint style="info" %}
**Der Eintrag in `/UI2/SEMOBJ` ist Komfort, keine Voraussetzung.** Aufgelöst wird der Intent über das Target Mapping; die Tabelle liefert die Wertehilfe im Launchpad Designer. Ohne Eintrag läuft alles — nur muss der Name beim Anlegen des Target Mapping exakt getippt werden, und ein Tippfehler dort meldet sich nirgends. Eine Minute, die sich lohnt.
{% endhint %}

#### Berechtigungen

| Wer | Was er braucht |
| --- | --- |
| Der Bearbeiter | die Fiori My Inbox wie bisher, **plus** den Katalog mit dem Target Mapping in einer seiner Rollen — sonst löst der Absprung nicht auf und das Workitem sieht aus wie vorher |
| Der Bearbeiter | Zugriff auf den OData-Service `/C09/CFL_INBOX_SRV`, wie für jeden Gateway-Service |
| Niemand | **eine eigene conFLOW-Berechtigung.** Wer entscheiden darf, darf eingeben |

**Das letzte ist wichtiger, als es aussieht.** conFLOW prüft nicht über ein Berechtigungsobjekt, sondern über das Workitem: schreiben darf, wer **aktueller Bearbeiter des offenen Workitems** ist — oder dessen **aktive Vertretung**. Alle anderen sehen dasselbe Formular als reine Anzeige, mit dem Hinweis *Nur Anzeige — dieses Workitem ist nicht für Sie offen*. Wer also nach einer Rolle sucht, die „Eingabefelder darf schreiben" erlaubt, sucht vergeblich; es gibt sie nicht und sie wäre auch falsch.

#### Und dann je Workflow

Das Semantic Object in `C08-VISU` eintragen. Das ist alles.

**Das BAdI wird dafür nicht gebraucht.** `set_inbox_ui( )` bleibt für den Sonderfall, dass ein Schritt eine *andere* App öffnen soll.

## Ein Antrag ohne Beleg

Ein Urlaubsantrag, ein Schulungsantrag, eine Meldung — Vorgänge, zu denen es keinen SAP-Beleg gibt. Dann füllt ein Formular die Werte, **bevor** der Workflow startet:

| Parameter in `C08` | Wert |
| --- | --- |
| `VIEW_ID` | die Feldgruppe des Antragsformulars |
| `TYPEID` | ein eigener Objekttyp für den Vorgang |
| `NRANGE` | ein Nummernkreisobjekt; conFLOW zieht den Schlüssel daraus, Intervall `01` |

**`NRANGE` ist nicht Komfort, sondern notwendig.** conFLOW prüft beim Start über Definition, Objekttyp und Schlüssel, ob schon ein Vorgang offen ist. Ohne eigenen Schlüssel fände der zweite Antrag den ersten und würde als Doppelstart abgelehnt — ein Fehler, der erst beim zweiten Antrag auftritt.

Die Werte gehen dem Start **mit**, nicht hinterher. Ein erster Hintergrundschritt kann also sofort mit ihnen rechnen und seine Regel darauf anwenden.

## Prüfen, in dieser Reihenfolge

1. **Im SAP GUI**, am Workitem: erscheint das Formular? Stehen die Bezeichnungen in Ihrer Sprache? Sind Pflichtfelder mit Stern markiert?
2. **Entscheiden, ohne ein Pflichtfeld zu füllen** — die Entscheidung muss abgelehnt werden, mit einer Meldung, die das Feld nennt.
3. **In `/C09/CFL_S04` nachsehen:** dort steht der Wert **intern** — ein Datum als `JJJJMMTT`, eine Zahl mit Punkt. Das ist richtig so; die Oberfläche rechnet um.
4. **In `/C09/CFL_S05`:** je Änderung eine Zeile mit altem Wert, neuem Wert, Benutzer und Kanal.
5. **Erst dann Fiori.** Wenn etwas im SAP GUI nicht stimmt, liegt es nicht an der App.

## Die Fallen

{% hint style="warning" %}
**Texte gehören in die `*t`-Tabellen**, nicht in `C13` oder `C14` selbst. Wer sie in der Stammtabelle sucht, findet sie nicht — und wer sie dort pflegen will, verliert sie bei der nächsten Sprache.
{% endhint %}

**Pflicht greift beim Start und beim Entscheiden, nicht beim Speichern.** Wer ein Pflichtfeld leert, darf das speichern. Sonst zeigte das Bild leer, im Container stünde der alte Wert, und entschieden würde mit dem alten — obwohl der Bearbeiter etwas anderes gesehen hat.

**Eine Container-Zeile fasst 132 Zeichen.** Längere Werte verteilt conFLOW selbst auf mehrere Zeilen und setzt sie beim Lesen wieder zusammen; Sie merken davon nichts. Nur wer **selbst** in den Container schreibt, muss selbst verteilen.

**Ein Anzeigefeld ist einmal beschreibbar: beim Start.** Danach wird ein Wert dafür abgelehnt, nicht stillschweigend verworfen. So bleibt eine Originaleingabe nachvollziehbar.

**Prüfen Sie die zweite Sprache.** Was nur in einer Sprache existiert, fällt genau dem nicht auf, der es gepflegt hat.

## Wenn das Customizing nicht reicht

Abhängige Wertelisten, kontextabhängige Pflicht, abgeleitete Vorschläge: dafür gibt es den **Feld-Exit** — eine Klasse hinter dem Parameter `FIELD_EXIT` in `C08`, die die fertige Feldliste noch einmal in die Hand bekommt, bevor die Oberfläche sie zeichnet.

Eine **eigene Oberfläche** brauchen Sie erst für etwas anderes als Felder: eine Belegvorschau, eine Simulation, eine Bedienung, die es so nicht gibt. Der Weg dorthin steht im [How-To zur Fiori-Oberfläche](fiori-oberflaeche-im-workitem/README.md).

Die Tabellen, das Datenmodell und die Grenzen im Einzelnen: [Technische Dokumentation, Abschnitt 14](../technische-dokumentation/technische-dokumentation.md).
