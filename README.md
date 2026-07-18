# GearUP Game Booster — Patched APK

> **Patched build:** `gearup_v3_patched.apk`  
> Signature check bypassed · Root bypass · All validation patches applied

---

## Install

```bash
adb install -r gearup_v3_patched.apk
```

If upgrading from v2: uninstall first if you hit a signature mismatch (signing key changed between patch sessions).

---

## How GearUP Actually Boosts Your Internet in Games

Most people think a "game booster" just clears RAM or kills background apps. GearUP works at the **network layer** — and that's why it actually changes ping.

### The Real Problem: Your Route to the Game Server Is Bad

When you connect to a game server, your packets don't travel in a straight line. They hop through your ISP's routers, then regional hubs, then a national backbone, then a peering point, then finally the server:

```
You → ISP Router → Regional Hub → National Backbone → Peering Point → Game Server
```

Every hop adds latency. Some hops are congested (overloaded routers at peak hours). Some go in the wrong direction before correcting. Your ISP doesn't care about your ping — they route packets down whatever path they have a contract for, not the fastest path.

This is why you can have 100 Mbps fiber and still get 120ms to a game server that's physically 300 km away.

---

### What GearUP Does Internally (Reverse-Engineered from the APK)

By decompiling the APK and extracting the native libraries, the full architecture is visible:

#### Step 1 — Android VpnService captures all traffic
The app creates a VPN tunnel on-device (`com/divider2/vpn/`). This uses the Android `VpnService` API to intercept every IP packet leaving your phone — no root needed.

#### Step 2 — TProxy classifies game traffic
The native library `libdivider2.so` runs a transparent proxy (`tproxy_connection_tree`). It inspects each packet's destination IP + port. Game traffic is identified using `DSL$TProxyTrafficIdentifyRule` — a list of IP ranges and port ranges per game title. Non-game traffic bypasses the tunnel entirely.

#### Step 3 — libRouteTable.so looks up the best relay
`libRouteTable.so` is a binary trie (radix tree) IP lookup table encoded in Protocol Buffers. It maps game server IP ranges to specific relay nodes. The mapping is downloaded from `gearupportal.com` and updated dynamically.

#### Step 4 — Encrypted UDP tunnel to the relay
Game traffic is wrapped in **AES-128-GCM** encryption and sent to the relay node (`sproxy` = secure proxy). The key is negotiated per-session (`aesGcmPwd`). The relay then forwards the decrypted packets to the actual game server.

#### Step 5 — P2P attempted first, relay as fallback
For some routes the app attempts a P2P direct path first. If `p2p_max_retries` is exceeded or `p2p_timeout` fires, it falls back to the `sproxy` relay automatically.

#### Step 6 — SNI routing for HTTPS game traffic
For game traffic that goes over HTTPS, `libdivider2.so` uses SNI proxying (`getSniDomain`) — it reads the SNI field in the TLS handshake to route to the right relay without decrypting the game's own TLS traffic.

---

### What "Private BGP Peering" Actually Means (Plain English)

BGP (Border Gateway Protocol) is the system that controls how the entire internet routes traffic between networks. Every ISP, every data center, every cloud provider runs BGP to tell each other "I can reach these IP addresses, send traffic to me."

When your ISP routes a packet to a game server, it follows whatever BGP routes it has received from its upstream providers — usually whoever the ISP has a paid transit contract with. That provider has their own BGP routes from their upstream, and so on. The path is determined by **business contracts**, not by what's actually fastest.

**Private peering** means two networks agree to connect directly to each other at a **physical location** (called an Internet Exchange Point, or IXP) and exchange BGP routes with each other. No transit provider in between, no extra hops, packets jump directly from one network to the other.

GearUP's relay nodes have private peering at IXPs near major gaming data centers. So:

```
Normal path (your ISP):
You → ISP → Transit A → Transit B → Peering Point → Game Server
     +5ms    +12ms       +8ms         +15ms           = ~120ms total

GearUP path:
You → GearUP Relay ——[private peering]——→ Game Server
     +8ms           direct exchange         = ~30ms total
```

