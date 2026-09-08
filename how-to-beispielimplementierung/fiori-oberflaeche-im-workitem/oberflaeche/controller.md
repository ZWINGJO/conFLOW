# Die Controller Extension
*Das Herzstück der Oberfläche — und die Datei mit den meisten Fallen.*

**`ext/DecisionSection.controller.js`**

```js
sap.ui.define([
    "sap/ui/core/mvc/ControllerExtension",
    "sap/ui/model/json/JSONModel",
    "sap/m/MessageBox",
    "sap/m/ObjectStatus",
    "sap/base/Log"
], function (ControllerExtension, JSONModel, MessageBox, ObjectStatus, Log) {
    "use strict";

    // Der Name der RAP-Aktion aus der Behavior Definition.
    var ACTION_NAME = "setDecision";

    // Das Entity Set der Werteliste, so wie ZCFL_00500_SD es exponiert.
    var VH_ENTITY_SET = "/ActionValueHelp";

    // Zugleich der Name der Controller Extension und die KOMPONENTE fuer
    // sap/base/Log. Damit laesst sich in den DevTools nach dieser einen
    // Erweiterung filtern, statt die Meldungen an einem selbstgebauten
    // Textpraefix zu erkennen - dafuer ist das dritte Argument da.
    //
    // Sichtbar wird das Protokoll erst ab dem passenden Log-Level:
    //
    //     ?sap-ui-log-level=INFO      im URL-Parameter
    //     sap.base.Log.setLevel(3)    zur Laufzeit in der Konsole
    //
    // Das ist der Unterschied zu console: die Ausgaben sind noch da,
    // aber der Anwender sieht sie nicht mehr im Normalbetrieb.
    var LOGGER = "zcfl00500inbox.ext.DecisionSection";

    /**
     * Ein Fehlerobjekt in den Detailtext von sap/base/Log falten.
     *
     * Log erwartet dort einen STRING. Reicht man ein Error-Objekt direkt
     * durch, steht im Protokoll "[object Error]" - also genau die
     * Meldung nicht, wegen der man hineinschaut. Bei OData V4 ist das
     * besonders bitter, weil die fachliche Begruendung ohnehin schon
     * eine Ebene tiefer in error.details[] liegt.
     */
    function detail(vError) {
        if (!vError) {
            return "";
        }
        return vError.message || String(vError);
    }

    // Wie lange nach dem letzten Tastendruck gewartet wird, bevor die
    // Notiz weggeschrieben wird. Kurz genug, dass sie steht, bevor
    // jemand unten auf einen conFLOW-Knopf klickt; lang genug, dass
    // nicht jedes Wort eine Anfrage ausloest.
    var NOTE_DELAY_MS = 800;

    // KEIN Prozesswissen mehr in dieser Datei.
    //
    // Hier standen EDITABLE_STEPS = ["01","03"] und STEP_ESCALATION =
    // "02" - eine zweite Kopie der Regel, die im ABAP schon stand. Beim
    // naechsten zusaetzlichen Schritt waeren die beiden auseinander-
    // gelaufen, und zwar unbemerkt: die Oberflaeche haette geoeffnet,
    // was das Backend abweist, oder umgekehrt.
    //
    // Stattdessen liefert der Service drei Felder, gefuellt aus
    // ZCL_CFL_00500_RULES:
    //
    //   IsDecisionEditable          darf hier entschieden werden
    //   DecisionStatusText          warum nicht, im Klartext
    //   DecisionStatusCriticality   wie der Text zu faerben ist
    //
    // Der Controller legt GEN_STAT nicht mehr aus. Er bindet Felder,
    // meldet Fehler und kennt den Prozess nicht - so soll es sein.

    /**
     * Den vollqualifizierten Aktionsnamen aus den Metadaten holen.
     *
     * Er lautet <Namensraum>.<Aktion>, und der Namensraum wird beim
     * Publish erzeugt - er folgt der Service-URI:
     *
     *   .../srvd/sap/zcfl_00500_sd/0001/
     *        -> com.sap.gateway.srvd.sap.zcfl_00500_sd.v0001
     *
     * Ihn hart einzutragen ist die haeufigste Fehlerquelle: ein
     * fehlendes Namensraum-Segment, und der Aufruf meldet nur
     * "action not found". Der Entity Container traegt denselben
     * Namensraum, also wird er von dort abgeleitet - dann stimmt er
     * auch nach einem Umzug in ein anderes Paket oder System.
     */
    function fullActionName(oModel, sAction) {
        try {
            var sContainer = oModel.getMetaModel().getObject("/$EntityContainer");
            if (sContainer && sContainer.lastIndexOf(".") > 0) {
                return sContainer.substring(0, sContainer.lastIndexOf(".")) + "."
                       + (sAction || ACTION_NAME);
            }
        } catch (e) {
            Log.error("Namensraum nicht ermittelbar", detail(e), LOGGER);
        }
        return null;
    }

    return ControllerExtension.extend(LOGGER, {

        override: {
            // Der Erweiterungspunkt liegt unter "routing", nicht auf
            // oberster Ebene - das ist die haeufigste Verwechslung bei
            // Controller Extensions in Fiori Elements V4. Er laeuft bei
            // jedem Binden, also auch beim Wechsel des Workitems.
            routing: {
                onAfterBinding: function (oContext) {
                    this.setDecisionModel(oContext);
                    this.wireControls();
                }
            }
        },

        /**
         * Die Werteliste aus dem Service holen.
         *
         * Programmatisch, nicht ueber items="{/ActionValueHelp}" im
         * XML: die deklarative Bindung gegen den OData-Service ist in
         * dieser UI5-Version stumm leer geblieben - dieselbe Klasse
         * von Problem wie beim Aktionsaufruf, und dieselbe Loesung.
         * Ueber die Modell-API sieht man in der Konsole, ob etwas
         * ankommt.
         *
         * Geladen wird einmal je Sitzung. Die vier Aktionen haengen
         * nicht am Workitem, und ein Nachladen bei jedem Wechsel waere
         * eine Anfrage ohne neuen Inhalt.
         *
         * KEINE fest verdrahtete Ersatzliste. Genau die war der Grund
         * fuer diesen Umbau: die Aktionen standen an drei Stellen
         * (Konstantenklasse, Werteliste, Fragment) und liefen
         * auseinander. Eine Notfassung im Controller waere die vierte
         * - und die heimtueckischste, weil sie stillschweigend
         * einspringt. Der Service liefert nicht? Dann ist das Dropdown
         * leer, es steht in der Konsole, und es steht am Schirm.
         * Sichtbar kaputt ist besser als unsichtbar veraltet.
         */
        loadActions: function (oModel) {
            var that = this;

            if (this._pActions) {
                return this._pActions;
            }

            this._pActions = new Promise(function (resolve) {
                try {
                    var oList = oModel.bindList(VH_ENTITY_SET);
                    oList.requestContexts(0, 100).then(function (aContexts) {
                        var aRows = aContexts.map(function (oCtx) {
                            return {
                                ActionKey:  oCtx.getProperty("ActionKey"),
                                ActionText: oCtx.getProperty("ActionText")
                            };
                        });
                        if (!aRows.length) {
                            Log.warning("Werteliste ist leer - " + VH_ENTITY_SET, "", LOGGER);
                            // Nicht merken: ein leeres Ergebnis kann ein
                            // Aussetzer sein, beim naechsten Binden noch
                            // einmal versuchen.
                            that._pActions = null;
                        } else {
                            Log.info("Werteliste geladen: " + aRows.length, "", LOGGER);
                        }
                        resolve(aRows);
                    }).catch(function (oError) {
                        Log.error("Werteliste nicht ladbar", detail(oError), LOGGER);
                        that._pActions = null;
                        resolve([]);
                    });
                } catch (e) {
                    Log.error("Werteliste nicht bindbar", detail(e), LOGGER);
                    that._pActions = null;
                    resolve([]);
                }
            });

            return this._pActions;
        },

        /**
         * Modell aufsetzen und mit dem vorbelegen, was im Container
         * steht - der in B1 berechneten Empfehlung.
         *
         * Werteliste und Vorbelegung werden ZUSAMMEN gesetzt. Kaeme
         * selectedKey vor der Liste, stuende die ComboBox leer da,
         * obwohl der Wert stimmt - sie findet den Schluessel dann
         * schlicht in keinem Eintrag.
         */
        setDecisionModel: function (oContext) {
            var oView     = this.base.getView(),
                oDecision = oView.getModel("decision"),
                oOData    = oView.getModel(),
                that      = this;

            if (!oDecision) {
                oDecision = new JSONModel();
                oView.setModel(oDecision, "decision");
            }

            oDecision.setData({ Reason: "", Note: "", Reasons: [],
                                Editable: false });
            this._sLastSaved = undefined;
            this.cancelNoteTimer();
            this.setStatus("");

            // Zu, bis feststeht, dass geoeffnet werden darf. Zwischen
            // hier und der Antwort des Servers liegen ein paar hundert
            // Millisekunden - lang genug, um auf einem Nur-Lese-Workitem
            // zu tippen, wenn die Felder solange offen stehen.
            this.applyEditable(false);

            if (!oContext) { return Promise.resolve(); }

            // Laufnummer gegen ueberlappende Durchlaeufe.
            //
            // onAfterBinding feuert mehr als einmal, und beide Laeufe
            // sind asynchron - der aeltere kann NACH dem juengeren
            // fertig werden und schreibt dann seinen veralteten Stand
            // ueber den aktuellen.
            //
            // Vorsorge, kein Befund: als "Action list unavailable" am
            // Schirm stand, sah das nach genau diesem Rennen aus. Es
            // war es nicht - der Query-Provider der Werteliste hat
            // GET_PAGING nicht gerufen und RAP hat die Anfrage
            // abgewiesen. Die Laufnummer bleibt trotzdem: dass
            // onAfterBinding mehrfach feuert, steht schon laenger im
            // Kommentar oben, und zwei ueberlappende Laeufe waeren ein
            // Fehler, den man erst im Termin sieht.
            //
            // Nur der juengste Lauf darf Modell und Statuszeile
            // anfassen. Alle aelteren steigen still aus.
            var iRun = (this._iRun || 0) + 1;
            this._iRun = iRun;

            // requestProperty statt getProperty!
            //
            // Seit das Facet "Your decision" aus der Metadata Extension
            // raus ist, fordert Fiori Elements diese Felder nicht mehr
            // an - sie stehen schlicht nicht im $select. getProperty
            // liefert dann leer, ohne dass etwas kaputt waere.
            // requestProperty laedt sie nach und liefert ein Promise.
            //
            // Die drei Decision-Status-Felder sind aus demselben Grund
            // dabei: sie stehen nicht in der Metadata Extension und
            // werden deshalb nicht von selbst angefordert.
            return Promise.all([
                oContext.requestProperty(["DecisionReason", "DecisionNote",
                                          "IsDecisionEditable", "DecisionStatusText",
                                          "DecisionStatusCriticality"]),
                this.loadActions(oOData)
            ]).then(function (aResult) {
                if (iRun !== that._iRun) {
                    Log.info("Lauf " + iRun + " ueberholt - verworfen", "", LOGGER);
                    return;
                }

                var aValues   = aResult[0],
                    aActions  = aResult[1],
                    bEditable = aValues[2] === true || aValues[2] === "X",
                    sStatus   = aValues[3] || "",
                    iCrit     = aValues[4];

                oDecision.setData({
                    Reasons:     aActions,
                    Reason:      aValues[0] || "",
                    Note:        aValues[1] || "",
                    Editable:    bEditable
                });

                // Readonly ZWEIMAL: ueber die Bindung im Fragment und
                // hier noch einmal von Hand.
                //
                // Guertel und Hosentraeger, mit Grund. Fragment und
                // Controller werden getrennt ausgeliefert und getrennt
                // gecacht - das XML sitzt im IndexedDB-View-Cache, das
                // JS nicht. Laeuft ein alter Fragment-Stand, fehlt die
                // Bindung und die Felder sind offen, obwohl das Modell
                // "nur lesen" sagt. Genau so ist es aufgefallen: die
                // Statuszeile stand richtig, die Felder waren trotzdem
                // eingebbar.
                //
                // Ein setEditable auf einem Control, das ohnehin schon
                // gebunden ist, kostet nichts und macht die Regel
                // unabhaengig davon, welche XML-Fassung im Browser
                // liegt.
                that.applyEditable(bEditable);

                // Auf einem Nur-Lese-Schritt sagt die Zeile, warum
                // nichts einzugeben ist. Ohne sie sieht ein graues
                // Feld nach einem Fehler aus. Text UND Faerbung kommen
                // aus dem Backend - hier wird nichts mehr entschieden.
                if (!bEditable) {
                    that.setStatus(sStatus, that.critToState(iCrit));
                } else if (!aActions.length) {
                    // Ein leeres Dropdown ohne Erklaerung sieht aus wie
                    // "es gibt nichts zu waehlen". Es gibt aber sehr
                    // wohl etwas - der Service liefert es nur nicht.
                    that.setStatus("Action list unavailable - please contact support", "Warning");
                }

                Log.info("Modell gesetzt", JSON.stringify(oDecision.getData()), LOGGER);
            }).catch(function (oError) {
                if (iRun !== that._iRun) { return; }
                Log.error("Vorbelegung fehlgeschlagen", detail(oError), LOGGER);

                // Die Felder starten geschlossen und werden erst
                // geoeffnet, wenn der Zustand feststeht. Faellt die
                // Ermittlung aus, bleiben sie zu - richtig so, denn
                // ohne die Antwort des Servers weiss niemand, ob hier
                // entschieden werden darf. Aber es muss am Schirm
                // stehen, sonst
                // sitzt der Bearbeiter vor zwei grauen Feldern ohne
                // jeden Hinweis und haelt es fuer Absicht.
                that.setStatus("Decision context could not be loaded - please reopen the work item", "Error");
            });
        },

        /**
         * Ein Control der Custom Section suchen.
         *
         * Nicht ueber byId: Fiori Elements stellt Fragment-IDs ein
         * generiertes Praefix voran, der schlichte Name findet nichts.
         * Gesucht wird deshalb ueber die Aggregation der View nach dem
         * Namensbestandteil.
         */
        findControl: function (sIdPart, sType) {
            var aFound = this.base.getView().findAggregatedObjects(true, function (oCtrl) {
                return oCtrl.isA(sType) && oCtrl.getId().indexOf(sIdPart) > -1;
            });
            return aFound.length ? aFound[0] : null;
        },

        /**
         * Ereignisse programmatisch anhaengen statt im XML.
         *
         * Handler-Referenzen der Form ".extension.<Name>.<Methode>"
         * haben in dieser UI5-Version nicht aufgeloest - das Control
         * wurde daraufhin gar nicht gerendert. Programmatisch ist es
         * unabhaengig davon, wie die Extension registriert ist.
         *
         * Das Anhaengen passiert einmal je Control; ein Merker am
         * Control verhindert Doppelungen beim naechsten Binden.
         */
        wireControls: function () {
            var that = this;

            var oBox = this.findControl("idDecisionSelect", "sap.m.ComboBox");
            if (oBox && !oBox.data("wired")) {
                oBox.attachSelectionChange(function () { that.onDecide(); });
                oBox.data("wired", true);
                Log.info("Auswahl verdrahtet", "", LOGGER);
            }

            var oNote = this.findControl("idDecisionNote", "sap.m.TextArea");
            if (oNote && !oNote.data("wired")) {
                // Zweimal, mit Absicht.
                //
                // "change" feuert erst beim Verlassen des Feldes. Das
                // allein ist ein Rennen gegen die conFLOW-Knoepfe in
                // der Fussleiste: die liegen ausserhalb dieser App, und
                // wer aus dem Notizfeld direkt dorthin klickt, loest
                // Sichern und Entscheiden fast gleichzeitig aus.
                //
                // "liveChange" mit kurzer Verzoegerung schliesst diese
                // Luecke - nach einer Schreibpause steht die Notiz
                // schon im Container, bevor irgendwer klickt.
                oNote.attachChange(function () {
                    that.cancelNoteTimer();
                    that.onDecide();
                });
                oNote.attachLiveChange(function () {
                    that.cancelNoteTimer();
                    that._iNoteTimer = window.setTimeout(function () {
                        that._iNoteTimer = null;
                        that.onDecide();
                    }, NOTE_DELAY_MS);
                });
                oNote.data("wired", true);
                Log.info("Notizfeld verdrahtet", "", LOGGER);
            }

            if (!oBox) {
                Log.warning("Auswahl NICHT gefunden", "", LOGGER);
            }
        },

        /**
         * Die beiden Eingaben schalten.
         *
         * Nur lesen heisst NICHT ausgegraut: der Teamlead soll sehen,
         * was der Customer Service gewaehlt und geschrieben hat.
         * "editable" laesst den Text schwarz, "enabled" wuerde ihn
         * grau machen und schlechter lesbar.
         */
        applyEditable: function (bEditable) {
            var oBox  = this.findControl("idDecisionSelect", "sap.m.ComboBox"),
                oNote = this.findControl("idDecisionNote", "sap.m.TextArea");

            if (oBox && oBox.setEditable) {
                oBox.setEditable(!!bEditable);
            }
            if (oNote && oNote.setEditable) {
                oNote.setEditable(!!bEditable);
            }
        },

        /**
         * Criticality des Backends in einen ObjectStatus-Zustand.
         *
         * Dieselbe Skala wie ueberall in Fiori Elements: 0 neutral,
         * 1 rot, 2 gelb, 3 gruen. Die Zuordnung ist Darstellung und
         * gehoert deshalb hierher - die BEWERTUNG, welcher Schritt
         * welche Criticality bekommt, steht im ABAP.
         */
        critToState: function (iCrit) {
            switch (Number(iCrit)) {
                case 1:  return "Error";
                case 2:  return "Warning";
                case 3:  return "Success";
                default: return "None";
            }
        },

        /**
         * Speichervorgaenge nacheinander, nicht nebeneinander.
         *
         * Ohne diese Kette kann ein aelterer Stand zuletzt im Container
         * landen:
         *
         *   Save A startet -> Benutzer aendert -> Save B startet
         *   -> B kommt zuerst zurueck -> A kommt danach an
         *   -> SET_VAL ueberschreibt B mit A
         *
         * Der Merker _sLastSaved schuetzt davor NICHT: er verhindert
         * identische Doppelaufrufe, nicht das Ueberholen. Und im
         * Backend gewinnt schlicht, wer zuletzt schreibt.
         *
         * Wahrscheinlich ist das nicht - OData V4 buendelt Anfragen und
         * der Browser schickt sie der Reihe nach. Aber ein Audit Trail,
         * der unter Last die falsche Reihenfolge festhaelt, bricht sein
         * Versprechen, und die Absicherung kostet fuenf Zeilen.
         *
         * Das zweite Argument im then/catch ist Absicht: nach einem
         * gescheiterten Save muss die Kette weiterlaufen, sonst waere
         * ein einziger Fehler das Ende jeder weiteren Speicherung.
         */
        queueSave: function (fnSave) {
            this._pChain = (this._pChain || Promise.resolve()).then(fnSave, fnSave);
            return this._pChain;
        },

        cancelNoteTimer: function () {
            if (this._iNoteTimer) {
                window.clearTimeout(this._iNoteTimer);
                this._iNoteTimer = null;
            }
        },

        /**
         * Die Zeile unter den Feldern.
         *
         * Ohne Knopf muss der Bearbeiter sehen, dass etwas passiert
         * ist - sonst ist "wird automatisch gespeichert" eine
         * Behauptung, die er glauben muss.
         *
         * sState faerbt: "Success" gruen fuer gesichert, "Error" rot
         * fuer abgewiesen, "None" neutral fuer alles Erklaerende.
         */
        setStatus: function (sText, sState) {
            // Zwei Typen, mit Absicht.
            //
            // Die Zeile war frueher ein sap.m.Text und ist jetzt ein
            // sap.m.ObjectStatus (wegen der Faerbung). Fragment und
            // Controller werden aber getrennt ausgeliefert und getrennt
            // gecacht - das XML sitzt im IndexedDB-View-Cache, das JS
            // nicht. Sucht der Controller nur den neuen Typ und laeuft
            // noch das alte Fragment, findet er nichts und schreibt ins
            // Leere: keine Meldung, kein Fehler, kein Hinweis worauf.
            //
            // Genau so ist es aufgefallen. Beide Typen zu akzeptieren
            // kostet zwei Zeilen und macht die Meldung unabhaengig
            // davon, welche Fassung der Browser gerade haelt.
            var oStatus = this.findControl("idDecisionStatus", "sap.m.ObjectStatus")
                       || this.findControl("idDecisionStatus", "sap.m.Text")
                       || this._createStatus();

            if (!oStatus) {
                Log.warning("Statuszeile nicht gefunden: " + sText, "", LOGGER);
                return;
            }

            oStatus.setText(sText || "");

            // setState gibt es nur am ObjectStatus. Am alten Text ist
            // die Meldung dann eben schwarz - lesbar bleibt sie.
            if (oStatus.setState) {
                oStatus.setState(sState || "None");
            }
        },

        /**
         * Die Statuszeile notfalls selbst anlegen.
         *
         * Sie steht im Fragment - aber Fragment und Controller werden
         * getrennt ausgeliefert und getrennt gecacht: das XML sitzt im
         * IndexedDB-View-Cache, das JS nicht. Laeuft eine aeltere
         * Fragment-Fassung, in der die Zeile fehlt oder anders heisst,
         * schreibt setStatus ins Leere - keine Rueckmeldung, kein
         * Fehler, kein Hinweis worauf. Das ist heute dreimal passiert
         * und hat jedesmal wie ein Logikfehler ausgesehen.
         *
         * Hier ist es kein Schoenheitsfehler, sondern der Kern: ohne
         * Speichern-Knopf IST die Zeile die Rueckmeldung. Fehlt sie,
         * muss der Bearbeiter glauben, dass etwas passiert ist.
         *
         * Also: gibt es sie nicht, wird sie angelegt und neben das
         * Notizfeld gehaengt. Einmal je View, danach findet der
         * normale Weg sie wieder.
         */
        _createStatus: function () {
            var oNote = this.findControl("idDecisionNote", "sap.m.TextArea"),
                oForm = oNote && oNote.getParent();

            if (!oForm || !oForm.addContent) {
                return null;
            }

            var oStatus = new ObjectStatus(
                this.base.getView().createId("idDecisionStatus"),
                { text: "", state: "None" });

            oStatus.addStyleClass("sapUiSmallMarginBegin");
            oStatus.addStyleClass("sapUiTinyMarginTop");
            oForm.addContent(oStatus);

            Log.info("Statuszeile nachtraeglich angelegt", "", LOGGER);
            return oStatus;
        },

        /**
         * Aus einem OData-Fehler den Satz herausholen, der den
         * Bearbeiter angeht.
         *
         * Die Meldung aus dem Behavior Pool ("An escalation needs a
         * reason - please add a note") steckt bei V4 in den Details,
         * nicht in oError.message - dort steht der technische
         * Rahmentext ("HTTP request was not processed because $batch
         * failed"). Wer nur message zeigt, zeigt dem Anwender die
         * Verpackung statt des Inhalts.
         */
        errorText: function (oError) {
            var oDetail = oError && oError.error;

            if (oDetail) {
                if (oDetail.details && oDetail.details.length) {
                    // Die erste Detailmeldung mit Text ist die
                    // fachliche - die technischen kommen danach.
                    for (var i = 0; i < oDetail.details.length; i++) {
                        if (oDetail.details[i] && oDetail.details[i].message) {
                            return oDetail.details[i].message;
                        }
                    }
                }
                if (oDetail.message) {
                    return oDetail.message;
                }
            }

            return (oError && oError.message) || "Saving failed";
        },

        /**
         * Speichern - die RAP-Aktion direkt ueber das OData-V4-Modell.
         *
         * NICHT ueber extensionAPI.invokeAction. Der Weg ist am
         * 21.08.2026 nachweislich gescheitert: der Aufruf ging los
         * (Konsole), aber die Zusage wurde weder erfuellt noch
         * abgelehnt - kein .then, kein .catch, keine Anfrage, kein Satz
         * im Container, kein Dump. invokeAction laeuft durch den
         * EditFlow und den Parameterdialog von Fiori Elements; beide
         * treffen hier auf eine Entity ohne Draft und ohne Sticky und
         * bleiben stumm stehen.
         *
         * bindContext( "<Aktion>(...)" ) ist der dokumentierte Weg der
         * OData-V4-Modell-API und kennt weder EditFlow noch Dialog: das
         * Modell schickt die Anfrage, execute( ) liefert Erfolg oder
         * Fehler. Genau das, was hier gebraucht wird - die Validierung
         * liegt ohnehin im ABAP, der Client reicht nur weiter.
         */
        onDecide: function () {
            var oView    = this.base.getView(),
                oContext = oView.getBindingContext(),
                oModel   = oView.getModel(),
                oData    = oView.getModel("decision").getData(),
                that     = this;

            if (!oContext) {
                Log.error("Kein Bindungskontext", "", LOGGER);
                return;
            }

            // Auf einem Nur-Lese-Schritt gar nicht erst losschicken.
            //
            // Die Felder sind dort nicht eingebbar, ein Aufruf kaeme
            // also nur ueber einen Umweg zustande - und das Backend
            // weist ihn ohnehin ab (IS_EDITABLE in SETDECISION). Der
            // Unterschied ist die Fehlermeldung: ohne diese Zeile
            // saehe der Bearbeiter eine rote Box fuer etwas, das er
            // gar nicht getan hat.
            if (!oData.Editable) {
                Log.info("Schritt ist nur lesend - nicht gespeichert", "", LOGGER);
                return;
            }

            // Ohne gewaehlten Grund nicht speichern - sonst meldete das
            // Backend "Please choose a reason", sobald jemand die Notiz
            // vor der Auswahl tippt und wegklickt.
            if (!oData.Reason) {
                Log.info("Noch kein Grund gewaehlt - nicht gespeichert", "", LOGGER);
                return;
            }

            var sAction = fullActionName(oModel);
            if (!sAction) {
                Log.error("Aktionsname nicht ermittelbar", "", LOGGER);
                return;
            }

            // Dropdown und Notizfeld melden sich getrennt. Wer die
            // Aktion waehlt und danach ins Notizfeld klickt, loest zwei
            // Aufrufe mit demselben Inhalt aus - der zweite ist nur
            // Laerm auf der Leitung und ein zweiter Hinweis am Schirm.
            var sPayload = JSON.stringify([oData.Reason, oData.Note || ""]);
            if (this._sLastSaved === sPayload) {
                Log.info("unveraendert - nicht gespeichert", "", LOGGER);
                return;
            }
            this._sLastSaved = sPayload;

            Log.info("setDecision -> " + sAction, JSON.stringify(oData), LOGGER);

            var oOperation;
            try {
                oOperation = oModel.bindContext(sAction + "(...)", oContext, {
                    $$inheritExpandSelect: false
                });
                oOperation.setParameter("DecisionReason", oData.Reason);
                oOperation.setParameter("DecisionNote",   oData.Note || "");
            } catch (e) {
                Log.error("Aktion nicht bindbar", detail(e), LOGGER);
                MessageBox.error("The decision action is not available in this service.");
                return;
            }

            var sStamp = new Date().toLocaleTimeString();

            // Welche Instanz wird hier gespeichert?
            //
            // queueSave( ) serialisiert ueber die Lebensdauer des
            // Controllers - und der ueberlebt den Wechsel des
            // Workitems, weil My Inbox die Component wiederverwendet.
            // Ohne diesen Merker koennte die Rueckmeldung eines Saves
            // von Workitem A in Workitem B landen:
            //
            //   A aendern -> Save A startet -> sofort B oeffnen
            //   -> setDecisionModel(B) laeuft -> Save A endet
            //   -> "Saved 09:26" erscheint bei B, obwohl dort nichts
            //      gespeichert wurde
            //
            // Und im Fehlerfall schlimmer: das Zuruecknehmen von
            // _sLastSaved traefe den Merker von B.
            //
            // Der Save von A soll fertig werden - er gehoert zu A und
            // ist richtig. Nur die ANZEIGE darf B nicht anfassen.
            var sSavedPath = oContext.getPath();

            var stillOnSameInstance = function () {
                var oNow = oView.getBindingContext();
                return !!oNow && oNow.getPath() === sSavedPath;
            };

            return this.queueSave(function () {
                return oOperation.execute();
            }).then(function () {
                Log.info("gespeichert", "", LOGGER);

                // NICHT nachladen.
                //
                // Hier stand requestSideEffects gefolgt von
                // setDecisionModel - "die Objektseite nachziehen". Das
                // war falsch und hat Eingaben gefressen:
                //
                //   Notiz aendern -> liveChange -> nach 800 ms sichern
                //   -> Erfolg -> Felder vom Server neu laden -> der
                //   gelesene Wert landet im Feld, IN DEM DER CURSOR
                //   NOCH STEHT.
                //
                // Sichtbar wurde es beim Loeschen: "Saved" stimmte, und
                // gleich darauf stand der alte Text wieder da.
                //
                // Nachzuladen gibt es ohnehin nichts. Die Anzeige haengt
                // am JSON-Modell "decision", und dessen Inhalt haben wir
                // gerade selbst gesendet - beim Speichern ist der Client
                // die Wahrheit, nicht der Server. Ob es wirklich
                // angekommen ist, beantwortet execute( ): kommt kein
                // Fehler, steht es im Container.
                if (!stillOnSameInstance()) {
                    Log.info("gespeichert, aber Workitem gewechselt - keine Anzeige", "", LOGGER);
                    return;
                }

                // Fehlt zum Abschluss noch etwas?
                //
                // Die Aktion gibt die geaenderte Instanz zurueck
                // ("result [1] $self"), und darin steht DecisionHint -
                // aus ZCL_CFL_00500_RULES, also aus derselben Quelle
                // wie alle anderen Prozessregeln. Kein zweiter
                // Roundtrip, und vor allem kein Prozesswissen hier:
                // WANN etwas fehlt, entscheidet das Backend, der
                // Controller zeigt nur den Satz.
                //
                // Gespeichert ist trotzdem - der Hinweis sagt, was zum
                // ABSCHLUSS fehlt, nicht was das Speichern verhindert
                // hat. Verbindlich abgelehnt wird spaeter im
                // conFLOW-Decision-Exit.
                var sHint = "";
                try {
                    var oResult = oOperation.getBoundContext();
                    sHint = (oResult && oResult.getProperty("DecisionHint")) || "";
                } catch (e) {
                    // Kein Grund, deshalb die Erfolgsmeldung zu
                    // unterschlagen - gespeichert wurde ja.
                    Log.warning("Hinweis nicht lesbar", detail(e), LOGGER);
                }

                if (sHint) {
                    that.setStatus("Saved " + sStamp + " - " + sHint, "Warning");
                } else {
                    that.setStatus("Saved " + sStamp, "Success");
                }
            }).catch(function (oError) {
                Log.error("setDecision FEHLGESCHLAGEN", detail(oError), LOGGER);

                if (!stillOnSameInstance()) {
                    // Weder Meldung noch Merker anfassen - beides
                    // gehoert inzwischen einem anderen Workitem.
                    return;
                }

                // Merker zuruecknehmen, sonst gilt der gescheiterte
                // Stand als gespeichert und ein zweiter Versuch mit
                // demselben Inhalt wird stillschweigend verworfen.
                that._sLastSaved = undefined;

                // KEINE MessageBox.
                //
                // Sie ist modal, und ausgeloest wird sie vom
                // liveChange-Timer: wer bei "Escalate" ohne Notiz
                // weiterschreibt, bekaeme sie nach jeder Schreibpause
                // erneut vor die Nase und muesste sie wegklicken, um
                // genau das zu tun, was sie verlangt.
                //
                // Die rote Zeile unter den Feldern sagt dasselbe, steht
                // solange bis der Grund behoben ist, und laesst die
                // Hand auf der Tastatur.
                that.setStatus(that.errorText(oError), "Error");
            });
        }

    });
});
```

