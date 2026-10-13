# Lab 04 - Host Discovery and Port Scanning with Nmap

Academic lab (Ethical Hacking Lab, Sem III) performed on Windows with
**Nmap 7.991** against a self-owned lab target.

**Topics:** `nmap` · `port-scanning` · `host-discovery` · `network-mapping` · `cybersecurity` · `ethical-hacking`

## About
This lab covers host discovery, TCP port scanning, service/version detection, and
saving results with **Nmap**. I interpreted open and closed ports, identified the
services behind them, and assessed which exposures matter from a defensive point
of view.

## Objective
Discover active hosts, scan TCP ports, identify running services and versions,
and recommend basic hardening based on what was exposed.

## Environment
| Item | Detail |
|------|--------|
| Scanner | Nmap 7.991 on Windows 11 (Command Prompt) |
| Target | `192.168.56.1`, the host-only adapter of my own Windows machine (see [screenshot 03](screenshots/03-ipconfig-adapters.png)) |
| Scope | Only my own machine; no third-party systems were scanned |
| Run dates | 30-31 Aug 2026 (two sessions) |

> The target is the scanning machine's own host-only adapter. That is
> authorized and safe, but it is a single host, so this lab demonstrates
> scanning technique rather than mapping a multi-host network.

## Commands Used
| Purpose | Command | Screenshot |
|---------|---------|------------|
| Verify install | `nmap --version` | [02](screenshots/02-nmap-version.png) |
| Host discovery | `nmap -sn 192.168.56.1` | [04](screenshots/04-host-discovery-sn.png) |
| Default scan | `nmap 192.168.56.1` | [05](screenshots/05-basic-scan.png) |
| TCP connect scan | `nmap -sT 192.168.56.1` | [06](screenshots/06-tcp-connect-scan-sT.png) |
| Service detection | `nmap -sV 192.168.56.1` | [07](screenshots/07-service-detection-sV.png) |
| Specific ports | `nmap -p 22,80,443 192.168.56.1` | [08](screenshots/08-specific-ports-22-80-443.png) |
| Combined scan | `nmap -sT -sV 192.168.56.1` | [09](screenshots/09-combined-scan-sT-sV.png) |
| Save output | `nmap -sV 192.168.56.1 -oN nmap_result.txt` | [10](screenshots/10-save-results-oN.png) |

## Results

### Host discovery
| Host | Result |
|------|--------|
| `192.168.56.1` | Up |
| `192.168.56.18` | Down (no response to discovery probes) |
| `0.0.0.0` | Down (not a valid scan target; used to see Nmap's "host seems down" output) |

### Open TCP ports on 192.168.56.1
| Port | State | Service | Version / notes |
|------|-------|---------|-----------------|
| 135/tcp | open | msrpc | Microsoft Windows RPC |
| 139/tcp | open | netbios-ssn | Microsoft Windows netbios-ssn |
| 445/tcp | open | microsoft-ds | SMB (version not identified, shown as `microsoft-ds?`) |
| 902/tcp | open | ssl/vmware-auth | VMware Authentication Daemon 1.10 |
| 912/tcp | open | vmware-auth | VMware Authentication Daemon 1.0 |
| 5357/tcp | open | http (wsdapi) | Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP) |

The first scans (30 Aug) showed only 135, 139 and 445 as open. The later scans
(31 Aug) also showed 902, 912 and 5357, so the set of listening services on a
workstation changes between sessions. Nmap reports the state at the moment of
the scan.

### Closed TCP ports
22 (SSH), 80 (HTTP) and 443 (HTTPS) were closed, so no SSH or web server was
listening on this interface. The full scans reported 994 other closed ports.

## Findings

| # | Finding | Evidence | Why it matters | Recommended response |
|---|---------|----------|----------------|----------------------|
| 1 | **SMB/RPC/NetBIOS exposed** (135, 139, 445) | [05](screenshots/05-basic-scan.png), [09](screenshots/09-combined-scan-sT-sV.png) | These are common Windows lateral-movement and worm targets; they should not be reachable from untrusted networks. | Restrict with host firewall rules to trusted subnets; disable NetBIOS over TCP/IP if unused; ensure SMBv1 is disabled; keep Windows patched. |
| 2 | **VMware Authentication Daemon** (902, 912) | [09](screenshots/09-combined-scan-sT-sV.png), [10](screenshots/10-save-results-oN.png) | Expected when VMware Workstation is installed, but it adds management surface on the host-only adapter. | Keep it reachable only from the host-only network; stop the service when VMware is not in use. |
| 3 | **WSDAPI / HTTPAPI on 5357** | [10](screenshots/10-save-results-oN.png) | Windows device-discovery service; useful on home networks but unnecessary exposure on others. | Turn off network discovery on non-private networks or block the port in the firewall. |
| 4 | **No SSH or web service** (22, 80, 443 closed) | [08](screenshots/08-specific-ports-22-80-443.png) | Reduces attack surface. | Keep it that way unless a service is needed. |

## Observations worth noting
- **Different "closed" messages.** The default scan and `-sV` report closed
  ports as `reset`, while `-sT` reports `conn-refused`. With admin rights Nmap
  defaults to a SYN scan (raw packets); `-sT` uses the OS to complete a full
  connection. Same result, different technique.
- **Version detection is imperfect.** Port 445 stays `microsoft-ds?` and 5357
  appears as `wsdapi?` in one scan and as `http` in another, so `-sV` output
  needs cross-checking.
- **Scan time grows with `-sV`:** about 0.8 s for a plain scan versus 7-17 s
  with version detection.

## Limitations and next steps
- **UDP scan not completed.** I did not capture UDP results, so I make no claim about UDP ports. Planned command (needs an administrator prompt):
  `nmap -sU -p 53,67,69,123,161 192.168.56.1`
- Host discovery was run on single IPs, not a subnet sweep. Next step: `nmap -sn 192.168.56.0/24` against a lab network with several VMs.
- Planned extension: scan a Kali or other lab VM while capturing the traffic in Wireshark, to see the SYN/RST packets behind each scan result (see [Lab 05](../05-wireshark-traffic-analysis/)).

## What I learned
- The difference between host discovery, port scanning and service detection.
- How to read open, closed, and filtered states and what each implies.
- Why exposed Windows networking services (SMB, RPC, NetBIOS) are a hardening priority.
- Scan results are a snapshot, not a permanent truth.

## Ethics
Scanning was limited to my own machine. Port scanning systems without explicit
permission can be illegal. Findings were used only for learning and defensive
hardening, and no discovered service was exploited.
