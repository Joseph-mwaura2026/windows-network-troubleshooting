# Windows Network Troubleshooting: Wrong Static Configuration and Stale Route

- **Type:** IT support and networking case study
- **Status:** Original incident resolved and verified. Evidence screenshots come from a lab reproduction (see the evidence note).
- **Environment:** Windows 11 laptop on a mobile hotspot (original incident); Windows 11 VM in a VirtualBox lab (reproduction)
- **Tools:** Command Prompt, PowerShell, `ipconfig`, `netsh`, `route`, `ping`, `nslookup`, `Test-NetConnection`

## Summary

A Windows laptop could not reach the internet through a mobile hotspot. The Wi-Fi adapter had a manual IPv4 configuration that did not match the hotspot network, and an obsolete persistent default route was still in the routing table. I restored a correct configuration, removed the stale route, and verified recovery in layers: gateway, public IP, DNS and TCP port 443.

## Evidence note

The screenshots from the original incident were not retained, so the incident is documented in section 2 as text records of the commands and results. The screenshots in section 4 come from a **controlled lab reproduction of the same type of fault**, performed in October 2026 on a Windows 11 VM. The addresses, latency figures and the way the adapter was restored differ from the original incident, and the differences are explained in section 4.

## 1. Problem statement

The laptop was connected to a mobile hotspot but had no internet access.

## 2. Original incident record (text)

**Symptoms**

| Test | Result |
|---|---|
| `ping 8.8.8.8 -n 20` | 20 sent, 0 received (100% loss) |
| Ping to the configured gateway `192.168.8.1` | 100% loss |
| Wi-Fi configuration | DHCP disabled; two manual IPv4 addresses (`192.168.1.240/24` and `192.168.8.240/24`) |
| Routing table | Default route via `192.168.8.1` using a source address from a different subnet; a persistent default route via `192.168.0.240` |

**Fix applied**

1. Opened an elevated Command Prompt.
2. Returned the adapter to DHCP: `netsh interface ipv4 set address name="Wi-Fi" source=dhcp`
3. Returned DNS to DHCP: `netsh interface ipv4 set dnsservers name="Wi-Fi" source=dhcp`
4. Requested a new lease: `ipconfig /renew "Wi-Fi"`
5. Confirmed the hotspot assigned `10.246.208.208/24` with gateway `10.246.208.51`.
6. Removed the stale persistent route: `route delete 0.0.0.0 mask 0.0.0.0 192.168.0.240` (returned `OK!`, and the persistent routes list then showed `None`).

**Verification after the fix**

| Test | Result |
|---|---|
| `ping 10.246.208.51 -n 10` (gateway) | 10/10 replies, 0% loss, about 6 ms average |
| `ping 8.8.8.8 -n 20` | 20/20 replies, 0% loss, about 46 ms average |
| `ping 8.8.8.8 -n 30` | 30/30 replies, 0% loss, about 44 ms average |
| `nslookup google.com` | Answer returned. The network's resolver mapped google.com to `forcesafesearch.google.com`, which is DNS filtering behaviour and not a DNS fault. |
| `Test-NetConnection www.google.com -Port 443` | `TcpTestSucceeded: True` |

## 3. Diagnosis

- The static addresses did not belong to the hotspot's network, so the configured gateway was unreachable. Ping to the gateway failed, and so did ping to the internet.
- The active default route used a source address from one subnet while pointing at a gateway in another.
- A stale persistent default route via `192.168.0.240` could interfere with normal routing.
- Testing the gateway, the public IP, DNS and TCP/443 **separately** made it clear that the fault was local configuration, not the hotspot or DNS.

## 4. Evidence: lab reproduction

**Setup.** A Windows 11 VM with a snapshot taken beforehand. The adapter `Ethernet 2` sits on the lab LAN (`10.0.1.0/24`) and reaches the internet through the pfSense gateway `10.0.1.1`. Its working baseline was a **static** configuration: `10.0.1.12/24`, gateway `10.0.1.1`, DNS `10.0.1.2`. Because the baseline was static (not DHCP like the hotspot case), recovery meant **restoring the original static values**.

