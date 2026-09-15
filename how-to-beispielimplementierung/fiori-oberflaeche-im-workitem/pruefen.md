# Einspielen, prüfen, Notaus

*Nach jedem Schritt eine Probe. Hält sie nicht, lohnt der nächste Schritt nicht, und die Ursache ist noch in Sichtweite.*

## Reihenfolge mit Probe

| # | Schritt | Probe |
| --- | --- | --- |
| 1 | SWFVMD1 prüfen | `SE16` → `SWFVGTP`, `TASK = TS00388601`: `SEMANTIC_OBJECT`, `ACTION`, `QUERY_PARAM00` dynamisch vorhanden |
| 2 | Datenklasse anlegen, in der Workflow-Klasse zum `GLOBAL FRIEND` erklären | `SE24` → Methode `READ` mit der Instanz-GUID testen: Text, Gründe, `Editable` kommen |
| 3 | SEGW-Modell, generieren, DPC_EXT füllen | alle Klassen aktiv |
| 4 | Service registrieren | `/sap/opu/odata/sap/ZCFL_00500_V2_SRV/$metadata` zeigt `Decision`, `DecisionSet`, `SetDecision` |
| 5 | Service fachlich | `…/DecisionSet('<id>')?$format=json` liefert den Vorgang; eine fremde GUID liefert **404** |
| 6 | App bauen und hochladen, Caches | `/sap/bc/ui5_ui5/sap/zcfl_00500_uiv2/index.html` meldet „No work item key was passed". **Das ist der Erfolgsfall.** |
| 7 | Semantic Object, Target Mapping, Rolle | `…/flp#ZCFLOrderPromiseV2-openInInbox?CFLQueryObject00=<id>` öffnet den Vorgang |
| 8 | BAdI: `set_inbox_ui( )` einspielen | **neues** Workitem erzeugen; im Workitem-Container stehen die drei `/C09/CFL_VISU_*`-Werte |
| 9 | My Inbox | App im Detailbereich, conFLOW-Knöpfe darunter, „Show Details" in der Fußleiste |

`<id>` ist die conFLOW-Instanz (`/C09/CFL_S03-ID`) eines offenen Dialog-Workitems, als 32-stellige Hex-Kette.

## Abnahme im Workitem

| | Erwartung |
| --- | --- |
| Grund wählen | Statuszeile grün: *Saved hh:mm:ss* |
| Notiz tippen | nach einer Schreibpause erneut *Saved* |
| Pflichtangabe fehlt | gespeichert **und** gelber Hinweis, was zum Abschluss fehlt |
| Schritt ohne Eingabe (z. B. Eskalation) | Felder grau, Zeile sagt warum |
| zweites Workitem anklicken | Titel, Text **und** Felder wechseln, nicht nur die Knöpfe |
| anderer Benutzer, fremde GUID | nichts zu sehen, nichts zu schreiben |
| `/C09/CFL_S04` | Grund und Notiz stehen da, `AENAM` ist der Bearbeiter |

## Die Fallen

| Bild | Ursache |
| --- | --- |
| Workitem zeigt weiter den **Textblock** | Workitem älter als die Umstellung · Semantic Object anders geschrieben · `openMode` als *Default Value* · Rolle fehlt · SWFVMD1 leer |
| Detailbereich bleibt **weiß** | App-Fehler beim Start → Browser-Konsole. Gegenprobe: `openMode` auf `embedIntoDetails` (dann ohne „Show Details") |
| beim zweiten Workitem wechseln **nur die Knöpfe** | `navigateBasedOnStartupParameter( )` fehlt in `Component.js` |
| App zeigt eine **alte** Fassung | App-Index und Site data, siehe [Customizing](customizing.md) |
| ein Feld bleibt **leer**, ohne Fehler | ABAP-Feldname im SEGW ≠ Komponente der Struktur (`MOVE-CORRESPONDING`) · Metadaten-Cache nicht gelöscht |
| langer Text **abgeschnitten** | `Edm.String` **mit** Max Length erzeugt `CHAR(n)`; für Text ohne feste Länge das Feld leer lassen |
| Notiz nach 132 Zeichen **weg** | Grenze von `/C09/CFL_S04-VALUE` |
| Speichern meldet Fehler **ohne Text** | `/IWFND/ERROR_LOG` — dort stehen Text, Programm und Zeile |
| Funktionsbaustein wirft zur Laufzeit, Syntaxcheck war grün | Parameter mit `TYPE i` statt dem DDIC-Typ des Bausteins übergeben |
| in SE80 geänderte Datei **wirkt nicht** | die App läuft aus `Component-preload.js`; nur über Build und Upload ändern |

{% hint style="danger" %}
**Bevor etwas nachgebaut wird, das nach Workitem-Standard klingt** (Anhänge, Notizen, Objektlinks), erst auf **„Show Details"** klicken. Das Verhalten der My Inbox ist dabei **beobachtet, nicht zugesichert**: Es stammt aus deren Quelltext und nicht aus einer Erweiterungsschnittstelle. Nach einem UI5- oder S/4-Upgrade `openMode`, „Show Details" und den Workitem-Wechsel erneut prüfen.
{% endhint %}

## Notaus und Rückbau

| Ziel | Handgriff | wirkt auf |
| --- | --- | --- |
| sofort weg, auch in offenen Workitems | Target Mapping im Katalog löschen | **alle** Workitems, ohne Transport |
| neue Workitems ohne App | `mc_inbox_ui = abap_false` | nur **neue** Workitems |
| ganz entfernen | Target Mapping, Semantic Object, BSP-Anwendung, Service, SEGW-Projekt, Datenklasse, Aufruf im BAdI | — |

{% hint style="success" %}
**Der Rückfall ist keine Notlösung, sondern der Normalfall** für jedes Workitem ohne die drei Container-Elemente. Deshalb funktioniert er zuverlässig: Er ist ständig in Gebrauch und nicht nur im Fehlerfall.
{% endhint %}
