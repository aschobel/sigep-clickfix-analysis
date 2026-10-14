# macOS payload analysis

The final URL in the clipboard command returned a 129,234-byte obfuscated JXA program with content disposition `WOF4lkeeirPc.js`.

```text
SHA-256  032d571c30c889a9cd2d459f5378eb50b87da0e077898ec93b8e4dcea7ca3c19
SHA-1    c37e8ef6c14a81208792cb0747bd5872fd3d02c5
MD5      28498430ff6225e460b3508caed547e1
```

It was inspected as text. No portion was passed to `osascript`, `eval`, `Function`, a JavaScript runtime, or a native loader.

## Static deobfuscation

The first pass contained 303 self-contained constant-string decoders. A restricted parser folded only literal arrays, arithmetic, XOR operations, `String.fromCharCode`, `map`, `reduce`, and string-building loops. This recovered imports and API names without evaluating the sample.

The program imports Cocoa, IOKit, and CoreFoundation, then binds native functions including:

- `sysctlbyname`
- `CFLocaleCopyCurrent`, `CFLocaleGetIdentifier`, and `CFLocaleGetValue`
- IOKit registry functions
- `ptrace`
- `csr_get_active_config`
- `malloc` and `free`

The remaining body was encrypted with an RC4-like routine and authenticated with a 16-byte value derived from the plaintext. The decryption key came from eight environment checks. Enumerating their finite branch outcomes produced exactly one key that validated all three encrypted blobs, allowing the embedded text to be recovered as data.

## Target qualification

The validated path describes a physical, non-debugged Mac with normal hardware signals and System Integrity Protection enabled, outside the excluded locale set.

The loader checks:

- `kern.hv_vmm_present` for a hypervisor;
- `hw.model`, `machdep.cpu.brand_string`, `IOPlatformExpertDevice`, and `AppleARMPE` for strings such as `virtual`, `vbox`, `vmware`, `parallels`, `qemu`, `kvm`, `utm`, `corellium`, and others;
- the size of the `AppleSMC` registry property set;
- `ptrace(PT_DENY_ATTACH)`, debugger-related environment variables, and timing delays;
- the active System Integrity Protection configuration;
- keyboard layouts and input modes associated with Russian and nearby languages;
- locale and country codes including `RU`, `BY`, `KZ`, `AM`, `AZ`, `KG`, `MD`, `TJ`, `UZ`, `TM`, `GE`, and `UA`; and
- time zones covering Moscow, Minsk, Kyiv, Tbilisi, Almaty, and other selected cities.

If these checks do not produce the expected combination, the integrity test for the hidden stage fails. This both avoids selected victims and frustrates automated analysis environments.

## Recovered loader

The decrypted JXA stage has SHA-256:

```text
97a6ed9d6d101dc6dd5a46a2afaff58bb3c1cbb577092337abfffd6d9a8c0f51
```

Its configuration contains a defanged Mach-O URL and installation directory:

```text
hxxps://policysync[.]pro/1kzpao/0G7Xcm07FD8I
~/Library/Caches/com.apple.finder
```

The code then performs the following actions:

1. Expands the tilde and creates `~/Library/Caches/com.apple.finder/Finder.app/Contents/MacOS`.
2. Exits immediately if `Contents/MacOS/3F9730CF` already exists.
3. Downloads the Mach-O with an ephemeral `NSURLSession`, retrying up to three times.
4. Writes it as `Contents/MacOS/3F9730CF` with POSIX mode `0755`.
5. Copies the legitimate Finder icon from `/System/Library/CoreServices/Finder.app/Contents/Resources/Finder.icns`.
6. Creates `Info.plist` with bundle name and display name `Finder`, bundle identifier `apple.finder.runtime`, `LSUIElement=true`, and `LSBackgroundOnly=true`.
7. Launches the impostor bundle with `NSWorkspace.openURL()` and exits.

This JXA layer is a delivery and concealment stage. It does not itself contain the credential-theft implementation, and it does not install a LaunchAgent. The downloaded Mach-O can implement additional behavior.

## Final Mach-O

The origin returned HTTP 404 when queried during analysis. Public threat intelligence nevertheless contains the exact filename:

```text
Filename  0G7Xcm07FD8I
Size      1,927,680 bytes
SHA-256   ddb7afd3e2d53100759643bb3b94b5cf011840d518d5553f33ae5c780af784e6
SHA-1     1e100d61e0e920a7812b9f626661291f035c3a3e
MD5       9a4c2d13b7fa04aaa55c701d15a0c357
Type      Universal Mach-O
```

[MalwareBazaar's record](https://bazaar.abuse.ch/sample/ddb7afd3e2d53100759643bb3b94b5cf011840d518d5553f33ae5c780af784e6/) reports the file as first seen on October 12, 2026. It records a malicious `Trojan[PSW]/MacOS.Agent` verdict from Hybrid Analysis and generic suspicious/malicious results from other services. The record also contains YARA matches related to script injection, loader behavior, and XCSSET-like strings. Those labels support a credential-stealing assessment but do not establish a precise malware family.

The final binary was not available from the live origin and was not executed or loaded during this investigation.
