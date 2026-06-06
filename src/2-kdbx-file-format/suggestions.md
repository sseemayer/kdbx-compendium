# Suggestions for improvement

As implementors of the KDBX file format, we have identified several areas where the format could be improved in a future version. This chapter outlines some of these sugestions.

## Serialize tags as XML elements
Instead of the [heterogeneous way tags are processed from strings](./inner.md#tags), the list of tags could be stored using XML elements, with one XML element per tag.

## Identify attachments by UUIDs
Currently, attachments in the [inner header](./inner.md#binary-attachments) are [referenced in the XML database by their order of appearance](./inner.md#attachment-references). This is the only case where elements in the XML database are not identified by UUIDs, and having attachment UUIDs would simplify unambiguous identification of attachments, e.g. during merge operations.

