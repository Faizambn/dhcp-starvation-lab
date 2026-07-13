# DHCP Lab — My Notes

These are my notes from building a DHCP starvation lab: I set up an
`isc-dhcp-server` on Debian inside an isolated VMware network, and honestly
spent way more time debugging why it wouldn't start than actually attacking it.
Writing this down so I remember what everything meant, and so future-me knows
what to check when a Linux service refuses to run.

---

## Contents

1. [DHCP — what I was actually building](#1-dhcp)
2. [IP addresses and subnets](#2-ip-addresses-and-subnets)
3. [VMware network types](#3-vmware-network-types)
4. [Linux services and systemd](#4-linux-services-and-systemd)
5. [Linux basics I picked up](#5-linux-basics)
6. [Stuff I'll need next](#6-stuff-ill-need-next)
7. [My troubleshooting checklist](#7-my-troubleshooting-checklist)
8. [Command cheat sheet](#8-command-cheat-sheet)

---

## 1. DHCP

DHCP (Dynamic Host Configuration Protocol) is basically the thing that hands out
IP addresses automatically when a device joins a network. Without it, someone
has to type an IP into every single machine by hand.

The way it works is a four-step conversation called **DORA**. I actually saw all
four of these in my server logs, which made it click:

| Step | Who talks | What they say |
|------|-----------|---------------|
| **D**iscover | Client | "Is there a DHCP server anywhere?" (it has no IP yet so it shouts to everyone) |
| **O**ffer | Server | "Sure, you can have 192.168.174.100" |
| **R**equest | Client | "Okay, I'll take that one" |
| **A**ck | Server | "Done, it's yours" |

A **lease** is just a rented address with a time limit. My config had:

```
default-lease-time 600;   # renew after 600 seconds
max-lease-time 7200;      # absolute max
```

Once I understood this, the attack made total sense: if I flood the server with
fake **Discover** messages using made-up MAC addresses, it gives away its whole
pool to fake clients, and real devices can't get an address. I built the victim
here — the attack part comes later.

I set my pool really small on purpose (only `.100` to `.110`, so 11 addresses)
so it runs out fast when I attack it. Easier to show in screenshots.

---

## 2. IP addresses and subnets

Took me a bit to properly get this. An address like `192.168.174.10` is really
two things squished together:

- which **network** you're on
- which **device** you are on that network

The **subnet mask** (`255.255.255.0`) is what decides where the split is.

`255.255.255.0` = "first three numbers are the network, last number is the
device." So `.1` through `.254` are all on the same network and can talk to each
other directly. The `/24` you see written after addresses means the same thing,
just shorter.

This is why everything in my lab had to start with `192.168.174`:

| Thing | Address |
|-------|---------|
| My server | 192.168.174.10 |
| The pool it hands out | 192.168.174.100 – .110 |
| Gateway in the config | 192.168.174.2 |

If they didn't share the `192.168.174` part they literally couldn't see each
other. This bit me later — the server had a leftover `169.254` address and dhcpd
refused to work because that's a totally different network it had no config for.

### About that 169.254 thing

`169.254.x.x` is what a machine gives *itself* when it wanted DHCP but nobody
answered. So if I see it, it basically means **"I didn't get a real address."**
Good thing to recognise instantly, it saved me time once I knew.

---

## 3. VMware network types

There are three adapter modes and they're really about *how cut off* the VM is
from the real world:

| Mode | What it means | When I used it |
|------|---------------|----------------|
| **Bridged** | VM acts like a real device on my actual home network | Never — attack traffic would hit real stuff, bad idea |
| **NAT** | VM borrows the host's internet, hidden behind it | Used this to download packages (`apt install`) |
| **Host-only (VMnet2)** | Private sealed network, no way out to the internet | This is the safe lab network |

I ended up switching back and forth — NAT when I needed to install things,
host-only for the actual lab. At first that felt annoying but it's actually the
correct way to do it: internet only when you need it, isolated the rest of the
time. Apparently that's how real labs are run so I'll keep doing it.

---

## 4. Linux services and systemd

A **service** (or daemon) is just a program that runs in the background all the
time — like my DHCP server. On modern Linux, **systemd** is what manages them,
and you talk to it with `systemctl`:

```bash
systemctl start   isc-dhcp-server
systemctl stop    isc-dhcp-server
systemctl restart isc-dhcp-server
systemctl status  isc-dhcp-server   # check if it's running + when it last tried
systemctl enable  isc-dhcp-server   # auto-start on boot
```

### The thing that confused me the most

When I ran the server by hand it worked perfectly:

```bash
dhcpd -4 -f -d ens33     # WORKS — I typed the interface myself
```

But starting it through systemd kept failing:

```bash
systemctl start isc-dhcp-server   # FAILS
```

Turns out systemd reads which interface to use from a **config file**
(`/etc/default/isc-dhcp-server`), and that file still said `eth0` from earlier —
but my actual interface was `ens33`. So by hand it used `ens33` (worked), and
systemd used `eth0` (didn't exist, failed). Fixed the file to say `ens33` and it
finally ran.

Lesson I'm keeping: **if a program works one way but fails another, check where
each one is getting its settings from.**

---

## 5. Linux basics

Quick notes on stuff I was typing without fully knowing why.

### root / sudo / su

- **root** = the admin account that can do anything. A lot of the config files I
  needed to edit can only be touched by root, which is why I kept getting
  permission errors.
- **`su -`** = become root (asks for root's password).
- **`sudo command`** = run just one command as root.

### Where things live

| Folder | What's in it |
|--------|--------------|
| `/etc` | config files (edited a bunch of these) |
| `/var` | changing stuff like logs and my `dhcpd.leases` file |
| `/sbin`, `/usr/sbin` | admin programs |

### PATH (why "command not found" happened)

The shell only looks in certain folders for commands. When `poweroff` said
"command not found," it wasn't missing — it lives in `/sbin` which wasn't in my
user's search path. Running `/sbin/poweroff` (full path) worked, and so did
doing it as root since root's path includes `/sbin`.

### `>` vs `>>` (I need to remember this one)

```bash
echo text >  file    # OVERWRITES the whole file
echo text >> file    # ADDS to the end
```

Getting these mixed up wipes files, so I'm being careful.

### The `ip` commands

```bash
ip a                                      # show interfaces + addresses
ip addr add 192.168.174.10/24 dev ens33   # add an address
ip addr del 192.168.174.10/24 dev ens33   # remove one
ip addr flush dev ens33                   # remove ALL addresses
ip link set ens33 up                      # turn the interface on
```

### Interface names caught me out

I assumed my interface was `eth0` like in old tutorials, but modern Debian uses
names like `ens33`. I only found the real name by running `ip a`. Note to self:
**never assume the interface name, always check first.** Hardcoding `eth0` broke
both my static IP and the service.

---

## 6. Stuff I'll need next

For the attack and defense parts coming up:

### MAC addresses

A MAC address (like `00:0c:29:8d:fa:aa`) is the hardware ID of the network card.
So there are two "addresses" for a device:

- **MAC** = hardware level (who you are on the local wire)
- **IP** = network level (who you are on the network)

The starvation attack works by faking loads of MAC addresses so the server
thinks tons of new devices showed up and gives each one a lease.

### Broadcast vs unicast

- **Broadcast** = message to everyone (DHCP Discover is this, because the client
  doesn't have an address or know who the server is yet)
- **Unicast** = message to one specific device

### Wireshark

This is my next tool — it shows the actual packets going across the network. I
should be able to literally see the DORA messages, and during the attack, see
the flood of Discover packets with fake MACs. That'll be my best screenshot for
the report.

---

## 7. My troubleshooting checklist

This is the part I most want to remember, because I fixed like five separate
things tonight and there was a pattern to all of them.

The big realisation: **the computer wasn't broken or being difficult — it was
doing exactly what it was told. The bug was always the gap between what it was
told and what I meant.** dhcpd wasn't ignoring me, it was faithfully looking at
`eth0` because a file said `eth0`.

My steps, in order:

1. **Actually read the error, don't skim it.** "No subnet declaration for eth0"
   literally told me the wrong interface. The answer was right there.

2. **Run the service in the foreground to see the real error.**
   `systemctl status` gives a short summary, but running it directly shows the
   full complaint:
   ```bash
   dhcpd -4 -f -d ens33      # -f foreground, -d debug
   ```
   This is what actually unstuck me a couple of times.

3. **Read the full log if I can't run it in foreground.**
   ```bash
   journalctl -xeu isc-dhcp-server.service    # scroll to the BOTTOM
   ```

4. **Check my assumptions with quick read commands instead of trusting them:**
   ```bash
   ip a                 # what addresses REALLY exist
   cat  file            # what's REALLY in the config (mine was all comments!)
   ls -l file           # does it even exist / is it empty (size 0)
   systemctl status x   # is it really running, and WHEN did it last try
   ```
   The timestamp thing caught me — I was looking at an old failure from minutes
   ago and thinking it was a new one.

5. **Fix one thing, then test again.** If the error *changes*, I fixed something
   real. If it's the *same* error, I didn't fix the actual problem.

### The actual bugs I hit (so I recognise them next time)

| What I saw | What I assumed | What it really was | How I found it |
|------------|----------------|--------------------|----------------|
| "Bad archive mirror" | picked a bad mirror | no internet at all (host-only network) | realised the VM had no way out |
| "device eth0 doesn't exist" | it's called eth0 | it was `ens33` | `ip a` |
| static IP wouldn't stick | I set it correctly | something kept re-adding a `169.254` address | it regenerated every time I deleted it |
| "No subnet declaration" | I wrote the config | the file was all `#` comments — my edits never saved because I couldn't type `{ }` | `cat` the file |
| systemd failed but manual worked | same program, should behave same | they read the interface from different places | compared the two commands |

That last config one had a dumb root cause: my keyboard layout wouldn't type
curly braces, so the config was never valid. Fixed it with `loadkeys us`.

---

## 8. Command cheat sheet

```bash
# ---- systemd services ----
systemctl start|stop|restart|status|enable <service>
journalctl -xeu <service>              # full log, newest at bottom

# ---- networking ----
ip a                                   # interfaces + addresses
ip addr add <ip>/<mask> dev <iface>    # add address
ip addr del <ip>/<mask> dev <iface>    # remove address
ip addr flush dev <iface>              # remove all addresses
ip link set <iface> up|down            # interface on/off
ip link show                           # list interface names

# ---- files ----
cat <file>                             # print file
tail -n 20 <file>                      # last 20 lines
nano <file>                            # edit (Ctrl+O save, Ctrl+X exit)
ls -l <file>                           # exists? size? permissions?
echo text >  <file>                    # OVERWRITE
echo text >> <file>                    # APPEND

# ---- users / power ----
su -                                   # become root
sudo <command>                         # run one command as root
/sbin/poweroff                         # shut down (full path if PATH complains)
reboot

# ---- if { } or symbols won't type ----
loadkeys us                            # set US keyboard layout

# ---- my DHCP server files ----
# config:          /etc/dhcp/dhcpd.conf
# which interface: /etc/default/isc-dhcp-server  ->  INTERFACESv4="ens33"
# active leases:   /var/lib/dhcp/dhcpd.leases
dhcpd -4 -f -d <iface>                 # run in foreground to see real errors
```

---

*My lab setup: isolated VMware host-only network (VMnet2), Debian running
isc-dhcp-server, static IP 192.168.174.10, small pool .100–.110 so it drains
fast when attacked.*
