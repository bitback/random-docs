# Windows Network Routing & Metrics - Technical Reference

## Executive Summary

Windows uses two separate metric values that combine to determine routing priority:
- **InterfaceMetric** (automatic, based on link speed and medium type)
- **RouteMetric** (0 for DHCP routes, 256 for static/persistent routes)
- **Final metric in routing table** = InterfaceMetric + RouteMetric

**Examples:**

Desktop with DHCP:
```
1 Gbps Ethernet → InterfaceMetric 25 + RouteMetric 0 = 25 (in route print)
```

Server with static IP:
```
1 Gbps Ethernet → InterfaceMetric 25 + RouteMetric 256 = 281 (in route print)
```

This explains why the same hardware shows different metrics in `route print` - it's the RouteMetric difference, not the InterfaceMetric.

---

## Complete InterfaceMetric Tables

### Standard Ethernet Interfaces (Windows 10/11, Server 2016+)

Physical medium types: `NdisPhysicalMedium802_3`, `NdisPhysicalMediumUnspecified`

| Link Speed | InterfaceMetric |
|------------|-----------------|
| ≥100 Gbps | 5 |
| ≥40 Gbps and <100 Gbps | 10 |
| ≥10 Gbps and <40 Gbps | 15 |
| ≥2 Gbps and <10 Gbps | 20 |
| **≥200 Mbps and <2 Gbps** | **25** ← **1 Gbps is here** |
| ≥80 Mbps and <200 Mbps | 35 |
| ≥20 Mbps and <80 Mbps | 45 |
| ≥4 Mbps and <20 Mbps | 55 |
| ≥500 Kbps and <4 Mbps | 65 |
| <500 Kbps | 75 |

### Wireless Interfaces

Physical medium types: `NdisPhysicalMediumWirelessLan`, `NdisPhysicalMediumWirelessWan`, `NdisPhysicalMediumNative802_11`

| Link Speed | InterfaceMetric |
|------------|-----------------|
| ≥2 Gbps | 25 |
| ≥500 Mbps and <2 Gbps | 30 |
| ≥200 Mbps and <500 Mbps | 35 |
| ≥80 Mbps and <200 Mbps | 40 |
| ≥20 Mbps and <80 Mbps | 45 |
| ≥4 Mbps and <20 Mbps | 50 |
| ≥500 Kbps and <4 Mbps | 55 |
| <500 Kbps | 60 |

**Important:** Loopback interface always gets **75** regardless of speed.

---

## Metric Types Explained

### 1. InterfaceMetric
- **Calculated automatically** by Windows based on link speed and physical medium type
- Visible in: `Get-NetIPInterface`
- Can be overridden manually (disable "Automatic metric" in adapter settings)

### 2. RouteMetric
- **Per-route value**, varies by system and configuration:
  - **Windows Desktop (DHCP)**: typically **0** for default gateway
  - **Windows Server (Static IP/Persistent)**: typically **256** for default gateway
- Manually added routes can specify custom RouteMetric
- Visible in: `Get-NetRoute`

### 3. Final Routing Table Metric
- **Sum of InterfaceMetric + RouteMetric**
- This is what you see in `route print` output
- Lower value = higher priority

### Example Calculation

**Windows Desktop with DHCP (typical):**
```
Intel 1 Gbps Ethernet with DHCP default gateway:
InterfaceMetric: 25  (automatic for 1 Gbps)
RouteMetric:      0  (DHCP-assigned route)
─────────────────────
route print:     25  ← final metric
```

**Windows Server with Static IP:**
```
HPE 1 Gbps Ethernet with static configuration:
InterfaceMetric: 25  (automatic for 1 Gbps)
RouteMetric:    256  (static/persistent route default)
─────────────────────
route print:    281  ← final metric
```

---

## RouteMetric Behavior: 0 vs 256

Windows assigns different RouteMetric values to default gateway routes depending on configuration method:

