# DHCP Starvation Attack — Lab Report

**A red-and-blue home-lab exercise: attacking and detecting DHCP pool exhaustion in an isolated environment**

| | |
|---|---|
| **Author** | Faiza Mobeen |
| **Repository** | `dhcp-starvation-lab` (github.com/<your-handle>/dhcp-starvation-lab) |
| **Environment** | Isolated VMware host-only network (VMnet2) |
| **Date** | July 2026 |
| **Classification** | Educational / portfolio — no production systems involved |

---

## 1. Disclaimer & Scope

This document describes a security exercise performed entirely within a **personal, isolated laboratory environment**. All activity took place on a VMware **host-only** virtual network (VMnet2) with **no route to any physical, corporate, or public network**. No real systems, third-party systems, or production data were involved at any point.

DHCP starvation is a denial-of-service technique. Performing it against any network you do not own and have explicit authorisation to test is illegal. It is presented here strictly as a controlled learning exercise.

**In scope:** a single virtual DHCP server and a single attacker VM, both confined to VMnet2.
**Out of scope:** any system outside VMnet2; any real-world target.

---

## 2. Executive Summary

This exercise demonstrates, end to end, how a **DHCP starvation attack** works, the impact it has on legitimate users, and how it can be detected.

A DHCP server issues IP addresses from a limited pool. An attacker who floods the server with address requests — each disguised as a different device using a spoofed hardware (MAC) address — can consume the entire pool. Once the pool is empty, **legitimate devices can no longer obtain an IP address and are unable to join the network** — a denial of service.

In this lab a Debian DHCP server was configured with a small 11-address pool. A custom attack tool written in Python (Scapy) exhausted the entire pool in seconds using spoofed MAC addresses. A legitimate client was then observed being **denied service** ("no available address"). Finally, the attack was **detected** through network-traffic analysis, which clearly isolated the attack signature.

The exercise confirms that DHCP, by default, has no protection against this attack, and that the effective defences (DHCP snooping and port security) live on managed network switches rather than on the DHCP server itself.

---

## 3. Environment & Methodology

### 3.1 Environment

| Component | Detail |
|---|---|
| Virtualisation | VMware Workstation |
| Network | VMnet2, host-only, isolated; VMware's own DHCP **disabled** |
| Subnet | 192.168.174.0/24 |
| DHCP server | Debian 13, `isc-dhcp-server`, static IP 192.168.174.10, interface `ens33` |
| DHCP pool | 192.168.174.100 – 192.168.174.110 (11 addresses, deliberately small) |
| Attacker | Kali Linux, interface `eth0`, tooling: Python 3 + Scapy |

*Figure 1 — Isolation: VMnet2 configured host-only with VMware DHCP disabled.*
![Figure 1](images/ss%201.png)

*Figure 2 — Attacker VM network adapter bound to VMnet2.*
![Figure 2](images/ss%202.png)

*Figure 3 — Lab topology (both VMs isolated on VMnet2).*
![Figure 3](images/diagram_topology.svg)

### 3.2 Methodology

The exercise followed a standard engagement structure:

1. **Setup** — build an isolated network and a working DHCP server.
2. **Baseline (Recon)** — confirm normal DHCP operation.
3. **Attack** — exhaust the DHCP pool with spoofed requests.
4. **Impact** — demonstrate denial of service to a legitimate client.
5. **Detection** — identify the attack from network traffic.
6. **Remediation** — document the real-world defences.

---

## 4. Setup & Baseline

### 4.1 DHCP server build

A minimal Debian server was installed and configured with `isc-dhcp-server`. The server was given a static address (192.168.174.10) because a DHCP server cannot obtain its own address via DHCP. The service was configured to listen on `ens33` and to serve the small pool `.100`–`.110`.

*Figure 4 — DHCP server VM created and attached to VMnet2.*
![Figure 4](images/ss%205.png)

*Figure 5 — DHCP service active and running.*
![Figure 5](images/ss%206.png)

*Figure 6 — Service status with a clean, single-lease baseline.*
![Figure 6](images/ss%207.png)

### 4.2 Baseline — normal operation

Before attacking, normal DHCP operation was verified. The attacker VM (acting as an ordinary client) successfully obtained a lease of **192.168.174.101** from the server, completing the full DHCP four-step exchange (Discover → Offer → Request → Acknowledge, "DORA").

*Figure 7 — Legitimate client obtains a valid lease (192.168.174.101) under normal conditions.*
![Figure 7](images/ss%208.png)

*Figure 8 — Normal DORA exchange captured on the wire (Wireshark).*
![Figure 8](images/ss%209.png)

This baseline is important: it establishes what "healthy" looks like, against which the attack's impact is measured.

