# KDBX file format

The KDBX format can be conceptually divided into two layers: the [outer database](./outer.md) with its header, and the compressed and encrypted HMAC block stream containing the [inner database](./inner.md) with its header.

{{#drawio path="diagrams/kdbx-structure.drawio" page=0}}

## Encoding

 * All numbers are stored in little-endian format, so the 32-bit hex number `0x01234567` will be stored as the byte sequence `67 45 23 01`
 * All boolean values are stored as a single byte, with `0x00` representing `false` and `0x01` representing `true`.
 * All UUIDs are stored as 16 bytes in big-endian format, so the UUID `123e4567-e89b-12d3-a456-426614174000` will be stored as the byte sequence `12 3E 45 67 E8 9B 12 D3 A4 56 42 66 14 17 40 00`.
 * All strings are UTF-8 encoded, so the string `Hello 😃` will be stored as the byte sequence `48 65 6C 6C 6F 20 F0 9F 98 83`.
