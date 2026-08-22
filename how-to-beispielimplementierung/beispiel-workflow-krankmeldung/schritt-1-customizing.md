# Schritt 1: Customizing anlegen

In diesem Schritt legen wir die Workflow-Definition an und konfigurieren den Laufweg. Am Ende ist der Workflow technisch lauffähig und kann getestet werden -- die fachliche Logik folgt in den weiteren Schritten.

---

## 1.1 Workflow-Definition anlegen

Einstieg über die Transaktion **`/C09/CONFLOW_C`**. Legen Sie eine neue Workflow-Definition an oder kopieren Sie die Kopiervorlage 1. Die Workflow-Nummer (z.B. `00100`) identifiziert den Prozess im gesamten System -- sie taucht im Customizing, in der BAdI-Klasse und im Reporting wieder auf.

{% hint style="info" %}
**Tipp:** Prüfen Sie vor dem Anlegen, ob die gewünschte Nummer in `/C09/CFL_C06` noch frei ist. Jede Nummer darf im System nur einmal vorkommen.
{% endhint %}

<figure><img src="../../.gitbook/assets/Folie6.png" alt="Workflow-Definition anlegen"><figcaption><p>Transaktion /C09/CONFLOW_C: Workflow-Definition anlegen</p></figcaption></figure>

---

## 1.2 Genehmigungsschritte definieren

Im Unterordner **"Genehmigungsschritte"** legen Sie die einzelnen Schritte an. Jeder Schritt hat einen Status-Code (`gen_stat`) und einen Typ:

| Typ | Status-Code | Bedeutung |
| --- | --- | --- |
| Start | `X0` | **Pflicht** -- der erste Schritt jedes Workflows |
| Dialog | `01`, `02`, ... | Erzeugt ein Workitem in der Inbox des Bearbeiters |
| Hintergrund | `B1`, `B2`, ... | Automatische Verarbeitung, kein Workitem |
| Ende | `X1`, `X2`, ... | Beendet den Workflow. Jeder Status mit `X` am Anfang ist ein Endstatus |

{% hint style="warning" %}
**Best Practice:** Für Hintergrundschritte den Typ `BACK_BATCH` verwenden (nicht `BACK`). `BACK_BATCH` läuft als technischer User `WF-BATCH`, `BACK` im Dialog des aktuellen Benutzers. Die ältere Variante `BACK` ist weiterhin möglich, erzeugt aber ein Workitem in der Inbox.
{% endhint %}

<figure><img src="../../.gitbook/assets/Folie7 (2).png" alt="Genehmigungsschritte definieren"><figcaption><p>Genehmigungsschritte: Typ, Status-Code und Reihenfolge</p></figcaption></figure>

---

## 1.3 Bearbeiter-Rollen definieren

Unter **"User Status Definition"** legen Sie die Rollen an, die in diesem Workflow vorkommen. Eine Rolle ist ein Bearbeiter-Key (`gen_stat_user`) mit einer Bezeichnung.

Zwei Rollen sind als Best Practice vordefiniert:

| Key | Bedeutung |
| --- | --- |
| `BU` | Hintergrund-User (`WF-BATCH`) für automatische Schritte |
| `WI` | Workflow-Initiator -- der Benutzer, der den Workflow gestartet hat |

Weitere Rollen legen Sie je nach Prozess an, z.B. `01` für die Personalabteilung.

<figure><img src="../../.gitbook/assets/Folie8 (1).png" alt="User Status Definition"><figcaption><p>Bearbeiter-Rollen (gen_stat_user) definieren</p></figcaption></figure>

---

## 1.4 Bearbeiter zuordnen

Unter **"User Status Zuordnung"** ordnen Sie den Rollen die tatsächlichen Bearbeiter zu. Die Zuordnung erfolgt über SAP-Organisationsobjekte:

| Objekttyp (`otype`) | Beispiel | Bedeutung |
| --- | --- | --- |
| `US` | `MEIER` | Einzelner SAP-Benutzer |
| `S` | `50000123` | SAP-Planstelle |
| `AC` | `Z_HR_ADMIN` | PFCG-Rolle |
| `US` | `WF-BATCH` | Technischer Hintergrund-User |
| `US` | `WF_INITIATOR` | Workflow-Initiator (dynamisch) |

