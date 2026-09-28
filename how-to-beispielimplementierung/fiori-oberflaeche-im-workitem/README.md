# Eigene Fiori-Oberfläche statt der mitgelieferten

*Wann eine eigene App nötig ist, was conFLOW dafür anbietet — und was Sie nicht nachbauen sollten.*

## Erst die Frage: brauchen Sie sie?

**Für Eingabefelder nicht.** Ein Auswahlfeld, ein Datum, ein Betrag, eine Notiz, eine kleine Tabelle: das macht eine Feldgruppe im Customizing, und die mitgelieferte App zeigt sie in der Fiori My Inbox — siehe [Eingabefelder am Workitem einrichten](../eingabefelder-einrichten.md).

Eine eigene Oberfläche lohnt sich, wenn der Bearbeiter etwas braucht, das **kein Feld** ist:

- eine Vorschau auf den Beleg, eine Zeichnung, ein Dokument
- eine Simulation — was passiert, wenn ich so entscheide
- eine Bedienung, die es so nicht gibt: eine Karte, ein Zeitstrahl, ein Vergleich zweier Belege

## Was conFLOW dafür anbietet

### 1 · Die Weiterleitung, ohne eine Zeile Code

Der Parameter `VISU` in `C08` benennt das Semantic Object. Das Framework setzt beim Anlegen des Workitems drei Container-Elemente: Semantic Object, Action und die Vorgangs-ID als 32-stellige Hex-Kette — so passt sie durch die URL.

**Das gilt für jede App, die ein Target Mapping hat**, auch für Ihre eigene. Der Weg ist derselbe wie für die mitgelieferte; nur URL und Component-ID im Target Mapping sind Ihre.

### 2 · Eine andere App je Schritt

Soll nicht der ganze Workflow dieselbe Oberfläche öffnen, sondern jeder Schritt eine andere, setzen Sie die drei Container-Elemente selbst — im BAdI-Hook [`get_after_creation_workitem`](../badi-referenz/lebenszyklus/get-after-creation-workitem.md), der einmal je Workitem läuft, nach dem Anlegen und vor der ersten Anzeige.

{% hint style="success" %}
**Fangen Sie Fehler dort still ab.** Eine Oberfläche, die nicht kommt, darf kein Workitem verhindern — ohne die Container-Elemente zeigt die Inbox einfach ihren Textblock, und das ist ein brauchbarer Zustand.
{% endhint %}

### 3 · Die Daten

Ihre App liest und schreibt nicht selbst auf den Tabellen, sondern über das Produkt:

| Wofür | Was |
| --- | --- |
| Felder einer Feldgruppe lesen, prüfen, schreiben | `/C09/CFL_CL_FIELDS_0101` — `GET_FIELDS`, `CHECK_VALUES`, `SAVE` |
| einen langen Wert lesen, über mehrere Zeilen verteilt | `/C09/CFL_CL_FIELDS_0101=>READ_VALUE` |
| freie Container-Werte | `GET_ATTRIBUT_VALUE` / `SET_ATTRIBUT_VALUE` der Workflow-Klasse |
| den Workitem-Text, wie ihn SAP GUI und Inbox zeigen | `SAP_WAPI_WORKITEM_DESCRIPTION` |

**Schreiben Sie immer über `SET_ATTRIBUT_VALUE`** — nur so entsteht das Änderungsprotokoll in `/C09/CFL_S05`, mit Benutzer, Zeitpunkt und Kanal. Wer daran vorbeischreibt, verliert den Audit Trail, und zwar unbemerkt.

## Die Einrichtung ist dieselbe

Service, Semantic Object, Target Mapping, Katalog, Berechtigungen, `openMode`, Notaus: alles steht in [Eingabefelder am Workitem einrichten, Schritt 4](../eingabefelder-einrichten.md). Für Ihre App ändern sich nur zwei Werte im Target Mapping — die URL Ihrer BSP-Anwendung und die Component-ID.

## Was Sie nicht nachbauen sollten

| | Warum nicht |
| --- | --- |
| Die Entscheidungsknöpfe | die kommen vom Task-Gateway, samt Farbe und Kommentarpflicht aus `C09` |
| Die Entscheidung selbst | entschieden wird mit den Knöpfen der Inbox; Ihre App hält nur fest, **warum** |
| Den Workitem-Text | `SAP_WAPI_WORKITEM_DESCRIPTION` liefert genau den Block, den SAP GUI und Inbox zeigen. Ändert er sich im Customizing, zieht Ihre App ohne Änderung mit |
| Die Pflichtprüfung | sie hängt im Kern und greift in beiden Oberflächen und über die Workflow-API |
| Das Protokoll | `S03`, `S04`, `S05` entstehen von selbst |

**Eine App, die entscheidet, ist der teuerste Fehler dieser Bauweise.** conFLOW bleibt die eine Stelle, an der Bearbeiterfindung, Fristen, Vertretung und Protokoll zusammenlaufen. Ihre App zeigt und erfasst — mehr nicht.

## Voraussetzung

| | |
| --- | --- |
| ABAP | SAP_BASIS **7.50**, SAP_GWFND — kein RAP nötig |
| UI5 | SAPUI5 **1.71** oder neuer |
| Inbox | Fiori My Inbox (BSP `CA_FIORI_INBOX`) |

Ohne Gateway-Komponente läuft alles andere weiter: SAP GUI, Mail, conMOBILE. Nur die Fiori-Oberfläche entfällt.
