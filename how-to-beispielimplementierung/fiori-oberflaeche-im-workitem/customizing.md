# Customizing und Launchpad

*Vier Stellen, und keine davon meldet sich, wenn sie fehlt. Das Workitem zeigt dann einfach wieder den Textblock.*

## 1 · SWFVMD1: einmal je System

Die Verbindung von Workitem und Intent pflegt man **nicht je Workflow**, sondern einmal für den generischen conFLOW-Task `TS00388601`. Die Einträge sind **dynamisch**, das heißt, sie lesen die Container-Elemente, die das BAdI gesetzt hat.

Transaktion `SWFVMD1` → Task `TS00388601` → Aufgabenvisualisierung `INTENT`:

| Parameter | Wert | Dynamisch |
| --- | --- | --- |
| `SEMANTIC_OBJECT` | `{&/C09/CFL_VISU_SEMANTIC_OBJECT&}` | ✓ |
| `ACTION` | `{&/C09/CFL_VISU_ACTION&}` | ✓ |
| `QUERY_PARAM00` | `CFLQueryObject00={&/C09/CFL_VISU_QUERY_OBJ00&}` | ✓ |
| `QUERY_PARAM01` … `05` | analog mit `…OBJ01` … `…OBJ05` | ✓ |

{% hint style="info" %}
**Oft ist das schon da.** Ist in einem System bereits eine conFLOW-App an ein Workitem angebunden, stehen die Sätze bereits. Probe: `SE16` → `SWFVGTP` mit `TASK = TS00388601`. Stehen dort die Zeilen oben, ist hier nichts zu tun.
{% endhint %}

{% hint style="danger" %}
**SWFVMD1 gewinnt vor SWFVISU.** Die Laufzeit liest zuerst `SWFVGT`/`SWFVGTP` und erst danach SWFVISU. Wer in SWFVISU pflegt, obwohl SWFVMD1 einen Satz hat, pflegt ins Leere, und es kommt keine Fehlermeldung.
{% endhint %}

## 2 · Semantic Object

Transaktion `/UI2/SEMOBJ` → neuer Eintrag:

| Feld | Wert |
| --- | --- |
| Semantic Object | `ZCFLOrderPromiseV2` — **zeichengenau** wie die Konstante im BAdI |
| Beschreibung | frei |

Unterstriche sind erlaubt, ein Bindestrich nicht, denn der trennt Semantic Object und Action.

## 3 · Target Mapping, Katalog und Rolle

Launchpad Designer (`/UI2/FLPD_CUST`) → eigener Katalog, z. B. `Z_CFL_00500` → **Target Mapping, keine Kachel**. Die App wird nie aus dem Launchpad gestartet, nur aus dem Workitem.

| Feld | Wert |
| --- | --- |
| Semantic Object | `ZCFLOrderPromiseV2` |
| Action | `openInInbox` |
| Application Type | SAPUI5 Fiori App |
| Title | frei |
| URL | `/sap/bc/ui5_ui5/sap/zcfl_00500_uiv2` (die BSP-Anwendung) |
| ID | `zcfl00500v2` — die `sap.app.id` aus der `manifest.json` |
| Device Types | Desktop · Tablet · Phone |
| Allow additional parameters | ✓ |
| Parameter | Name `openMode` · **Mandatory ✓** · **Value** `embedIntoDetailsNestedRouter` · Default Value **leer** |

{% hint style="danger" %}
**`openMode` gehört in die Spalte *Value*, nicht in *Default Value*.** My Inbox fragt den Intent sechsmal ab, einmal je Modus, und verlangt **genau einen** Treffer. *Value* wirkt als Filter und lässt genau einen Modus passen. *Default Value* filtert nicht, dann passen alle sechs. Sechs Treffer behandelt die Inbox wie keinen: Sie fällt ohne Meldung auf den Textblock zurück.
{% endhint %}

{% hint style="info" %}
**Warum `embedIntoDetailsNestedRouter`.** Der Name führt in die Irre, die App braucht keinen Router. Der Modus schaltet zwei Dinge:

**1.** den Knopf **„Show Details"**, der rechts Notizen, Anhänge und Objektlinks der Inbox einblendet. Den gibt es nur in diesem Modus, deshalb baut man Anhänge nicht nach.

**2.** die **Wiederverwendung** der App beim Sprung auf ein anderes Workitem. Die App muss dafür zwei Methoden mitbringen, siehe [Die App](app.md).
{% endhint %}

**Rolle:** Katalog in eine PFCG-Rolle aufnehmen und **Profil generieren**. Ohne Rolle löst der Intent nicht auf. Das ist die häufigste Ursache für „bei mir geht es, bei dir nicht".

## 4 · OData-Service registrieren

`/IWFND/MAINT_SERVICE` → *Service hinzufügen* → System-Alias `LOCAL` → technischer Servicename `ZCFL_00500_V2_SRV` → *Hinzufügen*.

Danach den OData-Service in derselben Rolle berechtigen (`S_SERVICE`), sonst lädt die App, bekommt aber keine Daten.

## 5 · Nach jedem Upload der App: die Caches

| Was | wofür |
| --- | --- |
| `/UI5/APP_INDEX_CALCULATE` | ohne diesen Lauf liefert der Server weiter die alte Fassung, **auch im Inkognito-Fenster** |
| `/UI2/INVALIDATE_CLIENT_CACHES` | Cache-Buster des Launchpads |
| Browser: DevTools → Application → **Clear site data** | UI5 legt Views in IndexedDB ab, und „Cache leeren" fasst das nicht an |

Nach einer Änderung am SEGW-Modell zusätzlich: `/IWFND/MAINT_SERVICE` → Service markieren → *Metadaten löschen*.

{% hint style="info" %}
**Diagnose:** Inkognito zeigt, was der Server ausliefert, das normale Fenster zeigt den Cache. Sind **beide** alt, fehlt der App-Index. Ist nur **eines** alt, sitzt es im Browser-Speicher.
{% endhint %}