---

## 5. Findings

### Finding 1 — DHCP Pool Exhaustion via MAC Spoofing (Starvation)

| | |
|---|---|
| **Severity** | High |
| **Type** | Denial of Service (network availability) |
| **Affected component** | DHCP service (`isc-dhcp-server`) on the isolated segment |

#### Description

The DHCP protocol identifies clients by their hardware (MAC) address and issues each a lease from its address pool. The protocol performs **no verification** that a MAC address corresponds to a genuine, unique device. An attacker can therefore generate a large number of **spoofed MAC addresses** and request an address for each, causing the server to reserve and issue leases until the pool is fully consumed.

A custom tool (`dhcp_starve.py`, written in Python with Scapy) was developed to demonstrate this. For each iteration it:

1. Generates a random, locally-administered MAC address (prefix `02:`).
2. Crafts and sends a DHCP **Discover** from that MAC.
3. Listens for the server's **Offer**.
4. Sends a matching **Request** to claim the offered address.

The tool automatically stops when the server ceases to respond — i.e. when the pool is exhausted.

#### Evidence

Running the tool against the 11-address pool claimed every available address in seconds, each with a distinct spoofed MAC, then correctly detected exhaustion and halted:

*Figure 9 — Attack tool output: nine remaining pool addresses claimed with distinct spoofed MACs, then "pool exhausted".*
![Figure 9](images/ss%2011.png)

The attack traffic on the wire shows repeated DORA cycles, each from `0.0.0.0` (client with no address) broadcasting to `255.255.255.255`:

*Figure 10 — Attack flood: repeated DHCP cycles captured during the attack.*
![Figure 10](images/ss%2012.png)

The server's own lease table confirms the outcome. Every pool address is bound, and the attacker-held leases are clearly distinguishable by their **locally-administered MAC prefix (`02:`)**, whereas the two legitimate hosts retain the VMware vendor prefix (`00:0c:29:`):

*Figure 11 — Post-attack lease table: pool consumed by spoofed MACs (02:xx = attacker; 00:0c:29 = legitimate hosts).*
![Figure 11](images/ss%2013.png)

*Figure 12 — Attack flow overview.*
![Figure 12](images/diagram_attack_flow.svg)

#### Business Impact

With the pool exhausted, **no new device can join the network**. In a real environment this denies service to any client whose lease expires or any newly connecting device — laptops, phones, servers, or infrastructure — producing a network-wide outage that is simple to launch and requires no credentials.

---

### Finding 2 — Legitimate Client Denied Service (Demonstrated Impact)

| | |
|---|---|
| **Severity** | High (consequence of Finding 1) |
| **Type** | Denial of Service (confirmed impact) |

#### Description

To confirm real impact — not merely that a tool ran — a legitimate client attempted to obtain an address **while the pool was exhausted**. Because a DHCP server honours existing reservations for known MAC addresses, the test was performed with a **fresh MAC** (one the server had never seen), simulating a genuinely new device with no prior claim.

#### Evidence

The connection attempt failed with an explicit error stating no address could be reserved:

*Figure 13 — Legitimate client denied: NetworkManager reports "no available address".*
![Figure 13](images/ss%2014.png)

This is the denial of service in plain terms: a real device, correctly requesting an address, is unable to join the network because the attack has consumed the entire pool.

#### Note on lease dynamics

DHCP leases expire (here, after the configured lease time), after which unused addresses return to the pool. A starvation attack must therefore be **sustained** to hold the network down, or the effect is temporary. This is a meaningful operational nuance: the attack denies service to *new* clients while the pool is held, but existing clients with active reservations are unaffected until their leases lapse.

---

## 6. Detection

While prevention requires switch-level controls (Section 7), the attack is **detectable** from its network signature. Detection analysis was performed with `tshark` (the command-line form of Wireshark) against the captured traffic.

### 6.1 Signature

A DHCP starvation attack produces a characteristic pattern:

- An **abnormal volume** of DHCP Discover requests in a short time.
- Requests originating from **many distinct MAC addresses**, most of them **locally-administered** (`02:`/`06:` prefixes) — the hallmark of spoofing.
- Often, a **single legitimate client repeatedly retrying** because it cannot obtain a lease — a visible symptom of the impact.

### 6.2 Results

Analysis of the capture identified **297 DHCP Discover requests** and, among them, **11 unique spoofed (locally-administered) MAC addresses** — consistent with exhaustion of the 11-address pool. A single legitimate MAC was also observed issuing over 200 repeated requests, evidencing a client locked out by the attack.

