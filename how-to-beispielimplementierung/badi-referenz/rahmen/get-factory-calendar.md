# `get_factory_calendar`

> **Dieser Hook bleibt in der Referenzklasse leer.** Warum, steht unten.

| | |
|---|---|
| **Wann** | Wenn conFLOW eine Frist ausrechnet - also beim Anlegen eines Workitems mit Fristangabe in /C09/CFL_C02. |
| **Rein** | IS_CFL_S01           die Instanz |
| **Raus** | CV_FACTORY_CALENDAR  der Fabrikkalender (SCAL-FCALID) |

```
WOFÜR   Damit "zwei Tage Frist" nicht am Freitagnachmittag abläuft,
       weil Samstag und Sonntag mitgezählt wurden.
```

**IN KEINER DER UNTERSUCHTEN PRODUKTIVIMPLEMENTIERUNGEN GEFÜLLT.**

Das heißt NICHT, dass Fristen ohne Kalender richtig sind - es heißt, dass die untersuchten Prozesse Fristen in Stunden rechnen oder gar keine haben.

**WANN MAN IHN BRAUCHT**

Sobald eine Frist in TAGEN läuft und die Eskalation eine Wirkung hat, die jemanden ärgert. Ein Workitem, das über Ostern eskaliert, weil vier Feiertage mitgezählt wurden, ist der klassische erste Produktivfehler.

Der Kalender kommt dann meistens aus dem Werk oder der Buchungskreis-Zuordnung des Belegs - also aus den Belegdaten, nicht aus einer Konstante.

**BLEIBT LEER.**
