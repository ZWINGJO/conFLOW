# Service Definition und App-Descriptor

**`ZCFL_00500_SD`**

```abap
@EndUserText.label: 'Order Promise Exception - My Inbox'
define service ZCFL_00500_SD {
  expose ZCFL_00500_C_EXCEPTION as OrderPromiseException;
  expose ZCFL_00500_C_ACTIONVH  as ActionValueHelp;

  // Die Einteilungen. Muss exponiert sein, damit Fiori Elements die
  // Association aufloesen kann - eine Tabelle im Facet holt ihre
  // Zeilen ueber einen eigenen Request.
  expose ZCFL_00500_C_SCHEDLINE as ScheduleLine;
}
```
Das Binding wird als **OData V4 · UI** angelegt und publiziert. Ein neu exponiertes Entity Set erscheint nach dem Aktivieren der Service Definition; ein erneutes Publizieren ist nicht nötig.

**`manifest.json`**

```json
{
  "_version": "1.65.0",
  "sap.app": {
    "id": "zcfl00500inbox",
    "type": "application",
    "i18n": {
      "bundleUrl": "i18n/i18n.properties",
      "supportedLocales": [
        ""
      ],
      "fallbackLocale": ""
    },
    "applicationVersion": {
      "version": "0.0.1"
    },
    "title": "{{appTitle}}",
    "description": "{{appDescription}}",
    "resources": "resources.json",
    "sourceTemplate": {
      "id": "@sap.adt.sevicebinding.deploy:lrop",
      "version": "1.0.0",
      "toolsId": "70A8A561D1C71FD1A7A7C79E27DEC000"
    },
    "dataSources": {
      "mainService": {
        "uri": "/sap/opu/odata4/sap/zcfl_00500_sb/srvd/sap/zcfl_00500_sd/0001/",
        "type": "OData",
        "settings": {
          "odataVersion": "4.0"
        }
      }
    }
  },
  "sap.ui": {
    "technology": "UI5"
  },
  "sap.ui5": {
    "flexEnabled": true,
    "resources": {
      "js": [],
      "css": []
    },
    "dependencies": {
      "minUI5Version": "1.136.0",
      "libs": {
        "sap.fe.templates": {}
      },
      "components": {}
    },
    "models": {
      "i18n": {
        "type": "sap.ui.model.resource.ResourceModel",
        "settings": {
          "bundleName": "zcfl00500inbox.i18n.i18n",
          "supportedLocales": [
            ""
          ],
          "fallbackLocale": ""
        }
      },
      "@i18n": {
        "type": "sap.ui.model.resource.ResourceModel",
        "uri": "i18n/i18n.properties",
        "settings": {
          "supportedLocales": [
            ""
          ],
          "fallbackLocale": ""
        }
      },
      "": {
        "dataSource": "mainService",
        "preload": true,
        "settings": {
          "operationMode": "Server",
          "autoExpandSelect": true,
          "earlyRequests": true
        }
      }
    },
    "routing": {
      "config": {},
      "routes": [
        {
          "pattern": ":?query:",
          "name": "OrderPromiseExceptionList",
          "target": "OrderPromiseExceptionList"
        },
        {
          "pattern": "OrderPromiseException({OrderPromiseExceptionKey}):?query:",
          "name": "OrderPromiseExceptionObjectPage",
          "target": "OrderPromiseExceptionObjectPage"
        }
      ],
      "targets": {
        "OrderPromiseExceptionList": {
          "type": "Component",
          "id": "OrderPromiseExceptionList",
          "name": "sap.fe.templates.ListReport",
          "options": {
            "settings": {
              "entitySet": "OrderPromiseException",
              "variantManagement": "Page",
              "initialLoad": "Enabled",
              "navigation": {
                "OrderPromiseException": {
                  "detail": {
                    "route": "OrderPromiseExceptionObjectPage"
                  }
                }
              },
              "controlConfiguration": {}
            }
          }
        },
        "OrderPromiseExceptionObjectPage": {
          "type": "Component",
          "id": "OrderPromiseExceptionObjectPage",
          "name": "sap.fe.templates.ObjectPage",
          "options": {
            "settings": {
              "entitySet": "OrderPromiseException",
              "editableHeaderContent": false,
              "allowDeepLinking": true,
              "navigation": {},
              "controlConfiguration": {
                "@com.sap.vocabularies.UI.v1.Facets": {}
              },
              "content": {
                "body": {
                  "sections": {
                    "DecisionSection": {
                      "template": "zcfl00500inbox.ext.DecisionSection",
                      "title": "{i18n>decisionSection}",
                      "position": {
                        "placement": "After",
                        "anchor": "SchedLine"
                      }
                    }
                  }
                }
              }
            }
          }
        }
      }
    },
    "extends": {
      "extensions": {
        "sap.ui.controllerExtensions": {
          "sap.fe.templates.ObjectPage.ObjectPageController": {
            "controllerName": "zcfl00500inbox.ext.DecisionSection"
          }
        }
      }
    }
  },
  "sap.fiori": {
    "archeType": "transactional"
  }
}
```

## Der Einhängepunkt der Custom Section
```
"content": { "body": { "sections": {
  "DecisionSection": {
    "template": "zcfl00500inbox.ext.DecisionSection",
    "title": "{i18n>decisionSection}",
    "position": { "placement": "After", "anchor": "SchedLine" }
} } } }
```
{% hint style="info" %}
**Der `anchor` ist der Name eines Facets aus der [Metadata Extension](../backend/mdx.md)**, nicht der einer Feldgruppe und nicht der Abschnittstitel. Er entscheidet, *wo* die eigene Section landet — hier hinter der Einteilungstabelle. Trifft der Name kein Facet, rutscht der Abschnitt kommentarlos ans Ende der Seite; das sieht nach einer Geschmacksfrage aus und ist ein Tippfehler.
{% endhint %}
{% hint style="info" %}
**Die App-ID aus `sap.app.id`** ist das, was im Target Mapping als *ID* einzutragen ist — **nicht** der BSP-Name. Verwechslung führt zu einem Intent, der auflöst und nichts lädt.
{% endhint %}
{% hint style="info" %}
**`supportedLocales: [""]` und `fallbackLocale: ""`** stehen dreimal, weil es genau *ein* Textbündel ohne Sprachsuffix gibt. Ohne die Angabe sucht UI5 zur Laufzeit nach Sprachvarianten, die es nie geben wird, und der [Build](quellprojekt.md) mahnt einen englischen Fallback an. Die Datei liegt im Quellprojekt, nicht in SE80 — sie ist die Stelle, an der App und Auslieferung zusammenkommen.
{% endhint %}