**Fault created:**
```
netsh interface ipv4 set address name="Ethernet 2" static 192.168.8.240 255.255.255.0 192.168.8.1
route -p add 0.0.0.0 mask 0.0.0.0 192.168.8.99 metric 5
```

### Step 1: Internet ping fails
`ping 8.8.8.8 -n 20`: 20 sent, 17 timed out. The 3 packets counted as "received" were **"Destination host unreachable" replies from the VM's own wrong address** (`192.168.8.240`), not replies from 8.8.8.8, so there was no real connectivity.

![Internet ping failing](images/01-internet-ping-failure.png)

### Step 2: Wrong static configuration
`netsh interface ipv4 show config name="Ethernet 2"` shows DHCP disabled, address `192.168.8.240`, gateway `192.168.8.1`, and the lab DNS server `10.0.1.2` still set.

![Static configuration](images/02-static-configuration.png)

### Step 3: Conflicting routes
`route print -4` shows two default routes (via `192.168.8.1` and via `192.168.8.99`) in the active table and two in the persistent table.

![Route table with stale route](images/03-route-print-conflict.png)

### Step 4: Correct configuration restored
The adapter is set back to its working static values: `10.0.1.12/24`, gateway `10.0.1.1`, DNS `10.0.1.2`.

```
netsh interface ipv4 set address name="Ethernet 2" source=static address=10.0.1.12 mask=255.255.255.0 gateway=10.0.1.1 gwmetric=256
```

![Configuration restored](images/04-configuration-restored.png)

### Step 5: Stale persistent route removed
`route delete 0.0.0.0 mask 0.0.0.0 192.168.8.99` returns `OK!`. The route table then shows a single default route via `10.0.1.1`, and the persistent routes list holds only that gateway (no `192.168.8.99` entry).

![Persistent route removed](images/05-route-delete-persistent-routes.png)

### Step 6: Connectivity restored
Gateway `10.0.1.1`: 10/10 replies, 0% loss, 1 ms average. Internet `8.8.8.8`: 30/30 replies, 0% loss, 44 ms average.

![Ping verification](images/06-ping-verification.png)

### Step 7: DNS tested in layers
The default lookup to the lab DNS server `10.0.1.2` timed out, although the server itself answered ping (4/4). A lookup through a public DNS server (`nslookup google.com 8.8.8.8`) returned an answer. This shows the network path is repaired and that the remaining problem is the **lab DNS server not answering client queries**, a separate issue.

![Layered DNS test](images/07a-dns-layered-test.png)

### Step 8: HTTPS port test
`Test-NetConnection 8.8.8.8 -Port 443` returns `TcpTestSucceeded : True`, sourced from `10.0.1.12` on `Ethernet 2`.

![HTTPS port test](images/07b-https-port-test.png)

## 5. Lessons learned

- Inspect the network configuration and the routing table before changing anything, and record the working values first so they can be restored.
- Test in layers: gateway, public IP, DNS, then the application port.
- Read ping results carefully. A "reply" from your own address saying *Destination host unreachable* is a failure, even though ping counts it as received.
- A stale **persistent** route survives reboots, so check the persistent routes list as well as the active one.
- A failing DNS server does not mean the network is down. Testing the same name through a public DNS server separates the two problems.
- Take a VM snapshot before deliberately breaking settings, and keep screenshots and command output as you work.

## 6. Skills demonstrated

- Windows network troubleshooting (`ipconfig`, `netsh`, `route`, `ping`, `nslookup`, `Test-NetConnection`)
- IPv4 addressing, default gateways and routing table analysis
- Static vs DHCP configuration and restoring a known-good configuration
- Layered diagnosis and verification
- Evidence-based technical documentation

## 7. Scope and limitations

- The original incident is documented from retained command output, not original screenshots. The images are a reproduction, as described in the evidence note.
- The lab DNS server (`10.0.1.2`) was not answering client queries during verification. That is a separate issue that was not investigated in this case study.
- All testing was performed on my own devices and lab network.
