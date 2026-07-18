# GearUP Game Booster — Patched APK

> **Patched build:** `gearup_v3_patched.apk`  
> Signature check bypassed · Root bypass · All validation patches applied

---

## How GearUP Actually Boosts Your Internet in Games

Most people think a "game booster" just clears RAM or kills background apps. GearUP does something at the **network layer** — and that's why it actually works.

### The Real Problem: Your Route to the Game Server Is Trash

When you connect to a game server, your packets don't travel in a straight line. They hop through your ISP's routers, then through backbone providers, then to the server. The path looks something like:

```
You → ISP Router → Regional Hub → National Backbone → Peering Point → Game Server
```

Every hop adds latency. Some hops are **congested** (overloaded routers at peak hours). Some hops go in the **wrong geographical direction** before correcting. Your ISP doesn't care about your ping — they just send your packets down the cheapest route they have a contract for.

This is why you can have 100 Mbps fiber and still get 120ms ping on a game server that's physically 300 km away.

---

### What GearUP Does: Intelligent Route Optimization

GearUP runs a **dedicated relay network** — their own servers placed at key peering points globally. When you boost a game:

1. **Traffic is redirected** through a GearUP relay node that has a **faster/less congested path** to the game server.
2. GearUP's infrastructure uses **optimized BGP routing** — they have private peering agreements that your ISP doesn't, giving them shorter, faster routes.
3. The relay node is chosen **dynamically** — the app tests multiple routes and picks the one with the lowest latency to your specific game server at that moment.

Your packets now travel:

```
You → GearUP Relay (optimized path) → Game Server
```

The relay leg (You → GearUP) might add 5ms, but eliminating 3 bad ISP hops saves 40ms. Net result: **lower ping**.

---

### Why Ping Stays Low and Stable (Not Just Lower)

Ping spikes (jitter) are worse than consistently high ping. A spike from 30ms to 200ms for one frame causes rubber-banding and missed shots. GearUP reduces jitter through two mechanisms:

#### 1. UDP Tunnel with Packet Prioritization
Games run on **UDP** — connectionless, fire-and-forget packets. GearUP wraps your game traffic in its own UDP tunnel between your device and the relay. Inside that tunnel it applies **QoS (Quality of Service)** tagging — game packets get priority over anything else going through the relay.

#### 2. Packet Loss Compensation (FEC)
On lossy mobile/WiFi connections, GearUP applies **Forward Error Correction** — it sends redundant data alongside your game packets so the relay can reconstruct a dropped packet without waiting for a retransmit. A retransmit on UDP causes a spike. FEC eliminates that spike entirely.

---

### The "Strange" Boost — Why It Feels Like More Than Just Ping

Users report that games feel smoother even when their original ping wasn't terrible (e.g., 40ms → 25ms doesn't sound dramatic). Here's why it feels significant:

- **Jitter removal** matters more than raw ping. A stable 40ms is smoother than a 25ms average with ±30ms variance.
- **Download/upload asymmetry correction** — mobile ISPs often throttle upload bursts. Game packets (your inputs) are upload. GearUP's tunnel prevents your ISP from throttling those bursts.
- **ISP deep packet inspection bypass** — some ISPs throttle UDP traffic they identify as gaming/P2P. Since your game traffic is tunneled inside GearUP's protocol, the ISP sees encrypted traffic to a known data center and doesn't throttle it.
- **Last-mile compression** on some game titles — GearUP has game-specific profiles that know which packet types are high-priority vs. redundant.

---

## Why Psiphon Does NOT Do the Same Thing

Psiphon is a **censorship circumvention tool**, not a game booster. The architecture is fundamentally different.

### Psiphon's Architecture

```
You → Psiphon Server (TCP proxy / VPN / SSH tunnel) → Internet
```

Psiphon is built around **TCP** — specifically to carry web traffic (HTTP/HTTPS) reliably through censored networks. It works using:
- HTTP proxy
- SOCKS5 proxy
- SSH tunneling
- L2TP/IPSec VPN

### Why TCP Destroys Game Performance

Games use **UDP** for a reason: UDP has no built-in acknowledgment, retransmit, or ordering. When a packet is lost, UDP lets the game decide what to do (usually: ignore it and move on). This is fine for games — a missed position update is irrelevant 50ms later.

TCP **never drops packets**. If a packet is lost, TCP:
1. Detects the loss (after a timeout)
2. Retransmits the packet
3. **Holds back all subsequent packets** until the lost one arrives (head-of-line blocking)

If you tunnel UDP game traffic inside TCP (what happens when you run a game through Psiphon):
- One lost packet stalls the entire stream
- You get **ping spikes on every packet loss event**, not just higher average ping
- The retransmit timeout adds 100–300ms of stall per event
- On a mobile connection (which drops packets constantly), this happens every few seconds

This is called the **TCP-over-TCP problem** — it's a known fundamental issue, not something Psiphon can fix. Psiphon was never designed for real-time traffic. It's designed to let you load a blocked website reliably.

### The Actual Difference at a Glance

| Feature | GearUP Booster | Psiphon |
|---|---|---|
| Primary protocol | UDP tunnel | TCP proxy / VPN |
| Purpose | Game latency optimization | Censorship circumvention |
| Game packet handling | Native UDP, priority QoS | Wrapped in TCP → head-of-line blocking |
| Route optimization | Dynamic relay selection, BGP-optimized paths | Fixed server, no route optimization |
| Jitter reduction | FEC + packet prioritization | None (TCP adds jitter on loss) |
| ISP throttle bypass | Yes (game traffic hidden in encrypted UDP) | Yes (but with TCP overhead) |
| Usable for gaming | ✅ Yes | ❌ No — makes it worse |

---

## Patch Notes

### v3 (current)
- **Fixed:** `AppUtils.getSignatureMD5()` now returns the correct Play Store cert MD5 (`E750009C76C0FC310884547EC0123E19`). Previous patch had the wrong value — server rejected every API request, causing "Failed to load."
- **Fixed:** `s3/V$c.smali` — `NativeUtils.checkDeviceRoot()` bypassed, always returns false. Rooted devices were having process boost silently disabled.

### v2
- Removed Play Store signature verification
- Set `debuggable=true`
- All `isValid()` checks in 51 response/model classes patched to return true

---

## Install

```bash
adb install -r gearup_v3_patched.apk
```

If upgrading from v2: uninstall first if you get a signature mismatch error (the signing key changed between patch sessions).
