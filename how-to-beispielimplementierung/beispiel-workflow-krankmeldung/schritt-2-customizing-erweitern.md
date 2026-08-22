# Schritt 2: Customizing erweitern

Der Workflow läuft jetzt grundsätzlich. In diesem Schritt verfeinern wir das Customizing: Hintergrundschritte bekommen Funktionalität, Workitems eine Beschreibung, Buttons eine Beschriftung, und der Mailversand wird eingerichtet.

---

## 2.1 Erweiterte Konfiguration im Überblick

<figure><img src="../../.gitbook/assets/Folie14 (1).png" alt="Customizing erweitern"><figcaption><p>Überblick: was im erweiterten Customizing hinzukommt</p></figcaption></figure>

---

## 2.2 Hintergrundschritte mit Methode hinterlegen

Ein Hintergrundschritt (Typ `BACK_BATCH`) führt automatisch eine statische ABAP-Methode aus. Die Methode wird im Customizing hinterlegt -- Klasse und Methodenname in `/C09/CFL_C01`, Felder `clsname` und `cmpname`.

Die Methode muss eine feste Signatur haben:

| Parameter | Richtung | Typ | Bedeutung |
| --- | --- | --- | --- |
| `IS_CFL_S03` | Importing | `/C09/CFL_S03` | Aktuelle Workitem-Zeile |
| `ET_BAPIRET2` | Exporting | `BAPIRET2_T` | Meldungen (bei E/A-Meldung: Ausgang wird NOK) |
| `EV_DECISION_KEY` | Exporting | `SWR_DECIKEY` | Entscheidung (bestimmt den nächsten Schritt) |

{% hint style="warning" %}
**Immer `EV_DECISION_KEY` setzen.** Ohne Rückgabe setzt das Framework den Ausgang auf `OK` -- bei einer Fehlermeldung in `ET_BAPIRET2` aber auf `NOK`. Damit ist `NOK` am Hintergrundschritt für den Fehlerfall reserviert; fachliche Ergebnisse gehören auf `UC1`-`UC5`.
{% endhint %}

<figure><img src="../../.gitbook/assets/Folie15.png" alt="Hintergrundschritt Methode"><figcaption><p>Hintergrundschritt: Klasse und Methode im Customizing</p></figcaption></figure>

---

## 2.3 Workitem-Beschreibung mit SO10-Texten

Damit der Bearbeiter am Workitem versteht, worum es geht, hinterlegen Sie **SO10-Texte** im Customizing. Die Textnamen stehen in `/C09/CFL_C01`, Feld `tdname`. Im Text können **Platzhalter** in der Form `§{FELDNAME}` verwendet werden -- conFLOW ersetzt sie zur Laufzeit mit Werten aus dem Container.

<figure><img src="../../.gitbook/assets/Folie16 (1).png" alt="SO10 Texte"><figcaption><p>SO10-Texte: Workitem-Beschreibung mit Platzhaltern</p></figcaption></figure>

---

## 2.4 Entscheidungsalternativen und Button-Beschriftung

Neben `OK` und `NOK` können Sie bis zu fünf weitere Entscheidungsalternativen (`UC1` bis `UC5`) anlegen. Diese werden in `/C09/CFL_C09` konfiguriert -- je Schritt eine eigene Zeile.

Die **Beschriftung** der Buttons (auch für `OK` und `NOK`) wird in der Texttabelle `/C09/CFL_C09T` gepflegt. Ohne Eintrag dort zeigt das Workitem die Standardtexte des SAP-Tasks.

{% hint style="info" %}
**Button-Steuerung:** Setzen Sie `nodisplay = 'X'` in `/C09/CFL_C09`, um einen Button für einen bestimmten Schritt auszublenden. Damit lässt sich z.B. auf dem ersten Schritt die Stornierung verbieten, während sie auf dem Eskalationsschritt verfügbar ist.
{% endhint %}

<figure><img src="../../.gitbook/assets/Folie17 (2).png" alt="Entscheidungsalternativen"><figcaption><p>Entscheidungsalternativen UC1-UC5 und ihre Beschriftung</p></figcaption></figure>

---

## 2.5 Mailversand konfigurieren

Im Unterordner **"Mailversand"** konfigurieren Sie, wer bei welchem Schritt eine E-Mail erhält. Die Mail-Inhalte werden als SO10-Texte gepflegt und können dieselben Platzhalter (`§{...}`) verwenden wie die Workitem-Beschreibung.

Die dynamische Ersetzung der Platzhalter erfolgt in der BAdI-Methode `GET_DATASOURCE_MAIL` (siehe [Schritt 3](schritt-3-badi-implementierung.md)).

<figure><img src="../../.gitbook/assets/Folie18 (2).png" alt="Mailversand"><figcaption><p>Mailversand: Empfänger und SO10-Texte je Schritt</p></figcaption></figure>
