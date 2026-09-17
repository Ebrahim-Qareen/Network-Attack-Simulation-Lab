# Build Guide — Network Attack Simulation Lab

## pfSense

Three interfaces: **WAN** (NAT to the hypervisor host), **LAN** (victim net `10.10.20.0/24`), **OPT1** (attacker net `192.168.50.0/24`). The attacker sits on its own segment so all attack traffic crosses pfSense and is inspected.

Firewall rules: allow OPT1 → LAN only on the ports each scenario needs, log every rule. Everything else is denied and logged — the deny log is half your evidence.

## Snort (pfSense package)

Install `Services → Snort`. Enable it on **both** LAN and OPT1 interfaces (inspect the attack leaving the attacker net *and* arriving at the victim net).

- Rule sets: Snort VRT/GPLv2 + Emerging Threats Open.
- Preprocessors: keep `portscan` (sfPortscan) and the frag/stream reassembly on — several scenarios depend on them.
- Custom rules: drop `custom.rules` in the interface's rules directory, add it under `Custom rules`, and load suppressions from `suppress.conf`.
- IPS mode: run in IDS (alert-only) first to confirm signatures fire, then flip to IPS (block) per scenario so you can see the difference between "seen" and "stopped".

**Check:** `nmap -sS` from Kali → alerts appear under `Services → Snort → Alerts` on OPT1.

## Lab safety

- No attack is ever pointed at the WAN interface — it exists only for the victim VMs' outbound needs and is NAT to the host.
- Every VM has a clean snapshot taken before each scenario; DoS scenarios in particular leave the state table dirty.
- Attacker net is host-only. Nothing in this lab touches a real network.

## Reset between scenarios

```
1. Stop the attack.
2. Snort → Alerts → Clear.
3. pfSense → Diagnostics → States → Reset (after DoS scenarios).
4. Revert victim VM to snapshot if it was exploited.
```