*Figure 14 — Detection via `tshark`: 297 Discover requests; 11 unique spoofed MACs each issuing a single Discover (the attack), plus one legitimate MAC retrying ~200 times (the impact).*
![Figure 14](images/ss%2015.png)

The detection logic, expressed as reproducible commands:

```bash
# Count DHCP Discover requests
tshark -r capture.pcapng -Y "dhcp.option.dhcp == 1" | wc -l

# List source MACs by frequency (reveals the spoofing pattern)
tshark -r capture.pcapng -Y "dhcp.option.dhcp == 1" -T fields -e eth.src \
  | sort | uniq -c | sort -rn

# Count unique spoofed (locally-administered) MACs — the alert figure
tshark -r capture.pcapng -Y "dhcp.option.dhcp == 1" -T fields -e eth.src \
  | grep -iE "^(02|06|0a|0e):" | sort -u | wc -l
```

### 6.3 Detection at scale (production)

In a real environment this signature would be detected by dedicated tooling rather than manual analysis:

- **IDS (Suricata / Snort):** rules that alert on a high rate of DHCP Discovers or an anomalous count of unique MACs per interval.
- **Zeek:** anomaly analysis over `dhcp.log` metadata.
- **SIEM (Wazuh / Elastic / Splunk):** central correlation of DHCP-server logs and IDS alerts, alerting on lease-rate or unique-MAC thresholds.

> **Follow-up work:** a dedicated lab (`dhcp-attack-detection`) will implement this detection with industry tooling (Suricata/Zeek and a SIEM), moving from manual traffic analysis to real-time alerting.

---

## 7. Remediation

DHCP starvation cannot be prevented on the DHCP server alone; the effective controls operate at the **network switch** layer:

| Control | How it mitigates starvation |
|---|---|
| **DHCP Snooping** | The switch tracks DHCP activity per port and drops DHCP traffic that violates its binding table; combined with rate limiting, it throttles the Discover flood. |
| **Port Security** | Limits the number of MAC addresses permitted on a switch port; a single port suddenly presenting dozens of MACs is blocked or shut down — directly defeating MAC spoofing. |
| **DHCP rate limiting** | Caps DHCP messages per port per second, blunting the flood. |
| **Monitoring / alerting** | Detection controls (Section 6) so that an attack in progress is surfaced quickly even if not fully prevented. |

Because the lab uses a VMware **host-only virtual network**, which provides no configurable managed switch, DHCP snooping and port security **cannot be demonstrated in this environment**. They are documented here as the correct real-world remediation.

> **Follow-up work:** a dedicated lab (`dhcp-snooping-prevention`) will rebuild the topology in GNS3/EVE-NG with a virtual managed switch to demonstrate DHCP snooping and port security defeating the same attack — providing the before/after prevention proof.

---

## 8. Conclusion

This exercise demonstrated the full lifecycle of a DHCP starvation attack in a safe, isolated lab: a working DHCP server was built, its normal operation baselined, its address pool exhausted using a custom Scapy tool, the resulting denial of service to a legitimate client confirmed, and the attack detected through traffic analysis. The real-world remediations were documented, with hands-on prevention and industrial-scale detection scoped as dedicated follow-up labs.

The key takeaways:

- DHCP performs no client verification, so pool exhaustion via MAC spoofing is trivial and unauthenticated.
- The impact is a genuine network-availability outage for new/expiring clients.
- The attack has a clear, detectable network signature.
- Effective prevention lives on managed switches (DHCP snooping, port security), not on the DHCP server.

---

## Appendix A — Tooling

| Tool | Purpose |
|---|---|
| VMware Workstation | Virtualisation and isolated networking |
| Debian 13 + `isc-dhcp-server` | DHCP server (target) |
| Kali Linux | Attacker platform |
| Python 3 + Scapy | Custom attack tool (`dhcp_starve.py`) |
| Wireshark / `tshark` | Traffic capture and detection analysis |

## Appendix B — Key configuration

**`/etc/dhcp/dhcpd.conf` (pool definition):**

```
default-lease-time 600;
max-lease-time 7200;
authoritative;

subnet 192.168.174.0 netmask 255.255.255.0 {
  range 192.168.174.100 192.168.174.110;
  option routers 192.168.174.2;
  option domain-name-servers 8.8.8.8;
}
```

**Listening interface — `/etc/default/isc-dhcp-server`:**

```
INTERFACESv4="ens33"
```

## Appendix C — Attack tool

The custom tool `dhcp_starve.py` (Python + Scapy) is included in the repository. It crafts spoofed DHCP Discover/Request packets to exhaust the pool and halts automatically on exhaustion. See the repository for the fully commented source.

---

*Prepared as a portfolio lab exercise. All activity was confined to an isolated virtual network; no production or third-party systems were involved.*
