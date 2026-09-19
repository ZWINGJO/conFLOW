# conFLOW: die App ans Workitem hängen

*Eine Methode im BAdI und drei Konstanten. Am conFLOW-Customizing ändert sich nichts.*

## Der Hook

[`get_after_creation_workitem`](../badi-referenz/lebenszyklus/get-after-creation-workitem.md) läuft einmal je Workitem, nach dem Anlegen und vor der ersten Anzeige. Genau dort wird die Oberfläche angehängt.

```abap
  METHOD /c09/cfl_if_badi_0101~get_after_creation_workitem.
    " ... was der Workflow hier sonst tut (Priorität o. ä.)

    set_inbox_ui( iv_wi_id = is_swr_wihdr-wi_id
                  iv_id    = is_data_step-id ).
  ENDMETHOD.
```

## `set_inbox_ui( )`

Gesetzt werden drei Container-Elemente **am Workitem**. Das dritte ist der Schlüssel, mit dem die App ihren Vorgang findet: die conFLOW-Instanz als 32-stellige Hex-Kette, weil sie so durch die URL passt.

```abap
  METHOD set_inbox_ui.
    DATA lv_key TYPE c LENGTH 32.

    IF zcl_cfl_const_00500=>mc_inbox_ui = abap_false.      " Notaus
      RETURN.
    ENDIF.

    IF iv_wi_id IS INITIAL OR iv_id IS INITIAL.
      RETURN.
    ENDIF.

    lv_key = iv_id.                                        " RAW16 -> Hex

    TRY.
        DATA(lo_container) = cl_swf_run_workitem_context=>get_instance(
                               im_wiid = iv_wi_id )->if_wapi_workitem_context~get_wi_container( ).

        lo_container->set( name  = zcl_cfl_const_00500=>mc_visu-semantic_object
                           value = zcl_cfl_const_00500=>mc_ui_semantic_object ).
        lo_container->set( name  = zcl_cfl_const_00500=>mc_visu-action
                           value = zcl_cfl_const_00500=>mc_ui_action ).
        lo_container->set( name  = zcl_cfl_const_00500=>mc_visu-query_obj00
                           value = lv_key ).

      CATCH cx_swf_ifs_exception.
        RETURN.
    ENDTRY.
  ENDMETHOD.
```

{% hint style="success" %}
**Das stille `CATCH` ist Absicht.** Fällt hier etwas aus, bleibt es beim Standard-Textblock. Eine Oberfläche, die nicht kommt, darf kein Workitem verhindern.
{% endhint %}

## Die Konstanten

```abap
    CONSTANTS mc_inbox_ui           TYPE abap_bool VALUE abap_true.
    CONSTANTS mc_ui_semantic_object TYPE string    VALUE 'ZCFLOrderPromiseV2' ##NO_TEXT.
    CONSTANTS mc_ui_action          TYPE string    VALUE 'openInInbox'        ##NO_TEXT.

    CONSTANTS:
      BEGIN OF mc_visu,
        semantic_object TYPE swfdname VALUE '/C09/CFL_VISU_SEMANTIC_OBJECT',
        action          TYPE swfdname VALUE '/C09/CFL_VISU_ACTION',
        query_obj00     TYPE swfdname VALUE '/C09/CFL_VISU_QUERY_OBJ00',
      END OF mc_visu.
```

{% hint style="danger" %}
**Die Schreibweise zählt zeichengenau.** `mc_ui_semantic_object` muss exakt dem Semantic Object im Launchpad entsprechen, Groß- und Kleinschreibung eingeschlossen. Weicht ein Buchstabe ab, löst der Intent nicht auf, und das Workitem zeigt ohne Meldung wieder den Textblock.
{% endhint %}

Die drei `/C09/CFL_VISU_*`-Namen gibt conFLOW vor. Sie sind in jedem Workflow gleich. `/C09/CFL_VISU_QUERY_OBJ01` bis `…05` stehen für weitere Parameter bereit, falls eine App mehr als den Schlüssel braucht.

## Die Entscheidung fällt beim Anlegen

Die Werte wandern in den Workitem-Container und bleiben dort. Daraus folgt:

- **Ein bestehendes Workitem wechselt die Oberfläche nicht.** Wer `mc_inbox_ui` umstellt oder das BAdI ändert, braucht zum Testen ein **neues** Workitem.
- Eine andere App für einen bestimmten Schritt ist ein `IF` auf `is_data_step-gen_stat` vor dem `set`.