The relay costs 8ms to reach but eliminates 35ms of unnecessary hops. The relay has a direct BGP session with the game server's network, which your ISP doesn't have.

#### Why Your ISP Doesn't Just Do This

They could. But private peering has costs — you pay for the physical cross-connect at the IXP, you need engineers to manage the BGP sessions, and you need to peer with thousands of different networks to cover all possible destinations. ISPs do peer where it makes business sense (high-volume destinations like Google, Netflix). Game servers don't generate enough traffic for your ISP to justify the peering cost. GearUP's entire business model is doing exactly this for gaming destinations.

---

### Why Ping Stays Low and Stable (Jitter Reduction)

Ping spikes are worse than consistently high ping. A jump from 30ms to 200ms causes rubber-banding and missed shots even if your average is fine. GearUP reduces jitter two ways:

**UDP tunnel with priority QoS** — Game packets get priority over everything else going through the relay. Your ISP naturally deprioritizes UDP bursts (they look like P2P to DPI systems). Inside GearUP's tunnel, the ISP sees encrypted data center traffic and doesn't throttle it.

**Forward Error Correction (FEC)** — On lossy mobile/WiFi connections, the tunnel sends redundant data alongside packets so the relay can reconstruct a dropped packet without waiting for a retransmit. A UDP retransmit causes a spike. FEC eliminates that spike at the cost of ~10% more bandwidth.

---

## Why Psiphon Does NOT Do the Same Thing

Psiphon is a **censorship circumvention** tool, not a game booster. Completely different design goal.

### Psiphon's Architecture

```
You → Psiphon Server (HTTP proxy / SSH tunnel / L2TP) → Internet
```

Psiphon is built around **TCP** to carry web traffic (HTTP/HTTPS) reliably through censored networks.

### Why TCP Destroys Game Performance

Games use **UDP** because UDP has no acknowledgment, retransmit, or ordering. When a UDP packet is lost, the game moves on — a missed position update is irrelevant 50ms later.

TCP **never drops a packet**. When one is lost:
1. TCP detects it after a timeout (~200ms on a mobile connection)
2. Retransmits the lost packet
3. **Holds back all subsequent packets** until the lost one arrives (head-of-line blocking)

If you tunnel UDP game traffic inside TCP (which happens through Psiphon):
- One lost packet stalls the entire stream for 200-300ms
- On a mobile connection that drops packets every few seconds, this fires constantly
- Your "ping" becomes: normal ping + TCP retransmit timeout = massive spikes

This is the **TCP-over-TCP meltdown problem** — a known fundamental issue described in academic literature. Psiphon can't fix it because the design choice to use TCP is load-bearing for its censorship-circumvention use case (firewalls block UDP but not HTTPS).

**Psiphon is also not route-optimized.** It picks a fixed server for circumvention, not the server with the lowest latency path to your game server. Even if the TCP problem didn't exist, Psiphon's server might be geographically worse for your game than your normal connection.

### The Difference at a Glance

| | GearUP Booster | Psiphon |
|---|---|---|
| Core protocol | UDP tunnel (AES-128-GCM) | TCP proxy / SSH / L2TP |
| Purpose | Game latency optimization | Censorship circumvention |
| Game packets | Native UDP, TProxy transparent | Wrapped in TCP → head-of-line blocking |
| Route selection | Dynamic per-game, BGP-optimized | Fixed server, no optimization |
| Jitter control | FEC + QoS prioritization | None |
| ISP throttle bypass | Yes (looks like encrypted DC traffic) | Yes (looks like HTTPS) |
| Works for gaming | ✅ Yes | ❌ No — makes it worse |

---

## Free Open-Source Alternatives That Actually Work for Gaming

GearUP's relay network is proprietary — there's no truly free equivalent that runs the same infrastructure. But these get close in different ways:

