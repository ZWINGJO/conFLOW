# Zwanzig Punkte, an denen es schiefgeht
*Jeder mindestens einmal live erlebt. Die rechte Spalte ist das, was man tatsächlich sieht — **zehn von zwanzig melden sich gar nicht**, drei melden sich an einer Stelle, die mit der Ursache nichts zu tun hat, und eine ist gar kein Fehler, sieht aber wie einer aus.*
| # | Falle | Erkennungsmerkmal |
| --- | --- | --- |
| 1 | `get_paging( )` nicht gerufen | `RAP_RUNTIME/014` — Liste leer, **Klasse aktiviert sauber** |
| 2 | `@UI.facet` vor der Entity statt am Schlüsselfeld | „used at wrong position (wrong scope)" für *jedes* Unterattribut |
| 3 | `@UI`-Annotation an einer Custom Entity | dieselbe Meldung — alles Darstellende gehört in die Metadata Extension |
| 4 | `openMode` fehlt im Target Mapping | Intent löst auf, Detailbereich bleibt **weiß** |
| 5 | Parameter nicht umbenannt | App lädt, filtert aber nicht — zeigt die erste beliebige Instanz |
| 6 | SWFVISU gepflegt, während SWFVMD1 einen Satz hat | Pflege wirkt nicht, **keine Meldung** |
| 7 | `/UI5/APP_INDEX_CALCULATE` vergessen | alte Fassung läuft weiter — **auch im Inkognito-Fenster** |
| 8 | IndexedDB-View-Cache *(nur ohne [Quellprojekt](../oberflaeche/20-quellprojekt.md))* | neues JS, altes XML — sieht aus wie ein Logikfehler |
| 9 | `refreshForStartupParameter` fehlt | Buttons wechseln, **Inhalt nicht** |
| 10 | `update;` im BDEF | Aktionen laufen über den EditFlow und bleiben **stumm stehen** |
| 11 | `cl_abap_tx=>save( )` in der Aktion | `BEHAVIOR_ILLEGAL_STATEMENT` |
| 12 | `invokeAction` aus einer Custom Section | weder `.then` noch `.catch` |
| 13 | Rolle fehlt beim Bearbeiter | „bei mir geht es, bei dir nicht" — Workitem zeigt den Textblock |
| 14 | `Edm.Binary` in einer V4-Aktion, ohne aktives Virenscan-Profil | `Virus scan profile /IWBEP/V4/ODATA_UPLOAD is not active` — die Ausnahme fliegt **vor** dem eigenen Handler, der Fehler steht also nicht dort, wo man ihn sucht |
| 15 | Standardfunktionen der Inbox nachgebaut | **meldet sich nie.** Es funktioniert ja — nur doppelt. Fällt erst auf, wenn jemand „Show Details" drückt |
| 16 | Objekte gelöscht, bevor die Verwender bereinigt sind | `Type "…" is unknown` in der Behavior Definition — und danach vier weitere Objekte, eines nach dem anderen |
| 17 | Leseroutine formatiert für die Anzeige und wird als Schlüsselquelle benutzt | Belegnummer ohne führende Nullen → `SELECT` findet nichts → **leere Tabelle, keine Meldung**. Gegenmittel: `ALPHA = IN` |
| 18 | Einzeldatei in SE80 geändert, während ein gebautes `Component-preload.js` in der BSP-Anwendung liegt | **meldet sich nie.** UI5 lädt das Bündel, die Datei wird gar nicht erst angefordert. Notausgang: das Bündel löschen |
| 19 | Deploy aus einem unvollständigen Projektordner | **meldet sich nie.** Der Deploy *ersetzt* die Anwendung — was nicht im Projekt liegt, ist danach im System weg. Gegenmittel: `deploy-test` |
| 20 | **Kein Fehler, sieht aber wie einer aus:** SE80 zeigt nach dem Upload generierte Dateinamen | Im BSP-Baum fehlen `Component.js` und `Component-preload.js`; stattdessen stehen dort Einträge `UI5<Hash>` und eine `UI5RepositoryPathMapping.xml`. Das ist die **physische** Ablage des SAPUI5-Repositories, die Mapping-Datei ist die Übersetzung. Die **logische** Sicht — ADT und die Laufzeit — ist vollständig. **Nicht im Baum prüfen, im Netzwerk-Tab** |

{% hint style="danger" %}
**Die Fallen 14 bis 16 hängen zusammen** und ergeben zusammen den teuersten Umweg dieses Projekts: eine Dokumentenverwaltung wurde nachgebaut (15), scheiterte am Virenscanner (14), und beim Rückbau ging die Reihenfolge schief (16). Nichts davon wäre passiert, wenn zu Beginn jemand auf „Show Details" geklickt hätte.
{% endhint %}
