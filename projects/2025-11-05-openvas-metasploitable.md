---
date: 2025-11-05
layout: page
---

[← Back to Projects](/projects/)

# Lab: Kali (OpenVAS) → Metasploitable - Multi-vulnerability Remediation

**Author:** Tung Nguyen  
**Date:** 2025-11-05  
**Environment:** Kali attacker VM + Metasploitable target VM (host-only network)

---

## Objective
Perform discovery and vulnerability remediation on a Metasploitable VM using Greenbone/OpenVAS (GVM). Document each finding, investigation steps, remediation, verification, and artifacts so the work is reproducible and reviewable.

---

## Environment & VM configuration
- Hypervisor: VirtualBox / VMware  
- Kali VM: 6 GB RAM, 2 CPU, host-only network  
- Metasploitable VM: 1 GB RAM, 1 CPU, host-only network  
- Tools used: Greenbone/OpenVAS (GVM), nmap, ss/netstat, bash, basic Linux utilities  

---

## Workflow Summary
1. Boot Kali and Metasploitable VMs on an isolated host-only network.  
2. Confirm target IP with host discovery (`nmap`).  
3. Start GVM on Kali (`gvm-start`) and run a target scan via the GSA web UI.  
4. Review scan results; triage vulnerabilities.  
5. Inspect target VM configuration / services.  
6. Remediate each vulnerability in turn.  
7. Re-scan with OpenVAS and verify remediation.  

---

## Table of Contents
- [Vulnerability: Ingreslock Backdoor (Port 1524)](#vulnerability-ingreslock-backdoor-port-1524)  
- [Vulnerability: Rexec / r-services (Port 512)](#vulnerability-rexec-r-services-port-512)  
- [Vulnerability: MySQL / MariaDB Default Credentials (Port 3306)](#vulnerability-mysql-mariadb-default-credentials-port-3306)

<!-- ---

## Vulnerability Template
Use this template to add additional vulnerabilities:

### Vulnerability: <SHORT TITLE> — <SERVICE / PORT>
**Severity:** <High / Medium / Low>  
**OpenVAS ID / Reference:** <OpenVAS NVT or CVE>  

**Description (short):**  
A one-sentence summary of the issue.

**Evidence (pre-remediation):**
- GVM finding screenshot: `/images/<filename>`  
- `nmap` or port scan output:
# Example
nmap -sT -p 1-65535 <target-ip> -->

---

### Vulnerability: Ingreslock Backdoor (Port 1524) {#vulnerability-ingreslock-backdoor-port-1524}
**Severity:** High  
**OpenVAS ID / Reference:** NVT – *TCP Port 1524: Ingreslock Backdoor*

**Description (short):**  
The `inetd` configuration contained an entry that spawned `/bin/bash` as root when the service was contacted. This effectively functions as a backdoor, allowing anyone who can connect to that port to obtain a root shell.

**Evidence (pre-remediation):**
- OpenVAS finding screenshot:  
  ![OpenVAS finding](../images/IngresLock-OpenVAS.png)
- `nmap` scan showing port 1524 open (before remediation):
  ![Port 1524 open before remediation](../images/ingreslock-scan-test.png)
- Snippet of `/etc/inetd.conf` showing the malicious entry:
  ![inetd.conf with malicious entry](../images/Inetd.conf-file.png)

**Root cause analysis:**  
A line in `/etc/inetd.conf` mapped the `ingreslock` service to `/bin/bash` and ran it as `root`. Since `inetd` launches the configured program with root privileges, any network connection to that service resulted in a root shell being spawned.

**Remediation performed:**  
1. Removed the malicious `ingreslock` line (`ingreslock stream tcp nowait root /bin/bash bash -i`) from `/etc/inetd.conf`.
2. Reboot to apply the change.
3. Scan again using nmap to make sure the port is closed and unable to netcat to it anymore.

---

### Vulnerability: Rexec / r‑services (Port 512) {#vulnerability-rexec-r-services-port-512}

**Severity:** High  
**OpenVAS ID / Reference:** NVT – *TCP Port 512: rexec (r‑services)*

**Description (short):**  
The target exposes the legacy `rexec` (r‑services) daemon on TCP port 512. R‑services transmit credentials in plaintext and are considered insecure; an exposed `rexec` allows remote command execution and is a significant misconfiguration.

**Evidence (pre-remediation):**
- OpenVAS finding screenshot:  
  ![OpenVAS finding - rexec](/images/rexec-openvas.png)
- `nmap` scan showing port 512 open (before remediation):  
  ![nmap port 512 before remediation](/images/rexec-nmap-scan.png)
- `inetd` configuration showing the `rexec` entry (example):  
  ![inetd.conf with rexec entry](/images/rexec-inetd.conf-file.png)  
  _Observed line (lab):_  
exec stream tcp nowait root /usr/sbin/tcpd /usr/sbin/in.rexecd

**Root cause analysis:**  
The `rexec` service was enabled via the system's `inetd` configuration. Because rexec performs plaintext authentication and is rarely required, having it enabled on a networked host exposes credentials and allows remote execution.

**Remediation performed:**  
1. Removed/disabled the `rexec` entry (`exec stream tcp nowait root /usr/sbin/tcpd /usr/sbin/in.rexecd`) from the inetd configuration.
2. Reboot to apply the change.
3. Scan again using nmap to make sure the port is closed.

---

### Vulnerability: MySQL / MariaDB Default Credentials (Port 3306) {#vulnerability-mysql-mariadb-default-credentials-port-3306}

**Severity:** High  
**OpenVAS ID / Reference:** NVT – *MySQL / MariaDB Default Credentials (MySQL Protocol)*

**Description (short):**  
The MySQL server allowed authentication as `root` with an empty password, permitting administrative access to the database service.

**Evidence (pre-remediation):**
- OpenVAS finding screenshot:  
  ![OpenVAS MySQL finding](../images/mysql-openvas.png)
- Lab test: successful login as `root` with no password (before remediation):  
  ![Successful login without password](../images/mysql-test-login.png)

**Root cause analysis:**  
The MySQL instance was left with a blank/weak root password and accepted remote connections. This allowed unauthenticated administrative access. On many intentionally vulnerable images (like Metasploitable) default credentials are present for testing, but on real systems this represents a critical misconfiguration.

**Remediation performed:**  
1. Logged into the MySQL server locally (lab) and set a strong root password. 
2. Test login again without password -> Failed
![Change password and test login without it](../images/mysql-change-password.png)
3. Test login again with password -> Success
![Successful login with the correct password](../images/mysql-test-login.png)
