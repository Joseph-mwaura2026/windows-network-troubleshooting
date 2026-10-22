# Windows Network Troubleshooting

## Project Overview

This project documents practical troubleshooting of Windows network connectivity problems using built-in Windows networking tools.

The investigation covered IP configuration, default routing, gateway connectivity, internet reachability, and DNS troubleshooting.

**Project type:** IT Support and Networking  
**Environment:** Windows, mobile hotspot, VirtualBox, Hyper-V  
**Tools:** Command Prompt, PowerShell, ipconfig, ping, route, tracert, nslookup

## Objectives

- Inspect Windows network configuration.
- Identify potential IP addressing and routing problems.
- Test connectivity to a local gateway and external IP addresses.
- Understand the difference between DNS resolution and IP connectivity.
- Verify connectivity after corrective actions.
- Document troubleshooting evidence and lessons learned.

## 1. Problem Statement

During troubleshooting, the laptop experienced connectivity problems while connected to a mobile hotspot.

The investigation revealed unexpected network configuration, including multiple IPv4 addresses on the Wi-Fi adapter and an incorrect default route that required correction.

A related session also involved a browser timeout when accessing Notion.

## 2. Diagnostic Commands

### Inspect network configuration

```cmd
ipconfig /all
```

Displays IP addresses, subnet masks, default gateways, DNS servers, and DHCP settings.

### Test gateway connectivity

```cmd
ping 192.168.8.1
```

Tests whether the laptop can communicate with the configured gateway.

### Test external IP connectivity

```cmd
ping 8.8.8.8
```

Tests reachability to an external IP address without requiring DNS resolution.

### Inspect IPv4 routing

```cmd
route print -4
```

Displays IPv4 routes and helps identify which gateway Windows uses for outgoing traffic.

### Check DNS resolution

```cmd
nslookup google.com
```

Queries DNS to resolve a hostname to an IP address.

## 3. Findings and Results

| Test | Observed result |
|---|---|
| Network configuration | Two IPv4 addresses and DHCP disabled in one diagnostic session |
| Gateway connectivity | 13 packets sent, 0 received in one test |
| External IP connectivity | 100% packet loss in one test |
| Routing investigation | An incorrect default route was removed |
| Verification | A subsequent external connectivity test received 30 out of 30 replies |

The successful retest demonstrated that external IP connectivity had been restored at that point.

The exact cause should be supported by comparing the routing table before and after the correction.

## 4. Troubleshooting Lessons

- Inspect network configuration before changing settings.
- Identify the active default route.
- Test gateway connectivity separately from internet connectivity.
- Test DNS resolution separately from IP connectivity.
- Repeat tests after making corrections.
- Preserve command output and screenshots as evidence.

## 5. Skills Demonstrated

- Windows network troubleshooting
- IPv4 addressing and default gateways
- Routing table analysis
- Command Prompt and PowerShell
- DNS diagnostics
- Connectivity testing
- Technical documentation
- Evidence-based problem solving

## 6. Evidence

Screenshots will be added to an `images/` folder as the project documentation is completed.

Planned evidence:

- `01-ipconfig-all.png`
- `02-gateway-ping.png`
- `03-internet-ping.png`
- `04-route-print.png`
- `05-routing-correction.png`
- `06-successful-ping.png`

## 7. Conclusion

This project demonstrates a structured approach to Windows networking diagnostics.

By inspecting network configuration, investigating routing, and repeating connectivity tests, I documented a troubleshooting process that restored external IP reachability during the verification test.

The project will be expanded with redacted screenshots and additional evidence to strengthen the root-cause analysis.

**Environment:** Personal Windows laptop and virtual lab.

**Authorization:** Testing was performed on systems and networks used for personal learning and troubleshooting.