## `bindContext` statt `invokeAction`
`extensionAPI.invokeAction` ist die Bequemlichkeitsschicht von Fiori Elements: sie bringt den Parameterdialog mit und läuft durch den EditFlow. Beides ist hier falsch — der Dialog ist überflüssig (die Felder stehen auf der Seite), und der EditFlow bleibt ohne Draft/Sticky **stumm stehen**.
{% hint style="info" %}
**Merksatz**

Ein Aufruf, der weder `.then` noch `.catch` auslöst, **hat das Netz nie verlassen**. Erst wenn ein Fehlertext kommt, redet man über Backend.
{% endhint %}

## Speichern ohne Knopf — fünf Bedingungen
| # |  | warum |
| --- | --- | --- |
| 1 | zwei Ereignisse am Textfeld | `change` feuert erst beim Verlassen — ein Rennen gegen die Buttons, die außerhalb der App liegen. `liveChange` mit ~800 ms **verkleinert das Fenster**, schließt es aber nicht |
| 2 | eine sichtbare Statuszeile | ohne Rückmeldung muss der Bearbeiter glauben, dass etwas passiert ist |
| 3 | ein Merker gegen den Doppelaufruf | Auswahlfeld und Textfeld melden sich getrennt |
| 4 | eine Queue | sonst kann ein **älterer** Stand zuletzt im Container landen |
| 5 | ein Instanz-Guard im Callback | die Queue überlebt den Workitem-Wechsel — sonst erscheint „Saved" beim falschen |

