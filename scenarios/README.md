# Scenarios

Eight attacks, each run from Kali on the attacker net against a victim on the LAN. Every one: command → what Snort sees → what pfSense shows → what the analyst concludes. Rule SIDs match `snort/custom.rules`.

## 1. Reconnaissance — T1046
`nmap -sS -sV 10.10.20.0/24`
Snort: ET SCAN Nmap signatures + sfPortscan preprocessor. pfSense: a burst of denied states on closed ports from one source. **Conclusion:** scan in progress; note the source and whether it moves on to specific ports (targeting) vs sweeping (discovery). Not itself an incident, but the precursor to one.

## 2. SYN flood — T1499.001
`hping3 -S -p 80 --flood 10.10.20.20`
Snort: SID `1000001` (100+ SYN to one host in 10s). pfSense: state table climbs fast, SYN-proxy kicks in if enabled. **Conclusion:** availability attack on the web server; confirm it's spoofed vs real source before blocking (blocking a spoofed source does nothing).

## 3. ICMP / UDP flood — T1498.001
`hping3 --icmp --flood 10.10.20.20`
Snort: SID `1000002` (200+ echo in 10s). pfSense: rate-limit and block. **Conclusion:** volumetric DoS; the value here is confirming your rate limits actually engage.

## 4. ARP spoofing / MITM — T1557.002
`arpspoof -i eth0 -t 10.10.20.20 10.10.20.1` (+ ettercap)
Snort: ARP preprocessor flags duplicate IP→MAC. **Conclusion:** someone on the LAN is intercepting traffic; ARP is unauthenticated so the *duplicate mapping* is the only signal — treat any duplicate as hostile in this lab.

## 5. DNS spoofing / rogue resolver — T1557 · T1071.004
`ettercap` dns_spoof, or a rogue `dnsmasq`
Snort: SID `1000005` — DNS query to a non-authorised resolver. **Conclusion:** clients are being answered by something other than your DNS; check which host is replying and whether responses were poisoned.

## 6. DNS tunnelling — T1071.004 · T1048.003
`iodine` / `dnscat2`
Snort: SID `1000003`/`1000004` — oversized TXT and NULL-record queries, high query rate to one domain. **Conclusion:** likely data exfiltration or C2 over DNS; the tell is *volume and record type to a single domain*, not any one query.

## 7. Malware C2 beacon — T1071.001 · T1571
`msfvenom` reverse TCP + Metasploit multi/handler
Snort: SID `1000006` — outbound to 4444, plus ET MALWARE payload rules. pfSense: periodic outbound on an odd port. **Conclusion:** an endpoint is beaconing out; the periodicity (fixed interval) distinguishes C2 from normal traffic. Isolate the source host.

## 8. Brute force (SSH/FTP) — T1110
`hydra -l admin -P rockyou.txt ssh://10.10.20.30`
Snort: SID `1000007` — 10+ connections to port 22 from one source in 60s. **Conclusion:** credential attack; check for a subsequent *successful* auth, which changes this from noise to an incident.
