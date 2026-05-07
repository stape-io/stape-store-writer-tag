# Stape Store Writer tag for Google Tag Manager Server Side

The **Stape Store Writer tag** saves data to the Stape Store, a persistent key-value database for sGTM.
The stored data can be queried and retrieved later.

## Core Features

- Create new documents with auto-generated IDs or update existing ones.
- Includes all event data in the stored document.
- Allows adding or overriding data with custom fields.
- Supports both overwriting entire documents and merging data.

## Configuration

- **Document ID:** Specify a document to update it. Leave it empty to create a new document.
- **Add Event Data:** Check this to include all event data in your document.
- **Custom Data:** Add custom fields. These will override any fields with the same name from the event data.
- **Merge document keys:** When checked, the tag merges your data with the existing document. If unchecked, it overwrites the entire document.
- **Stape Store Collection Name:** The name of the Stape Store collection to write to. Defaults to `default`.

## Useful Resources

- [How to use the Stape Store Writer tag](https://stape.io/helpdesk/documentation/stape-store-feature#how-to-use-the-stape-store-writer-tag)

## Open Source

The **Stape Store Writer Tag for GTM Server Side** is developed and maintained by [Stape Team](https://stape.io/) under the Apache 2.0 license.

### GTM Gallery Status
🟢 [Listed](https://tagmanager.google.com/gallery/#/owners/stape-io/templates/stape-store-writer-tag)
