# Was der Standard schon kann — und was man deshalb nicht baut
*Das Kapitel, das am meisten Arbeit spart. Es steht vor der Objektliste, weil es die Objektliste kürzt.*

Die eigene App ersetzt **nur den Info-Bereich** des Workitems. Die My Inbox behält alles andere: **Notizen**, **Anhänge** und **Objektlinks**. Sie liegen nicht dahinter und sind nicht verdeckt — sie werden über einen Knopf in der Fußleiste eingeblendet, den das Framework selbst dazustellt.

```
┌──────────────────────────────┬──────────────────────┐
│  eigene RAP-App              │ Comments │ Attachments│
│  (ersetzt nur den Info-Tab)  │   More ∨             │
│                              │                      │
├──────────────────────────────┴──────────────────────┤
│  … Show Log │ Show Details │ Claim │ Forward │ ⋯    │
└─────────────────────────────────────────────────────┘
```

Das Layout dahinter ist ein `sap.ui.layout.DynamicSideContent`: die eigene App im Hauptbereich, die Reiter als Side Content. Belegt im Quelltext der Inbox selbst — BSP-Anwendung `CA_FIORI_INBOX`, Datei `Component-preload.js`:

```
// Der Knopf entsteht NUR bei diesem openMode
if (e === "embedIntoDetailsNestedRouter" && l.taskSupportsCommAttRelObj(d)) {
    s.push({ sId: "DetailsButtonID",
             sI18nBtnTxt: l.bShowDetails ? "XBUT_HIDEDETAILS" : "XBUT_SHOWDETAILS",
             onBtnPressed: function (t) { l.onDetailsBtnPress() } });
}

// und er schaltet den Side Content mit den Standard-Reitern frei
else if ((t.TaskSupports.Attachments || t.TaskSupports.TaskObject || t.TaskSupports.Comments)
         && this.bShowDetails
         && this._getEmbedIntoDetailsNestedRouter()) {
    this.fnCreateSelectedTab(this.byId("tabBarDetails").getSelectedKey());
    this.setShowSideContent(true)
}
```

| Beobachtung | Bedeutung für den Nachbau |
| --- | --- |
| Bedingung `_getEmbedIntoDetailsNestedRouter()` | Der Side Content ist **ausschließlich** für diesen openMode vorgesehen. Die anderen fünf (`embedIntoDetails`, `genericEmbedIntoDetails`, `genericEmbeddedInboxOnly`, `replaceDetails`, `external`) bekommen ihn nicht — sie rendern aber ohnehin keine Fiori-Elements-App. |
| Der Info-Reiter fehlt rechts | Den *ist* die eigene App. Keine Doppelung, sondern eine Teilung. |
| `TaskSupports.Comments \|\| Attachments \|\| TaskObject` | Kommt aus dem Task-Gateway. Fehlt der Knopf, liegt es hier — nicht an der App. |
| `bShowDetails = false` im Konstruktor | Startzustand ist zu, der Knopf ist ein Umschalter. |
| Breakpoint `S` | `setShowMainContent(false)` — auf schmalen Geräten verdeckt der Side Content die App, solange er offen ist. |

{% hint style="danger" %}
**Die teuerste Fehlannahme dieses Projekts stand genau hier.** Der Satz „die eigene Oberfläche ersetzt den Detailbereich vollständig, der Standardbereich ist für den Bearbeiter nicht erreichbar" klang plausibel, stand drei Tage als Kommentar im Code und trug am Ende **fünf Objekte, zwei RAP-Aktionen und rund 430 Zeilen JavaScript** — eine nachgebaute Dokumentenliste mit Upload, Öffnen und Löschen. Alles zurückgebaut. Der Nachbau hatte zusätzlich einen Preis, den der Standard nicht hat — siehe den Abschnitt gleich darunter.
{% endhint %}

## Falls du doch Binärdaten brauchst: der Virenscanner steht davor
Sobald eine OData-**V4**-Anfrage eine Eigenschaft vom Typ `Edm.Binary` oder `Edm.Stream` trägt, scannt `/IWBEP/CL_V4_VIRUS_SCANNER` sie — **bevor** der eigene Handler erreicht wird. Ist kein Profil aktiv, kommt keine Warnung, sondern eine Ausnahme:

```
Virus scan profile /IWBEP/V4/ODATA_UPLOAD is not active (refer to SAP note 3027559)
```

```
// /IWBEP/CL_V4_VIRUS_SCANNER=>CREATE_VIRUS_SCANNER_INSTANCE
WHEN 2. " No active virus scan profile found. Continue anyway?
  IF mv_virus_scan_profile = gcs_vscan_profile-cp        " nur Client-Proxy
  OR ...->is_virus_scan_ignored( ) = abap_true.          " oder konfiguriert
    RETURN.
  ENDIF.
  RAISE EXCEPTION ... profile_not_active.
```

| Prüfen | Wo | Erwartet |
| --- | --- | --- |
| Profil aktiv? | `VSCAN_PROF`, Feld `ACTIVE` | `X` |
| Scan ignorieren? | `/IWBEP/C_V4_SETT`, Name `VIRUS_SCANER_IGNORED` | `X` |

Ist beides leer, schlägt jeder Upload fehl. Der Tippfehler *SCANER* steht so im Standard. Ein UI dafür gibt es nicht — `/IWBEP/GLOBAL_CONFIG` zeigt unter „OData V4 Options" nur die Recommended Services, und für `/IWBEP/C_V4_SETT` ist kein Pflegedialog registriert. Es bleibt eine Zeile ABAP:

```
/iwbep/cl_v4_settings_conf_fac=>create_settings_config(
  )->set_is_virus_scan_ignored( abap_true ).
COMMIT WORK AND WAIT.
```

{% hint style="danger" %}
**Nur Uploads sind betroffen.** Das Download-Profil `/IWBEP/CP/ODATA_DOWNLOAD` fällt unter die `cp`-Ausnahme und läuft ohne Scan durch — Lesen und Löschen funktionieren also, nur Schreiben nicht. Ein Bild, das leicht in die falsche Richtung führt. Und dazugesagt gehört, was die Einstellung tut: **sie schaltet eine Sicherheitsprüfung ab.** In einem System ohne Scan-Server ist das folgenlos; bei einem Kunden mit aktivem Virenscanner ist die richtige Antwort ein **aktives Profil** (Transaktion `VSCANPROFILE`), nicht das Abschalten. Systemweite Einstellung — sie gehört nicht in den Transport der Anwendung, sondern auf die Liste der Voraussetzungen.
{% endhint %}

{% hint style="success" %}
**Die Regel daraus:** Bevor du in einer eingebetteten Inbox-App irgendetwas nachbaust, das nach Workitem-Standard klingt — Anhänge, Notizen, Objektlinks, Bearbeiterhistorie —, **klick erst auf „Show Details"**. Und wenn du es dort nicht findest: im Bundle nachlesen, nicht schließen. Ein Screenshot beweist nur, was gerade konfiguriert ist.
{% endhint %}