### RouteMetric = 0 (Lower final metric in route print)
**Typical scenarios:**
- **DHCP-assigned default gateway** on Windows Desktop (10/11)
- Routes added dynamically by network stack
- Some VPN clients

**Result in route print:**
```
0.0.0.0    0.0.0.0    192.168.35.1    192.168.35.2    25
                                                        ↑
                                      InterfaceMetric (25) + RouteMetric (0) = 25
```

### RouteMetric = 256 (Higher final metric in route print)
**Typical scenarios:**
- **Static IP configuration** on Windows Server
- **Persistent routes** (`route add -p`)
- Some manually configured gateways

**Result in route print:**
```
0.0.0.0    0.0.0.0    10.11.0.254    10.11.0.10    281
                                                     ↑
                                    InterfaceMetric (25) + RouteMetric (256) = 281
```

### Why Does This Matter?

**It usually doesn't** - if you have only one default gateway, the RouteMetric difference is irrelevant. Windows will use the only available route.

**It matters when:**
- You have multiple NICs with default gateways (e.g., primary LAN + backup LAN)
- One has RouteMetric 0 and another has 256
- Lower total metric (0 + InterfaceMetric) wins over higher (256 + InterfaceMetric)

### Checking RouteMetric

```powershell
# View RouteMetric for default gateway
Get-NetRoute -DestinationPrefix 0.0.0.0/0 | Select-Object NextHop, RouteMetric, InterfaceAlias

# Check if IP is DHCP or static
Get-NetIPAddress -InterfaceAlias "Ethernet" | Select-Object IPAddress, PrefixOrigin, SuffixOrigin
# PrefixOrigin = Dhcp → likely RouteMetric 0
# PrefixOrigin = Manual → likely RouteMetric 256
```

---

## Routing Decision Process

Windows routes packets based on:

1. **Most specific match first** (longest prefix match)
   - Example: `192.168.33.0/24` beats `0.0.0.0/0` even with higher metric

2. **If multiple routes to same destination** → lowest metric wins

3. **If metrics are equal** → binding order (adapter priority in Advanced Settings)

### Practical Example

```
Active Routes:
Network Destination    Netmask         Gateway        Interface      Metric
0.0.0.0               0.0.0.0         10.11.0.254    10.11.0.10     281
192.168.33.0          255.255.255.0   192.168.1.1    192.168.1.100  26
```

**Traffic routing:**
- To `192.168.33.5` → uses second route (more specific match)
- To `8.8.8.8` → uses first route (default gateway)
- Metric only matters when multiple routes match the same destination

---

## PowerShell Diagnostic Commands

### View All Adapters with Metrics
```powershell
Get-NetAdapter | Select-Object Name, InterfaceDescription, LinkSpeed, InterfaceMetric
```
Shows adapter names, descriptions, negotiated speeds, and interface metrics.

---

### View IP Configuration with Interface Metrics
```powershell
Get-NetIPInterface -AddressFamily IPv4 | Select-Object InterfaceAlias, InterfaceMetric, AutomaticMetric
```
Displays interface metrics and whether automatic metric is enabled for each adapter.

---

### View Routing Table (PowerShell way)
```powershell
Get-NetRoute -AddressFamily IPv4 | Select-Object DestinationPrefix, NextHop, RouteMetric, InterfaceAlias | Sort-Object RouteMetric
```
Shows all IPv4 routes with their route metrics and associated interfaces.

---

### View Default Gateway Routes
```powershell
Get-NetRoute -AddressFamily IPv4 -DestinationPrefix 0.0.0.0/0 | Select-Object DestinationPrefix, NextHop, RouteMetric, InterfaceAlias
```
Lists only default gateway routes (internet-bound traffic) with their RouteMetric values.

---

### Check IP Configuration Method (DHCP vs Static)
```powershell
Get-NetIPAddress -InterfaceAlias "Ethernet" | Select-Object IPAddress, PrefixOrigin, SuffixOrigin
```
Shows whether IP is assigned via DHCP (PrefixOrigin = Dhcp) or manually (PrefixOrigin = Manual). This often correlates with RouteMetric values (0 for DHCP, 256 for static).

