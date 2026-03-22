# TrueNAS SCALE — Fix Incus DHCP Leak on Local Network

> **TL;DR** — After a reboot or update, TrueNAS SCALE's Incus bridge (`incusbr0`) may leak its internal DHCP server onto your local network, breaking connectivity for all devices. This guide explains how to detect and permanently fix it.

---

## The Problem

TrueNAS SCALE uses **Incus** (a container/VM system) internally. Its network bridge `incusbr0` can end up listening on your physical network interface and responding to DHCP requests **instead of your router**.

When this happens, devices on your network receive:
- A broken subnet mask `/30` (`255.255.255.252`) instead of `/24` (`255.255.255.0`)
- A gateway pointing to TrueNAS instead of your router
- Unexpected IPv6 addresses even if IPv6 is disabled on your router
- No internet access

The issue is **random and intermittent** — it reappears after each TrueNAS reboot or system update because Incus resets its network config to defaults.

---

## Symptoms

- WiFi and wired devices not getting an IP address
- "Internet may not be available" warning on Android devices
- Devices getting IPv6 addresses even though IPv6 is disabled on the router
- Some devices get an IP but with mask `255.255.255.252` instead of `255.255.255.0` → no internet
- Problem disappears and comes back randomly, often after a TrueNAS reboot

---

## Root Cause

Incus configures `incusbr0` with an IP in the same subnet as the local network (e.g. `192.168.1.x/30`), causing its `dnsmasq` DHCP server to respond to broadcast DHCP requests from all devices on the LAN instead of being isolated to the internal bridge only.

It also sends Router Advertisements (`--enable-ra`) which explains the unwanted IPv6 addresses.

---

## Detection

Run this from any Linux machine on your network (e.g. a Raspberry Pi):

```bash
sudo nmap --script broadcast-dhcp-discover -e eth0 2>/dev/null \
  | grep -E "Server Identifier|Router|Subnet Mask|Domain Name"
```

**Normal result** (single server, /24 mask):
```
Server Identifier: 192.168.1.254
Subnet Mask: 255.255.255.0
Router: 192.168.1.254
```

**Abnormal result** (two servers or /30 mask):
```
Server Identifier: 192.168.1.254
Subnet Mask: 255.255.255.0
Router: 192.168.1.254
Server Identifier: 192.168.1.x    ← TrueNAS parasite
Subnet Mask: 255.255.255.252      ← broken mask
Domain Name: incus                ← Incus signature
```

You can also verify directly on TrueNAS:

```bash
# Should show incusbr0:67, NOT 0.0.0.0:67
sudo ss -ulnp | grep :67

# Find the rogue dnsmasq process
ps aux | grep dnsmasq
```

---

## Permanent Fix

### Step 1 — Create the fix script

Store the script on a persistent ZFS pool (survives updates):

```bash
sudo mkdir -p /mnt/YOUR_POOL/scripts
sudo nano /mnt/YOUR_POOL/scripts/fix-incus-network.sh
```

Paste this content (replace `10.10.10.1/24` with any private subnet that doesn't overlap your LAN):

```bash
#!/bin/bash
# Fix Incus DHCP leak — isolate incusbr0 to a private subnet
sleep 10
incus network set incusbr0 ipv4.address 10.10.10.1/24
incus network set incusbr0 ipv4.dhcp true
incus network set incusbr0 ipv6.address none
incus network set incusbr0 ipv6.nat false
incus network set incusbr0 dns.mode managed
```

Make it executable:

```bash
sudo chmod +x /mnt/YOUR_POOL/scripts/fix-incus-network.sh
```

---

### Step 2 — Register the script in TrueNAS

This uses the official TrueNAS init/shutdown mechanism — it survives updates:

```bash
sudo midclt call initshutdownscript.create '{
  "type": "SCRIPT",
  "script": "/mnt/YOUR_POOL/scripts/fix-incus-network.sh",
  "when": "POSTINIT",
  "enabled": true,
  "timeout": 10,
  "comment": "Fix Incus DHCP leak"
}'
```

You can verify it was registered in **TrueNAS WebUI → System → Advanced Settings → Init/Shutdown Scripts**.

---

### Step 3 — Apply immediately (no reboot needed)

```bash
sudo /mnt/YOUR_POOL/scripts/fix-incus-network.sh
```

---

### Step 4 — Verify the fix

```bash
sudo ss -ulnp | grep :67
```

✅ **Correct** — DHCP isolated on the internal bridge:
```
UNCONN 0  0  0.0.0.0%incusbr0:67  0.0.0.0:*  users:(("dnsmasq",...))
```

❌ **Still broken** — DHCP exposed on all interfaces:
```
UNCONN 0  0  0.0.0.0:67  0.0.0.0:*  users:(("dnsmasq",...))
```

Run the detection nmap command again from another machine to confirm only your router responds to DHCP.

---

## After TrueNAS Updates

After each TrueNAS update, check:

```bash
sudo ss -ulnp | grep :67
```

If the problem is back, the registered init script will fix it automatically on the next reboot. To fix immediately without rebooting:

```bash
sudo /mnt/YOUR_POOL/scripts/fix-incus-network.sh
```

---

## Environment

| Component | Version |
|-----------|---------|
| TrueNAS SCALE | 25.04+ |
| Incus | bundled with TrueNAS |
| Affected since | TrueNAS SCALE Electric Eel+ |

---

## Why not just disable Incus?

TrueNAS Apps (the application marketplace) relies on Incus internally — disabling it would break the Apps system entirely. The fix above isolates Incus to a private subnet while keeping everything functional.

