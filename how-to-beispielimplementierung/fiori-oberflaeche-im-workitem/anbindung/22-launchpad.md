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

## Berechtigung — Teil eins: kommt der Anwender hin?
{% hint style="danger" %}
**Ohne Rolle löst der Intent nicht auf** — das Workitem zeigt kommentarlos wieder den Textblock. **Die häufigste Ursache für „bei mir geht es, bei dir nicht".**
{% endhint %}

## Berechtigung — Teil zwei: darf er *diesen* Vorgang sehen?
{% hint style="danger" %}
**Das prüft hier niemand.** Die Rolle gewährt Zugriff auf den *Service*, nicht auf *bestimmte Instanzen*. Wer authentifiziert ist und eine gültige Workflow-ID kennt, liest deren Container — Kunde, Material, Mengen, Termine, Entscheidungsgrund — und kann über die Aktion hineinschreiben.
{% endhint %}

RAP macht das nicht von selbst. In der hier gezeigten Fassung gibt es **keine** der drei Stellen, an denen es entstehen würde:
| Wo Instanzschutz normalerweise sitzt | hier |
| --- | --- |
| `AUTHORITY-CHECK` im Query-Provider oder im Handler | **nein** |
| DCL / Access Control auf der Entity | **nein** — eine Custom Entity hat keine |
| `get_global_authorizations` im Behavior Pool | **nein** |

{% hint style="info" %}
**Warum das im Referenzprojekt vertretbar war**

Es ist ein Showcase auf Demo-Daten, die Entscheidung ist dort bewusst getroffen und dokumentiert. Und der Angriffsweg ist schmal: der ungefilterte Listenmodus wurde gestrichen, es gibt also keinen Weg, *alle* laufenden Vorgänge zu lesen, ohne eine einzige ID zu kennen. Man braucht eine gültige GUID. **Für einen produktiven Nachbau reicht das nicht.** Eine Workflow-ID ist kein Geheimnis — sie steht in URLs, Protokollen und Mails.
{% endhint %}

**Was zu tun ist**, wenn die App produktiv geht: dieselbe Frage beantworten, die der Workflow ohnehin schon beantwortet — *ist der angemeldete Benutzer Bearbeiter dieses Workitems?* Die Auskunft steht in `/c09/cfl_s03` neben der Instanz. Ein Vergleich gegen `sy-uname` im Query-Provider (lesen) **und** im Action-Handler (schreiben) schließt die Lücke, ohne ein neues Berechtigungsobjekt zu erfinden. Dieselbe Regel zweimal gerufen, nicht zweimal geschrieben — wie bei `is_editable( )`.

{% hint style="info" %}
**Die Probe:** Workflow-ID aus einem fremden Workitem nehmen, mit einem Benutzer anmelden, der damit nichts zu tun hat, und das Entity Set direkt aufrufen. Kommen Daten, ist die Lücke offen — und zwar unabhängig davon, was die Oberfläche ausgraut. Feature Control sieht nur, wer durch die App geht.
{% endhint %}

## Der Cache
UI5 liefert die App-Dateien unter einem Hash-Pfad aus, der aus dem App-Index stammt. Eine Änderung in SE80 ändert ihn **nicht**.
{% hint style="success" %}
**Die Hälfte davon erledigt sich mit dem [Quellprojekt](../oberflaeche/20-quellprojekt.md).** Wer baut und deployt, liefert Controller und Fragment als *ein* Bündel mit *einem* Hash aus — der Zustand „neues JS, altes XML" kann dann nicht mehr entstehen. Die Server-Zeilen der Tabelle bleiben trotzdem: nach jedem Deploy einmal `/UI5/APP_INDEX_CALCULATE`.
{% endhint %}
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
**Fragment und Controller werden getrennt gecacht** — solange sie als Einzeldateien ausgeliefert werden. Das XML sitzt im IndexedDB-View-Cache, das JS nicht; sie können also **unterschiedlich alt** sein. Neues Verhalten mit alter Oberfläche sieht aus wie ein Logikfehler und ist keiner. **Mit gebautem `Component-preload.js` ist dieser Fall weg**, an seine Stelle tritt ein anderer: eine in SE80 geänderte Einzeldatei wird gar nicht mehr angefordert.
{% endhint %}
