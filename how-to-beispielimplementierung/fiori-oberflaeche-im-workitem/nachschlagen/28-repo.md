# conFLOW auf GitHub
*Wo das Produkt liegt, zu dem dieses How-To gehört.*

{% hint style="info" %}
**Repository**

[github.com/ZWINGJO/conFLOW](https://github.com/ZWINGJO/conFLOW)
{% endhint %}

Dieses Dokument beschreibt **eine** Erweiterung von conFLOW: die Ablösung des Workitem-Textblocks durch eine eigene Fiori-Oberfläche. Es ersetzt weder die Produktdokumentation noch das Customizing-Handbuch — es setzt beide voraus.

| Wenn du das suchst | schau hier |
| --- | --- |
| Das Produkt, Releases, Installation | [github.com/ZWINGJO/conFLOW](https://github.com/ZWINGJO/conFLOW) |
| Customizing der Tabellen `C01`–`C10`, Bearbeiterfindung, WF-Start | Produktdokumentation — dieses How-To setzt sie voraus |
| Die BAdI-Hooks im Überblick | [Kapitel „Die Anbindung im conFLOW-BAdI"](../anbindung/21-badi.md) — hier stehen nur die drei, die die App anbinden |
| Den vollständigen Code des Referenz-Workflows | in diesem Dokument, eingebettet in den jeweiligen Kapiteln |

{% hint style="info" %}
**Zum Aktualisieren.** Das How-To wird erzeugt, nicht gepflegt: `python3 tools/build_howto_doc.py` liest jede Quelle frisch aus `src/` und schreibt die HTML-Datei neu. Wer eine Kopie ablegt — in einer Hilfe, einem Wiki, einem Repo —, legt damit eine **Momentaufnahme** ab. Nach Änderungen am Referenz-Workflow neu erzeugen und die Kopie ersetzen, sonst weicht sie vom Code ab. Genau das zu verhindern war der Grund, den Generator überhaupt zu bauen.
{% endhint %}
