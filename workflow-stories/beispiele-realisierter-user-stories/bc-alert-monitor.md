# BC Alert Monitor

## Die Anforderung

Systemwarnungen (Alerts) im SAP-Basis-Betrieb müssen zuverlässig beim richtigen Bearbeiter ankommen und nachvollziehbar behandelt werden. Klassische Alerts laufen in den CCMS Alert Monitor oder per E-Mail -- beides ohne Quittierung, ohne Eskalation und ohne dokumentierten Bearbeitungsweg.

## Was conFLOW hier leistet

Der Workflow nimmt einen Alert entgegen und erzeugt ein Workitem für den zuständigen Basis-Betreuer. Wird der Alert nicht innerhalb einer definierten **Frist** bearbeitet, eskaliert conFLOW automatisch an die nächste Stufe. Die Frist wird im Customizing gepflegt, nicht im Code -- eine Anpassung erfordert keinen Transport.

Am Ende steht ein vollständiger Audit Trail: wann wurde der Alert ausgelöst, wer hat ihn wann bearbeitet, wie wurde entschieden. Das ist für regulierte Umgebungen und für die Auswertung wiederkehrender Probleme gleichermaßen relevant.

### User Story

<figure><img src="../../.gitbook/assets/Folie11 (1).png" alt="User Story BC Alert Monitor"><figcaption><p>Fachliche User Story: Systemüberwachung mit Eskalation</p></figcaption></figure>

### Workflow

<figure><img src="../../.gitbook/assets/Folie12 (2).png" alt="Workflow BC Alert Monitor"><figcaption><p>Technischer Laufweg mit Frist und Eskalation</p></figcaption></figure>
