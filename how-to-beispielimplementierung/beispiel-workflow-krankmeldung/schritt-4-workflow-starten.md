# Schritt 4: Workflow starten und testen

Der Workflow ist konfiguriert und die BAdI-Klasse implementiert. In diesem letzten Schritt richten wir den Trigger ein, befüllen den Container mit Fachdaten und testen den gesamten Prozess.

---

## 4.1 Überblick

<figure><img src="../../.gitbook/assets/Folie32.png" alt="Workflow starten Überblick"><figcaption><p>Vom Trigger zum laufenden Workflow</p></figcaption></figure>

---

## 4.2 Trigger-Varianten

Es gibt mehrere Wege, einen conFLOW-Workflow auszulösen. Die Wahl hängt vom Anwendungsfall ab:

| Variante | Wann einsetzen | Wie |
| --- | --- | --- |
| **Report / Programm** | Demos, Tests, manuelle Auslösung | `start_workflow_int( )` direkt aufrufen |
| **Anwendungs-BAdI** | Automatisch bei Belegänderung | Im Save-Exit des Anwendungsobjekts prüfen und starten |
| **Statusverwaltung** | Automatisch bei Statuswechsel | SAP-Statusverwaltung (BSVW) wirft BOR-Ereignis, SWETYPV-Kopplung löst den Workflow aus |

{% hint style="warning" %}
**Mehrfachstart-Schutz:** `start_workflow_int( )` prüft **nicht**, ob bereits ein Workflow für dasselbe Objekt läuft. Rufen Sie vorher `check_open_workflow( )` auf -- sonst entstehen parallele Instanzen für denselben Beleg.
{% endhint %}

<figure><img src="../../.gitbook/assets/Folie33.png" alt="Trigger Variante 1"><figcaption><p>Variante A: Workflow aus einem Report oder Programm starten</p></figcaption></figure>

<figure><img src="../../.gitbook/assets/Folie34.png" alt="Trigger Variante 2"><figcaption><p>Variante B: Workflow aus einem Anwendungs-BAdI starten</p></figcaption></figure>

<figure><img src="../../.gitbook/assets/Folie35.png" alt="Trigger Variante 3"><figcaption><p>Variante C: Workflow über Statusverwaltung und BOR-Ereignis</p></figcaption></figure>

---

## 4.3 Container befüllen

Der **conFLOW-Container** (`/C09/CFL_S04`) speichert beliebige Fachdaten als Key-Value-Paare. Geschrieben wird über die Framework-Methoden:

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
**Best Practice:** Attribut-Namen als Konstanten in eine eigene Klasse `ZCL_CFL_CONST_<nnnnn>` legen. Diese Klasse wird von der BAdI-Klasse, von Hintergrundschritten und ggf. von der UI gemeinsam genutzt -- eine Quelle für alle Attribut-Namen.
{% endhint %}

<figure><img src="../../.gitbook/assets/Folie36.png" alt="Container"><figcaption><p>SET_ATTRIBUT_VALUE und GET_ATTRIBUT_VALUE: der conFLOW-Container</p></figcaption></figure>

---

## 4.4 Nachlauflogik bei parallelen Schritten

Bei parallelen Workitems entscheiden mehrere Bearbeiter gleichzeitig. Dann stellt sich eine Frage, die das Customizing allein nicht beantwortet: **Was soll gelten?** Die erste Entscheidung, oder die Mehrheit?

Die BAdI-Methode `GET_AFTER_EXECUTION_WORKITEM` wird nach **jeder einzelnen** Entscheidung aufgerufen. Sie bekommt drei Dinge:

| Parameter | Inhalt |
| --- | --- |
| `IS_DATA_STEP` | der Schritt, inklusive Instanz-`ID`, `GEN_STAT` und der eigenen `WI_ID` |
| `IS_SWR_WIHDR` | der Workitem-Kopf |
| `IV_KEY` | der gewählte Ausgang (`0001` = OK, `0002` = NOK, `0003`–`0007` = `UC1`–`UC5`) |

### Variante A: die erste Ablehnung beendet den Parallelschritt

Der häufigste Fall. Lehnt einer ab, sollen die Workitems der anderen verschwinden — sonst arbeitet jemand an einem Vorgang, der schon entschieden ist.

```abap
" GET_AFTER_EXECUTION_WORKITEM
IF iv_key = '0002'.                              " nur bei NOK

  SELECT * FROM /c09/cfl_s03 INTO TABLE lt_cfl_s03
    WHERE id       = is_data_step-id
      AND gen_stat = is_data_step-gen_stat       " nur dieser Schritt
      AND wi_id   NE is_data_step-wi_id.         " das eigene nicht

  LOOP AT lt_cfl_s03 INTO ls_cfl_s03.
    CALL FUNCTION 'SAP_WAPI_WORKITEM_COMPLETE'
      EXPORTING
        workitem_id = ls_cfl_s03-wi_id
        set_obsolet = 'X'.
  ENDLOOP.

ENDIF.
```

