# Mail

Wer bekommt eine Benachrichtigung, in welcher Sprache, mit welchem Inhalt

conFLOW verschickt Mails aus einem SO10-Textbaustein. Darin stehen Platzhalter der Form &STRUKTUR-FELD&. GET_DATASOURCE_MAIL füllt sie, indem es ganze STRUKTUREN übergibt - nicht einzelne Werte.

Das ist der Kern und die Ursache der meisten Missverständnisse: man liefert Daten, nicht Text. Welche davon im Mail landen, entscheidet der Textbaustein - also der Kunde, ohne Transport.

## Die Hooks

- [`get_send_mail_user`](get-send-mail-user.md) - Empfänger ergänzen oder entfernen
- [`get_mail_language`](get-mail-language.md) - Sprache des Mails
- [`get_status_mail_dynamic`](get-status-mail-dynamic.md) - Welcher Bearbeiterkreis angeschrieben wird
- [`get_datasource_mail`](get-datasource-mail.md) - Die Daten für den Mailtext und den Workitem-Text
- [`get_add_attachments`](get-add-attachments.md) - Anhänge - Archiv, Belegdateien, SAP-Shortcut
