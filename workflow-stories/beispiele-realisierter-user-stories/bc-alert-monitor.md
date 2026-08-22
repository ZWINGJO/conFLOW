# BC Alert Monitor

## Die Anforderung

Systemwarnungen (Alerts) im SAP-Basis-Betrieb muessen zuverlaessig beim richtigen Bearbeiter ankommen und nachvollziehbar behandelt werden. Klassische Alerts laufen in den CCMS Alert Monitor oder per E-Mail -- beides ohne Quittierung, ohne Eskalation und ohne dokumentierten Bearbeitungsweg.

## Was conFLOW hier leistet

Der Workflow nimmt einen Alert entgegen und erzeugt ein Workitem fuer den zustaendigen Basis-Betreuer. Wird der Alert nicht innerhalb einer definierten **Frist** bearbeitet, eskaliert conFLOW automatisch an die naechste Stufe. Die Frist wird im Customizing gepflegt, nicht im Code -- eine Anpassung erfordert keinen Transport.

Am Ende steht ein vollstaendiger Audit Trail: wann wurde der Alert ausgeloest, wer hat ihn wann bearbeitet, wie wurde entschieden. Das ist fuer regulierte Umgebungen und fuer die Auswertung wiederkehrender Probleme gleichermassen relevant.

### User Story

<figure><img src="../../.gitbook/assets/Folie11 (1).png" alt="User Story BC Alert Monitor"><figcaption><p>Fachliche User Story: Systemueberwachung mit Eskalation</p></figcaption></figure>

### Workflow

<figure><img src="../../.gitbook/assets/Folie12 (2).png" alt="Workflow BC Alert Monitor"><figcaption><p>Technischer Laufweg mit Frist und Eskalation</p></figcaption></figure>