{% hint style="danger" %}
**Nach dem Speichern NICHT nachladen.** `requestSideEffects` im Erfolgspfad schreibt den gelesenen Wert in das Feld, **in dem der Cursor noch steht**. Beim Speichern ist der Client die Wahrheit — der gesendete Inhalt ist der aktuelle.
{% endhint %}

## Der Fehlertext steckt in `details[]`
Bei OData V4 steht in `oError.message` nur der Rahmentext („HTTP request was not processed because $batch failed"). Die fachliche Meldung aus dem Behavior Pool liegt in `error.details[]`. Wer nur `message` zeigt, zeigt dem Anwender die Verpackung statt des Inhalts.

{% hint style="info" %}
**Keine `MessageBox` im Auto-Save-Pfad.** Sie ist modal und wird vom Timer ausgelöst — wer bei einer Pflichtangabe weiterschreibt, müsste sie nach jeder Schreibpause wegklicken, um genau das zu tun, was sie verlangt.
{% endhint %}

{% hint style="success" %}
**Kein Prozesswissen in dieser Datei.** Ob hier entschieden werden darf, welcher Text bei Nur-Lesen steht, was noch fehlt — alles Felder aus der Regelklasse. Der Controller bindet, zeigt an und meldet Fehler.
{% endhint %}
