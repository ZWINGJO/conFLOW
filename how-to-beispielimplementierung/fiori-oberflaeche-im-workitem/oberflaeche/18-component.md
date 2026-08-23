# Der Wechsel zwischen zwei Workitems
*Das undokumentierteste Stück der ganzen Konstruktion.*

My Inbox legt je Intent **genau ein** Routing-Target an und verwendet die Component danach wieder — beim zweiten Klick wird sie nicht neu gebaut. Der S3-Controller der Inbox ruft stattdessen zwei Methoden auf, die eine generierte Fiori-Elements-App **nicht mitbringt**:
```
refreshForStartupParameter( params, false )
navigateBasedOnStartupParameter( params )
```
{% hint style="danger" %}
Fehlen sie, zeigt die Oberfläche beim Wechsel weiter die Daten des *ersten* Workitems — während die **Buttons daneben korrekt umschalten**, weil die vom Task-Gateway kommen. Dieses Bild sieht aus wie ein Fehler in der Datenbindung und ist keiner.
{% endhint %}

**`Component.js`**

```js
sap.ui.define(
    ["sap/fe/core/AppComponent", "sap/base/Log"],
    function (Component, Log) {
        "use strict";

        // Der Schluessel steckt in den Intent-Parametern, als Array.
        //
        // ZWEI Namen, und das ist kein Versehen: Beim ERSTEN Oeffnen loest
        // die Shell den Intent auf und wendet dabei das Renaming aus dem
        // Target Mapping an - dort kommt "WfId" an. Beim WECHSEL auf ein
        // anderes Workitem reicht My Inbox die Intent-Parameter roh durch,
        // also unter conFLOWs Originalnamen "CFLQueryObject00".
        // Wer nur einen der beiden liest, hat genau den halben Fall.
        function wfId(oParams) {
            var v = oParams && (oParams.WfId || oParams.CFLQueryObject00);
            if (!v) { return null; }
            return Array.isArray(v) ? v[0] : v;
        }

        return Component.extend("zcfl00500inbox.Component", {

            metadata: {
                manifest: "json"
            },

            /**
             * Umschalten auf ein anderes Workitem desselben Intents.
             *
             * My Inbox legt je Intent ("ZCFLOrderPromise-openInInbox-detail")
             * genau EIN Routing-Target an und benutzt die Component danach
             * wieder - beim zweiten Klick wird sie nicht neu gebaut. Der
             * S3-Controller der Inbox ruft stattdessen
             *
             *     refreshForStartupParameter( params, false )
             *     navigateBasedOnStartupParameter( params )
             *
             * Eine generierte Fiori-Elements-App bringt beide nicht mit.
             * Fehlen sie, zeigt die Oberflaeche beim Wechsel weiter die
             * Daten des ersten Workitems - waehrend die Buttons daneben
             * korrekt umschalten, weil die vom Task-Gateway kommen.
             * Genau dieses Bild kostete den halben Nachmittag.
             */
            navigateBasedOnStartupParameter: function (oParams) {
                var sWfId = wfId(oParams);

                if (!sWfId) { return; }

                try {
                    // OData V4 mit einem einzigen Schluesselfeld: WfId='...'
                    this.getRouter().navTo(
                        "OrderPromiseExceptionObjectPage",
                        { OrderPromiseExceptionKey: "WfId='" + sWfId + "'" },
                        true   // Historie ersetzen - die Navigation fuehrt die Inbox
                    );
                } catch (oError) {
                    // Wir laufen im Aufrufstapel der Inbox. Eine Ausnahme von
                    // hier reisst deren Detailbereich mit - lieber die alte
                    // Anzeige stehen lassen und die Ursache protokollieren.
                    Log.error("Navigation auf " + sWfId + " fehlgeschlagen",
                              oError, "zcfl00500inbox.Component");
                }
            },

            /**
             * Zweite Haelfte desselben Vertrags. Die Inbox ruft sie nur,
             * wenn es sie gibt; das eigentliche Umschalten macht die
             * Navigation oben. Bleibt bewusst leer, statt zu fehlen -
             * so steht der Vertrag vollstaendig im Code.
             */
            refreshForStartupParameter: function () {
                return;
            }

        });
    }
);
```

## Zwei Parameternamen — kein Versehen
| Erstes Öffnen | die Shell löst den Intent auf und wendet das **Renaming** aus dem Target Mapping an → `WfId` |
| --- | --- |
| Wechsel | My Inbox reicht die Intent-Parameter **roh** durch → `CFLQueryObject00` |
{% hint style="info" %}
**Wer nur einen der beiden liest, hat genau den halben Fall** — und findet den Fehler erst, wenn jemand im Termin zweimal klickt.
{% endhint %}
{% hint style="danger" %}
**Bekannte Unsicherheit:** `navigateBasedOnStartupParameter` ist **nicht dokumentiert**; der Vertrag wurde aus dem Inbox-Bundle gelesen. Stabil genug, dass SAP die Methode *abfragt* statt sie vorauszusetzen — aber ein Support Package kann sie umbenennen. Dann fällt die Oberfläche stumm auf das alte Verhalten zurück.
{% endhint %}
