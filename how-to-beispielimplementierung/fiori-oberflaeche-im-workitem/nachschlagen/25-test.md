# Testmatrix — vor jeder Übergabe
*Die Fälle 2 bis 5 findet man über die Oberfläche nicht. Sie brauchen einen direkten OData-Aufruf.*
| # | Fall | Soll-Ergebnis |
| --- | --- | --- |
| 1 | Normale Instanz über die Inbox lesen | Kontext vollständig, Werte = Textblock |
| 2 | **Fremde GUID lesen** (zweiter Benutzer mit Rolle) | sollte abgewiesen werden |
| 3 | **Fremde GUID ändern** | dito |
| 4 | **Read-only-Schritt per OData ändern** | Abweisung mit Klartext |
| 5 | **Ungültigen Action-Key senden** | Abweisung — Wertehilfe ist keine Zusicherung |
| 6 | Pflichtfeld-Aktion ohne Zusatzfeld | **wird gespeichert**, Hinweis gelb |
| 7 | Zwei Saves unmittelbar hintereinander | beide kommen an, letzter Stand gewinnt |
| 8 | Tippen und sofort den Workflow-Button drücken | Workflow läuft; Eingabe *kann* verloren gehen |
| 9 | **Vorbelegung unverändert lassen, dann abschließen** | Container enthält den Wert — **nicht leer** |
| 10 | Zwischen zwei Workitems wechseln | Inhalt *und* Buttons wechseln |
| 11 | Notaus ziehen | sauberer Rückfall auf den Textblock |
| 12 | Rolle fehlt | Intent löst nicht auf, Textblock |
| 13 | **Workitem-Wechsel während eines Saves** | Save wird fertig; **keine** Meldung im anderen Workitem |
| 14 | **Dieselbe Instanz in zwei Tabs** | *last writer wins*, ohne Warnung — bewusst so |
| 15 | **„Show Details" in der Fußleiste** | rechts erscheinen Notizen, Anhänge, Objektlinks — links bleibt die App |
| 16 | Im Anhänge-Reiter eine Datei hochladen | geht durch, **ohne** Virenscan-Meldung — der Standard nutzt OData V2 |
| 17 | **Nach dem Deploy: Netzwerk-Tab beim Öffnen des Workitems** | `Component-preload.js` wird geladen — **nicht** die Einzeldateien. Wird es nicht, läuft ein alter Stand |
{% hint style="success" %}
**Fall 9 ist der wertvollste.** Er prüft, ob ein angezeigter Wert auch ein gespeicherter Wert ist. Ein Test, der nur über die Oberfläche geht, findet ihn nie — dort sieht alles richtig aus.
{% endhint %}
{% hint style="info" %}
**Fall 14 soll nicht „bestehen"**, sondern das erwartete Verhalten belegen. Es gibt keinen Konfliktschutz, und das ist eine dokumentierte Entscheidung.
{% endhint %}
{% hint style="success" %}
**Fall 15 und 16 gehören an den Anfang, nicht ans Ende.** Sie prüfen nicht die eigene App, sondern was der Standard ohnehin liefert — und beantworten damit die Frage, ob man ein Stück Funktionalität überhaupt bauen muss. Im Referenzprojekt wurde das zu spät gefragt und kostete fünf Objekte.
{% endhint %}
