# Der Behavior Pool
*Zwei Klassen in einem Include: der Handler und die Sicherungsklasse. Der Handler sammelt in einen statischen Puffer, geschrieben wird erst in `SAVE`.*

**`ZCL_CFL_00500_BEHV (global)`**

```abap
CLASS zcl_cfl_00500_behv DEFINITION
  PUBLIC
  ABSTRACT
  FINAL
  FOR BEHAVIOR OF zcfl_00500_c_exception .

  PUBLIC SECTION.

*--------------------------------------------------------------------*
* Schreibt die Entscheidung in den conFLOW-Container.
*
* Sitzt in der globalen Klasse, obwohl nur die lokale Handler-Klasse
* sie ruft: die Freundschaft zu ZCL_CFL_WORKFLOW_00500 ist der
* globalen Klasse gewaehrt, und darauf will ich mich verlassen und
* nicht darauf, ob sie sich auf lokale Klassen des Pools vererbt.
*--------------------------------------------------------------------*
    CLASS-METHODS save_decision
      IMPORTING !iv_id     TYPE guid_16
                !iv_reason TYPE clike
                !iv_note   TYPE clike .

*--------------------------------------------------------------------*
* Aus einem Schluessel wird Sprache: PARTIAL_DELIVERY -> "Partial
* delivery". Steht hier aus demselben Grund wie SAVE_DECISION: die
* Freundschaft zu ZCL_CFL_WORKFLOW_00500 hat die globale Klasse, nicht
* die lokale Handler-Klasse, die sie braucht.
*--------------------------------------------------------------------*
    CLASS-METHODS pretty
      IMPORTING !iv_code        TYPE clike
      RETURNING VALUE(rv_text)  TYPE string .

ENDCLASS.

CLASS zcl_cfl_00500_behv IMPLEMENTATION.

  METHOD save_decision.

*--------------------------------------------------------------------*
* Geschrieben werden GRUND und NOTIZ - nicht die Aktion.
*
* PROPOSED_ACTION gehoert dem conFLOW-Button: vorbelegt in B1 mit der
* Empfehlung, festgeschrieben beim Abschluss aus dem geklickten
* Ausgang (ZCL_CFL_WORKFLOW_00500=>CHECK_COMPLETION). Die App fasst es
* nicht an - sonst konkurrieren zwei Bedienelemente um dieselbe
* Entscheidung.
*--------------------------------------------------------------------*
    zcl_cfl_workflow_00500=>set_val( iv_element = zcl_cfl_const_00500=>mc_prop-decision_reason
                                     iv_id      = iv_id
                                     iv_value   = iv_reason ).

    zcl_cfl_workflow_00500=>set_val( iv_element = zcl_cfl_const_00500=>mc_prop-decision_note
                                     iv_id      = iv_id
                                     iv_value   = iv_note ).

  ENDMETHOD.

  METHOD pretty.

    rv_text = zcl_cfl_workflow_00500=>pretty_code( iv_code ).

  ENDMETHOD.

ENDCLASS.
```
**`ZCL_CFL_00500_BEHV (Local Types)`**