---

### View Traditional Routing Table
```cmd
route print
```
Classic command showing complete routing table with final metrics (InterfaceMetric + RouteMetric).

---

### Check Adapter Speed/Duplex Settings
```powershell
Get-NetAdapterAdvancedProperty -Name "Adapter Name" -DisplayName "Speed & Duplex"
```
Shows speed/duplex negotiation settings for specified adapter.

---

### View Detailed Adapter Info
```powershell
Get-NetAdapter "Adapter Name" | Select-Object Name, LinkSpeed, MediaType, DriverVersion, Status
```
Displays comprehensive adapter information including driver version and operational status.

---

### View Physical Medium Type (requires WMI)
```powershell
Get-WmiObject -Namespace root\wmi -Class MSNdis_PhysicalMediumType
```
Shows what physical medium type the driver reports (determines which metric table Windows uses).

---

## Common Scenarios

### Scenario 1: Two NICs with Same Default Gateway
**Problem:** Both adapters have identical speeds (e.g., both 1 Gbps)

**Final metric depends on:**
1. **InterfaceMetric** (both will have 25 for 1 Gbps)
2. **RouteMetric** (0 for DHCP, 256 for static - if different, lower wins)
3. **Binding order** (if metrics are completely identical)

**Example:**
```
NIC1: DHCP    → InterfaceMetric 25 + RouteMetric 0   = 25  (wins)
NIC2: Static  → InterfaceMetric 25 + RouteMetric 256 = 281
```

**Best practice for manual control:**
```powershell
# Set preferred adapter
Set-NetIPInterface -InterfaceAlias "Primary NIC" -InterfaceMetric 10

# Set backup adapter
Set-NetIPInterface -InterfaceAlias "Backup NIC" -InterfaceMetric 50
```

This overrides automatic metric and ensures predictable routing regardless of DHCP/static configuration.

---

### Scenario 2: Persistent Static Routes
```cmd
# Add persistent route (survives reboot)
route add -p 192.168.33.0 mask 255.255.255.0 192.168.1.1 metric 1

# View persistent routes only
route print | findstr "Persistent"
```

Static routes appear in `Persistent Routes` section of `route print` and are stored in registry at:
`HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters\PersistentRoutes`

---

### Scenario 3: VPN Split Tunneling
VPN adapters typically get high automatic metrics (e.g., Wintun 100 Gbps → metric 5) but can be manually adjusted to control which traffic goes through VPN vs. local gateway.

---

## Historical Note: Algorithm Changes

**Windows XP SP2 - Windows 7 / Server 2008 R2:**
```
≥2 Gbps   → metric 5
>200 Mbps → metric 10
>20 Mbps  → metric 20
(etc.)
```

**Windows 10 / Server 2016 and newer:**
Uses more granular tables shown above with separate handling for different physical medium types.

**Impact:** Documentation from pre-2016 era is outdated and will show incorrect metric values.

---

## Key Takeaways

1. **1 Gbps Ethernet = InterfaceMetric 25** is correct and expected behavior
2. **route print shows sum** of InterfaceMetric + RouteMetric
3. **RouteMetric varies**: 0 (DHCP/Desktop) vs 256 (Static/Server) - this explains why same adapter shows 25 vs 281
4. **Physical medium type** from driver determines which metric table applies
5. **No difference** between Windows Server and Desktop versions in InterfaceMetric calculation (same algorithm since Windows 10/Server 2016)
6. **Most specific route wins** regardless of metric
7. **Metric matters** only when multiple routes match the same destination

---

## References

- Microsoft Documentation: Automatic Metric Calculation
- NDIS Physical Medium Types (OID_GEN_PHYSICAL_MEDIUM)
- TCP/IP Routing in Windows Server 2016+

**Document version:** 2025-11-14
**Applies to:** Windows 10, Windows 11, Windows Server 2016, 2019, 2022
