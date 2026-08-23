# Die Bearbeiterfindung

Bearbeiterfindung - zentral, nicht in der Workflow-Klasse.

**WARUM EINE EIGENE KLASSE**

Die Rollenauflösung ist immer derselbe Block: Funktionsbaustein rufen, sortieren, entdoppeln, Präfix davor. Sechs Zeilen, die nichts mit dem einzelnen Workflow zu tun haben.

Steht sie in der WF-Klasse, steht sie beim dritten Workflow dreimal im System - und beim ersten Fehler korrigiert man zwei davon. Außerdem: dieselbe Rolle wird typischerweise von mehreren Workflows gebraucht ("Einkauf", "Buchhaltung"). Eine gemeinsame Klasse ist der Ort, an dem man nachsieht, wer das eigentlich ist.

**WAS HIER NICHT HINEINGEHÖRT**

Die ENTSCHEIDUNG, welcher Bearbeiterkreis welchen Helfer bekommt. Die steht in GET_ACTORS der Workflow-Klasse, weil sie workflow-spezifisch ist. Hier steht nur, WIE man an die Personen kommt - nicht, WANN.

**DAS PRÄFIX IST DER HÄUFIGSTE FEHLER**

Ein Eintrag in der Bearbeiterliste ist immer ein typisiertes Org-Objekt:

```
US<uname>      Benutzer
S<planstelle>  Planstelle
O<orgeinheit>  Organisationseinheit
AC<rolle>      Rolle
```

Ein blanker Benutzername erzeugt kein Workitem und keine Fehlermeldung. Deshalb setzen alle Methoden hier das Präfix selbst - der Aufrufer soll gar nicht erst in die Lage kommen, es zu vergessen.

## Die ganze Klasse

