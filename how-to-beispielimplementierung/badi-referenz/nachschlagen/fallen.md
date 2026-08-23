# Die Fallen

Gesammelt aus dieser Referenz und aus Produktivimplementierungen. Sie
haben eines gemeinsam: **keine erzeugt eine Fehlermeldung.** Deshalb
kosten sie Zeit.

## Beim Bauen

**1 · `US` vergessen**

Ein Eintrag in `CT_ACTORS` ist immer ein typisiertes Org-Objekt -
`US<uname>`, `S<planstelle>`, `AC<rolle>`. Ein blanker Benutzername
erzeugt kein Workitem und keinen Fehler. Es landet einfach bei
niemandem.

**2 · Inline-`DATA()` an einem klassischen `CALL FUNCTION`**

An einem `CALL FUNCTION ... IMPORTING` ist eine Inline-Deklaration
nicht erlaubt. In einem conFLOW-BAdI sind solche Aufrufe häufig.
Dieser Fehler meldet sich immerhin beim Aktivieren.

**3 · Der Container ohne `C07`**

`SET_ATTRIBUT_VALUE` schreibt nur Elemente, die in `/C09/CFL_C07`
gepflegt sind. Fehlt der Eintrag, wird der Wert verworfen - kein
Fehler, kein Satz in `S04`, und beim Lesen kommt leer zurück.

**4 · Die BAdI-Implementierung ohne Filter**

Dann läuft die Klasse für jeden Workflow im System.

## Bei der Anzeige

**5 · `<b>` statt `<H>`**

Der Workitem-Text ist SAPscript-ITF, kein HTML - obwohl der
Parametertyp `/C09/CFL_HTML_TABLE_TT` heißt. `<b>` wird
stillschweigend entfernt: kein Fett, kein sichtbares Tag, keine
Meldung. Richtig ist `<H>Text</>`, geschlossen mit `</>`.

**6 · Nur einen der beiden Objekt-Link-Hooks pflegen**

`GET_OBJECT_INFO` wirkt in Fiori, `GET_NEW_PREVIEW_DESCR` im
SAP-GUI. Wer einen weglässt, hat in der anderen Oberfläche den
Framework-Klassennamen stehen.

**7 · Den Präfix ungeprüft abschneiden**

`CV_DESCRIPTION` kommt mit 13 Zeichen Technik davor. Ist der Text
kürzer, läuft der Offset ins Leere - und der Kurzdump kommt zur
Anzeigezeit.

**8 · `DECISION_TEXT` anfassen**

In `GET_FIORI_TASK_DEC_OP_ACT` prüft das Framework direkt nach dem
Aufruf, ob der Text noch ein `-` enthält, und überspringt sonst die
`C09T`-Übersetzung. Ergebnis: unbeschriftete Buttons.

**9 · Buttons über den Text erkennen**

Zum Zeitpunkt des Hooks steht dort schon der übersetzte Text.
`DECISION_KEY` ist der stabile Schlüssel.

## Zur Laufzeit

**10 · `SAP_WAPI_CHANGE_WORKITEM_PRIO` im After-Create-Hook**

Die WAPI liest `SWWWIHEAD` von der Datenbank. Dort steht das Workitem
zu diesem Zeitpunkt noch nicht. Der Aufruf läuft still ins Leere.

**11 · Das fehlende `COMMIT` nach `SET_WORKITEM_OBSOLET`**

Die Methode ruft `SAP_WAPI_WORKITEM_COMPLETE` mit
`DO_COMMIT = FALSE`. Ohne das `COMMIT` des Aufrufers bleiben die
Workitems offen.

**12 · `EV_DECISION_KEY` nicht gesetzt**

Der häufigste Grund für "der Workflow hängt im Hintergrundschritt".

**13 · Eine lokale Struktur an `ADD_DATASOURCE_MAIL`**

Die Methode arbeitet über RTTI und braucht einen DDIC-Header, um die
Platzhalternamen zu bauen. Eine lokale Struktur hat keinen und wird
kommentarlos ignoriert - das Mail hat dann Lücken.

**14 · Formatierte Belegnummern als Suchschlüssel**

Eine Leseroutine, die für die Anzeige formatiert, taugt nicht als
Schlüsselquelle. Ohne führende Nullen findet der `SELECT` nichts -
und meldet keinen Fehler, sondern liefert eine leere Tabelle.

**15 · Where-Used glauben**

`GET_OBJECT_INFO` und `GET_FIORI_TASK_DEC_OP_ACT` sehen im
Aufrufnachweis tot aus, weil sie über Enhancements gerufen werden.
Sie laufen trotzdem. Bei conFLOW-BAdI-Hooks: **ausprobieren statt
Where-Used glauben.**

**16 · Priorität 1**

SAP verschickt Workitems der Priorität 1 als Express-Nachricht. Im
Testsystem fällt das nicht auf; im Produktivsystem bekommt der
Bearbeiter ein Popup. 4 ist die höchste Stufe ohne diesen Effekt.
