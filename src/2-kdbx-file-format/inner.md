# Inner database

The inner database is the result of decrypting and decompressing the HMAC block stream. It consists of the inner header and the XML database.

{{#drawio path="diagrams/kdbx-structure.drawio" page=5}}

## Inner Header

The inner header consists of an arbitrary number of entries, each with the following structure:

{{#drawio path="diagrams/kdbx-structure.drawio" page=6}}

| Byte(s) | Description                                                                                             |
|---------|---------------------------------------------------------------------------------------------------------|
| 0       | Entry type ID                                                                                           | 
| 1-4     | Entry length                                                                                            |
| 4-...   | Entry value                                                                                             |

The header must be terminated by an entry with type `0x00` and length `0x00000000`.

Valid entry types are:

| Name                       | Type ID | Type                                    | Length    | Description                                                                                        |
|----------------------------|---------|-----------------------------------------|-----------|----------------------------------------------------------------------------------------------------|
| End of header              | 0       | -                                       | 0         | Marks the end of the header                                                                        |
| Inner encryption           | 1       | UInt32                                  | 4         | The ID of the encryption algorithm used for the inner database                                     |
| Inner encryption key       | 2       | Bytes                                   | Variable  | The key used for encrypting the inner database. This should be re-generated on each database save. |
| Binary attachment          | 3       | Bytes                                   | Variable  | A binary file attachment                                                                           |

### Inner encryption

Inner encryption is used to encrypt protected fields within the XML database. The inner encryption key is specified as a field of the inner header, and independent of the outer encryption key.

Valid inner encryption algorithm IDs are:

| Name           | ID     | Description                                         |
|----------------|--------|-----------------------------------------------------|
| AES            | 1      | AES encryption in CBC mode with PKCS7 padding.      |
| Twofish        | 2      | Twofish encryption in CBC mode with PKCS7 padding.  |
| ChaCha20       | 3      | ChaCha20 encryption.                                |


### Binary attachments

Binary attachments are used to store binary files within the database. They are referenced in the XML database by their ID, which is the index of the binary attachment entry in the inner header.

Binary attachments have the following structure:

| Byte(s) | Description                                                                                                                                        |
|---------|----------------------------------------------------------------------------------------------------------------------------------------------------|
| 0       | Flags. `0x01` indicates the binary content should be protected in process memory, but does not indicate that the content is encrypted in the file. |
| 1-...   | Binary content                                                                                                                                     |

## XML Database

The XML database is a UTF-8 encoded XML document that contains most actual data of the database, including groups, entries, and their fields. The structure of the XML database is defined by the [KDBX XML Schema](https://keepass.info/help/download/KDBX_XML.xsd). This section fills in some details about the XML database that are important for implementors.


### Value types
Some value types have special encoding rules within the XML database, which are described here.

#### (Nullable) Booleans

There are two boolean types in the XML database - `TBool` and `TNullableBoolEx`. `TBool` values are stored as the string `True` or `False`, while `TNullableBoolEx` values are stored as `True`, `False`, `true`, `false`, or the string `Null` or `null`. Be sure to check the XML schema for the correct type of boolean to use.


#### Colors

Colors are stored as RGB values in the format "#RRGGBB". Empty strings should be treated as default values, chosen by the application.

**Example:** an orangeish color could be expressed as `#FF6600`.

#### UUID values

UUID values are stored as 128-bit unsigned integers that are base64 encoded.

**Example:** the UUID `00010203-0405-0607-0809-0a0b0c0d0e0f` would be stored as the byte sequence `00 01 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f`, which is base64 encoded as `AAECAwQFBgcICQoLDA0ODw==`.

#### Timestamps

Timestamps should be stored as seconds since the special epoch of `2000-01-01T00:00:00Z` in UTC, as a 64-bit signed integer that is base64 encoded.

Some older versions of the KDBX format used ISO 8601 datetime strings, so parsing from this format should be supported for backward compatibility.

**Example:** the timestamp `2000-01-01T00:00:01Z`, one second after the epoch, would be stored as the byte sequence `01 00 00 00 00 00 00 00`, which is base64 encoded as `AQAAAAAAAA==`.

### Protected values

### Attachment references

Attachments are stored as [binary entries in the inner header](#binary-attachments), and referenced in the XML database by their ID, which is the index of the binary attachment entry in the inner header.
