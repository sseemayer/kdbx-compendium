# Introduction

Welcome to the KDBX compendium! This book is intended as an overview of the KeePass KDBX file format for developers implementing support for KeePass databases in their applications. 



## Notable KDBX format implementations

This book aims to compare different implementations of the KDBX file format, with direct references to the source code of each implementation. This is toidentify gotchas that developers might encounter.

**Example:** For Argon2, some implementations<sup>[KP](https://github.com/ralish/KeePass/blob/c238e8ee9bb62f4df4e102e67e5a731971a23852/KeePassLib/Cryptography/KeyDerivation/Argon2Kdf.cs#L72), [KP2A](https://github.com/PhilippC/keepass2android/blob/df20f34a1e5983c2d2fa515bb7c36e8904e2dc81/src/KeePassLib2Android/Cryptography/KeyDerivation/Argon2Kdf.cs#L70), [STBX](https://github.com/strongbox-password-safe/Strongbox/blob/26ecccf4633921a20dd354a1548a740fbcb07ab7/model/keepass/Argon2KdfCipher.m#L31)</sup> default to 2 lanes of parallelism, while some default to 4<sup>[KPXC](https://github.com/keepassxreboot/keepassxc/blob/b9bcbfc592e4b23d44e084ae2c2f3fdf8c9ed5cd/src/crypto/kdf/Argon2Kdf.h#L26), [KPDX](https://github.com/Kunzisoft/KeePassDX/blob/8bb87558c9b4ee76f5f9b8348b965a3232b2c0b0/database/src/main/java/com/kunzisoft/keepass/database/crypto/kdf/Argon2Kdf.kt#L204), [KPIU](https://github.com/keepassium/KeePassium/blob/e651df4f89b0b8550371566ca9aa55a964d2904e/KeePassiumLib/KeePassiumLib/db/kdf/Argon2KDF.swift#L33), </sup>. Libraries generally don't have a default, and require the caller to specify the number of lanes.

### GUI applications

| Application                                                         | Shortcode | Platforms               | Language           | Notes                                                                                           |
|---------------------------------------------------------------------|-----------|-------------------------|--------------------|-------------------------------------------------------------------------------------------------|
| [KeePass](https://keepass.info)                                     | KP        | Windows, (Linux, MacOS) | C#                 | No public git repo, [will use unofficial mirror](https://github.com/ralish/KeePass/tree/mirror) |
| [KeePassXC](https://github.com/keepassxreboot/keepassxc)            | KPXC      | Windows, Linux, MacOS   | C++                |                                                                                                 |
| [Keepass2Android](https://github.com/PhilippC/keepass2android)      | KP2A      | Android                 | Java               |                                                                                                 |
| [KeePassDX](https://github.com/kunzisoft/keepassdx)                 | KPDX      | Android                 | Kotlin             |                                                                                                 |
| [KeePassium](https://github.com/keepassium/keepassium)              | KPIU      | iOS                     | Swift              |                                                                                                 |
| [Strongbox](https://github.com/strongbox-password-safe/strongbox)   | STBX      | iOS, macOS              | Objective-C, Swift |                                                                                                 |


### Libraries

| Library                                                | Shortcode | Language   | Read support               | Write support         | Notes                                    |
|--------------------------------------------------------|-----------|------------|----------------------------|-----------------------|------------------------------------------|
| [pykeepass](https://github.com/libkeepass/pykeepass)    | PYKP      | Python     | KDBX3, KDBX4               | ?                     |                                          |
| [keepass-rs](https://github.com/sseemayer/keepass-rs)  | KPRS      | Rust       | KDB, KDBX3, KDBX4, KDBX4.1 | experimental, KDBX4.1 | maintained by the author of this book    |
| [kdbxweb](https://github.com/keeweb/kdbxweb)           | KDBW      | TypeScript | KDBX3, KDBX4, KDBX4.1      | KDBX3, KDBX4          | seems unmaintained, last commit Dec 2024 |


## Other resources

 * [KeePass KDBX file format specification](https://keepass.info/help/kb/kdbx.html)
 * [KeePass KDBX XML Schema (partial)](https://keepass.info/help/download/KDBX_XML.xsd)
 * [Almost Secure: "Documenting KeePass KDBX4 file format"](https://palant.info/2023/03/29/documenting-keepass-kdbx4-file-format/)

## Conventions in this Document

 * All byte offsets are zero-based, so the first byte of a structure is at offset `0`.
 * All byte ranges are non-inclusive, so the byte range `0-4` includes bytes at offsets `0`, `1`, `2`, and `3`, but not the byte at offset `4`.
 * All lengths are in bytes, so a length of `4` means that the value occupies 4 bytes.
 * All hexadecimal numbers are prefixed with `0x`, so the number `255` will be represented as `0xFF`.