```abap
*--------------------------------------------------------------------*
* Schreibseite der Inbox-App.
*
* RAP trennt Aendern und Sichern: die Aktion sammelt nur im
* transaktionalen Puffer, geschrieben wird erst in SAVE. Das ist hier
* kein Formalismus - die App laeuft im selben Roundtrip wie das
* Workitem, und ein Container-Schreibvorgang mitten in der Bearbeitung
* waere nicht zurueckzunehmen, wenn der Benutzer abbricht.
*
* Geschrieben wird ueber ZCL_CFL_WORKFLOW_00500=>SET_VAL, also ueber
* /c09/cfl_cl_workflow_0101=>SET_ATTRIBUT_VALUE. Kein direkter
* Tabellenzugriff auf /C09/ - Framework nie anfassen.
*
* Kein LOCK-Handler: die Entity ist kein "lock master", und das kann
* sie bei UNMANAGED auch nicht werden. Gesperrt wird ohnehin an einer
* anderen Stelle - wer das Workitem nicht reserviert hat, bekommt die
* App gar nicht erst zu sehen.
*--------------------------------------------------------------------*
CLASS lhc_orderpromiseexception DEFINITION INHERITING FROM cl_abap_behavior_handler.

  PUBLIC SECTION.

    " Ruft die Sicherungsklasse LSC_... - zwei lokale Klassen desselben
    " Includes sind nicht automatisch befreundet. Deshalb zwei
    " oeffentliche Methoden statt eines direkten Zugriffs auf GT_BUFFER.
    CLASS-METHODS flush.

    " Gegenstueck zu FLUSH: den Puffer verwerfen, ohne ihn zu schreiben.
    CLASS-METHODS discard.

  PRIVATE SECTION.

    TYPES: BEGIN OF ty_buffer,
             wf_id  TYPE guid_16,
             reason TYPE zcfl_00500_c_exception-decisionreason,
             note   TYPE zcfl_00500_c_exception-decisionnote,
           END OF ty_buffer.

    CLASS-DATA gt_buffer TYPE SORTED TABLE OF ty_buffer WITH UNIQUE KEY wf_id.

    METHODS get_instance_features FOR INSTANCE FEATURES
      IMPORTING keys REQUEST requested_features FOR OrderPromiseException RESULT result.

    METHODS read FOR READ
      IMPORTING keys FOR READ OrderPromiseException RESULT result.

    METHODS setDecision FOR MODIFY
      IMPORTING keys FOR ACTION OrderPromiseException~setDecision RESULT result.

    CLASS-METHODS to_guid
      IMPORTING !iv_key      TYPE zcfl_00500_c_exception-wfid
      RETURNING VALUE(rv_id) TYPE guid_16.

    " Die Zeile, wie sie der Client sehen soll: Container plus das, was
    " in dieser Transaktion schon geaendert, aber noch nicht gesichert
    " wurde.
    CLASS-METHODS row_of
      IMPORTING !iv_id        TYPE guid_16
      RETURNING VALUE(rs_row) TYPE zcfl_00500_c_exception.

ENDCLASS.

CLASS lhc_orderpromiseexception IMPLEMENTATION.

  METHOD to_guid.
    " char(32) -> guid_16 ist die eingebaute C-nach-X-Konvertierung:
    " die Zeichen werden als Hex-Ziffern gelesen. Siehe ZCL_CFL_00500_QUERY.
    DATA lv_hex TYPE c LENGTH 32.

    lv_hex = iv_key.
    TRANSLATE lv_hex TO UPPER CASE.

    IF lv_hex CN '0123456789ABCDEF'.
      RETURN.
    ENDIF.

    rv_id = lv_hex.
  ENDMETHOD.

  METHOD row_of.

    rs_row = zcl_cfl_00500_query=>read_one( iv_id ).
    IF rs_row-wfid IS INITIAL.
      RETURN.
    ENDIF.

*--------------------------------------------------------------------*
* Was im Puffer steht, hat Vorrang vor dem Container - sonst springt
* das Feld auf den alten Wert zurueck, solange nicht gesichert ist.
*--------------------------------------------------------------------*
    READ TABLE gt_buffer INTO DATA(ls_buf) WITH KEY wf_id = iv_id.
    IF sy-subrc = 0.
      rs_row-decisionreason     = ls_buf-reason.
      rs_row-decisionreasontext = zcl_cfl_00500_behv=>pretty( ls_buf-reason ).
      rs_row-decisionnote       = ls_buf-note.

      " Der Hinweis gehoert zum gepufferten Stand, nicht zum
      " gespeicherten. Sonst gibt die Aktion zurueck, was VOR ihr
      " galt - und der Bearbeiter sieht "Notiz fehlt", obwohl er sie
      " gerade geschrieben hat.
      rs_row-decisionhint = zcl_cfl_00500_rules=>decision_hint(
                              iv_reason = ls_buf-reason
                              iv_note   = ls_buf-note ).
    ENDIF.

  ENDMETHOD.

  METHOD read.

    LOOP AT keys INTO DATA(ls_key).
      DATA(lv_id) = to_guid( ls_key-WfId ).
      IF lv_id IS INITIAL.
        APPEND VALUE #( %tky = ls_key-%tky ) TO failed-orderpromiseexception.
        CONTINUE.
      ENDIF.

      DATA(ls_row) = row_of( lv_id ).
      IF ls_row-wfid IS INITIAL.
        APPEND VALUE #( %tky = ls_key-%tky ) TO failed-orderpromiseexception.
        CONTINUE.
      ENDIF.

      APPEND ls_row TO result.
    ENDLOOP.

  ENDMETHOD.

  METHOD get_instance_features.
*--------------------------------------------------------------------*
* Dynamische Steuerung: darf auf diesem Schritt entschieden werden?
*
* Grundlage ist der conFLOW-Schritt, nicht die App. Eine Instanz, die
* gerade in einem Hintergrundschritt steht oder schon abgeschlossen
* ist, hat kein offenes Dialog-Workitem - dann ist GEN_STAT leer und
* es gibt nichts zu entscheiden.
*
* Die Regel selbst steht in ZCL_CFL_00500_RULES - hier wird sie nur
* gerufen. Sie wird an drei Stellen gebraucht: hier als Auskunft an
* die Oberflaeche, in SETDECISION als verbindliche Abweisung, und im
* Query-Provider als Feld ISDECISIONEDITABLE, damit der Controller
* GEN_STAT nicht mehr selbst auslegen muss.
*--------------------------------------------------------------------*

    LOOP AT keys INTO DATA(ls_key).

      DATA(lv_id) = to_guid( ls_key-WfId ).

      DATA(lv_gen_stat) = COND /c09/cfl_gen_stat(
        WHEN lv_id IS INITIAL THEN space
        ELSE row_of( lv_id )-genstat ).

      DATA(lv_open) = zcl_cfl_00500_rules=>is_editable( lv_gen_stat ).

      APPEND VALUE #(
        %tky                   = ls_key-%tky
        %action-setDecision    = COND #( WHEN lv_open = abap_true
                                         THEN if_abap_behv=>fc-o-enabled
                                         ELSE if_abap_behv=>fc-o-disabled ) )
        TO result.

    ENDLOOP.

  ENDMETHOD.

  METHOD setDecision.
*--------------------------------------------------------------------*
* Die Entscheidung erfassen.
*
* Gespeichert wird SOFORT - nicht erst mit dem Genehmigen-Button der
* Inbox. Das ist Absicht: die Buttons laufen ueber den Task-Gateway in
* die conFLOW-Logik und nicht durch diese App; es gaebe keinen
* verlaesslichen Weg, einen ungespeicherten Bildschirmzustand beim
* Entscheiden einzusammeln.
*
* Fachlich ist es ohnehin richtig so. Der Container ist der Audit
* Trail: ein Bearbeiter, der eine Aktion waehlt und dann zoegert,
* hinterlaesst eine Spur. Ein Wert, der nur im Browser lebt, ist keine
* Entscheidung.
*
* Die Wirkung entsteht spaeter und woanders: der Hintergrundschritt
* nach der Entscheidung liest PROPOSED_ACTION mit GET_VAL aus dem
* Container und handelt danach.
*--------------------------------------------------------------------*

    LOOP AT keys INTO DATA(ls_key).

      DATA(lv_id) = to_guid( ls_key-WfId ).
      IF lv_id IS INITIAL.
        APPEND VALUE #( %tky = ls_key-%tky ) TO failed-orderpromiseexception.
        CONTINUE.
      ENDIF.

      DATA(ls_row) = row_of( lv_id ).
      IF ls_row-wfid IS INITIAL.
        APPEND VALUE #( %tky = ls_key-%tky ) TO failed-orderpromiseexception.
        APPEND VALUE #( %tky = ls_key-%tky
                        %msg = new_message_with_text(
                                 severity = if_abap_behv_message=>severity-error
                                 text     = 'Workflow instance not found' ) )
               TO reported-orderpromiseexception.
        CONTINUE.
      ENDIF.

      DATA(ls_param) = ls_key-%param.

*--------------------------------------------------------------------*
* Pflichtpruefung - zum zweiten Mal.
*
* Im Dialog steht die Aktion schon als Pflichtfeld (FieldControl in
* ZCFL_00500_D_DECISION). Trotzdem wird hier erneut geprueft: eine
* Oberflaeche ist eine Bequemlichkeit fuer den Anwender, keine
* Zusicherung fuer den Server. Wer den Service direkt aufruft,
* umgeht jede Annotation.
*--------------------------------------------------------------------*
      IF ls_param-DecisionReason IS INITIAL.
        APPEND VALUE #( %tky = ls_key-%tky ) TO failed-orderpromiseexception.
        APPEND VALUE #( %tky = ls_key-%tky
                        %msg = new_message_with_text(
                                 severity = if_abap_behv_message=>severity-error
                                 text     = 'Please choose a reason' ) )
               TO reported-orderpromiseexception.
        CONTINUE.
      ENDIF.

*--------------------------------------------------------------------*
* Nur zulaessige Gruende.
*
* Die Wertehilfe bietet genau sechs an - aber sie ist Bequemlichkeit
* fuer den Anwender, keine Zusicherung fuer den Server. Wer den
* OData-Service direkt ruft, kann jeden Text schicken, der in CHAR(20)
* passt, und ohne diese Pruefung landet er im Container.
*
* Der Schaden waere subtil: PRETTY_CODE findet keinen Klartext, und
* die ComboBox zeigt ein LEERES Feld, obwohl ein Wert gespeichert ist.
* Dasselbe Argument wie bei der Feature Control unten - eine
* Oberflaeche schuetzt den Server nicht.
*--------------------------------------------------------------------*
      IF zcl_cfl_00500_rules=>is_valid_reason( ls_param-DecisionReason ) = abap_false.
        APPEND VALUE #( %tky = ls_key-%tky ) TO failed-orderpromiseexception.
        APPEND VALUE #( %tky = ls_key-%tky
                        %msg = new_message_with_text(
                                 severity = if_abap_behv_message=>severity-error
                                 text     = 'Unknown reason code - not saved' ) )
               TO reported-orderpromiseexception.
        CONTINUE.
      ENDIF.

*--------------------------------------------------------------------*
* Nur-Lese-Schritte weisen die Aktion ab.
*
* Dieselbe Regel wie in GET_INSTANCE_FEATURES, hier aber verbindlich.
* Die Feature Control ist eine Auskunft an die Oberflaeche; wer den
* Service direkt ruft, sieht sie nie. Ohne diese Pruefung koennte der
* Teamlead die Entscheidung des Customer Service ueberschreiben,
* obwohl seine Felder grau sind.
*--------------------------------------------------------------------*
      IF zcl_cfl_00500_rules=>is_editable( ls_row-genstat ) = abap_false.
        APPEND VALUE #( %tky = ls_key-%tky ) TO failed-orderpromiseexception.
        APPEND VALUE #( %tky = ls_key-%tky
                        %msg = new_message_with_text(
                                 severity = if_abap_behv_message=>severity-error
                                 text     = 'This step is read only - the decision cannot be changed here' ) )
               TO reported-orderpromiseexception.
        CONTINUE.
      ENDIF.

*--------------------------------------------------------------------*
* Der Grund "Sonstiges" braucht eine Erlaeuterung - aber nicht sofort.
*
* Die Pflicht haengt an der gewaehlten AKTION, nicht am Schritt. Sie
* stand urspruenglich auf GEN_STAT = 02, also auf dem Workitem des
* Teamleads, und damit an der falschen Stelle: die Begruendung
* schreibt der, der abgibt, nicht der, der uebernimmt.
*
* Und sie hat bis zum 22.08.2026 die Aktion ABGELEHNT. Das war seit
* dem Wegfall des Speichern-Knopfes falsch:
*
*   Bearbeiter waehlt "Escalate"
*   -> selectionChange feuert SOFORT (Auto-Save)
*   -> Notiz noch leer -> failed -> NICHTS wird gespeichert
*   -> waehrend die Oberflaeche "Escalate" schon anzeigt
*
* Also genau der Zustand, den Befund 11.4.14 beseitigt hat: das UI
* zeigt etwas, das im Container nicht steht. Und der Bearbeiter sieht
* eine Fehlermeldung fuer einen Ablauf, der voellig richtig ist - erst
* die Aktion waehlen, dann begruenden.
*
* Jetzt: gespeichert wird, gemeldet auch. Die Meldung ist eine
* WARNUNG ohne FAILED, und derselbe Text kommt ueber das Feld
* DECISION_HINT mit der Antwort zurueck, sodass die Oberflaeche ihn
* sofort zeigen kann.
*
* Verbindlich abgelehnt wird die Eskalation ohne Begruendung spaeter
* im conFLOW-Decision-Exit - dort, wo der Zustandsuebergang passiert.
*--------------------------------------------------------------------*
      DATA(lv_hint) = zcl_cfl_00500_rules=>decision_hint(
                        iv_reason = ls_param-DecisionReason
                        iv_note   = ls_param-DecisionNote ).

      IF lv_hint IS NOT INITIAL.
        APPEND VALUE #( %tky = ls_key-%tky
                        %msg = new_message_with_text(
                                 severity = if_abap_behv_message=>severity-warning
                                 text     = lv_hint ) )
               TO reported-orderpromiseexception.
      ENDIF.

*--------------------------------------------------------------------*
* In den transaktionalen Puffer. Geschrieben wird in SAVE.
*--------------------------------------------------------------------*
      READ TABLE gt_buffer ASSIGNING FIELD-SYMBOL(<buf>) WITH KEY wf_id = lv_id.
      IF sy-subrc <> 0.
        INSERT VALUE #( wf_id = lv_id ) INTO TABLE gt_buffer ASSIGNING <buf>.
      ENDIF.

      <buf>-reason = ls_param-DecisionReason.
      <buf>-note   = ls_param-DecisionNote.

*--------------------------------------------------------------------*
* Die geaenderte Instanz zurueckgeben ("result [1] $self"). Damit
* aktualisiert Fiori Elements die Objektseite ohne zweiten Aufruf -
* der neue Wert steht sofort im Block "Your decision".
*--------------------------------------------------------------------*
      APPEND VALUE #( %tky   = ls_key-%tky
                      %param = row_of( lv_id ) ) TO result.

    ENDLOOP.

*--------------------------------------------------------------------*
* HIER STAND CL_ABAP_TX=>SAVE( ) - und das war ein Irrtum.
*
* Die Idee war, der Transaktion zu melden, dass es etwas zu sichern
* gibt. Die Methode tut aber etwas anderes: sie schaltet die
* transaktionale PHASE um, und das ist mitten in einer Aktion
* verboten. Das Ergebnis war ein Laufzeitfehler, kein Speichern:
*
*   BEHAVIOR_ILLEGAL_STATEMENT
*   "Statement SAVE( ) is not allowed with this status."
*
* Es braucht die Anmeldung auch gar nicht. RAP ruft die SAVE-Sequenz
* am Ende jeder aendernden OData-Anfrage von selbst, und eine Aktion
* ist eine aendernde Anfrage. Der Puffer oben genuegt.
*--------------------------------------------------------------------*

  ENDMETHOD.

  METHOD flush.

    LOOP AT gt_buffer INTO DATA(ls_buf).
      zcl_cfl_00500_behv=>save_decision( iv_id     = ls_buf-wf_id
                                         iv_reason = ls_buf-reason
                                         iv_note   = ls_buf-note ).
    ENDLOOP.

    CLEAR gt_buffer.

  ENDMETHOD.

  METHOD discard.

    CLEAR gt_buffer.

  ENDMETHOD.

ENDCLASS.

*--------------------------------------------------------------------*
* Die Sicherungsklasse.
*
* SAVE, nicht SAVE_MODIFIED: die zweite gehoert zum "additional save"
* eines MANAGED Behaviors und bekommt die geaenderten Instanzen als
* Parameter. Bei UNMANAGED fuehrt der Handler seinen Puffer selbst -
* und genau den schreibt SAVE hier weg.
*--------------------------------------------------------------------*
CLASS lsc_zcfl_00500_c_exception DEFINITION INHERITING FROM cl_abap_behavior_saver.
  PROTECTED SECTION.
    METHODS save REDEFINITION.
    METHODS cleanup_finalize REDEFINITION.
ENDCLASS.

CLASS lsc_zcfl_00500_c_exception IMPLEMENTATION.

  METHOD save.
    lhc_orderpromiseexception=>flush( ).
  ENDMETHOD.

  METHOD cleanup_finalize.

*--------------------------------------------------------------------*
* Der Puffer ist Zustand auf Klassenebene und ueberlebt die Anfrage.
* Wird sie abgebrochen, bleibt sonst Inhalt stehen und geraet in die
* naechste - ein Wert, den niemand geschickt hat, landet im Container.
*
* Hier stand vorher RETURN. Das war eine bewusste, aber falsche
* Entscheidung: FLUSH leert den Puffer nur auf dem Erfolgspfad.
*
* Der Umweg ueber DISCARD( ) ist noetig, nicht Geschmack: diese Methode
* gehoert zur SICHERUNGSKLASSE, und die sieht GT_BUFFER der
* Handler-Klasse nicht - zwei lokale Klassen desselben Includes sind
* nicht automatisch befreundet. Genau derselbe Grund wie bei FLUSH( ).
*--------------------------------------------------------------------*
    lhc_orderpromiseexception=>discard( ).

  ENDMETHOD.

ENDCLASS.
```

