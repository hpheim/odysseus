# UDM Pro VPN & Remote Access Runbook

**Scope:** Remote access to home LAN (10.10.10.0/24) via UDM Pro VPN — covers WireGuard, Teleport, DNS resolution, and RDP to 10.10.10.10.

---

## 1. VPN Options

### Teleport (UniFi's zero-config VPN)
- Works behind CGNAT / double-NAT (no port forwarding needed).
- Uses UniFi's relay infrastructure.
- Enable: UniFi Network → Settings → Teleport & VPN → Teleport → Enable.
- Client: WiFiman app (iOS/Android) or UniFi Identity app.
- Limitation: does not push custom DNS servers to clients by default.

### WireGuard (site-to-site or client-to-site)
- Requires UDP port 51820 forwarded to the UDM Pro WAN IP (or the VPS acting as relay).
- Enable: UniFi Network → Settings → Teleport & VPN → VPN Server → Create → WireGuard.
- Set the server address to the public WAN IP (or VPS public IP if relaying).
- Client config is downloaded from the UI or generated via QR code.
- Pushes DNS and routes as configured.

**Decision:** Use WireGuard if you have a static/known WAN IP or a VPS relay. Use Teleport if behind CGNAT with no VPS.

---

## 2. DNS Resolution Over VPN

### Problem
VPN tunnel is up but internal hostnames don't resolve (e.g., `nslookup myserver.local` fails).

### Fix — WireGuard
In the WireGuard server config on the UDM Pro, set the DNS server to your internal DNS IP (e.g., `10.10.10.1` or a Pi-hole/AdGuard at `10.10.10.x`). The client `.conf` must have:

```ini
[Interface]
DNS = 10.10.10.1
```

### Fix — Teleport
Teleport does not natively push DNS. Workaround: after connecting via Teleport, manually set the device DNS to the internal DNS IP, or use IP addresses directly.

---

## 3. RDP to 10.10.10.10

### Prerequisites
1. **Target machine must be powered on.** If asleep, use Wake-on-LAN via the UniFi Network app (Settings → Network → select the client → Wake).
2. **RDP must be enabled** on the Windows machine: Settings → System → Remote Desktop → On.
3. **Windows Firewall** must allow RDP (port 3389/TCP) on the network profile the VPN adapter uses.

### Common failure: Firewall profile mismatch
Windows often classifies a VPN adapter as "Public" network, where RDP is blocked by default.

**Fix (run on the Windows machine or via remote PowerShell):**
```powershell
# Check current profile
Get-NetConnectionProfile

# If the VPN adapter shows "Public", switch to Private:
Set-NetConnectionProfile -InterfaceAlias "WireGuard Tunnel" -NetworkCategory Private

# Or allow RDP on Public profile:
Set-NetFirewallRule -DisplayGroup "Remote Desktop" -Enabled True -Profile Public
```

### Connecting
From iPhone: use Microsoft Remote Desktop app → Add PC → enter `10.10.10.10`.
From laptop: `mstsc /v:10.10.10.10` or any RDP client.

---

## 4. Subnet Overlap

### Problem
Hotel/travel WiFi uses the same `10.10.10.0/24` range as home LAN. VPN connects but traffic stays local.

### Fix
Change the home LAN to a non-overlapping range (e.g., `10.10.50.0/24`) or use the VPN's AllowedIPs to force all traffic through the tunnel (`0.0.0.0/0`). If changing the LAN subnet:

1. UniFi Network → Settings → Networks → Default → change Gateway/Subnet.
2. Update DHCP range.
3. Update any static IP reservations (including 10.10.10.10 → 10.10.50.10).
4. Regenerate WireGuard client configs.

---

## 5. VPS as WireGuard Relay

If the UDM Pro is behind CGNAT (no direct port forward):

1. Run WireGuard on a VPS with a public IP.
2. UDM Pro connects to VPS as a WireGuard peer (site-to-site).
3. VPS forwards client traffic to UDM Pro's tunnel.
4. Clients connect to VPS public IP, traffic routes through to home LAN.

```
[Client phone] → VPS:51820 → WireGuard tunnel → UDM Pro → LAN 10.10.10.0/24
```

---

## 6. Diagnostic Commands (run via SSH)

```bash
# On UDM Pro (SSH in):
wg show                          # WireGuard interface status
ip route show table all          # routing tables
nslookup myhost.local 10.10.10.1 # test internal DNS
ping 10.10.10.10                 # test LAN reachability

# On VPS (if relay):
wg show
tcpdump -i wg0 -n               # watch tunnel traffic
ss -ulnp | grep 51820           # confirm WireGuard listening

# On Windows target:
Test-NetConnection -ComputerName 10.10.10.10 -Port 3389  # RDP reachability
Get-NetConnectionProfile         # check firewall profile
```

---

## 7. Quick Checklist — "I'm traveling and can't RDP"

1. [ ] VPN connected? (check WireGuard/Teleport app status)
2. [ ] Can you ping 10.10.10.1 (gateway)? If no → tunnel routing issue or subnet overlap.
3. [ ] Can you ping 10.10.10.10? If no → target machine off (Wake-on-LAN) or firewall.
4. [ ] Can you resolve internal DNS names? If no → DNS not pushed over tunnel.
5. [ ] RDP connection refused? → Windows Firewall profile is Public; switch to Private.
6. [ ] RDP black screen / timeout? → Machine asleep; send Wake-on-LAN.
