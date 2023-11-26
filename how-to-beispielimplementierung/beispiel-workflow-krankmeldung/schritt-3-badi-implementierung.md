# 📝 Schritt 3: BADI Implementierung

{% hint style="info" %}
Die nächsten Punkte zeigen, wie man mittels BADI Implementierung Anforderungen sehr einfach lösen kann.
{% endhint %}

<figure><img src="../../.gitbook/assets/Folie19.png" alt=""><figcaption></figcaption></figure>

{% hint style="info" %}
Über die Transaktion SE80 muss eine Erweiterungsimplementierung durchgeführt werden und diese mittels Filter auf die jeweilige Workflow-Definition eingeschränkt werden.
{% endhint %}

<figure><img src="../../.gitbook/assets/Folie20 (1).png" alt=""><figcaption></figcaption></figure>

{% hint style="info" %}
Ein Schritt kann auch dynamisch abhängig von diversen Eigenschaften ermittelt werden. Um dies zu ermöglichen, muss einfach im Genehmigungsschritt "BADI" ausgewählt werden.
{% endhint %}

<figure><img src="../../.gitbook/assets/Folie21 (1).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../.gitbook/assets/Folie22 (1).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../.gitbook/assets/Folie23 (1).png" alt=""><figcaption></figcaption></figure>

{% hint style="info" %}
Um einen parallelen Schritt zu definieren, müssen die parallel laufenden Userstatus zum Genehmigungsschritt hinterlegt werden.
{% endhint %}

<figure><img src="../../.gitbook/assets/Folie24.png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../.gitbook/assets/Folie25.png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../.gitbook/assets/Folie26.png" alt=""><figcaption></figcaption></figure>

{% hint style="info" %}
Um den Bearbeiter einer Rolle dynamisch zu ermitteln, kann die Methode "GET\_ACTORS" in der BADI-Implementierung verwendet werden.
{% endhint %}

<figure><img src="../../.gitbook/assets/Folie27.png" alt=""><figcaption></figcaption></figure>

{% hint style="info" %}
Die Meldungen aus den Hintergrundschritten werden in einen BALOG fortgeschrieben und können dementsprechend ausgewertet und abgefragt werden.
{% endhint %}

<figure><img src="../../.gitbook/assets/Folie28.png" alt=""><figcaption></figcaption></figure>

{% hint style="info" %}
Die dynamische Ersetzung der Platzhalter in den SO10-Texten erfolgt in der Methode GET\_DATASOURCE\_MAIL der BADI-Implementierung. In der ausgelieferten Klasse /C09/CFL\_CL\_BADI\_0101 findet man dazu auch ein Beispiel.
{% endhint %}

<figure><img src="../../.gitbook/assets/Folie29.png" alt=""><figcaption></figcaption></figure>

{% hint style="info" %}
Der Absprung aus dem Workitem in das gewünschte SAP-Objekt erfolgt in der Methode EXECUTE\_DEFAULT\_METHOD der BADI-Implementierung.
{% endhint %}

<figure><img src="../../.gitbook/assets/Folie30.png" alt=""><figcaption></figcaption></figure>

{% hint style="info" %}
Es gibt noch diverse Möglichkeiten, um die Entscheidungsaufgabe im SAP-GUI zu erweitern.
{% endhint %}

<figure><img src="../../.gitbook/assets/Folie31.png" alt=""><figcaption></figcaption></figure>

