# Die Custom Section — das Fragment
*Bemerkenswert daran ist, was **nicht** darin steht.*

**`ext/DecisionSection.fragment.xml`**

```xml
<core:FragmentDefinition
    xmlns="sap.m"
    xmlns:core="sap.ui.core"
    xmlns:f="sap.ui.layout.form">

    <!--
      Custom Section "Your decision".

      Absichtlich ohne jede Handler-Referenz und ohne Datenbindung an
      den Service. Beides hat in dieser UI5-Version nicht aufgeloest:
      der Knopf wurde gar nicht erst gerendert, die Werteliste blieb
      leer - beides ohne sichtbare Fehlermeldung.

      Gebunden wird stattdessen durchgehend an das JSON-Modell
      "decision", das der Controller fuellt. Es haelt drei Dinge:

        /Reason  /Note        die beiden Eingaben
        /Reasons               die Werteliste aus ZCFL_00500_C_ACTIONVH
        /Editable              darf auf diesem Schritt entschieden werden

      Das Auswahlfeld fragt nach dem GRUND, nicht nach der Aktion.
      Die Aktion entscheidet der conFLOW-Button in der Fussleiste -
      bis zum 22.08.2026 standen hier dieselben vier Werte, und wer
      oben etwas anderes waehlte als er unten klickte, bekam seine
      Auswahl stillschweigend ersetzt. Zwei Bedienelemente fuer
      dieselbe Frage sind nicht aufloesbar; also stellt dieses hier
      eine andere.

      Der Umweg ueber das JSON-Modell ist kein Selbstzweck. Die
      Werteliste kommt aus dem OData-Service, aber erst nachdem der
      Controller sie geholt hat - und sie muss stehen, BEVOR
      selectedKey gesetzt wird, sonst zeigt die ComboBox ein leeres
      Feld, obwohl der Wert stimmt. Ein Modell, ein Zeitpunkt, kein
      Rennen.
    -->
    <VBox>

        <f:SimpleForm layout="ResponsiveGridLayout"
                      editable="true"
                      labelSpanXL="3" labelSpanL="3" labelSpanM="3"
                      columnsXL="1" columnsL="1">
            <f:content>

                <Label text="{i18n>decisionReason}"/>
                <ComboBox id="idDecisionSelect"
                          width="20rem"
                          selectedKey="{decision>/Reason}"
                          editable="{decision>/Editable}"
                          items="{decision>/Reasons}"
                          placeholder="{i18n>decisionPlaceholder}">
                    <core:Item key="{decision>ActionKey}" text="{decision>ActionText}"/>
                </ComboBox>

                <Label text="{i18n>decisionNote}"/>
                <TextArea id="idDecisionNote"
                          width="100%"
                          rows="4"
                          growing="true"
                          value="{decision>/Note}"
                          editable="{decision>/Editable}"
                          placeholder="{i18n>decisionNotePlaceholder}"/>

            </f:content>
        </f:SimpleForm>

        <!--
          Kein Speichern-Knopf. Gesichert wird, sobald eine Auswahl
          getroffen ist und sobald die Notiz stehenbleibt - der
          Container ist der Audit Trail, und ein Knopf, den man
          vergessen kann, ist dort die schlechtere Loesung.

          Die Zeile darunter sagt, wann zuletzt gesichert wurde. Ohne
          sie waere "wird automatisch gespeichert" eine Behauptung,
          die der Bearbeiter glauben muss. Auf einem Nur-Lese-Schritt
          steht dort stattdessen, warum nichts einzugeben ist, und im
          Fehlerfall die Begruendung aus dem Backend.

          ObjectStatus statt Text: dieselbe Zeile traegt drei Aussagen
          (gesichert / nur lesen / abgewiesen), und "state" faerbt sie
          gruen, neutral oder rot. Ein Fehler, der aussieht wie eine
          Erfolgsmeldung, ist keiner - man liest ihn schlicht nicht.
        -->
        <ObjectStatus id="idDecisionStatus"
                      text=""
                      state="None"
                      class="sapUiSmallMarginBegin sapUiSmallMarginBottom sapUiTinyMarginTop"/>

    </VBox>

</core:FragmentDefinition>
```

{% hint style="danger" %}
**Keine Handler-Referenz, keine Bindung an den OData-Service.** Beides hat in dieser UI5-Version nicht aufgelöst — das Control wurde dann gar nicht erst gerendert bzw. die Liste blieb leer, jeweils **ohne Fehlermeldung**.
{% endhint %}
Gebunden wird stattdessen durchgehend an ein JSON-Modell, das der Controller füllt:
| Pfad | Inhalt |
| --- | --- |
| `/Reason` | der gewählte Entscheidungsgrund |
| `/Note` | die Notiz |
| `/Reasons` | die Werteliste aus dem Service |
| `/Editable` | darf auf diesem Schritt entschieden werden |
{% hint style="info" %}
**Werteliste und Vorbelegung zusammen setzen.** Käme `selectedKey` vor der Liste, stünde die ComboBox leer da, obwohl der Wert stimmt — sie findet den Schlüssel dann in keinem Eintrag.
{% endhint %}

## Warum das Feld nach dem *Grund* fragt und nicht nach der *Aktion*
Der naheliegende Entwurf war ein Dropdown mit denselben Aktionen, die auch die conFLOW-Buttons darunter anbieten. Er ist gescheitert, und zwar unrettbar: **zwei Bedienelemente für dieselbe Frage lassen sich nicht abgleichen.** Wer oben „Eskalieren" wählte und unten „Teillieferung" klickte, bekam seine Auswahl stillschweigend ersetzt — der Button gewinnt, er löst den Zustandsübergang aus.
Eine Warnmeldung hätte nicht geholfen: das Feld ist mit der berechneten *Empfehlung* vorbelegt, eine Abweichung davon ist also der Normalfall. Also stellt das Feld eine andere Frage.
| Bedienelement | beantwortet | gehört |
| --- | --- | --- |
| conFLOW-Button | **was** getan wird | dem Framework |
| Dropdown | **warum** | der App |
| Notizfeld | Details | der App |
{% hint style="success" %}
**Verallgemeinert:** Alles, was den Workflow weiterschaltet, gehört dem Framework. Die eigene Oberfläche ergänzt Kontext und Begründung — sie konkurriert nicht um die Entscheidung selbst.
{% endhint %}

## Die Texte
**`i18n/i18n.properties`**

```ini
#XTIT: Application title
appTitle=Order Promise Exception

#XTIT: Application description
appDescription=Order Promise Exception - My Inbox

#XFLD: Custom Section
decisionSection=Your decision
decisionReason=Reason
decisionNote=Note
decisionPlaceholder=Why did you decide this way?
decisionNotePlaceholder=Optional - mandatory for reason "Other"
```
{% hint style="info" %}
**Elf Zeilen, und trotzdem ein Befund.** Die Meldungstexte des Controllers und des Behavior Pools stehen *nicht* hier, sondern hartcodiert im jeweiligen Quelltext — für einen einsprachigen Showcase vertretbar, vor einem Produktivgang nicht. Wer das sauber will, nimmt für die ABAP-Seite eine Nachrichtenklasse und für die JavaScript-Seite dieses Bundle.
{% endhint %}