## Der einfache Weg: Parameter `VISU` im Customizing

`set_inbox_ui` muss man **nicht selbst** rufen. conFLOW setzt die drei Container-Elemente von sich aus, sobald am Workflow der Parameter `VISU` gepflegt ist.

Pflege in **c08 (Allgemeine Parameter)** je Workflow-Definition:

| Parameter | Wert |
| --- | --- |
| `VISU` | das Semantic Object der App, z. B. `ZCFLOrderPromiseV2` |

Daraus setzt das Framework beim Anlegen jedes Workitems `/C09/CFL_VISU_SEMANTIC_OBJECT` (aus `VISU`), `/C09/CFL_VISU_ACTION` (`openInInbox`) und `/C09/CFL_VISU_QUERY_OBJ00` (die Workflow-Instanz). **Leer = wie bisher**, das Workitem zeigt den Textblock. Kein Erben über `WF_DEF` — je Definition pflegen. Wirkt nur in Fiori.

{% hint style="info" %}
**Bestehende offene Workitems** bekommen die Elemente erst beim Anlegen. Der Report `/C09/CFL_MIGRATE_VISU` trägt sie nach — Vorgabe Simulation, geschrieben wird erst mit gesetztem Haken, über `SAP_WAPI_WRITE_CONTAINER`.
{% endhint %}

Der BAdI-Weg oben bleibt für **Sonderfälle**: eine andere App je Schritt oder ein berechnetes Semantic Object. Er übersteuert `VISU`, weil `get_after_creation_workitem` nach dem Framework läuft.

## Was die App vom Workflow braucht

Die App liest und schreibt **nur über conFLOW**. Eigene Tabellen braucht sie nicht.

| Tabelle | wofür die App sie braucht |
| --- | --- |
| `/C09/CFL_S01` | Kopf: gehört die Instanz zu **diesem** Workflow (`WF_DEFINITION`)? |
| `/C09/CFL_S03` | Schritte: welches Workitem (`WI_ID`) auf welchem Schritt (`GEN_STAT`) |
| `/C09/CFL_S04` | Container: die Werte, gelesen und geschrieben |

**Den Workitem-Text holt die App, statt ihn nachzubauen.** Das BAdI baut den Kontextblock ohnehin in `get_workitem_text`. Über `SAP_WAPI_WORKITEM_DESCRIPTION` bekommt die App genau diesen Block, also denselben, den SAP GUI und Standard-Inbox zeigen. Ändert sich der Text im BAdI, zieht die App ohne eine Zeile Änderung mit.

**Geschrieben wird über die Helfer der Workflow-Klasse**, zum Beispiel mit `GLOBAL FRIENDS` für die Datenklasse der App. Dann gibt es für Texte und Werte nur einen Rechenweg.

{% hint style="danger" %}
**Ein Container-Wert ist höchstens 132 Zeichen lang** (`/C09/CFL_S04-VALUE`). Längeres wird ohne Meldung abgeschnitten. Deshalb die Notiz in Modell, Eingabefeld und Serverprüfung auf 132 begrenzen.
{% endhint %}

## Optional: beim Entscheiden verbindlich prüfen

Die App speichert **immer**, auch Halbfertiges, sonst kann niemand arbeiten. Ob der Prozess **weiterlaufen** darf, prüft conFLOW nach dem Klick auf einen Knopf:

| Hook | Abbruch | Meldung an den Bearbeiter |
| --- | --- | --- |
| [`get_after_execution_mobile`](../badi-referenz/lebenszyklus/get-after-execution-mobile.md) | `cv_subrc = 9` | ja, `cs_t100msg` |
| [`get_after_execution`](../badi-referenz/lebenszyklus/get-after-execution.md) | `cv_subrc = 1` — **nicht 9**, sonst läuft die Entscheidung im SAP GUI trotzdem durch | nein |

Beide bekommen `iv_altkey`, also den tatsächlich geklickten Ausgang. Beide Hooks sollten dieselbe Prüfmethode rufen, dann gilt eine Bedingung auf beiden Wegen.

{% hint style="info" %}
**Die Regel:** Die App prüft, ob ein **Arbeitsstand** gespeichert werden darf. conFLOW prüft, ob der **Prozess** weiterlaufen darf. Eine fehlende Begründung ist eine Abschlussbedingung und keine Speicherbedingung.
{% endhint %}
