# BC Stammdaten Verteilung

## Die Anforderung

Stammdaten -- Materialstaemme, Konditionen, Organisationseinheiten -- muessen in einer verteilten Systemlandschaft konsistent gehalten werden. Aenderungen im fuehrenden System muessen geprueft, freigegeben und an die Empfaengersysteme verteilt werden. Fehler bei der Verteilung muessen erkannt und behandelt werden.

## Was conFLOW hier leistet

Der Workflow bildet den Freigabe- und Verteilungsprozess ab: eine Stammdatenaenderung wird erfasst, der zustaendige Fachbereich gibt sie frei, und ein Hintergrundschritt uebernimmt die technische Verteilung. Scheitert die Verteilung, erzeugt conFLOW ein Workitem fuer den Basis-Betreuer mit den relevanten Fehlerinformationen.

Die Meldungen aus Hintergrundschritten werden automatisch im Anwendungslog (SLG1) protokolliert und koennen dort ausgewertet werden -- ohne eigene Z-Tabelle.

### User Story

<figure><img src="../../.gitbook/assets/Folie13 (2).png" alt="User Story BC Stammdaten Verteilung"><figcaption><p>Fachliche User Story: Stammdatenverteilung mit Freigabe</p></figcaption></figure>

### Workflow

<figure><img src="../../.gitbook/assets/Folie14.png" alt="Workflow BC Stammdaten Verteilung"><figcaption><p>Technischer Laufweg: Freigabe und Verteilung</p></figcaption></figure>
