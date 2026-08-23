# Die Regelklasse — eine Datei, drei Aufrufer
*Nicht im Behavior Pool, nicht im Query-Provider — daneben.*

| Aufrufer | wozu |
| --- | --- |
| Query-Provider | liefert das Ergebnis als **Felder** an die Oberfläche |
| `get_instance_features` | Auskunft an Fiori Elements |
| `setDecision` | verbindliche **Abweisung** beim Schreiben |

Die Aufrufrichtung entscheidet
Lägen die Regeln im **Behavior Pool**, hinge der Query-Provider an der RAP-Infrastruktur — falsche Richtung. Lägen sie im **Query-Provider**, hinge der Handler am Lesen. Also stehen sie daneben, und beide rufen sie.
Der eigentliche Gewinn ist ein anderer: **wer wissen will, was dieser Workflow der Oberfläche erlaubt, liest eine Datei.**

**`ZCL_CFL_00500_RULES`**

```abap
CLASS zcl_cfl_00500_rules DEFINITION
  PUBLIC
  ABSTRACT
  FINAL
  CREATE PUBLIC .

*--------------------------------------------------------------------*
* Die Prozessregeln der Inbox-App - an genau einer Stelle.
*
* Warum eine eigene Klasse und nicht der Behavior Pool: die Regeln
* werden von ZWEI Seiten gebraucht, und die haben nichts miteinander zu
* tun.
*
*   ZCL_CFL_00500_QUERY   liefert sie als FELDER an die Oberflaeche
*                         ("darf hier entschieden werden?")
*   LHC_ORDERPROMISE...   erzwingt sie beim Schreiben
*
* Laegen sie im Behavior Pool, haenge der Query-Provider an der
* RAP-Infrastruktur - falsche Richtung. Laegen sie im Query-Provider,
* haenge der Handler am Lesen. Also stehen sie daneben, und beide
* rufen sie.
*
* Es ist ausserdem die Klasse, in der die Regeln fuer einen Pruefer
* zusammen stehen. Wer wissen will, was dieser Workflow der Oberflaeche
* erlaubt, liest EINE Datei.
*--------------------------------------------------------------------*
  PUBLIC SECTION.

    TYPES ty_action  TYPE c LENGTH 20 .
    TYPES ty_actions TYPE STANDARD TABLE OF ty_action WITH EMPTY KEY .

*--------------------------------------------------------------------*
* Die zulaessigen Aktionsschluessel - die EINE Liste.
*
* Die Aktionen stehen auf den conFLOW-Buttons; die App bietet sie
* nicht mehr an (dort steht seit dem 22.08.2026 der GRUND). Gebraucht
* wird die Liste trotzdem: ACTION_OF_KEY baut daraus die Umkehrung
* Ausgang -> Aktion, die der Decision-Exit braucht.
*
* Eine neue Aktion ist damit eine Zeile hier plus ein Ausgang im
* c09-Customizing - und KEY_OF_ACTION verbindet beide.
*--------------------------------------------------------------------*
    CLASS-METHODS get_valid_actions
      RETURNING VALUE(rt_action) TYPE ty_actions .

*--------------------------------------------------------------------*
* Aktion und conFLOW-Ausgang - dieselbe Zuordnung, beide Richtungen.
*
* Der Bearbeiter waehlt im Dropdown eine AKTION; entschieden wird aber
* ueber einen conFLOW-BUTTON, und der traegt einen DECISION-KEY. Beide
* meinen dasselbe, und die Zuordnung darf es deshalb nur einmal geben.
*
*   ACCEPT_NEW_DATE   ok    ->  B2
*   PARTIAL_DELIVERY  uc1   ->  B3
*   CANCEL_REMAINING  uc2   ->  B4
*   ESCALATE          uc3   ->  02
*
* Gebraucht wird sie an drei Stellen: fuer die gruene Faerbung des
* Empfehlungs-Buttons (beide Oberflaechen) und im Decision-Exit, wo
* aus dem geklickten Button die Aktion wird.
*--------------------------------------------------------------------*
    CLASS-METHODS key_of_action
      IMPORTING !iv_action    TYPE clike
      RETURNING VALUE(rv_key) TYPE swr_decikey .

    CLASS-METHODS action_of_key
      IMPORTING !iv_key          TYPE swr_decikey
      RETURNING VALUE(rv_action) TYPE ty_action .

*--------------------------------------------------------------------*
* Darf auf diesem conFLOW-Schritt entschieden werden?
*
* Entschieden wird auf den beiden Customer-Service-Schritten: 01 (RED)
* und 03 (YELLOW) - dort sitzt der Bearbeiter, der den Fall loest.
*
* Der Eskalationsschritt 02 ist Nur-Lesen. Der Teamlead sieht, was der
* Customer Service vorgeschlagen und wie er es begruendet hat, und
* entscheidet selbst ueber die conFLOW-Buttons; dort landet seine
* Entscheidung ohnehin im Audit Trail. Duerfte er in dieselben Felder
* schreiben, waere der Vorschlag des Customer Service ueberschrieben
* und die Eskalation hinterher nicht mehr nachvollziehbar. Ein
* Container-Feld, zwei Bearbeiter nacheinander - das macht die
* Historie kaputt.
*
* Hintergrund- und Endeschritte haben kein Dialog-Workitem; dort ist
* GEN_STAT leer.
*--------------------------------------------------------------------*
    CLASS-METHODS is_editable
      IMPORTING !iv_gen_stat    TYPE /c09/cfl_gen_stat
      RETURNING VALUE(rv_allow) TYPE abap_bool .

*--------------------------------------------------------------------*
* Die zulaessigen Entscheidungsgruende - das Vokabular des Dropdowns.
*
* Dieselbe Rolle wie GET_VALID_ACTIONS, nur fuer die andere Frage:
* die Aktionen beantworten WAS (und kommen vom Button), die Gruende
* WARUM (und kommen aus der App). Eine Liste, zwei Verwender - die
* Wertehilfe baut daraus ihre Zeilen, IS_VALID_REASON prueft dagegen.
*--------------------------------------------------------------------*
    CLASS-METHODS get_valid_reasons
      RETURNING VALUE(rt_reason) TYPE ty_actions .

    CLASS-METHODS is_valid_reason
      IMPORTING !iv_reason   TYPE clike
      RETURNING VALUE(rv_ok) TYPE abap_bool .

*--------------------------------------------------------------------*
* Fehlt noch etwas, bevor abgeschlossen werden kann?
*
* Geprueft wird der GRUND, nicht die Aktion. Das ist seit dem
* 22.08.2026 so, und der Unterschied ist wichtig: die Aktion kommt vom
* Button, den der Bearbeiter erst am Ende drueckt - eine Pflicht daran
* zu haengen hiess, waehrend der Bearbeitung etwas zu verlangen, das
* noch gar nicht feststeht.
*
* Der Grund dagegen ist eine Eingabe der App. Bei OTHER ist die Notiz
* Pflicht: ein Grund "Sonstiges" ohne Erlaeuterung ist keiner.
*
* Das ist KEINE Speicherbedingung, sondern eine Abschlussbedingung -
* und der Unterschied ist der Kern der Architektur:
*
*   RAP SETDECISION      schuetzt den Schreibzugriff auf den
*                        ARBEITSZUSTAND. Ein halbfertiger Stand darf
*                        gespeichert werden, sonst kann niemand
*                        arbeiten.
*   conFLOW Decision-Exit  schuetzt den ZUSTANDSUEBERGANG. Dort ist
*                        eine fehlende Begruendung verbindlich.
*
* Waehrend der Bearbeitung ist "ESCALATE ohne Notiz" ein voellig
* normaler Zwischenzustand: erst die Aktion waehlen, dann begruenden.
* Ihn abzulehnen hiess frueher, dass die Auswahl gar nicht erst im
* Container landete - waehrend die Oberflaeche sie schon anzeigte.
* Genau der Zustand, den Befund 11.4.14 beseitigt hat.
*
* Also: speichern, aber sagen was fehlt. Der Hinweis wandert als Feld
* mit der Antwort zurueck, der Bearbeiter sieht ihn sofort, und die
* verbindliche Ablehnung kommt spaeter beim Abschluss.
*--------------------------------------------------------------------*
    CLASS-METHODS decision_hint
      IMPORTING !iv_reason     TYPE clike
                !iv_note       TYPE clike
      RETURNING VALUE(rv_text) TYPE string .

*--------------------------------------------------------------------*
* Der Satz unter den Eingabefeldern, wenn dort nichts einzugeben ist.
*
* Steht hier und nicht im JavaScript: welcher Schritt was bedeutet,
* ist Prozesswissen. Der Controller soll GEN_STAT gar nicht mehr
* auslegen muessen - er bindet ein Feld und zeigt es an.
*
* Leerer Rueckgabewert heisst: es gibt nichts zu erklaeren, hier darf
* gearbeitet werden.
*--------------------------------------------------------------------*
    CLASS-METHODS status_text
      IMPORTING !iv_gen_stat   TYPE /c09/cfl_gen_stat
      RETURNING VALUE(rv_text) TYPE string .

*--------------------------------------------------------------------*
* Die Faerbung zum Text oben. 0 neutral, 1 rot, 2 gelb, 3 gruen.
*
* Nur-Lesen ist kein Fehler, also neutral - nicht rot. Rot bleibt dem
* vorbehalten, was der Bearbeiter abstellen soll.
*--------------------------------------------------------------------*
    CLASS-METHODS status_criticality
      IMPORTING !iv_gen_stat        TYPE /c09/cfl_gen_stat
      RETURNING VALUE(rv_crit)      TYPE int1 .

ENDCLASS.

CLASS zcl_cfl_00500_rules IMPLEMENTATION.

  METHOD key_of_action.

    rv_key = COND #(
      WHEN iv_action = zcl_cfl_const_00500=>mc_recommendation-accept_new_date
        THEN /c09/cfl_cl_workflow_0101=>mc_decision-ok
      WHEN iv_action = zcl_cfl_const_00500=>mc_recommendation-partial_delivery
        THEN /c09/cfl_cl_workflow_0101=>mc_decision-uc1
      WHEN iv_action = zcl_cfl_const_00500=>mc_recommendation-cancel_remaining
        THEN /c09/cfl_cl_workflow_0101=>mc_decision-uc2
      WHEN iv_action = zcl_cfl_const_00500=>mc_recommendation-escalate
        THEN /c09/cfl_cl_workflow_0101=>mc_decision-uc3 ).

  ENDMETHOD.

  METHOD action_of_key.

*--------------------------------------------------------------------*
* Die Umkehrung, aus derselben Tabelle gebaut. Kein zweites CASE -
* sonst waere es wieder eine Zuordnung an zwei Stellen, und die
* laeuft beim naechsten Ausgang auseinander.
*--------------------------------------------------------------------*
    LOOP AT get_valid_actions( ) INTO DATA(lv_action).
      IF key_of_action( lv_action ) = iv_key.
        rv_action = lv_action.
        RETURN.
      ENDIF.
    ENDLOOP.

  ENDMETHOD.

  METHOD is_editable.

    rv_allow = xsdbool(
      iv_gen_stat = zcl_cfl_const_00500=>mc_stat-decision_red OR
      iv_gen_stat = zcl_cfl_const_00500=>mc_stat-decision_yellow ).

  ENDMETHOD.

  METHOD get_valid_actions.

    rt_action = VALUE #(
      ( CONV ty_action( zcl_cfl_const_00500=>mc_recommendation-accept_new_date ) )
      ( CONV ty_action( zcl_cfl_const_00500=>mc_recommendation-partial_delivery ) )
      ( CONV ty_action( zcl_cfl_const_00500=>mc_recommendation-cancel_remaining ) )
      ( CONV ty_action( zcl_cfl_const_00500=>mc_recommendation-escalate ) ) ).

  ENDMETHOD.

  METHOD status_text.

    IF is_editable( iv_gen_stat ) = abap_true.
      RETURN.
    ENDIF.

    rv_text = COND #(
      WHEN iv_gen_stat = zcl_cfl_const_00500=>mc_stat-escalation
        THEN 'Decided by Customer Service - read only'
      ELSE 'No open decision on this step - read only' ).

  ENDMETHOD.

  METHOD decision_hint.

    IF iv_reason IS INITIAL.
      rv_text = 'Please choose a reason for your decision'.
      RETURN.
    ENDIF.

    IF iv_reason = zcl_cfl_const_00500=>mc_reason-other
       AND iv_note IS INITIAL.
      rv_text = 'Reason "Other" needs an explanation - please add a note'.
    ENDIF.

  ENDMETHOD.

  METHOD get_valid_reasons.

    rt_reason = VALUE #(
      ( CONV ty_action( zcl_cfl_const_00500=>mc_reason-customer_agreed ) )
      ( CONV ty_action( zcl_cfl_const_00500=>mc_reason-customer_request ) )
      ( CONV ty_action( zcl_cfl_const_00500=>mc_reason-stock_situation ) )
      ( CONV ty_action( zcl_cfl_const_00500=>mc_reason-production_confirmed ) )
      ( CONV ty_action( zcl_cfl_const_00500=>mc_reason-internal_policy ) )
      ( CONV ty_action( zcl_cfl_const_00500=>mc_reason-other ) ) ).

  ENDMETHOD.

  METHOD is_valid_reason.

    DATA(lv_reason) = CONV ty_action( iv_reason ).
    DATA(lt_valid)  = get_valid_reasons( ).

    READ TABLE lt_valid TRANSPORTING NO FIELDS WITH KEY table_line = lv_reason.
    rv_ok = xsdbool( sy-subrc = 0 ).

  ENDMETHOD.

  METHOD status_criticality.

    " Nur-Lesen ist ein Zustand, kein Missstand.
    rv_crit = 0.

  ENDMETHOD.

ENDCLASS.
```

