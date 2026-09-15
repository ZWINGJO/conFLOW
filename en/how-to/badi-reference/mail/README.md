# Mail

Who gets a notification, in which language, with which content

conFLOW sends mails from an SO10 standard text. It contains placeholders of the form &STRUCTURE-FIELD&. GET_DATASOURCE_MAIL fills them by passing entire STRUCTURES - not individual values.

This is the core and the cause of most misunderstandings: you supply data, not text. Which of it ends up in the mail is decided by the standard text - that is, by the customer, without a transport.

## The hooks

- [`get_send_mail_user`](get-send-mail-user.md) - Adding or removing recipients
- [`get_mail_language`](get-mail-language.md) - Language of the email
- [`get_status_mail_dynamic`](get-status-mail-dynamic.md) - Which group of agents is notified
- [`get_datasource_mail`](get-datasource-mail.md) - The data for the mail text and the work item text
- [`get_add_attachments`](get-add-attachments.md) - Attachments - archive, document files, SAP shortcut
