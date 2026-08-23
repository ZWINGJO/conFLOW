# Launchpad, Intent und Berechtigung
*Drei Zeilen entscheiden, ob die App erscheint — und keine davon meldet sich, wenn sie fehlt.*

## Semantic Object
Transaktion `/UI2/SEMOBJ`. Ohne Eintrag lässt sich im Target Mapping kein Intent anlegen.

## Target Mapping · `/UI2/FLPD_CUST`
Bewusst **nur ein Target Mapping, keine Kachel** — die App wird nie aus dem Launchpad heraus gestartet, sondern immer aus einem Workitem.
| Semantic Object | `ZCFLOrderPromise` |
| --- | --- |
| Action | `openInInbox` |
| Application Type | SAPUI5 Fiori App |
| URL | *leer* |
| ID | `zcfl00500inbox` — die **Component-ID** aus der manifest.json |
| Device Types | Desktop · Tablet · Phone |
| Allow additional parameters | ✔ — My Inbox reicht weitere Parameter durch |
| `openMode` | `embedIntoDetailsNestedRouter` · mandatory |
| `CFLQueryObject00` | Value leer · **Target Name** `WfId` |

## Berechtigung
{% hint style="danger" %}
**Ohne Rolle löst der Intent nicht auf** — das Workitem zeigt kommentarlos wieder den Textblock. **Die häufigste Ursache für „bei mir geht es, bei dir nicht".**
{% endhint %}

## Der Cache
UI5 liefert die App-Dateien unter einem Hash-Pfad aus, der aus dem App-Index stammt. Eine Änderung in SE80 ändert ihn **nicht**.
| Schicht | „Cache leeren" | Inkognito | Was hilft |
| --- | --- | --- | --- |
| HTTP-Cache | ja | leer | Hard Reload |
| localStorage / IndexedDB | **nein** | leer | DevTools → Application → **Clear site data** |
| App-Index (Server) | — | — | `/UI5/APP_INDEX_CALCULATE` |
| Cache-Buster (Server) | — | — | `/UI2/INVALIDATE_CLIENT_CACHES` |
{% hint style="info" %}
**Diagnose**

**Inkognito zeigt den Server, das normale Fenster zeigt den Cache.** Sind *beide* alt, fehlt der App-Index. Ist nur *eines* alt, sitzt es im Browser-Storage — und „Cache leeren" fasst IndexedDB nicht an.
{% endhint %}
{% hint style="info" %}
**Fragment und Controller werden getrennt gecacht.** Das XML sitzt im IndexedDB-View-Cache, das JS nicht — sie können **unterschiedlich alt** sein. Neues Verhalten mit alter Oberfläche sieht aus wie ein Logikfehler und ist keiner.
{% endhint %}
