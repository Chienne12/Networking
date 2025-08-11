# Networking## README – How to use the Cisco Packet Tracer files (quick tests and troubleshooting)

[Place this file in: `C:\Users\LaptopK1\Documents\Document Study\NetWorking\AssignMent\Part 2` next to the `.pkt` files]

## 1) Environment
- Cisco Packet Tracer 8.x or later
- Open the `.pkt` files directly: `Cisco_Config_Optimized.pkt`, `Cisco_Config_Initial.pkt`, `VTP_VLAN_Packet_Tracer_Config.pkt`

## 2) Quick start
1. Open a `.pkt` file (recommended: `Cisco_Config_Optimized.pkt`).
2. Wait for devices to boot (stable green LEDs).
3. Main components:
   - Core L3 Switch: SVIs for VLAN 10/20/30/40/50/70/80/81/82, `ip routing` enabled
   - Border/ISP Router: NAT and default/static routes
   - DHCP/DNS/Web/Print Servers (VLAN 70)
   - WLC/AP: Wi‑Fi mapped to VLAN 80/81/82

## 3) Addressing/VLAN quick reference
- Supernet 200.1.0.0/16; each VLAN /24; gateway .1
- Examples:
  - VLAN 10 Students: 200.1.10.0/24 (GW 200.1.10.1)
  - VLAN 20 Teachers: 200.1.20.0/24 (GW 200.1.20.1)
  - VLAN 30 Staff: 200.1.30.0/24 (GW 200.1.30.1)
  - VLAN 40 Management: 200.1.40.0/24 (GW 200.1.40.1)
  - VLAN 50 IT_Admin: 200.1.50.0/24 (GW 200.1.50.1)
  - VLAN 70 Servers: 200.1.70.0/24 (GW 200.1.70.1)
  - VLAN 80/81/82 Wireless: Students / Staff-Admin / Guest
- Servers: DHCP `200.1.70.10`, DNS `200.1.70.11`, Web `200.1.70.12`

## 4) Quick tests (Ping/DNS/NAT)
- DHCP: PC → Desktop → IP Configuration → DHCP (expect valid IP/gateway/DNS)
- Same VLAN ping: PC in VLAN10 → another in VLAN10: `ping 200.1.10.11` (expect success)
- Inter‑VLAN ping (allowed): VLAN20 → Web (VLAN70): `ping 200.1.70.12` (expect success)
- ACL‑blocked ping: VLAN10 → IT_Admin (VLAN50): `ping 200.1.50.10` (expect fail)
- DNS: PC → Command Prompt: `nslookup dhcp.education.local 200.1.70.11`
- NAT: internal PC → external host (e.g. `8.8.10.15`): `ping 8.8.10.15`; on router: `show ip nat translations`

## 5) State/health commands
- Core Switch:
```
show ip interface brief | include Vlan
show vlan brief
show interfaces trunk
show ip route
show access-lists
show running-config | section ^interface Vlan
```
- Border Router:
```
show ip route | beg Gateway
show ip nat translations
show running-config | include ip nat (inside|outside)|ip nat pool|access-list 1
```
- VTP (on switches):
```
show vtp status
show vtp counters
```

## 6) Common faults and quick fixes (playbook)
### 6.1 DHCP client gets APIPA 169.254.x.x (no IP)
- Check:
```
show ip interface brief | include Vlan 80|Vlan 10|Vlan 20|Vlan 30|Vlan 40|Vlan 50|Vlan 70
show access-lists Students-Wireless-ACL
```
- Fix Wi‑Fi ACL if DHCP (UDP 67/68) is blocked:
```
conf t
ip access-list extended Students-Wireless-ACL
 permit udp any eq 68 any eq 67
 permit udp any eq 67 any eq 68
 permit udp any any eq 67
 permit udp any any eq 68
exit
end
```
- Ensure SVI has helper: `ip helper-address 200.1.70.10`

### 6.2 Inter‑VLAN not working
- Check:
```
show run | include ^ip routing
show ip interface brief | include Vlan
show interfaces trunk
show access-lists
```
- Fix:
```
conf t
ip routing
! verify trunk allowed VLANs and SVI up/up
end
```

### 6.3 Students not blocked from IT_Admin/Management
- Verify standard ACL direction on destination SVIs:
```
show ip interface vlan 50
show ip interface vlan 40
```
- Place OUT on destination SVIs:
```
conf t
interface vlan 50
 no ip access-group 20 in
 ip access-group 20 out
interface vlan 40
 no ip access-group 10 in
 ip access-group 10 out
end
```

### 6.4 VTP not syncing (config digest errors)
- Check: `show vtp status`, `show vtp counters`
- Re‑align domain/password/version:
```
conf t
no vtp domain
no vtp password
no vtp version
vtp domain EDU-NET
vtp mode client
vtp password 123456
vtp version 2
end
! On Core set vtp mode server with the same settings
```

### 6.5 NAT cannot reach Internet
- Check:
```
show running-config | include ip nat (inside|outside)|access-list 1|ip nat pool
show ip route 0.0.0.0
```
- Example quick fix:
```
conf t
access-list 1 permit 200.1.0.0 0.0.255.255
ip nat inside source list 1 interface GigabitEthernet0/0 overload
end
```

## 7) Packet path example after ACL is applied
- 200.1.10.10 (Students) → 200.1.50.10 (IT_Admin): packet enters the Core via `Vlan10` (in), is routed to 200.1.50.0/24, exits via `Vlan50` (out) and is dropped by ACL 20 applied outbound on `Vlan50` (deny source 200.1.10.0/24). Result: ping fails and ACL 20 hit‑counter increases.

## 8) Screenshot suggestions
- Capture `show access-lists` before/after the ping
- Core `show ip int br` (SVIs up/up)
- `show vtp status` when fixing VTP
- `show ip nat translations` when verifying NAT

---
This guide lets you open the simulation, run quick tests, and apply a concise playbook to fix common issues (DHCP, ACL, VTP, NAT, inter‑VLAN). If you need an English/VI bilingual version or additional automated scenarios, tell me and I’ll add them.
