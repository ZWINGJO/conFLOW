# Was hier gebaut wird
*Ein Genehmigungs-Workitem zeigt in der SAP Fiori My Inbox statt eines Textblocks eine eigene Anwendung — mit gegliedertem Kontext, Ampel und einem Eingabebereich, der direkt in den Workflow-Container schreibt.*

| conFLOW-Standard | mit eigener Oberfläche |
| --- | --- |
| Ein Textblock im Beschreibungsfeld, gerendert aus SAPscript-ITF. Gegliedert, aber Fließtext — und ohne Eingabemöglichkeit. | Kopfzeile mit Ampel und Empfehlung, vier Reiter aus dem Container, ein Bereich **Your decision** mit Auswahlfeld und Notiz, der sofort speichert. Darunter unverändert die conFLOW-Buttons. |

Alle Werte stammen aus demselben Container und über dieselben Routinen, die auch den Textblock füllen — **eine Quelle, ein Rechenweg**. Zwei Rechenwege für dieselbe Zahl fallen erst im Kundentermin auf.

## Die Objekte
| Objekt |  | Rolle |
| --- | --- | --- |
| ZCFL_NNNNN_C_<Sache> | DDLS | Custom Entity — Anzeige-, Entscheidungs- und Statusfelder |
| ZCFL_NNNNN_C_<Sache>VH | DDLS | Custom Entity für die Werteliste |
| ZCFL_NNNNN_D_<Aktion> | DDLS | Abstract Entity — Parameter der Aktion |
| ZCFL_NNNNN_X_<Sache> | DDLX | Metadata Extension — Facets, Feldgruppen, Criticality |
| ZCFL_NNNNN_C_<Sache> | BDEF | Behavior Definition, `unmanaged`, eine Aktion |
| ZCL_CFL_NNNNN_RULES | CLAS | **die Prozessregeln** — von Query-Provider und Handler gerufen |
| ZCL_CFL_NNNNN_QUERY | CLAS | Query-Provider der Hauptentität |
| ZCL_CFL_NNNNN_VH | CLAS | Query-Provider der Werteliste |
| ZCL_CFL_NNNNN_BEHV | CLAS | Behavior Pool — global + Local Types |
| ZCFL_NNNNN_SD / _SB | SRVD | Service Definition und Binding, OData V4 · UI |
| ZCFL_NNNNN_UI | BSP | die UI5-Anwendung — **gebaut aus einem Quellprojekt**, nicht in SE80 gepflegt |
| ZCL_CFL_WORKFLOW_NNNNN | CLAS | conFLOW-BAdI — hier nur die Teile, die die App anbinden |
Dazu außerhalb von ABAP: Semantic Object, Katalog mit Target Mapping, Rolle.

{% hint style="info" %}
**Alle Quellen in diesem Dokument sind vollständig** und werden beim Erzeugen aus `src/` gelesen — nicht abgetippt. Das Dokument kann deshalb nicht vom Code abweichen.
{% endhint %}

## Woher die Beispiele stammen — und wie weit sie tragen
Jede Quelle in diesem Dokument ist echter, laufender Code aus **einem** Workflow: `ZCFL_00500`, „Order Promise Exception" — ein Vertriebs-Showcase mit Backorder-Ausnahmen aus dem Verkaufsbeleg. Belege wie `14668/10`, Material `MX_6122` und Kunde `BP1010` sind Demo-Daten aus dem eigenen System, kein Kundenprojekt.

Das ist Absicht und keine Einschränkung: **ein durchgehendes Beispiel trägt weiter als zwanzig Ausschnitte.** Wer nachbaut, sieht dieselben Objekte in jedem Kapitel wieder und kann sie Stück für Stück auf den eigenen Fall übersetzen. Was fachlich ist — Ampel, Empfehlung, Backorder-Regeln — steht in den Beispielen an genau den Stellen, wo im eigenen Workflow etwas anderes hingehört.

| Im Beispiel | Beim Nachbau |
| --- | --- |
| Severity, Empfehlung, Backorder-Prozent | die eigenen Container-Elemente — **austauschbar** |
| Sechs Entscheidungsgründe | die eigene Werteliste — **austauschbar** |
| Schritte `01`, `02`, `03` | die eigenen `gen_stat`-Codes — **austauschbar** |
| Query-Provider, Behavior Pool, Custom Section, Regelklasse | **Struktur bleibt** — nur die Feldnamen wechseln |
| Die siebzehn Fallen | **gelten unverändert** — sie hängen am Framework, nicht am Prozess |

{% hint style="info" %}
**Stand und Gültigkeit.** Erzeugt aus dem Referenzprojekt zu `ZCFL_00500`; dort liegt die Quelle und dort läuft der Generator (`tools/build_howto_doc.py`). Eine Kopie dieses Dokuments an anderer Stelle ist eine **Momentaufnahme** — bei Änderungen am Referenz-Workflow neu erzeugen, sonst driftet sie vom Code weg. Getestet auf S/4HANA mit `minUI5Version 1.136`; die Annotations-Scopes und das Verhalten der My Inbox sind releaseabhängig und im Zielsystem zu prüfen.
{% endhint %}

## Braucht es überhaupt eine App?
Nicht jedes Workitem verdient eine. Der conFLOW-Textblock kann mehr, als man denkt: Gliederung, fette Überschriften (SAPscript-ITF, `<H>Text</>`), Ampel als Emoji, Priorität, farbige Buttons, sprechenden Objekt-Link.
Der Umbau lohnt, wenn **mindestens zwei** zutreffen:

  - der Bearbeiter soll **etwas eingeben**, das in keinen Button passt
  - der Kontext braucht **Reiter** statt Fließtext
  - es gibt Werte zum **Auswählen** statt Freitext
  - die Optik ist Teil des Auftrags

{% hint style="info" %}
Trifft nur **eines** zu, ist der Textblock die bessere Investition. Er kostet einen Nachmittag, die App ein Projektkapitel — und sie bringt einen Auslieferungsprozess mit, den es vorher nicht gab: Repository, Build, Deploy. Siehe [Kapitel „Das Quellprojekt"](oberflaeche/20-quellprojekt.md).
{% endhint %}