## Die Reihenfolge der Prüfungen ist die Architektur
| Prüfung | Wirkung |
| --- | --- |
| Aktion nicht leer | `failed` — nichts wird gespeichert |
| Aktionsschlüssel gültig | `failed` |
| Schritt editierbar | `failed` |
| **Hinweis** (z. B. Notiz fehlt) | **Warnung ohne `failed` — es WIRD gespeichert** |
{% hint style="success" %}
**Der Unterschied, den man leicht übersieht:** eine Prüfung, die *speichern* verhindert, ist etwas anderes als eine, die *abschließen* verhindert. Beim Auto-Save ist ein halbfertiger Arbeitszustand normal. Wer ihn ablehnt, erzeugt genau die Lücke zwischen Oberfläche und Backend, die er schließen wollte.
{% endhint %}

## Zwei Fallstricke im Behavior Pool
{% hint style="danger" %}
**`cl_abap_tx=>save( )` gehört NICHT in eine Aktion.** Sie meldet nicht „hier gibt es etwas zu sichern", sondern schaltet die transaktionale *Phase* um — mitten in einer Aktion verboten: `BEHAVIOR_ILLEGAL_STATEMENT`. Nötig ist sie auch nicht: RAP ruft die SAVE-Sequenz am Ende jeder ändernden Anfrage von selbst.
{% endhint %}
{% hint style="info" %}
**Handler und Sicherungsklasse sind nicht befreundet.** Die Sicherungsklasse sieht `gt_buffer` nicht — deshalb die beiden öffentlichen Methoden `flush( )` und `discard( )`. Wer in `cleanup_finalize` direkt auf den Puffer zugreift, bekommt *„Field GT_BUFFER is unknown"*.
{% endhint %}
{% hint style="info" %}
**`cleanup_finalize` muss den Puffer leeren.** Er ist Zustand auf Klassenebene und überlebt die Anfrage; `flush( )` leert ihn nur auf dem Erfolgspfad. Bei einer abgebrochenen Anfrage geriete sonst Inhalt in die nächste.
{% endhint %}