## Die wichtigste Entwurfsfrage: was fragt das Dropdown?
{% hint style="danger" %}
**Der Fehler, der am längsten unentdeckt blieb.** Die App bot im Auswahlfeld dieselben *Aktionen* an wie die conFLOW-Buttons darunter — zwei Bedienelemente für dieselbe Entscheidung. Wer oben „Escalate" wählte und unten „Partial delivery" klickte, bekam seine Auswahl **stillschweigend ersetzt**.
{% endhint %}
Auflösen lässt sich das nicht durch eine Meldung: der Startwert des Dropdowns ist die *Empfehlung*, also wäre eine Abweichung der Normalfall. Und ob der Bearbeiter das Feld angefasst hat oder nur den Startwert stehenließ, ist nicht unterscheidbar.
|  | beantwortet | kommt von |
| --- | --- | --- |
| conFLOW-Button | **Was** wird getan | dem Klick — er *ist* die Entscheidung |
| Dropdown | **Warum** | der App, kontrolliertes Vokabular |
| Notizfeld | Details zum Grund | der App, Freitext |
{% hint style="success" %}
**Der Grund ist auswertbar, die Notiz nicht.** „In wie vielen Fällen war die Bestandssituation der Auslöser?" beantwortet ein kontrolliertes Vokabular, ein Freitextfeld nicht. Genau das macht aus einem Audit Trail, den man *nachlesen* kann, einen, den man *auszählen* kann.
{% endhint %}
{% hint style="info" %}
**Und die Pflichtfeld-Regel wandert mit:** statt „bei Eskalation ist die Notiz Pflicht" gilt jetzt „**beim Grund *Sonstiges* ist die Notiz Pflicht**". Dieselbe Mechanik, aber an einer Bedingung, die nicht mit dem Button kollidiert — die Aktion steht ja erst fest, wenn der Bearbeiter klickt.
{% endhint %}

