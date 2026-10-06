# `get_status_dynamic`

> **Dieser Hook bleibt in der Referenzklasse leer.** Warum, steht unten.

| | |
|---|---|
| **Wann** | Nur, wenn der Folgestatus, den conFLOW aus /C09/CFL_C02 ermittelt hat, mit Y beginnt oder in /C09/CFL_C01 das Attribut BADI trägt - dann nach dieser Ermittlung und bevor conFLOW ihn benutzt. Ist an diesem Status in C01 eine Klasse hinterlegt, läuft sie vorher; der Hook hat das letzte Wort. Bei jedem anderen Statuswechsel wird er NICHT gerufen. |
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

2. Reicht eine Regel? c09-BEDINGUNG am Hintergrundschritt ohne Methode, Felder aus dem c08 TEMPLATE, Dezimalzahl in Hochkommata: GESAMTWERT_RW > '10000.00'.

3. Mehrere Bearbeiter an einem Schritt: c01-DECI_RULE (Veto, erste Entscheidung, Mehrheit).

4. Geht es mit einem Hintergrundschritt, der über EV_DECISION_KEY verzweigt? (Die Verzweigung bleibt in C02 sichtbar.)

5. Erst dann dieser Hook.

Der Beispielprozess kommt mit 4 aus - siehe BACKGROUND_CLASSIFY( ) ganz unten, die genau das tut.

**WORAN MAN DENKEN MUSS**

Damit der Hook überhaupt läuft, braucht der Folgestatus in C01 das Attribut BADI (oder einen Namen, der mit Y beginnt). Tragen mehrere Status das Attribut, läuft er an allen - ohne eine Weiche am Anfang schreibt man Fälle um, die man nie gemeint hat.

**BLEIBT HIER LEER - mit Absicht, siehe oben. Das Muster steht**

auskommentiert darunter, damit man es hat, wenn man es braucht.

## Der Code

```abap
*   " Beispiel: Betragsgrenze entscheidet, ob eskaliert wird
*   " (Voraussetzung: Status 01 traegt in C01 das Attribut BADI)
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
