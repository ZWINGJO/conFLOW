# Wie das Workitem seine App findet
*Einmal verstanden, sind alle weiteren Details Innenleben einzelner Glieder.*

**`Die Kette`**

```
ABAP
  ZCL_CFL_WORKFLOW_NNNNN~get_after_creation_workitem( )
        │
        └─► set_inbox_ui( )   setzt DREI Container-Elemente am WORKITEM:
                /C09/CFL_VISU_SEMANTIC_OBJECT = 'ZCFLOrderPromise'
                /C09/CFL_VISU_ACTION          = 'openInInbox'
                /C09/CFL_VISU_QUERY_OBJ00     = <Instanz als Hex>
                             │
Customizing SWFVMD1          ▼
  Task TS00388601  — conFLOWs generischer Task, KEIN eigener Task!
  Eintraege DYNAMISCH, sie lesen die Container-Elemente:
       SEMANTIC_OBJECT   {&/C09/CFL_VISU_SEMANTIC_OBJECT&}
       ACTION            {&/C09/CFL_VISU_ACTION&}
       QUERY_PARAM00     CFLQueryObject00={&/C09/CFL_VISU_QUERY_OBJ00&}
                             │
                             ▼
       Intent:  #ZCFLOrderPromise-openInInbox?CFLQueryObject00=70A8A561...
                             │
Launchpad, Katalog            ▼
  Target Mapping
       App Type    SAPUI5 Fiori App, ID zcfl00500inbox
       Parameter   openMode         = embedIntoDetailsNestedRouter  (Pflicht)
       Parameter   CFLQueryObject00 → Target Name: WfId
                             │
                             ▼
       BSP → Fiori Elements → OData V4 → Query-Provider → /c09/cfl_s04
```

## Die drei Stellen, an denen es typischerweise scheitert
{% hint style="danger" %}
**Es gibt keinen eigenen Task.** conFLOW hat die Visualisierung *generisch* verdrahtet: `TS00388601` trägt dynamische SWFVMD1-Einträge, die zur Laufzeit Container-Elemente auflösen. **Welche App erscheint, entscheidet das einzelne Workitem** — nicht das Customizing. Das ist auch der Grund, warum der Notaus so einfach sein kann.
{% endhint %}
{% hint style="info" %}
**SWFVMD1 gewinnt vor SWFVISU.** `CL_SWF_UTL_URL_GENERATE` liest zuerst `SWFVGT`/`SWFVGTP` (Schlüssel `WLC = SAPUI5`) und erst danach SWFVISU. Wer in SWFVISU pflegt, während SWFVMD1 einen Satz hat, pflegt ins Leere — ohne Fehlermeldung.
{% endhint %}
{% hint style="info" %}
**`openMode = embedIntoDetailsNestedRouter`** ist der Schalter, der die App *einbettet*. Ohne ihn kennt die Inbox den Intent, tut aber nichts: der Detailbereich bleibt weiß.
{% endhint %}

## Die Parameter-Umbenennung, die eine Zusatzentity spart
conFLOW liefert den Schlüssel unter seinem eigenen Namen `CFLQueryObject00`. Die Custom Entity nennt ihr Schlüsselfeld `WfId`. Statt in der Entity ein Zusatzfeld anzulegen, benennt das Target Mapping den Parameter um — Spalte **Target Name**. Damit filtert Fiori Elements ohne eine Zeile Code.
Voraussetzung: das Feld muss `@UI.selectionField` sein, sonst wendet Fiori Elements den Startup-Parameter nicht als Filter an.
