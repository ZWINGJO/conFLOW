# Schritt 0 — die Vertragsmatrix
*Vor der ersten Objektanlage. Zwanzig Minuten am Whiteboard; jede dieser Fragen hat im Referenzprojekt später Zeit gekostet, weil sie nicht vorher gestellt wurde.*
| Fachlicher Instanzschlüssel? | eine Instanz je *was* — Auftrag, Position, Beleg? |
| --- | --- |
| Wie kommt er ins Frontend? | Container → SWFVMD1 → Intent → Target Mapping |
| Passt er durch die URL? | GUID als `char(32)` Hex. Die *Anforderung* ist allgemein, der *Typ* nicht |
| Welche Werte nur lesen? |  |
| Welche schreiben? |  |
| **Wann gilt ein UI-Wert als persistiert?** | sofort · nach Pause · erst beim Abschluss |
| **Welcher Folgeschritt konsumiert sie?** | wenn keiner: das sagen. Nicht „ist vorgesehen" |
| Welche Framework-Tabellen lesend? | und was passiert beim Produkt-Upgrade? |
| Wer darf lesen / schreiben? | Rolle allein reicht nicht — sie gilt für den *Service*, nicht für die *Instanz* |
| Auf welchen Schritten? | `gen_stat`-Liste festlegen |
| Welche Aktionen zulässig, **woher die Liste**? | eine Quelle für Wertehilfe *und* Serverprüfung |
| **Was bedeutet ein vorbelegter Wert?** | Empfehlung · Arbeitswert · bestätigter Wert — **drei verschiedene Dinge** |
| **Was passiert mit einem laufenden Save,** wenn abgeschlossen oder gewechselt wird? | die Transaktionsgrenze zwischen UI und Workflow |
| Bietet das Framework einen Exit für den Zustandsübergang? | conFLOW: ja — dort gehört die finale Prüfung hin |
{% hint style="info" %}
**Die vier fett gesetzten Fragen** sind die, an denen das Referenzprojekt Fehler gemacht hat. Jede einzelne war später ein Befund mit Schweregrad S1 oder S2.
{% endhint %}
