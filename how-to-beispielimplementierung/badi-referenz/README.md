# Referenz: alle 26 BAdI-Methoden

Wer einen conFLOW-Workflow baut, legt als erstes eine Klasse an, die
`/C09/CFL_IF_BADI_0101` implementiert. Das Interface verlangt
**26 Methoden**. Sie heißen alle ähnlich, ihre Signaturen sagen
wenig darüber, wozu sie gut sind, und die meisten braucht man nicht.

Nur weiß man am Anfang nicht, welche.

Dieses Kapitel ist eine **vollständige, aktivierbare Beispielklasse**
mit allen 26 Hooks. Jeder trägt denselben Kopf: wann er läuft,
was er bekommt, was er ändern darf, und ob man ihn überhaupt braucht.

**9 Hooks sind absichtlich leer.** Das ist kein Rest, den noch
jemand ausfüllen muss - es ist die Antwort. Bei ihnen steht im Kopf,
wofür sie gedacht sind und woran man merkt, dass der eigene Fall
dazugehört. Ein Beispiel, das alle 26 Methoden füllt, wäre
bequemer zu lesen und in der Sache falsch.

## Was hier liegt

| | |
|---|---|
| `ZCL_CFL_WORKFLOW_00900` | die Referenzklasse - alle 26 Hooks, ein Hintergrundschritt, elf Helfer |
| `ZCL_CFL_CONST_00900` | die Konstantenklasse - jeder Schlüssel aus dem Customizing an einer Stelle |
| `ZCL_CFL_GET_ACTORS` | die Bearbeiterfindung, zentral statt in der Workflow-Klasse |

Alle drei sind gegen ein echtes System syntaxgeprüft. Abhängigkeiten
außerhalb des conFLOW-Standards gibt es keine.

## Wie man liest

Von vorne, wenn man den ersten Workflow baut - die Hooks stehen in
sechs Gruppen entlang des Ablaufs, nicht in der Reihenfolge des
Interface.

Über die [Landkarte](grundlagen/landkarte.md), wenn man einen
bestimmten Hook sucht.

## Der Beispielprozess

Ein **Freigabeprozess** - der Fall, für den conFLOW in neun von zehn
Projekten gebaut wird. Hier auf einem SAP-Standardbeleg, damit der
Datenteil keine Aufmerksamkeit kostet und die beim Workflow bleibt.
Siehe [Der Prozess](grundlagen/prozess.md).
