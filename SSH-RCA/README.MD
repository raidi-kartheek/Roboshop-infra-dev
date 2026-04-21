# 📌 Root Cause Analysis (RCA)

## Incident: SSH & Jenkins Agent Connectivity Failure on RHEL 9 Production Server

- **Environment:** Production  
- **Severity:** High (Operational access lost)  
- **Impact Window:** <insert time window>  

---

## 1️⃣ Incident Summary

Production servers running RHEL 9 became inaccessible via SSH, and Jenkins agents lost connectivity immediately after a routine package installation executed via automation (user-data/bootstrap).

Although the AMI remained unchanged, newer packages were pulled from live repositories, causing a cryptographic library mismatch that broke OpenSSH functionality.

---

## 2️⃣ Customer / Business Impact

- ❌ SSH access to production instances was disrupted  
- ❌ Jenkins agents disconnected → CI/CD pipeline failures  
- ❌ Required manual intervention via console access  
- ⚠️ Increased operational risk during outage  

---

## 3️⃣ Timeline (High Level)

| Time        | Event |
|------------|------|
| T0         | Instance launched using existing AMI |
| T0 + X     | User-data executed `dnf module enable nodejs:20` |
| T0 + X     | NodeJS dependency upgraded OpenSSL |
| T0 + X     | OpenSSH remained on older version |
| T0 + X     | SSH started failing |
| T0 + Y     | Issue identified via logs |

---

## 4️⃣ Technical Root Cause

### 🔴 Primary Cause: OpenSSL & OpenSSH Version Mismatch



OpenSSL version mismatch
Built against: 30000070 (OpenSSL 3.0.7)
Running with: 30500010 (OpenSSL 3.5.1)



### What happened:

- NodeJS 20 installation upgraded OpenSSL → 3.5.1  
- OpenSSH was NOT upgraded  
- SSHD binary still linked to OpenSSL 3.0.7  
- Result → Runtime crypto failure  

---

## 5️⃣ Why This Happened

### Key Factors:

1. **Live Repository Drift**
   - NodeJS pulled EL9.7 packages
   - Earlier builds used EL9.3

2. **AMI Misconception**
   - AMI ≠ fixed runtime packages  
   - `dnf install` always pulls latest repo packages  

3. **Partial Upgrade Issue**
   - OpenSSL upgraded  
   - OpenSSH NOT upgraded  

4. **No Post-Validation**
   - `sshd -t` not executed  

---

## 6️⃣ Detection & Evidence

### Commands Used:

```bash
sshd -t
journalctl -u sshd

Error:

OpenSSL version mismatch. Built against 30000070, you have 30500010

7️⃣ Immediate Resolution
Fix Applied:

dnf update -y openssh openssh-server openssh-clients
sshd -t
systemctl restart sshd

✅ SSH restored successfully


8️⃣ Permanent Corrective Actions
1️⃣ Package Consistency Enforcement

dnf update -y openssl* openssh*
sshd -t

2️⃣ Immutable / Golden AMI Strategy
Pre-bake dependencies (NodeJS, Docker, Java)
Use Packer
Minimize user-data scripts

3️⃣ Fail-Fast Automation

set -euo pipefail
sshd -t || exit 1

4️⃣ Repository Control

Pin RHEL versions (e.g., EL9.3 / EL9.5)
Use repo snapshots / internal mirrors
Avoid auto upgrades in PROD

5️⃣ Break-Glass Access

Enable EC2 Serial Console
Enable AWS SSM Session Manager

9️⃣ Preventive Measures

Risk	Prevention
Crypto mismatch	Upgrade OpenSSH with OpenSSL
Repo drift	Pin repo versions
Bootstrap issues	Avoid heavy user-data
Silent failures	Add validation checks
Prod lockout	Enable break-glass access

🔟 Lessons Learned

AMI immutability ≠ runtime immutability
Crypto upgrades are high-risk
SSHD can appear "running" but be broken
sshd -t is mandatory validation
Avoid OS upgrades in user-data
Do not rely on live repos in production