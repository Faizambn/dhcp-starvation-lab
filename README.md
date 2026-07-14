# DHCP Starvation Lab

A hands-on home-lab exercise that **performs, proves the impact of, and detects** a DHCP starvation attack — documented in a professional report format.

Built entirely inside an **isolated VMware host-only network**. No real, corporate, or public systems were involved.

![Topology](Images/ss%2017.png)

---

## What this lab covers

- **Setup** — an isolated network segment with a Debian `isc-dhcp-server` (small 11-address pool).
- **Attack** — a custom Python/Scapy tool (`dhcp_starve.py`) that exhausts the pool using spoofed MAC addresses.
- **Impact** — a legitimate client is denied service ("no available address").
- **Detection** — the attack signature is isolated from captured traffic using `tshark`.
- **Remediation** — the real-world defences (DHCP snooping, port security) are documented.

## Environment

| Component | Detail |
|---|---|
| Network | VMware VMnet2 — host-only, isolated (VMware DHCP disabled) |
| Subnet | 192.168.174.0/24 |
| DHCP server | Debian 13, `isc-dhcp-server`, static 192.168.174.10 |
| Pool | 192.168.174.100 – .110 (11 addresses) |
| Attacker | Kali Linux + Python 3 / Scapy |

## Attack in one picture

![Attack flow](Images/ss%2016.png)

## Key result

The 11-address pool was exhausted in seconds using distinct spoofed MACs, after which a
legitimate client could no longer obtain an address. Traffic analysis identified **297 DHCP
Discover requests** and **11 unique spoofed (locally-administered) MACs** — the attack
signature — alongside a legitimate client repeatedly retrying, evidencing the denial of service.

## Contents

| Path | Description |
|---|---|
| `Lab-report.md` | Full report (setup → attack → impact → detection → remediation) |
| `report/` | Formal report as a downloadable Word document |
| `dhcp_starve.py` | The custom Scapy attack tool (commented) |
| `Images/` | Screenshots and diagrams |
| `Notes/` | Personal study notes from building the lab |

## Roadmap

This is the first lab in a planned series:

1. **`dhcp-starvation-lab`** *(this repo)* — attack, impact, and traffic-based detection.
2. **`dhcp-attack-detection`** — detection at scale with industry tooling (Suricata / Zeek / SIEM).
3. **`dhcp-snooping-prevention`** — prevention on a managed switch (DHCP snooping, port security) in GNS3/EVE-NG.

## Disclaimer

DHCP starvation is a denial-of-service technique. Everything here was performed in a personal,
isolated lab. Running this against any network you do not own and are not explicitly authorised
to test is illegal. Provided for educational purposes only.

---

*Author: Faiza Mobeen*
