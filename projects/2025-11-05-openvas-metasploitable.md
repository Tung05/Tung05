---
title: "OpenVAS scan & Ingreslock remediation"
date: 2025-11-05
---

# Lab: Kali (OpenVAS) → Metasploitable scan & remediation

**Author:** Tung Nguyen  
**Date:** 2025-11-05  
**Environment:** Kali attacker VM + Metasploitable target VM (host-only network)

---

## Objective
Scan a vulnerable Metasploitable VM from a Kali attacker VM using Greenbone/OpenVAS (GVM) and remediate a discovered backdoor (Ingreslock on port 1524). Document steps and verification.

---

## Environment & VM configuration
- Host hypervisor: VirtualBox or VMware
- Kali VM: 6 GB RAM, 2 CPU, host-only network
- Metasploitable VM: 1 GB RAM, 1 CPU, host-only network
- Tools used: GVM (Greenbone/OpenVAS), nmap, ss/netstat, basic shell commands

> **Safety note:** All testing was performed on isolated VMs. Do not perform scans or exploits on machines you do not own or have explicit written permission to test.

---

## Workflow summary
1. Boot Kali and Metasploitable VMs on an isolated host-only network.  
2. Confirm target IP with host discovery (`nmap` or similar).  
3. Start GVM on Kali (`gvm-start`) and run a target scan via the GSA web UI.  
4. Review scan results; identify the Ingreslock backdoor on TCP port 1524.  
5. Inspect `/etc/inetd.conf` on Metasploitable and remove malicious line.  
6. Restart inetd/xinetd and/or reboot the VM.  
7. Re-scan with OpenVAS and confirm remediation.

---

## What the scan found
- **Finding:** Port 1524 — Ingreslock Backdoor  
- **Description (sanitized):** `inetd`/`xinetd` configuration contained a service mapping that would spawn an interactive shell as root when the service was contacted. This is a local daemon configuration backdoor that grants remote root shell access.

---

## Investigation & verification commands (safe / non-exploitive)
- Verify open ports (example):
```bash
nmap -sT -p 1-65535 <target-ip>
