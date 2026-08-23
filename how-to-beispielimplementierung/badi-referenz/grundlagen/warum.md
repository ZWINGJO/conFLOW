# Warum dieses Beispiel

## Das Problem, das es löst

Ein conFLOW-Workflow beginnt mit einer Klasse, die
`/C09/CFL_IF_BADI_0101` implementiert. Beim Anlegen erzeugt die
Entwicklungsumgebung 26 leere Methodenrümpfe.

Danach steht man vor drei Fragen, und keine davon beantwortet die
Signatur:

1. **Welche Hooks braucht mein Workflow?** Die Namen helfen nur
   begrenzt - `GET_AFTER_EXECUTION` und
   `GET_AFTER_EXECUTION_WORKITEM` klingen gleich und sind es nicht.
2. **Was darf ich in einem Hook ändern?** Manche Parameter sind
   `CHANGING` und werden trotzdem ignoriert. Andere wirken nur in
   einer der beiden Oberflächen.
3. **Was passiert, wenn ich einen weglasse?** Meistens nichts
   Sichtbares - und genau das ist das Problem. Ein fehlender Hook
   erzeugt selten einen Fehler. Er erzeugt eine Oberfläche, die
   etwas anderes zeigt als gedacht.

## Was der Standard mitbringt

Im Paket `/C09/CONFLOW_BACKEND_0101` liegt
**`/C09/CFL_CL_BADI_0101`** - eine Beispiel-Implementierung des
BAdI. Sie enthält Fragmente aus echten Projekten: Bearbeiterfindung
über die Aufbauorganisation, Archivanhänge, ein Beispiel für
`GET_STATUS_DYNAMIC`.

Die Fragmente sind überwiegend **auskommentiert** und stammen aus
verschiedenen Installationen. Als Ideengeber taugen sie; als Vorlage
zum Kopieren nicht, weil sie auf Objekte verweisen, die es nur im
jeweiligen Projekt gab.

Diese Referenz ergänzt sie um das, was fehlt: eine Fassung, die
**vollständig, aktivierbar und in sich schlüssig** ist, und bei der
zu jedem Hook eine Aussage steht - auch zu denen, die leer bleiben.

## Was hier anders ist

**Ein durchgehender Prozess statt Einzelbeispielen.** Alle Hooks
beziehen sich auf denselben Beleg und dieselben Container-Attribute.
Man kann von oben nach unten lesen.

**Nur Standard-Abhängigkeiten.** Die Klasse ruft
`/C09/CFL_CL_WORKFLOW_0101` und `/C09/CFL_CL_HELPER_0101` - beide
gehören zur Auslieferung. Eigene Hilfsklassen gibt es nicht, außer
den beiden, die mitgeliefert werden.

**Syntaxgeprüft gegen ein echtes System.** Nicht "sollte gehen",
sondern übersetzt.

**Die leeren Hooks sind Inhalt.** Bei sechs von 26 lautet die
richtige Antwort "lass ihn leer". Warum, steht dabei.

## Was hier nicht steht

Das **Customizing**. Diese Referenz beschreibt die Klasse, nicht die
Tabellen `/C09/CFL_C01` bis `C10`. Ohne gepflegtes Customizing läuft
kein Workflow, egal wie gut die Klasse ist - die
[Einbau-Reihenfolge](../nachschlagen/einbauen.md) sagt, was
zusammengehört.