<figure><img src="../../.gitbook/assets/Folie37.png" alt="GET_AFTER_EXECUTION_WORKITEM"><figcaption><p>Parallelschritt beenden: bei NOK die übrigen Workitems obsolet setzen</p></figcaption></figure>

{% hint style="info" %}
**Dasselbe gibt es fertig** — `/C09/CFL_CL_HELPER_0101=>SET_WORKITEM_OBSOLET( is_data_step = is_data_step )`. Ein Unterschied ist wichtig: der Helfer räumt **alle** offenen Dialog-Workitems des ganzen Workflows ab, das Coding oben nur die **desselben Schritts**. Laufen in Ihrem Prozess parallele Workitems in mehreren Schritten gleichzeitig, nehmen Sie die engere Variante. Und in beiden Fällen gilt: **das `COMMIT WORK` macht der Aufrufer**, sonst bleiben die Workitems offen, ohne Fehlermeldung.
{% endhint %}

### Variante B: die Mehrheit entscheidet

Soll nicht die erste Stimme zählen, sondern das Ergebnis aller, brauchen Sie einen **Sammelschritt**: einen `Y`-Schritt hinter dem Parallelschritt, auf den alle Ausgänge zeigen. Dort wird ausgezählt — in `GET_STATUS_DYNAMIC`, der genau für `Y`-Schritte gerufen wird.

Ausgezählt wird über das Workflow-Protokoll. Es wird von hinten nach vorn gelesen, bis der Anfang des Parallelblocks erreicht ist:

```abap
" GET_STATUS_DYNAMIC, am Sammelschritt
CALL FUNCTION 'SWL_GET_PROCESS_STEPLIST'
  EXPORTING
    wf_id          = cs_data-top_wi_id
    with_expansion = abap_true
    with_errors    = abap_true
  TABLES
    wfm_steplog    = lt_wfm_steplog.

lv_lines = lines( lt_wfm_steplog ) + 1.

DO.
  SUBTRACT 1 FROM lv_lines.
  READ TABLE lt_wfm_steplog ASSIGNING <fs_steplog> INDEX lv_lines.

  CASE <fs_steplog>-rc_intern.
    WHEN '0003'.  ADD 1 TO lv_plus.              " Zustimmung
    WHEN OTHERS.  ADD 1 TO lv_minus.
  ENDCASE.

  IF <fs_steplog>-node_p_ind = 1.                " Anfang des Parallelblocks
    EXIT.
  ENDIF.
ENDDO.

IF lv_plus GT lv_minus.
  cs_data-gen_stat = '01'.                       " Mehrheit dafuer
ELSE.
  cs_data-gen_stat = 'X1'.                       " Mehrheit dagegen
ENDIF.
```

<figure><img src="../../.gitbook/assets/Folie38.png" alt="Mehrheitsentscheidung"><figcaption><p>Mehrheitsentscheidung: auszählen im Sammelschritt über GET_STATUS_DYNAMIC</p></figcaption></figure>

{% hint style="warning" %}
**Welcher Ergebniscode als Zustimmung zählt, hängt von Ihrem Workflow ab.** Im Beispiel ist es `0003`. Prüfen Sie das am eigenen Protokoll nach, statt den Wert zu übernehmen — und denken Sie daran, dass `node_p_ind` die Abbruchbedingung ist: ohne sie zählen Sie den ganzen Workflow aus, nicht nur den Parallelblock.
{% endhint %}

Es gibt einen dritten Weg, wenn es nicht um Mehrheiten, sondern um unterschiedliche Antworten geht: geben Sie den Bearbeitern über `UC1`–`UC5` eigene Ausgänge und werten Sie diese in einem nachgelagerten Schritt aus.

---

## 4.5 Checkliste: erster Test

Bevor der Workflow produktiv geht, prüfen Sie folgende Punkte:

| Prüfpunkt | Wo prüfen |
| --- | --- |
| Workflow-Instanz wurde angelegt | `/C09/CFL_S01`: eine Zeile mit Ihrer `wf_definition` und `instid` |
| Workitems wurden erzeugt | `/C09/CFL_S03`: Zeilen mit der `id` aus `S01` |
| Bearbeiter stimmt | SWIA: Workitem öffnen, Bearbeiter prüfen |
| Container gefüllt | `/C09/CFL_S04`: Attribute mit der `id` aus `S01` |
| Entscheidung funktioniert | Workitem in SBWP oder Fiori My Inbox öffnen und entscheiden |
| Folgeschritt stimmt | Nach Entscheidung: nächster Schritt in `S01-gen_stat` |
| Workflow beendet sich | `S01-wf_end = 'X'` nach dem letzten Schritt |

{% hint style="success" %}
**Der Workflow läuft.** Ab hier ist der Prozess funktional vollständig. Weitere Verfeinerungen -- bessere Texte, differenzierte Bearbeiterfindung, Fiori-Darstellung, conMOBILE-App -- sind Ausbaustufen, keine Voraussetzungen.
{% endhint %}