```abap
CLASS zcl_cfl_get_actors DEFINITION
  PUBLIC
  FINAL
  CREATE PUBLIC.

*----------------------------------------------------------------------*
* Bearbeiterfindung - zentral, nicht in der Workflow-Klasse.
*
* WARUM EINE EIGENE KLASSE
*
* Die Rollenaufloesung ist immer derselbe Block: Funktionsbaustein
* rufen, sortieren, entdoppeln, Praefix davor. Sechs Zeilen, die
* nichts mit dem einzelnen Workflow zu tun haben.
*
* Steht sie in der WF-Klasse, steht sie beim dritten Workflow dreimal
* im System - und beim ersten Fehler korrigiert man zwei davon.
* Ausserdem: dieselbe Rolle wird typischerweise von mehreren
* Workflows gebraucht ("Einkauf", "Buchhaltung"). Eine gemeinsame
* Klasse ist der Ort, an dem man nachsieht, wer das eigentlich ist.
*
* WAS HIER NICHT HINEINGEHOERT
*
* Die ENTSCHEIDUNG, welcher Bearbeiterkreis welchen Helfer bekommt.
* Die steht in GET_ACTORS der Workflow-Klasse, weil sie
* workflow-spezifisch ist. Hier steht nur, WIE man an die Personen
* kommt - nicht, WANN.
*
* DAS PRAEFIX IST DER HAEUFIGSTE FEHLER
*
* Ein Eintrag in der Bearbeiterliste ist immer ein typisiertes
* Org-Objekt:
*
*     US<uname>      Benutzer
*     S<planstelle>  Planstelle
*     O<orgeinheit>  Organisationseinheit
*     AC<rolle>      Rolle
*
* Ein blanker Benutzername erzeugt kein Workitem und keine
* Fehlermeldung. Deshalb setzen alle Methoden hier das Praefix
* selbst - der Aufrufer soll gar nicht erst in die Lage kommen, es
* zu vergessen.
*----------------------------------------------------------------------*

  PUBLIC SECTION.

*--------------------------------------------------------------------*
* Die Rollennamen.
*
* Sie stehen als Konstanten hier und nicht als Literale in den
* Methoden - dann sieht man an einer Stelle, welche Rollen dieser
* Workflow-Baukasten voraussetzt. Das ist die Liste, die der
* Basis-Kollege beim Aufsetzen eines neuen Systems braucht.
*
* In einer groesseren Installation gehoeren sie in eine
* Customizing-Tabelle - Rollennamen unterscheiden sich zwischen
* Entwicklungs- und Produktivsystem oefter, als man denkt.
*--------------------------------------------------------------------*
    CONSTANTS mc_role_buyer      TYPE agr_name VALUE 'Z_CFL_BUYER' ##NO_TEXT.
    CONSTANTS mc_role_supervisor TYPE agr_name VALUE 'Z_CFL_SUPERVISOR' ##NO_TEXT.

    CLASS-METHODS buyer
      RETURNING VALUE(rt_actors) TYPE /c09/cfl_wf_tt_actors.

    CLASS-METHODS supervisor
      RETURNING VALUE(rt_actors) TYPE /c09/cfl_wf_tt_actors.

*--------------------------------------------------------------------*
* Der eine Baustein, den alle benutzen.
*
* Oeffentlich, damit eine Workflow-Klasse mit einer Sonderrolle sie
* aufrufen kann, ohne dass hier fuer jeden Einzelfall eine Methode
* dazukommt.
*--------------------------------------------------------------------*
    CLASS-METHODS by_role
      IMPORTING iv_role          TYPE agr_name
      RETURNING VALUE(rt_actors) TYPE /c09/cfl_wf_tt_actors.

ENDCLASS.


CLASS zcl_cfl_get_actors IMPLEMENTATION.

  METHOD buyer.
    rt_actors = by_role( mc_role_buyer ).
  ENDMETHOD.


  METHOD supervisor.
    rt_actors = by_role( mc_role_supervisor ).
  ENDMETHOD.


  METHOD by_role.
*--------------------------------------------------------------------*
* Alle Benutzer einer Rolle als Bearbeiter.
*
* TWP_GET_ROLE_USER_ASSIGNMENT ist der Standardweg. Zwei Parameter
* verdienen Aufmerksamkeit:
*
*   NO_USERS_FROM_COMPOSITE_ROLES = SPACE
*       Sammelrollen werden MITGENOMMEN. Das ist fast immer richtig
*       und fast nie offensichtlich: in vielen Installationen haengen
*       die Benutzer an Sammelrollen, und die Einzelrolle waere leer.
*       Wer den Parameter auf 'X' setzt, bekommt einen Workflow, der
*       im Entwicklungssystem laeuft und im Produktivsystem
*       niemanden findet.
*
*   Die Exceptions
*       ROLE_NOT_FOUND ist der Fall, der im Produktivsystem
*       tatsaechlich vorkommt - die Rolle wurde nicht angelegt oder
*       heisst anders. Er darf NICHT zu einem Kurzdump fuehren,
*       sondern zu einer leeren Liste. Der Auffangbearbeiter in
*       GET_ACTORS faengt das dann ab, und im Workitem steht, dass
*       niemand gefunden wurde.
*
* SORTIEREN UND ENTDOPPELN
*       Ein Benutzer kann ueber mehrere Rollenzuordnungen kommen. Ohne
*       DELETE ADJACENT DUPLICATES bekommt er dasselbe Workitem
*       mehrfach - was im SAP-GUI wie ein Fehler aussieht und in der
*       Fiori-Inbox wie drei offene Aufgaben.
*--------------------------------------------------------------------*

    DATA lt_users TYPE TABLE OF twpagruser.

    IF iv_role IS INITIAL.
      RETURN.
    ENDIF.

    CALL FUNCTION 'TWP_GET_ROLE_USER_ASSIGNMENT'
      EXPORTING  role_name                     = iv_role
                 no_users_from_composite_roles = space
      TABLES     role_user_assignment          = lt_users
      EXCEPTIONS role_not_found                = 1
                 no_role_user_assignment_found = 2
                 not_supported                 = 3
                 no_role_name_specified        = 4
                 OTHERS                        = 5.

    IF sy-subrc <> 0.
      RETURN.                                " leere Liste, kein Dump
    ENDIF.

    SORT lt_users BY uname.
    DELETE ADJACENT DUPLICATES FROM lt_users COMPARING uname.

    LOOP AT lt_users INTO DATA(ls_user).
      IF ls_user-uname IS NOT INITIAL.
        APPEND |US{ ls_user-uname }| TO rt_actors.
      ENDIF.
    ENDLOOP.

  ENDMETHOD.

ENDCLASS.
```