{% hint style="info" %}
**Dynamische Bearbeiterfindung:** Setzen Sie `user_badi = 'X'` in `/C09/CFL_C03`, um die Auflösung an die BAdI-Methode `GET_ACTORS` zu delegieren. Dann ermittelt der ABAP-Code den Bearbeiter zur Laufzeit, z.B. abhängig von Belegdaten. Siehe [Schritt 3: BAdI-Implementierung](schritt-3-badi-implementierung.md).
{% endhint %}

<figure><img src="../../.gitbook/assets/Folie9.png" alt="Bearbeiter zuordnen"><figcaption><p>Zuordnung der Rollen zu SAP-Benutzern oder Organisationsobjekten</p></figcaption></figure>

---

## 1.5 Laufweg definieren

Unter **"Genehmigungsschritte"** (Übergangstabelle) definieren Sie, welcher Schritt auf welchen folgt. Jeder Schritt hat zwei Standard-Ausgänge:

| Ausgang | Feld in `/C09/CFL_C02` | Bedeutung |
| --- | --- | --- |
| OK | `gen_stat_ok` | Genehmigt / Weiter |
| NOK | `gen_stat_nok` | Abgelehnt / Zurück |

Zusätzlich können bis zu fünf weitere Entscheidungsalternativen (`UC1`-`UC5`) definiert werden -- das folgt in [Schritt 2](schritt-2-customizing-erweitern.md).

{% hint style="info" %}
**Pflicht:** Der Einstieg ist immer `X0`. In der Übergangstabelle muss eine Zeile `gen_stat = 'X0'` mit dem ersten Dialogschritt in `gen_stat_ok` stehen. `X0` ist die einzige Status-Konstante, die das Framework hart kodiert erwartet.
{% endhint %}

<figure><img src="../../.gitbook/assets/Folie10 (1).png" alt="Laufweg definieren"><figcaption><p>Übergangstabelle: Schritt für Schritt durch den Prozess</p></figcaption></figure>

---

## 1.6 Schritt und Bearbeiter verknüpfen

Unter **"Zuordnung Userstatus"** (`/C09/CFL_C05`) verknüpfen Sie jeden Genehmigungsschritt mit der Rolle, die ihn bearbeiten soll.

{% hint style="info" %}
**Parallele Schritte:** Ordnen Sie einem Genehmigungsschritt **mehrere** Bearbeiter-Keys zu. conFLOW erzeugt dann für jeden ein eigenes Workitem, und der Workflow wartet, bis alle entschieden haben.
{% endhint %}

<figure><img src="../../.gitbook/assets/Folie11.png" alt="Zuordnung Userstatus"><figcaption><p>Verknüpfung: welche Rolle bearbeitet welchen Schritt</p></figcaption></figure>

---

## 1.7 Ergebnis: der Workflow ist lauffähig

Ab diesem Punkt ist der Workflow technisch lauffähig. Das Customizing definiert: welche Schritte es gibt, wer sie bearbeitet, in welcher Reihenfolge und mit welchen Ausgängen. Ein Start (z.B. über einen Testreport oder SWIA) erzeugt eine Workflow-Instanz und das erste Workitem.

<figure><img src="../../.gitbook/assets/Folie12 (1).png" alt="Workflow lauffähig"><figcaption><p>Ergebnis: der Workflow läuft und erzeugt Workitems</p></figcaption></figure>

---

## 1.8 Testen mit SWIA

Über die SAP-Transaktion **`SWIA`** können Sie den Workflow prüfen und testen. SWIA zeigt den aktuellen Zustand jeder Workflow-Instanz: welcher Schritt ist aktiv, welcher Bearbeiter ist zugeordnet, und wie sieht die Historie aus.

{% hint style="success" %}
**Empfehlung:** Testen Sie den Workflow nach jedem Customizing-Schritt in SWIA. So finden Sie Konfigurationsfehler (fehlender Bearbeiter, fehlender Übergang) sofort und nicht erst bei der BAdI-Implementierung.
{% endhint %}

<figure><img src="../../.gitbook/assets/Folie13.png" alt="SWIA Test"><figcaption><p>Transaktion SWIA: Workflow-Instanzen prüfen und testen</p></figcaption></figure>
