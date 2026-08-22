# SD Business Partner Synchronisierung

## Die Anforderung

Geschäftspartner-Stammdaten müssen zwischen Systemen oder Organisationseinheiten synchron gehalten werden. Änderungen an einem Business Partner -- neue Adresse, geänderte Bankverbindung, neuer Ansprechpartner -- müssen geprüft und in die Zielsysteme übernommen werden.

## Was conFLOW hier leistet

Der Workflow erkennt relevante Änderungen am Business Partner und startet automatisch einen Synchronisierungsprozess. Dabei können **Hintergrundschritte** die eigentliche technische Synchronisierung übernehmen, während ein Dialogschritt nur dann eingreift, wenn manuelle Prüfung oder Entscheidung nötig ist -- etwa bei Konflikten oder unvollständigen Daten.

Dieses Beispiel zeigt ein Muster, das typisch für conFLOW ist: der **Standardfall läuft automatisch durch** (Hintergrundschritte, kein Workitem in einer Inbox), und **nur die Ausnahme erzeugt Arbeit** für einen Menschen.

### User Story

<figure><img src="../../.gitbook/assets/Folie9 (2).png" alt="User Story SD Business Partner Synchronisierung"><figcaption><p>Fachliche User Story: Business Partner Synchronisierung</p></figcaption></figure>

### Workflow

<figure><img src="../../.gitbook/assets/Folie10.png" alt="Workflow SD Business Partner Synchronisierung"><figcaption><p>Technischer Laufweg mit Hintergrundschritten</p></figcaption></figure>
