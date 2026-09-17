# Network Attack Simulation Lab

![pfSense](https://img.shields.io/badge/pfSense-firewall-1F2937?style=flat-square)
![Snort](https://img.shields.io/badge/Snort-IDS%2FIPS-EF4444?style=flat-square)
![Kali](https://img.shields.io/badge/Kali-attacker-2563EB?style=flat-square)
![ATT&CK](https://img.shields.io/badge/MITRE-ATT%26CK%20mapped-6B7280?style=flat-square)

A segmented network lab where every attack is run on purpose and every alert is explained. pfSense sits between an attacker network and a victim network, Snort inspects both interfaces, and each scenario ends with the packet capture, the Snort alert, and what a network analyst should conclude from it.

## Topology

```
   ATTACKER NET 192.168.50.0/24          VICTIM NET 10.10.20.0/24
   ┌────────────────────┐                ┌────────────────────┐
   │ Kali 192.168.50.10 │                │ Windows 10 client  │
   └─────────┬──────────┘                │ Ubuntu web/DNS     │
             │                           │ Metasploitable 2   │
             │        ┌──────────┐       └─────────┬──────────┘
             └────────┤  pfSense ├─────────────────┘
                      │ Snort on │
                      │ LAN+OPT1 │──── WAN (NAT to host)
                      └──────────┘
```

| Component | Role |
|---|---|
| pfSense | Routing, firewall rules, NAT, Snort package host |
| Snort (pfSense package) | IDS on both internal interfaces; IPS (block offenders) enabled per scenario |
| Kali Linux | Attacker — Nmap, hping3, Ettercap/arpspoof, dnsmasq/Responder, msfvenom |
| Windows 10 / Ubuntu / Metasploitable 2 | Victims and services |

## Scenarios

| # | Scenario | Tool | ATT&CK | What Snort sees | What the firewall shows |
|---|---|---|---|---|---|
| 1 | Reconnaissance — SYN, UDP, version scan | `nmap` | T1046 | ET SCAN Nmap signatures, portscan preprocessor | Burst of denied states on closed ports |
| 2 | SYN flood DoS on web server | `hping3 -S --flood` | T1499.001 | `sfportscan` / rate-based DoS rules | State table growth, SYN-proxy behaviour |
| 3 | ICMP / UDP flood | `hping3`, `nping` | T1498.001 | ICMP flood signatures | Rate limit and block on WAN/LAN |
| 4 | ARP spoofing / MITM | `arpspoof`, `ettercap` | T1557.002 | ARP preprocessor anomalies, duplicate IP→MAC | Gratuitous ARP visible in packet capture |
| 5 | DNS spoofing / rogue DNS | `ettercap dns_spoof`, `dnsmasq` | T1557 · T1071.004 | DNS response anomalies, unexpected resolver | Clients querying non-authorised resolver |
| 6 | DNS tunnelling | `iodine` / `dnscat2` | T1071.004 · T1048.003 | Long TXT/NULL queries, high query rate to one domain | Abnormal DNS volume per host |
| 7 | Malware C2 beacon | `msfvenom` reverse TCP/HTTPS + Metasploit handler | T1071.001 · T1571 | ET MALWARE / Metasploit payload rules | Periodic outbound connections on odd ports |
| 8 | Brute force (SSH / FTP) | `hydra` | T1110 | ET SCAN / policy brute-force rules | Repeated connections, IPS block |

Each scenario is documented in [`scenarios/README.md`](scenarios/README.md) with: objective · command run · Snort alert(s) · pfSense view · analyst conclusion · false-positive notes · rule tuning.

## Repository layout

```
.
├── README.md
├── docs/
│   └── build-guide.md        # pfSense interfaces/rules/NAT, Snort setup, IPS mode, lab safety
├── scenarios/
│   └── README.md             # the 8 scenarios: command, alert, firewall view, analyst conclusion
└── snort/
    ├── custom.rules          # local rules written for this lab (SID 1000001+)
    └── suppress.conf         # documented suppressions
```

## Custom Snort rules (excerpt)

```
# SYN flood — more than 100 SYNs to one host in 10s
alert tcp any any -> $HOME_NET any (msg:"LAB DOS SYN flood"; flags:S; threshold:type both, track by_dst, count 100, seconds 10; classtype:attempted-dos; sid:1000001; rev:1;)

# DNS tunnelling — oversized TXT query
alert udp $HOME_NET any -> any 53 (msg:"LAB DNS possible tunnelling - long TXT query"; content:"|00 10 00 01|"; dsize:>150; classtype:policy-violation; sid:1000002; rev:1;)

# Metasploit reverse TCP default handler port
alert tcp $HOME_NET any -> $EXTERNAL_NET 4444 (msg:"LAB MALWARE outbound to 4444 - possible reverse shell"; flags:S; classtype:trojan-activity; sid:1000003; rev:1;)
```

## Lab safety

Attacks stay on isolated host-only networks. The WAN interface is NAT to the hypervisor host only and no attack is ever pointed at it. Every VM is snapshotted before each scenario.

## Author

Ebrahim Mohamed — SOC Analyst / Detection Engineer / Cybersecurity Instructor — [LinkedIn](https://linkedin.com/in/EbrahimMohamed)