## Vier Dinge, die man daran sehen kann
{% hint style="info" %}
**Zwei Listen, zwei Fragen.** `get_valid_actions( )` liefert die Aktionen — gebraucht für die Umkehrung *Ausgang → Aktion*, die der Decision-Exit braucht. `get_valid_reasons( )` liefert die Gründe für das Dropdown. Beide sind **je eine** Quelle: Wertehilfe und Serverprüfung greifen auf dieselbe zu. Stünde die Menge zweimal da, müsste man einen neuen Eintrag an zwei Stellen nachtragen — und wer die zweite vergisst, baut den unangenehmsten Fall: **die Oberfläche bietet an, was der Server ablehnt.**
{% endhint %}

{% hint style="info" %}
**`is_editable( )` kennt nur die Schritte, auf denen gearbeitet wird.** Der Eskalationsschritt ist bewusst nicht dabei: ein Container-Feld, zwei Bearbeiter nacheinander — das macht die Historie kaputt. Der Vorschlag des ersten wäre überschrieben und die Eskalation hinterher nicht mehr erklärbar.
{% endhint %}

{% hint style="success" %}
**`decision_hint( )` ist keine Speicherbedingung, sondern eine Abschlussbedingung.** Während der Bearbeitung ist „Grund gewählt, Notiz noch leer" ein völlig normaler Zwischenzustand: erst auswählen, dann schreiben. Ihn abzulehnen hieße, dass die Auswahl gar nicht erst im Container landet — während die Oberfläche sie schon anzeigt. **Gespeichert wird, gemeldet auch** — und beim Abschluss wird aus dem gelben Hinweis ein Abbruch.
{% endhint %}

{% hint style="info" %}
**`status_text( )` und `status_criticality( )` verhindern Prozesswissen im JavaScript.** Welcher Schritt was bedeutet, entscheidet das Backend; der Controller bindet ein Feld und zeigt es an.
{% endhint %}
