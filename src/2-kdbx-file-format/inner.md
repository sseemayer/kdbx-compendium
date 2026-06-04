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

There are two boolean types in the XML database - `TBool` and `TNullableBoolEx`. `TBool` values are stored as the string `True` or `False`, while `TNullableBoolEx` values are stored as `True`, `False`, or the string `Null`.

Most boolean fields in the XML database are of type `TBool`, which means that an unspecified value will default to a field-specific default value.
<sup>
[KP](https://github.com/ralish/KeePass/blob/c238e8ee9bb62f4df4e102e67e5a731971a23852/KeePassLib/Serialization/KdbxFile.Read.Streamed.cs#L834-L842),
[KPXC](https://github.com/keepassxreboot/keepassxc/blob/608967954c5e1de08a218a195eb28dcd39ae07aa/src/format/KdbxXmlReader.cpp#L1050-L1065)
</sup>

For `TNullableBoolEx` fields, an unspecified or `Null` value will be treated as inheriting the parent group's value. The two fields of type `TNullableBoolEx` are `EnableAutoType` and `EnableSearching`.
<sup>
[KP](https://github.com/ralish/KeePass/blob/c238e8ee9bb62f4df4e102e67e5a731971a23852/KeePassLib/Serialization/KdbxFile.Read.Streamed.cs#L844-L853),
[KPXC](https://github.com/keepassxreboot/keepassxc/blob/608967954c5e1de08a218a195eb28dcd39ae07aa/src/format/KdbxXmlReader.cpp#L557-L584)
</sup>

#### Colors

Colors are stored as RGB values in the format "#RRGGBB". Empty strings should be treated as default values, chosen by the application.
<sup>
[KP](https://github.com/ralish/KeePass/blob/c238e8ee9bb62f4df4e102e67e5a731971a23852/KeePassLib/Serialization/KdbxFile.Read.Streamed.cs#L444-L449),
[KPXC](https://github.com/keepassxreboot/keepassxc/blob/608967954c5e1de08a218a195eb28dcd39ae07aa/src/format/KdbxXmlReader.cpp#L1088-L1116)
</sup>

**Example:** an <span style="background: #ff9900; width: 1rem; height: 1rem; display: inline-block;"></span> orangeish color could be expressed as `#FF6600`.

#### UUID values

UUID values are stored as 128-bit unsigned integers that are base64 encoded.
<sup>
[KP](https://github.com/ralish/KeePass/blob/c238e8ee9bb62f4df4e102e67e5a731971a23852/KeePassLib/Serialization/KdbxFile.Read.Streamed.cs#L855-L860),
[KPXC](https://github.com/keepassxreboot/keepassxc/blob/608967954c5e1de08a218a195eb28dcd39ae07aa/src/format/KdbxXmlReader.cpp#L1128-L1141)
</sup>

**Example:** the UUID `00010203-0405-0607-0809-0a0b0c0d0e0f` would be represented as the byte sequence `00 01 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f`, which is base64 encoded as `AAECAwQFBgcICQoLDA0ODw==`.

#### Timestamps

Timestamps should be stored as seconds since the special epoch of `2000-01-01T00:00:00Z` in UTC, as a 64-bit signed integer that is base64 encoded.
<sup>
[KP](https://github.com/ralish/KeePass/blob/c238e8ee9bb62f4df4e102e67e5a731971a23852/KeePassLib/Serialization/KdbxFile.Read.Streamed.cs#L918-L947),
[KPXC](https://github.com/keepassxreboot/keepassxc/blob/608967954c5e1de08a218a195eb28dcd39ae07aa/src/format/KdbxXmlReader.cpp#L1067)
</sup>

Some older versions of the KDBX format used ISO 8601 datetime strings, so parsing from this format should be supported for backward compatibility.

**Example:** the timestamp `2000-01-01T00:00:01Z`, one second after the epoch, would be stored as the byte sequence `01 00 00 00 00 00 00 00`, which is base64 encoded as `AQAAAAAAAA==`.

### DataTransferObfuscation aka AutoTypeObfuscation

DataTransferObfuscation is an integer-valued field for entries in the XML database. It corresponds to an enumeration of obfuscation methods used for AutoType:

| Value  | Method                             |
|--------|------------------------------------|
| 0      | No obfuscation                     |
| 1      | Use clipboard                      |

### Protected values

Values in the XML database can be marked as protected with an attribute `Protected="True"`. Protected values are stored as Base64-encoded encrypted data, using the [encryption method and key specified in the inner header](#inner-encryption). There is also a `ProtectInMemory="True"` attribute, but it is only used for plaintext XML exports.

While some implementations allow protections on any field
<sup>
[KPXC](https://github.com/keepassxreboot/keepassxc/blob/608967954c5e1de08a218a195eb28dcd39ae07aa/src/format/KdbxXmlReader.cpp#L1027-L1048)
</sup>, the official KeePass implementation only allows the string and binary fields of entries to be protected
<sup>
[KP](https://github.com/ralish/KeePass/blob/c238e8ee9bb62f4df4e102e67e5a731971a23852/KeePassLib/Serialization/KdbxFile.Read.Streamed.cs#L509-L523)
</sup>


### Attachment references

In KDBX4, attachments are stored as [binary entries in the inner header](#binary-attachments), and referenced in the XML database by their ID, which is the index of the binary attachment entry in the inner header. In KDBX3, attachments are stored as Base64-encoded data with an optional `Compressed="True"` and `Protected="True"` attributes, and are not referenced by ID
<sup>
[KP](https://github.com/ralish/KeePass/blob/c238e8ee9bb62f4df4e102e67e5a731971a23852/KeePassLib/Serialization/KdbxFile.Read.Streamed.cs#L980-L1028),
[KPXC](https://github.com/keepassxreboot/keepassxc/blob/608967954c5e1de08a218a195eb28dcd39ae07aa/src/format/KdbxXmlReader.cpp#L877-L924)
</sup>.

KDBX4 example:

```xml
<Binary>
    <Key>myfile.txt</Key>
    <Value Ref="1"/>
</Binary>
```

KDBX3 example:

```xml
<Binary>
    <Key>myfile.txt</Key>
    <Value Compressed="True">H4sIAAAAAAAAA/NIzcnJVyjJyCxWAKLk/NyCotTi4tQUhZTEkkQuAFCFZIEeAAAA</Value>
</Binary>
```

