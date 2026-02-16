# 08 — Case Study: Phishing to Lateral Movement

## 🧠 The Attack Path
**Scenario:** User clicks link -> Attacker steals Session Cookie -> Moves to Admin Share.

```mermaid
graph LR
    A[Phish: AiTM] -->|Steals Cookie| B(MFA Bypass)
    B -->|Logon Type 3| C{Lateral Mov}
    C -->|SMB / WinRM| D[Admin Share Access]
    style B fill:#f96,stroke:#333,stroke-width:2px
    style D fill:#f00,stroke:#333,stroke-width:2px

```

## 1. The Requirement (PIR)

* **Question:** Do we have visibility into lateral movement *after* a potential identity compromise?
* **Gap:** We see the login, but do we see the pivot?

## 2. Hypothesis & Mapping

* **Technique:** [T1021.002 (SMB/Windows Admin Shares)](https://attack.mitre.org/techniques/T1021/002/)
* **Logic:** If `User_A` logs in from `New_IP` AND immediately connects to `Server_B` via SMB, alert.

## 3. The Logic (Pseudo-Code)

*This demonstrates how I translated the Intel into a correlation rule (Sigma/SIEM agnostic):*

```sql
rule Detect_Lateral_Movement_After_Logon {
  meta:
    description = "Detects SMB connection shortly after a remote logon from the same user"
    severity = "High"
    author = "Pratik K."
  events:
    $logon = EventCode=4624 AND LogonType=3 // Network Logon
    $share = EventCode=5140 // Network Share Object Accessed (C$, ADMIN$)
  condition:
    $logon.TargetUser == $share.SubjectUser AND
    NOT in_subnets($logon.IpAddress, Trusted_Subnets) AND
    time_diff($logon.time, $share.time) < 5 minutes
}

```

## 4. Reality Check (Tuning)

* **False Positive:** Initially, this triggered on the **SCCM Service Account** pushing patches (it behaves exactly like an attacker).
* **The Fix:** I added a whitelist for the specific `Trusted_Subnets` (IT Management VLAN) to reduce noise.
* **Result:** High-fidelity alerts only.

## 5. The Outcome

* **Validation:** Ran `Invoke-AtomicTest T1021.002` to simulate the traffic.
* **Result:** Detection logic triggered successfully in the test environment.
* **Metric:** Reduced detection time for pivoting from **4 hours** (manual log review) to **~10 minutes** (automated alert).

---

### ⚠️ Anti-Pattern → 🔧 The Fix

**Anti-Pattern:** Obsessing over the "Patient Zero" (the click) and ignoring the spread.
**The Fix:** **Assume Breach**. Build detections for the *next* step (Lateral Movement).
**Evidence:** Correlation rules linking Identity logs (Okta/Entra) with Endpoint logs (CrowdStrike/Sysmon).
