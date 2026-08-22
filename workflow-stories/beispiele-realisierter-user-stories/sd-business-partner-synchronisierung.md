# SD Business Partner Synchronisierung

## Die Anforderung

Geschaeftspartner-Stammdaten muessen zwischen Systemen oder Organisationseinheiten synchron gehalten werden. Aenderungen an einem Business Partner -- neue Adresse, geaenderte Bankverbindung, neuer Ansprechpartner -- muessen geprueft und in die Zielsysteme uebernommen werden.

## Was conFLOW hier leistet

Der Workflow erkennt relevante Aenderungen am Business Partner und startet automatisch einen Synchronisierungsprozess. Dabei koennen **Hintergrundschritte** die eigentliche technische Synchronisierung uebernehmen, waehrend ein Dialogschritt nur dann eingreift, wenn manuelle Pruefung oder Entscheidung noetig ist -- etwa bei Konflikten oder unvollstaendigen Daten.

Dieses Beispiel zeigt ein Muster, das typisch fuer conFLOW ist: der **Standardfall laeuft automatisch durch** (Hintergrundschritte, kein Workitem in einer Inbox), und **nur die Ausnahme erzeugt Arbeit** fuer einen Menschen.

### User Story

<figure><img src="../../.gitbook/assets/Folie9 (2).png" alt="User Story SD Business Partner Synchronisierung"><figcaption><p>Fachliche User Story: Business Partner Synchronisierung</p></figcaption></figure>

### Workflow

<figure><img src="../../.gitbook/assets/Folie10.png" alt="Workflow SD Business Partner Synchronisierung"><figcaption><p>Technischer Laufweg mit Hintergrundschritten</p></figcaption></figure>
