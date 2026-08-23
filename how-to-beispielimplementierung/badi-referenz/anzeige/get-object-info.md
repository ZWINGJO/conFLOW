# `get_object_info`

| | |
|---|---|
| **Wann** | Wenn die FIORI-Inbox ihren Reiter "Links" aufbaut. Der Weg dorthin führt über SAP_WAPI_GET_OBJECTS, nicht über<br>GET_NEW_PREVIEW_DESCR - deshalb wirkt dieser Hook dort und der andere nicht. |
| **Rein** | IS_LPOR    das Objekt, das beschriftet werden soll |
| **Raus** | CV_RETURN  der Präfix der Beschriftung |

**WAS PASSIERT, WENN MAN IHN WEGLÄSST**

In der Fiori-Inbox steht unter "Objects and attachments" der Klassenname des Frameworks:

```
    CON: conFLOW Worflow V.0101: 0004500001234
```

Mit dem Hook:

```
    Purchase order: 0004500001234
```

Die Instanz-ID hängt das Framework selbst an - CV_RETURN ersetzt nur den Teil davor.

**DER TEXT GEHÖRT IN EIN TEXTSYMBOL, NICHT IN DEN CODE**

TEXT-001 hängt am Textpool der Klasse. Damit ist er übersetzbar, und dieselbe Beschriftung steht für beide Oberflächen an genau einer Stelle.

FALLE BEIM TRANSPORT: Textsymbole hängen am Textpool, nicht am Code. Wer die Klasse über abapGit oder einen Transport umzieht und den Textpool vergisst, hat die Beschriftung verloren - und merkt es erst in der Inbox des Zielsystems.

**DIESER HOOK SIEHT IM AUFRUFNACHWEIS TOT AUS**

Where-Used findet ihn nicht, weil er über eine Enhancement gerufen wird. Er läuft trotzdem. Merksatz für conFLOW-BAdI- Hooks allgemein: AUSPROBIEREN STATT WHERE-USED GLAUBEN.

## Der Code

```abap
cv_return = text-001.
```