### 1. Cloudflare WARP — Best Free Option (Not Open-Source, But Free)
**How it works:** WireGuard tunnel (UDP-based) to Cloudflare's network, then Argo Smart Routing to the destination.  
**Why it's similar:** Cloudflare has private peering at hundreds of IXPs globally, same concept as GearUP's relay nodes.  
**Download:** [1.1.1.1 app](https://1.1.1.1/) — completely free tier.  
**Limitation:** Not game-specific, no per-game routing rules. But on many routes it genuinely lowers ping because Cloudflare peers closer to game servers than your ISP does.

### 2. WireGuard — Open-Source UDP Tunnel, Self-Hosted
**GitHub:** https://github.com/WireGuard/wireguard-android  
**How it works:** You rent a VPS ($5/month, DigitalOcean, Vultr, Hetzner) near your game's server region and install WireGuard. Traffic goes: You → Your VPS (WireGuard UDP) → Game Server.  
**Why it's better than Psiphon:** WireGuard is natively UDP. No TCP-over-TCP problem. Sub-millisecond overhead.  
**Why it works:** If you pick a VPS in the same city or datacenter as the game server, you cut out your ISP's bad routing entirely. A Hetzner VPS in Frankfurt costs €4/month and will outperform GearUP for European game servers.  
**Setup time:** ~20 minutes with a one-line install script.

### 3. Shadowsocks + AEAD — Open-Source, UDP Capable
**GitHub:** https://github.com/shadowsocks/shadowsocks-android  
**How it works:** Shadowsocks with AEAD ciphers supports UDP relay mode — game UDP packets are wrapped and forwarded, not dropped into TCP like older proxies.  
**Why it's relevant:** Widely deployed in Asia with many free public servers. Used heavily for gaming in China/SEA where ISP routing is especially bad.  
**Note:** Encryption is ChaCha20-IETF-Poly1305 or AES-256-GCM — similar to what GearUP uses internally.

### 4. V2Ray / Xray — Open-Source, True UDP Support
**GitHub:** https://github.com/XTLS/Xray-core  
**How it works:** XUDP (extended UDP) over VLESS/XTLS carries game UDP packets end-to-end without TCP wrapping.  
**Why it's better than Psiphon:** Designed from scratch to support real UDP proxying, not just TCP forwarding.  
**Limitation:** Complex to configure. Needs your own server.

### 5. SoftEther VPN Gate — Free Public Relays, Open-Source
**Website:** https://www.vpngate.net  
**GitHub:** https://github.com/SoftEtherVPN/SoftEtherVPN  
**How it works:** Volunteer-operated relay servers worldwide. Supports L2TP/IPSec (UDP-based) and OpenVPN.  
**Why it works for gaming:** L2TP/IPSec uses UDP port 500/4500 — no TCP-over-TCP problem. Some volunteer nodes are on fast connections near game server regions.  
**Limitation:** Server quality varies wildly. You have to test which node gives you the best route to your game.

### Which to Choose

| Situation | Recommendation |
|---|---|
| Want zero config, just works | **Cloudflare WARP** |
| Want full control, willing to rent a $5 VPS | **WireGuard** (best performance) |
| Playing games in SEA/Asia, need public free servers | **Shadowsocks** or **VPN Gate** |
| Want GearUP-level per-game routing, self-hosted | **Xray + XUDP** (advanced) |
| Avoid at all costs for gaming | Psiphon, OpenVPN TCP mode, any HTTP proxy |

---

## Secrets Found Inside the APK

Full static analysis of the decompiled smali code and native `.so` libraries exposed the following hardcoded credentials and endpoints.

### Sentry Error Tracking DSNs
The app sends crash reports and runtime errors to two separate Sentry instances. These DSNs include the auth token embedded in the URL:

```
https://54c7163c5088445f94f3dd2866800944@sentry.guinfra.com/27
https://88368f80614049beaee0a763b0df1545@sentry.easebar.com/136
```

`guinfra.com` is GearUP's own infrastructure domain. `easebar.com` is their parent company (Easebar Network Technology). Both DSNs allow submitting fake error reports to their monitoring dashboards.

### A/B Test Endpoint + Project Key
```
https://abtest.sc.gearupportal.com/api/v2/abtest/online/results
  ?project-key=E0AE1BAD954BC983B1A67F95967B924503599695
```
This is how the app fetches which UI experiments are active for a user. The project key is hardcoded — any client can query it.

### Firebase (Google Analytics + FCM Push)
```
Google API Key:       AIzaSyAEGASoAOHhuqd4YE6pA432ao2OpyuDjC4
Google App ID:        1:199832736237:android:42715638e2096bf86757e4
GCM Sender ID:        199832736237
Web Client ID:        199832736237-fv3cj0fkre9ddo8gvfrt0qh4daprobd9.apps.googleusercontent.com
```
The Google API key is restricted to this app by package name + SHA-1 cert on Firebase's side, but the raw value is visible to anyone who extracts the APK.

### Facebook SDK
```
App ID:        853347009696108
Client Token:  833dc01a1a02660ae13aef1518f80ed6
```
Used for Facebook Login and analytics. The client token allows calling Facebook Graph API endpoints scoped to this app.

### AppsFlyer Attribution
```
Dev Key: qCWrc7L2pa8SbLEvonfmja
```
This is the advertiser attribution key used to track installs and revenue attribution. With this key you could submit fake install events to their AppsFlyer dashboard and inflate their reported install numbers.

### Sentry ProGuard Source Map UUID
```
1c76e873-bb8b-4fe4-8db3-16b46c2725ab
```
This UUID maps minified ProGuard stack traces back to original source line numbers on their Sentry dashboard. It's what lets their developers see readable crash reports instead of obfuscated class names.

### Internal API Domain Pattern
```
^https://(?:[a-zA-Z0-9-]+\.)*gearupportal\.com(/|$)
```
All GearUP's backend APIs live under `gearupportal.com` subdomains. The app enforces SSL pinning on this pattern — any subdomain of `gearupportal.com` over HTTPS is trusted.

Known subdomains found:
- `abtest.sc.gearupportal.com` — A/B testing
- `*.gearupportal.com` — main API (acc config, boost lists, game lists)

### Contact / Support Channels (Hardcoded in App)
```
Discord:   https://discord.gg/z7FSytVDgY
WhatsApp:  https://wa.me/6591902075   (Singapore +65)
Facebook:  https://www.facebook.com/Gearup-Booster-103878812520939
Line:      https://lin.ee/L7Ycc0B
```

### Internal Architecture Secrets (From Native .so Strings)

**Encryption:** The relay tunnel uses `AES-128-GCM` with a session password negotiated via the key field `aesGcmPwd`. The cipher modes compiled into `libdivider2.so` include AES-128/192/256 in GCM, CBC, CTR, and OFB — GCM is the active mode for game traffic.

**Relay protocol name:** `sproxy` (secure proxy). Log strings: `sproxy auth connect`, `fallback to sproxy`. It's their internal name for the relay-node protocol.

**P2P with relay fallback:** The engine first attempts a P2P direct path (`addP2PRoute`). If `p2p_max_retries` is exceeded or `p2p_timeout` fires, it silently falls back to `sproxy`. Most users are always on sproxy because P2P requires the game server to cooperate.

**SNI routing:** `getSniDomain` in `libdivider2.so` reads the SNI field from TLS ClientHello packets to route HTTPS game traffic to the correct relay without breaking the game's own TLS encryption.

**Route table format:** `libRouteTable.so` is a binary trie (radix tree) IP lookup table. Routes are encoded in Protocol Buffers (`route_info`, `route_info_list` message types). The route table is downloaded from the API and stored locally for fast per-packet lookups.

**DNS override:** `114.114.114.114` is hardcoded as the DNS server — this is China Unicom's public DNS. IP range `163.163.0.1–163.163.255.255` is in the routing table, which is China Unicom's backbone range. The app is clearly built primarily for the Chinese gaming market and extended internationally.

---

## Patch Notes

### v3 (current)
- **Fixed:** `AppUtils.getSignatureMD5()` → correct Play Store cert MD5 `E750009C76C0FC310884547EC0123E19`. Previous patch had the wrong value; server rejected every API request.
- **Fixed:** `s3/V$c.smali` → `NativeUtils.checkDeviceRoot()` bypassed, always returns `false`. Rooted devices had process boost silently disabled.

### v2
- Removed Play Store signature verification
- Set `debuggable=true`
- All `isValid()` checks in 51 response/model classes patched to return `true`
