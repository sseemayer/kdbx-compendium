# Outer database

The outer database consists of an identifier followed by the database version, then the outer header, and finally hashes and HMACs for integrity and password verification. After the outer database, the HMAC block stream begins, which contains the compressed and encrypted inner database.

{{#drawio path="diagrams/kdbx-structure.drawio" page=1}}

## KDBX Identifier (4 bytes)

A KDBX file always starts with the bytes `03 D9 A2 9A` corresponding to the little-endian encoding of the 32-bit hex number `0x9aa2d903`. In the remainder of this chapter, numbers will be listed in their hex form, keep in mind that they are stored in little-endian format in the file.

## Version (8 bytes)

The next 8 bytes are the version number, with the following structure:

| Byte(s) | Description                                                                                             |
|---------|---------------------------------------------------------------------------------------------------------|
| 0-4     | Main version<br/>**KDB:** `0xb54bfb65`<br/>**KDB2:** `0xb54bfb66`<br/>**KDBX3 / KDBX4:** `0xb54bfb67`   |
| 4-6     | Minor version<br/>**KDBX4:** `0x0000`<br/>**KDBX4.1:** `0x001`                                          |
| 6-8     | Major version<br/>**KDBX3:** `0x0003`<br/>**KDBX4:** `0x004`                                            |

## Outer header

The outer header consists of an arbitrary number of entries, each with the following structure:

| Byte(s) | Description                                                                                             |
|---------|---------------------------------------------------------------------------------------------------------|
| 0       | Entry type ID                                                                                           | 
| 1-4     | Entry length                                                                                            |
| 4-...   | Entry value                                                                                             |

The header must be terminated by an entry with type `0x00` and length `0x00000000`.

{{#drawio path="diagrams/kdbx-structure.drawio" page=2}}

Valid entry types are:

| Name                       | Type ID | Type                                    | Length    | Description                                                                                                     |
|----------------------------|---------|-----------------------------------------|-----------|-----------------------------------------------------------------------------------------------------------------|
| End of header              | 0       | -                                       | 0         | Marks the end of the header                                                                                     |
| Outer encryption           | 2       | UUID                                    | 16        | The UUID of the encryption algorithm used for the HMAC block stream                                             |
| Compression                | 3       | UInt32                                  | 4         | The ID of the compression algorithm used for the inner database                                                 |
| Master seed                | 4       | Bytes                                   | 32        | The master seed used for key derivation                                                                         |
| Encryption IV / nonce      | 7       | Bytes                                   | 16 or 12  | The initialization vector (IV) or nonce used for encryption.<br/>AES-256 uses 16 bytes, ChaCha20 uses 12 bytes. |
| KDF parameters             | 11      | [VariantDictionary](#variantdictionary) | Variable  | The parameters for the key derivation function.                                                                 |
| Public custom data         | 12      | [VariantDictionary](#variantdictionary) | Variable  | Unencrypted custom data that may be used by applications.                                                       |


### Outer encryption

The outer encryption algorithm is used to encrypt the HMAC block stream. The outer encryption algorithm is specified by its UUID, which is stored in the outer header. The following outer encryption algorithms exist:

| Name       | UUID                                   | Notes                                                                                         |
|------------|----------------------------------------|-----------------------------------------------------------------------------------------------|
| AES-128    | `61ab05a1-9464-41c3-8d74-3a563df8dd35` | AES-128 in CBC mode with PKCS7 padding. This algorithm is deprecated.                         |
| AES-256    | `31c1f2e6-bf71-4350-be58-05216afc5aff` | AES-256 in CBC mode with PKCS7 padding.                                                       |
| Twofish    | `ad68f29f-576f-4bb9-a36a-d47af965346c` | Twofish in CBC mode with PKCS7 padding. This algorithm is deprecated.                         |
| ChaCha20   | `d6038a2b-8b6f-4cb5-a524-339a31dbb59a` | ChaCha20                                                                                      |


### Compression 

The compression algorithm is used to compress the inner database before encryption. The compression algorithm is specified by its ID, which is stored in the outer header. The following compression algorithms exist:

| Name       | ID     | Notes                     |
|------------|--------|---------------------------|
| None       | 0      | No compression.           |
| GZip       | 1      | GZip compression.         |

### Master seed and encryption IV / nonce

Every time a KDBX file is saved, a new random master seed and encryption IV / nonce should be generated.

### KDF parameters

Key derivation functions are specified via a UUID and a set of parameters. The following key derivation functions are defined:

| Name       | UUID                                   |
|------------|----------------------------------------|
| AES-KDF    | `c9d9f39a-628a-4460-bf74-0d08c18a4fea` |
| Argon2d    | `ef636ddf-8c29-444b-91f7-a9a403e30a0c` |
| Argon2id   | `9e298b19-56db-4773-b23d-fc3ec6f0a1e6` |

The parameters for the key derivation function (KDF) are stored in the outer header as a [VariantDictionary](#variantdictionary). 

The following parameters are defined:

| Key            | Type       | KDFs          | Description                                                                                                                                            |
|----------------|------------|---------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| `$UUID`        | UUID       | All           | The UUID of the key derivation function.                                                                                                               |
| `S`            | Byte[32]   | AES-KDF       | The salt used for key derivation. This should be re-generated on every database save.                                                                  |
| `R`            | UInt32     | AES-KDF       | The number of rounds.                                                                                                                                  |
| `V`            | UInt32     | Argon2        | The version number. This should be set to `0x13` for version 1.3 (recommended) or `0x10` for version 1.0.                                              |
| `S`            | Byte[]     | Argon2        | The salt used for key derivation. Minimum 8 bytes, maximum 0x3FFFFFFF bytes, 32 bytes recommended. This should be re-generated on every database save. |
| `I`            | UInt32     | Argon2        | The number of iterations. Minimum 1, maximum 0xFFFFFFFF                                                                                                |
| `M`            | UInt32     | Argon2        | The amount of memory in bytes. Minimum 8192, maximum 0x7FFFFFFF                                                                                        |
| `P`            | UInt32     | Argon2        | The degree of parallelism. Minimum 1, maximum 0x00FFFFFF                                                                                               |

### Public custom data

Public, unencrypted custom data stored as a [VariantDictionary](#variantdictionary). The keys should be unique and prefixed with the name of the application, e.g. `MyApplication_MyItem`

## VariantDictionary

{{#drawio path="diagrams/kdbx-structure.drawio" page=3}}

A variant dictionary is a typed key-value store used to store the parameters for the key derivation function and the public custom data in the outer header. It consists of an arbitrary number of entries, each with the following structure:

| Byte(s)             | Type    | Description                                      | 
|---------------------|---------|--------------------------------------------------|
| `0`                 | UInt8   | Value type ID                                    |
| `1-5`               | UInt32  | Key length `K`                                   |
| `5-(5+K)`           | String  | Key                                              |
| `(5+K)-(5+K+4)`     | UInt32  | Value length `V`                                 |
| `(5+K+4)-(5+K+4+V)` | Bytes   | Value (type depends on value type ID)            |

Valid value type IDs are:

| Type ID | Length   | Description             |
|---------|----------|-------------------------|
| `0x04`  | 4        | UInt32                  |
| `0x05`  | 8        | UInt64                  |
| `0x08`  | 1        | Boolean                 |
| `0x0c`  | 4        | Int32                   |
| `0x0d`  | 8        | Int64                   |
| `0x18`  | Variable | String (UTF-8 encoded)  |
| `0x42`  | Variable | Byte array              |

VariantDictionaries must be terminated by a single null byte (`0x00`).

## Outer header verification
All data from the start of the KDBX file up to and including the end of the outer header is used to verify the integrity of the outer header by computing a SHA-256 hash of this data and comparing it to the hash stored after the outer header. A mismatch indicates that the file is corrupted.


## Key derivation and verification
The outer header contains the parameters for the key derivation function (KDF) used to derive the master key from the user-provided secrets. 

```
CompositeKey = SHA-256(concatenate-if-present(
    SHA256(masterPassword),
    keyFromKeyfile,
    keyFromKeyProvider,
    keyFromWindowsUserAccount
))

TransformedKey = KDF(CompositeKey)
MasterKey = SHA-256(concatenate(MasterSeed, TransformedKey))
```

The AES-KDF is simply a repeated encryption of the composite key with itself as the key. The number of rounds is determined by the `R` parameter in the KDF parameters. The result of the final round is hashed with SHA-256 to produce the transformed key.

The Argon2 KDFs are used with the parameters specified in the KDF parameters and a 32 byte output length to produce the transformed key.

The correctness of the master key is verified by computing the HMAC of the header data as follows:

```
HMACKey = SHA-512(concatenate(MasterSeed, TransfomedKey, 0x01))
HMACBlockKey = SHA-512(concatenate(0xFFFFFFFFFFFFFFFF, HMACKey))
CheckHMAC = HMAC(HMACBlockKey, headerData)
```

The data in `CheckHMAC` is compared to the data stored after the outer header hash. A mismatch indicates that the master key is incorrect.

## HMAC block stream

The HMAC block stream is a stream of blocks (typically 1 MiB in size<sup>[KP](https://github.com/ralish/KeePass/blob/c238e8ee9bb62f4df4e102e67e5a731971a23852/KeePassLib/Serialization/HmacBlockStream.cs#L37), [KPXC](https://github.com/keepassxreboot/keepassxc/blob/b9bcbfc592e4b23d44e084ae2c2f3fdf8c9ed5cd/src/streams/HmacBlockStream.cpp#L27)</sup>) where each block is authenticated with an HMAC.

{{#drawio path="diagrams/kdbx-structure.drawio" page=4}}

For the `i`-th block, the HMAC key is verified as follows:

```
HMACBlockKey = SHA-512(concatenate(i as UInt64, HMACKey))
CheckHMac = HMAC(HMACBlockKey, blockData)
```

## From HMAC block stream to inner database

The HMAC block stream is decrypted using the outer encryption algorithm specified in the outer header, resulting in the compressed inner database. The compressed inner database is then decompressed using the compression algorithm specified in the outer header, resulting in the [inner database](./inner.md).
