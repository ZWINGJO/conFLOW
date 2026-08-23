# Fünfzehn Sätze, die sich bewährt haben
*Jeder steht für einen Fehler, der einmal Zeit gekostet hat.*
| 1 | **Erst den Service beweisen, dann die App bauen.** Zehn Sekunden im Browser schließen die halbe Fehlerklasse aus. |
| --- | --- |
| 2 | **Keine stillen Fallbacks.** Eine Notfassung, die heimlich einspringt, verdeckt genau den Fehler, den man nicht gebrauchen kann. |
| 3 | **Regeln ins Backend.** Was der Browser weiß, ist Bequemlichkeit. Was beide wissen müssen, gehört als *Feld* in die Entity — nicht als Kopie in beide. |
| 4 | **Bei „läuft durch, tut aber nichts": zuerst prüfen, ob System und Repo dasselbe sagen.** |
| 5 | **Ein Aufruf ohne `.then` und `.catch` hat das Netz nie verlassen.** |
| 6 | **Inkognito zeigt den Server, das normale Fenster den Cache.** Beide alt → App-Index. Nur eines alt → Browser-Storage. |
| 7 | **Beim Speichern ist der Client die Wahrheit.** Nach erfolgreichem Schreiben nicht nachladen — der Cursor steht möglicherweise noch im Feld. |
| 8 | **Ein UI-Feld darf nie einen Zustand suggerieren, den das Backend nicht besitzt.** Vorbelegung entweder persistieren oder sichtbar als Vorschlag kennzeichnen. |
| 9 | **Wertelisten sind Client-Komfort.** Was gespeichert werden darf, prüft der Server — gegen dieselbe *Liste*, nicht nur dieselben Konstanten. |
| 10 | **Detail-Apps verlangen zwingend einen Instanzschlüssel.** Ein ungefilterter Listenmodus nur bei echtem Anwendungsfall. |
| 11 | **Erst prüfen, ob eine Funktion gebraucht wird, bevor man sie absichert.** Die wirksamste Änderung im Referenzprojekt war eine Streichung. |
| 12 | **Die letzte fachliche Schranke gehört dorthin, wo der Zustandsübergang passiert.** |
| 13 | **Eine Prüfung, die *speichern* verhindert, ist etwas anderes als eine, die *abschließen* verhindert.** |
| 14 | **Eine Begründung, die niemand am Bildschirm geprüft hat, ist keine Begründung.** Sie steht als Kommentar im Code, wird beim Dokumentieren abgeschrieben und trägt irgendwann Objekte, die niemand braucht. |
| 15 | **Bei Framework-Verhalten im Bundle nachlesen, nicht schließen.** Ein Screenshot beweist nur, was gerade konfiguriert ist — im Referenzprojekt führte er einmal in jede Richtung. |
