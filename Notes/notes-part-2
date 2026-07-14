# DHCP Lab — My Notes (Part 2: The Attack)

Picking up from Part 1 (where I built the DHCP server and got it running). This
part is about actually *attacking* it — writing a DHCP starvation script in
Scapy, and everything I learned while doing it. Same as before: this is me
writing down what I actually did and what tripped me up, so future-me remembers.

---

## Contents

1. [What we were attacking and why](#1-what-we-were-attacking)
2. [Scapy — what it is and why I used it](#2-scapy)
3. [How I built the attack packet (layers)](#3-building-the-packet)
4. [Discover-only vs Discover+Request](#4-discover-vs-request)
5. [The big bug: srp not matching (and AsyncSniffer fix)](#5-the-srp-bug)
6. [DHCP behaviours that confused me](#6-dhcp-behaviours)
7. [Proving impact](#7-proving-impact)
8. [Command / concept cheat sheet](#8-cheat-sheet)

---

## 1. What we were attacking

The DHCP server I built in Part 1 hands out IP addresses from a small pool
(only `192.168.174.100` to `.110`, so 11 addresses). A **DHCP starvation
attack** works by flooding that server with fake "give me an address" requests,
each pretending to be a different device, until the pool is empty. Then real
devices can't get an address — that's the denial of service.

The trick that makes it work: each fake request uses a **different fake MAC
address**, so the server thinks each one is a brand-new device and gives each a
lease. Spoof enough MACs and the pool drains.

I set the pool deliberately tiny (11 addresses) so it drains in seconds and is
easy to show in screenshots.

---

## 2. Scapy

**Scapy** is a Python library for building and sending network packets by hand.
Normally the operating system builds packets for you and hides the details.
Scapy lets me build a packet piece by piece and put whatever I want in each
field — including fake values, which is the whole point of an attack.

### Why I used Scapy instead of a ready-made tool

There are prebuilt tools for this — **yersinia** (a big Layer 2 attack tool with
a menu, does lots of attacks) and **dhcpstarv** (small, does only DHCP
starvation). Both are point-and-shoot. I didn't use them because:

- They weren't installed on my Kali, and installing meant switching to NAT
  (which would've broken my running Wireshark capture and my lab network).
- Scapy was already installed.
- **Most importantly:** building it myself in Scapy means I actually understand
  the attack instead of just running a black box. I can explain every field.

The honest trade-off: prebuilt tools are faster in a real job; writing it
myself is slower but I learn way more. For a learning/portfolio lab, Scapy wins.
Good line for an interview: *"I built it in Scapy to understand the mechanics,
and I know yersinia/dhcpstarv exist as faster prebuilt options."*

---

## 3. Building the packet

A network packet is built in **layers**, wrapped one inside another like
envelopes. In Scapy you stack them with the `/` symbol. For a DHCP Discover, the
layers are:

| Layer | What it's for | What I put in it |
|-------|---------------|------------------|
| **Ether** | hardware / MAC level | fake random MAC as source, broadcast (`ff:ff:...`) as dest |
| **IP** | network level | `0.0.0.0` source (no IP yet), `255.255.255.255` dest (broadcast) |
| **UDP** | delivery, uses ports | source port 68 (client), dest port 67 (server) |
| **BOOTP** | base of the DHCP message | the fake MAC again (as bytes), random transaction ID, broadcast flag |
| **DHCP** | the actual message | marked as "discover" |

The `0.0.0.0 → 255.255.255.255` broadcast pattern is exactly what I saw in
Wireshark for a *real* client too — because a client asking for its first
address has no IP and doesn't know where the server is, so it shouts to
everyone.

### Two little helper functions I wrote

- **`random_mac()`** — makes a random fake MAC as text. Starts with `02:`
  on purpose, because that marks a MAC as "locally administered" (made-up, not
  real hardware). This is the honest way to fake one.
- **`mac_to_bytes()`** — converts that text MAC into raw bytes, because one
  field deep in the packet (`chaddr`) wants bytes, not text.

I kept the same fake MAC in *both* the Ether layer and the BOOTP `chaddr` field,
because a real client's MAC matches in both places — so mine has to as well or
it looks wrong.

### Reading a packet with `.show()`

Before sending anything, I used `packet.show()` to print the whole packet, layer
by layer, and check every field was right. Good habit — inspect before you fire.
It confirmed my Ether `src` and the BOOTP `chaddr` were the same fake MAC, the
IPs were `0.0.0.0`/broadcast, ports were 68→67, and it was marked as a discover.

---

## 4. Discover vs Request

DHCP is a 4-step conversation (DORA): **D**iscover, **O**ffer, **R**equest,
**A**ck.

- A **Discover-only** attack just floods Discovers. The server *offers* an
  address for each, and reserves it briefly — that alone can drain the pool.
  Simple, fire-and-forget.
- A **Discover + Request** attack goes further: for each fake client it waits
  for the server's Offer, then sends a Request to actually *claim* the address.
  This completes the lease and holds it more firmly. More realistic, but harder
  to code because now the script has to *listen* for the Offer, not just send.

I did the full **Discover + Request** version. The extra difficulty is that to
send a Request, you have to know *which address the server offered* — and you
only learn that by catching the Offer and reading it.

---

## 5. The srp bug

This was the main thing that broke, and the fix taught me something real.

### What went wrong

My first Discover+Request version used Scapy's `srp` ("send and receive
packet") — it's supposed to send a packet and hand back the reply. But it kept
returning **0 answers**. When I tested directly, Scapy said:

```
Received 9 packets, got 0 answers
```

So the replies *were* arriving (9 of them), but `srp` matched none of them to my
Discover. Turns out `srp` matches a reply to a request by expecting the reply to
"look like" a direct response — but DHCP Offers come back as **broadcasts** from
the server's MAC, not addressed neatly back to my fake client, so `srp`'s
auto-matching fails. Known DHCP + Scapy gotcha.

### The fix: listen myself, match by transaction ID

Instead of trusting `srp`, I sniffed for the Offer myself and picked it out by
its **transaction ID (xid)** — the number that ties a DORA conversation
together. If a packet's xid matched my Discover's xid and it was a *reply*, it's
mine.

### The second bug: timing (race condition)

My first sniff-based attempt sent the Discover *then* started sniffing — but the
Offer can arrive faster than the code gets to the sniff line, so I'd miss it.
The fix was **`AsyncSniffer`**: start a sniffer running in the *background*
first, *then* send. That guarantees I'm already listening before the packet goes
out. A tiny `time.sleep(0.2)` before sending makes sure the sniffer is fully up.

Lesson I'm keeping: **when you need to catch a fast reply, start listening
before you send, not after.**

---

## 6. DHCP behaviours that confused me

Two things surprised me and are worth remembering (and good interview material):

### Existing clients keep their lease

After I drained the pool, my Kali machine could still renew its address. Why?
Because the server had already reserved that address for Kali's real MAC, and it
*honours existing reservations* even during starvation. So starvation denies
addresses to **new** clients, not necessarily to ones that already hold a lease.

To properly prove impact, I had to test with a **fresh MAC** the server had
never seen (using `macchanger`), so it had no reservation to fall back on.

### Leases expire and the pool refills

When I re-ran the attack a bit later, it claimed 0 addresses — because the fake
leases from the first run were still held. But later still, addresses started
coming free again, because leases **expire** after the lease time (I set 600
seconds). So a starvation attack has to be either *sustained* (keep flooding) or
demonstrated *promptly* after draining, before leases time out.

---

## 7. Proving impact

The point of the attack isn't "I ran a script" — it's showing real users get
hurt. I proved this by trying to connect a fresh client while the pool was
drained. NetworkManager failed with:

```
Connection activation failed: IP configuration could not be reserved
(no available address, timeout, etc.)
```

That "**no available address**" is the denial of service in plain words — a
legitimate client couldn't get online because the attack ate the whole pool.
Pairing that with a "before" shot (client getting an address normally) is the
whole story: normal = works, under attack = can't connect.

---

## 8. Cheat sheet

```python
# ---- Scapy basics ----
from scapy.all import *      # load Scapy
packet.show()                # print a packet layer-by-layer (inspect before sending)
sendp(pkt, iface="eth0")     # send a packet (with Ethernet layer)

# ---- building layers (stack with / ) ----
Ether(src=fake_mac, dst="ff:ff:ff:ff:ff:ff")     # hardware layer
IP(src="0.0.0.0", dst="255.255.255.255")         # network layer
UDP(sport=68, dport=67)                          # DHCP client->server ports
BOOTP(chaddr=mac_bytes, xid=random_id, flags=0x8000)
DHCP(options=[("message-type", "discover"), "end"])

# ---- catching a reply reliably (the fix) ----
sniffer = AsyncSniffer(iface="eth0", lfilter=<test>)  # set up background listener
sniffer.start()              # start listening FIRST
sendp(discover, ...)         # THEN send
results = sniffer.stop()     # stop and get caught packets

# ---- reading fields from a caught packet ----
pkt[BOOTP].xid               # transaction ID
pkt[BOOTP].yiaddr            # "your IP address" = the address being offered
pkt[IP].src                  # who sent it (the server)
pkt[BOOTP].op                # 1 = request, 2 = reply
```

```bash
# ---- resetting the lab between runs (on the SERVER) ----
systemctl stop isc-dhcp-server
cp /var/lib/dhcp/dhcpd.leases /var/lib/dhcp/dhcpd.leases.backup   # keep as evidence
> /var/lib/dhcp/dhcpd.leases                                     # empty the file
systemctl start isc-dhcp-server
cat /var/lib/dhcp/dhcpd.leases                                   # verify empty

# ---- making a client look brand-new (on KALI) ----
sudo macchanger -r eth0      # random new MAC (interface must be down first)
# note: NetworkManager may randomise the MAC again on connect

# ---- checking a lease / failure ----
ip a                         # look at eth0 inet line: .10x = got lease, 169.254 = failed
```

### The fake-vs-real MAC tell (nice for the report)

- Fake MACs I generated start with **`02:`** (locally administered).
- Real VMware machines start with **`00:0c:29:`** (VMware's vendor prefix).

So in the drained lease table you can literally see which leases are the attack
(`02:...`) and which are real hosts (`00:0c:29:...`). Good evidence detail.

---

*Part 2 of my DHCP lab notes. Next stage: defense (DHCP snooping / port
security) — with the honest caveat that real snooping needs a managed switch,
which my VMware host-only network can't do, so that stage needs a different
approach (detection, or GNS3/EVE-NG).*
