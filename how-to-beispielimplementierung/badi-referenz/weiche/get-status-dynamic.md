# `get_status_dynamic`

> **Dieser Hook bleibt in der Referenzklasse leer.** Warum, steht unten.

| | |
|---|---|
| **Wann** | Bei jedem Statuswechsel, nachdem conFLOW den Folgestatus aus /C09/CFL_C02 ermittelt hat - und bevor er ihn benutzt. |
| **Rein und raus** | CS_DATA - die Instanz MIT dem vorgesehenen Folgestatus. Wer CS_DATA-GEN_STAT hier überschreibt, überschreibt das Customizing. CS_DATA_OLD hält den Stand davor. |

```
WOFÜR   Verzweigungen, deren Ziel erst zur Laufzeit feststeht:
       Genehmigungsstufen nach Betrag, überspringen einer Stufe,
       wenn sie fachlich entfällt, Rückkehr an die Stelle, an der
       weitergeleitet wurde.
```

**DAS VERHÄLTNIS ZU /C09/CFL_C02**

C02 sagt "nach Schritt 01 mit Ausgang OK kommt Schritt B2". Dieser Hook sagt "außer wenn ...". Wer ihn benutzt, hat eine Wegführung, die man im Customizing NICHT MEHR SIEHT - das ist der Preis, und er ist hoch.

**DESHALB DIE REIHENFOLGE DER FRAGEN**

1. Geht es mit einem zusätzlichen Status in C02?

2. Geht es mit einem Hintergrundschritt, der über EV_DECISION_KEY verzweigt? (Das ist der saubere Weg - die Verzweigung bleibt in C02 sichtbar.)

3. Erst dann dieser Hook.

Der Beispielprozess kommt mit 2 aus - siehe BACKGROUND_CLASSIFY( ) ganz unten, die genau das tut.

**WORAN MAN DENKEN MUSS**

Der Hook läuft bei JEDEM Statuswechsel, auch bei denen, die einen nichts angehen. Ohne eine Weiche am Anfang schreibt man Fälle um, die man nie gemeint hat.

**BLEIBT HIER LEER - mit Absicht, siehe oben. Das Muster steht**

auskommentiert darunter, damit man es hat, wenn man es braucht.

## Der Code

```abap
*   " Beispiel: Betragsgrenze entscheidet, ob eskaliert wird
*   IF cs_data-gen_stat <> zcl_cfl_const_00900=>mc_stat-approve.
*     RETURN.
*   ENDIF.
*
*   IF is_true( get_val( iv_element = zcl_cfl_const_00900=>mc_prop-limit_hit
*                        iv_id      = cs_data-id ) ) = abap_true.
*     cs_data-gen_stat = zcl_cfl_const_00900=>mc_stat-escalate.
*   ELSE.
*     cs_data-gen_stat = zcl_cfl_const_00900=>mc_stat-post.
*   ENDIF.
```
