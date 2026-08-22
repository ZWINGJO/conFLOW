# Schritt 4: Workflow starten und testen

Der Workflow ist konfiguriert und die BAdI-Klasse implementiert. In diesem letzten Schritt richten wir den Trigger ein, befuellen den Container mit Fachdaten und testen den gesamten Prozess.

---

## 4.1 Ueberblick

<figure><img src="../../.gitbook/assets/Folie32.png" alt="Workflow starten Ueberblick"><figcaption><p>Vom Trigger zum laufenden Workflow</p></figcaption></figure>

---

## 4.2 Trigger-Varianten

Es gibt mehrere Wege, einen conFLOW-Workflow auszuloesen. Die Wahl haengt vom Anwendungsfall ab:

| Variante | Wann einsetzen | Wie |
| --- | --- | --- |
| **Report / Programm** | Demos, Tests, manuelle Ausloesung | `start_workflow_int( )` direkt aufrufen |
| **Anwendungs-BAdI** | Automatisch bei Belegaenderung | Im Save-Exit des Anwendungsobjekts pruefen und starten |
| **Statusverwaltung** | Automatisch bei Statuswechsel | SAP-Statusverwaltung (BSVW) wirft BOR-Ereignis, SWETYPV-Kopplung loest den Workflow aus |

{% hint style="warning" %}
**Mehrfachstart-Schutz:** `start_workflow_int( )` prueft **nicht**, ob bereits ein Workflow fuer dasselbe Objekt laeuft. Rufen Sie vorher `check_open_workflow( )` auf -- sonst entstehen parallele Instanzen fuer denselben Beleg.
{% endhint %}

<figure><img src="../../.gitbook/assets/Folie33.png" alt="Trigger Variante 1"><figcaption><p>Variante A: Workflow aus einem Report oder Programm starten</p></figcaption></figure>

<figure><img src="../../.gitbook/assets/Folie34.png" alt="Trigger Variante 2"><figcaption><p>Variante B: Workflow aus einem Anwendungs-BAdI starten</p></figcaption></figure>

<figure><img src="../../.gitbook/assets/Folie35.png" alt="Trigger Variante 3"><figcaption><p>Variante C: Workflow ueber Statusverwaltung und BOR-Ereignis</p></figcaption></figure>

---

## 4.3 Container befuellen

Der **conFLOW-Container** (`/C09/CFL_S04`) speichert beliebige Fachdaten als Key-Value-Paare. Geschrieben wird ueber die Framework-Methoden:

```abap
" Wert schreiben
/c09/cfl_cl_workflow_0101=>set_attribut_value(
  iv_element = 'BELEGNR'
  iv_id      = ls_cfl_s03-id
  it_value   = VALUE #( ( lv_belegnr ) ) ).

" Wert lesen
DATA(lt_val) = /c09/cfl_cl_workflow_0101=>get_attribut_value(
                 iv_element = 'BELEGNR'
                 iv_id      = ls_cfl_s03-id ).
```

{% hint style="info" %}
**Best Practice:** Attribut-Namen als Konstanten in eine eigene Klasse `ZCL_CFL_CONST_<nnnnn>` legen. Diese Klasse wird von der BAdI-Klasse, von Hintergrundschritten und ggf. von der UI gemeinsam genutzt -- eine Quelle fuer alle Attribut-Namen.
{% endhint %}

<figure><img src="../../.gitbook/assets/Folie36.png" alt="Container"><figcaption><p>SET_ATTRIBUT_VALUE und GET_ATTRIBUT_VALUE: der conFLOW-Container</p></figcaption></figure>

---

## 4.4 Nachlauflogik bei parallelen Schritten

Bei parallelen Workitems entscheiden mehrere Bearbeiter. Die BAdI-Methode `GET_AFTER_EXECUTION_WORKITEM` wird nach jeder einzelnen Entscheidung aufgerufen und erhaelt den Kontext: welcher Bearbeiter hat wie entschieden, und sind noch weitere Workitems offen.

Damit laesst sich z.B. nach der letzten Entscheidung eine Zusammenfassung berechnen oder eine Folgeaktion ausloesen.

<figure><img src="../../.gitbook/assets/Folie37.png" alt="GET_AFTER_EXECUTION_WORKITEM"><figcaption><p>Nachlauflogik nach parallelen Entscheidungen</p></figcaption></figure>

<figure><img src="../../.gitbook/assets/Folie38.png" alt="Parallele Tasks Ergebnis"><figcaption><p>Zusammenfuehrung und Weiterverarbeitung</p></figcaption></figure>

---

## 4.5 Checkliste: erster Test

Bevor der Workflow produktiv geht, pruefen Sie folgende Punkte:

| Pruefpunkt | Wo pruefen |
| --- | --- |
| Workflow-Instanz wurde angelegt | `/C09/CFL_S01`: eine Zeile mit Ihrer `wf_definition` und `instid` |
| Workitems wurden erzeugt | `/C09/CFL_S03`: Zeilen mit der `id` aus `S01` |
| Bearbeiter stimmt | SWIA: Workitem oeffnen, Bearbeiter pruefen |
| Container gefuellt | `/C09/CFL_S04`: Attribute mit der `id` aus `S01` |
| Entscheidung funktioniert | Workitem in SBWP oder Fiori My Inbox oeffnen und entscheiden |
| Folgeschritt stimmt | Nach Entscheidung: naechster Schritt in `S01-gen_stat` |
| Workflow beendet sich | `S01-wf_end = 'X'` nach dem letzten Schritt |

{% hint style="success" %}
**Der Workflow laeuft.** Ab hier ist der Prozess funktional vollstaendig. Weitere Verfeinerungen -- bessere Texte, differenzierte Bearbeiterfindung, Fiori-Darstellung, conMOBILE-App -- sind Ausbaustufen, keine Voraussetzungen.
{% endhint %}
