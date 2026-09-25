# Microsoft Purview Enterprise Implementation: Healthcare & Clinical Research Data Protection (PHI / IP)

## 📌 Executive Summary

This repository documents the end-to-end engineering and deployment of an enterprise data classification, prevention, and lifecycle management architecture inside **Microsoft Purview**.

Simulating an international biomedical research & clinical healthcare provider, this project establishes strong guardrails against exfiltration and non-compliance risks under **GDPR (General Data Protection Regulation)** and **HIPAA (Health Insurance Portability and Accountability Act)**. The tenant is secured across **Exchange Online, SharePoint Online, OneDrive for Business, and Managed Endpoints**.

---

## 🏛️ Logical Architecture

```
                                  [ Unstructured Data Ingestion ]
                                  (Exchange, SharePoint, OneDrive)
                                                  │
                                                  ▼
                         ┌──────────────────────────────────────────────────┐
                         │      Data Classification & SIT Pipeline         │
                         │  - Custom SIT: Medical Patient Record ID         │
                         │  - Native SIT: Romania Personal Numerical Code   │
                         └────────────────────────┬─────────────────────────┘
                                                  │
                      ┌───────────────────────────┴───────────────────────────┐
                      ▼                                                       ▼
        ┌───────────────────────────┐                           ┌───────────────────────────┐
        │    Sensitivity Labels     │                           │   Auto-Labeling Engine    │
        │ - Healthcare - PHI        │                           │ - Simulated evaluation    │
        │ - Medical Research - IP   │                           │ - SharePoint / OneDrive   │
        └─────────────┬─────────────┘                           └───────────────────────────┘
                      │
                      ▼
        ┌──────────────────────────────────────────────────────────────────┐
        │               Data Loss Prevention (DLP) Policies               │
        │  Rule: Block External Exfiltration of PHI (SIT / Label trigger)   │
        │  Actions: Restrict External Access | Real-time Policy Tips      │
        └─────────────────────────────┬────────────────────────────────────┘
                                      │
                                      ▼
        ┌──────────────────────────────────────────────────────────────────┐
        │             Data Lifecycle Management & Retention               │
        │  Policy: Healthcare PHI 10-Year Retention (3,650 days)           │
        │  Target: Tamper-proof retention for clinical records             │
        └──────────────────────────────────────────────────────────────────┘
```

---

## 🛠️ Components & Implementation Details

### 1. Classification & Sensitive Information Types (SIT)
* **Custom SIT:** `Medical Patient Record ID`
  * **Pattern:** Structured prefix `MED-` followed by clinical sequence identifiers.
  * **Confidence Level:** High Confidence based on contextual evidence keywords (*diagnostic, fisa, pacient, tratament*).
* **Native SIT:** `Romania Personal Numerical Code (CNP)`
  * Integrated for regional regulatory compliance under EU GDPR.

### 2. Information Protection & Sensitivity Labels
* **`Healthcare - PHI (Patient Data)`:** Applied to medical records, diagnostics, prescription data, and identifiable health summaries.
* **`Medical Research - IP`:** Applied to laboratory trials, pharmacological patents, and proprietary formulas.
* **Label Policy:** `Healthcare & Research Data Protection Policy` published tenant-wide to enforce manual classification with justification prompts.
* **Auto-Labeling:** Configured simulation policy `Auto-Label PHI Healthcare Records` targeting OneDrive and SharePoint scopes.

### 3. Data Loss Prevention (DLP)
* **Policy Name:** `Healthcare & Patient Data DLP Policy`
* **Locations:** Exchange Online, SharePoint Online, OneDrive for Business.
* **Detection Triggers:**
  * Document/Message contains `Medical Patient Record ID` OR Sensitivity Label `Healthcare - PHI (Patient Data)`.
  * Condition: `Content is shared from Microsoft 365 with people outside my organization`.
* **Enforcement:**
  * Immediate external access revocation / blocked outbound delivery.
  * Native in-context **Policy Tip** alerting users of sensitive data violations.
  * High-severity alert generation for SecOps/SOC ingestion.

### 4. Data Lifecycle Management (DLM)
* **Policy Name:** `Healthcare PHI 10-Year Retention`
* **Coverage:** Exchange Mailboxes, SharePoint Sites, OneDrive Accounts.
* **Retention Window:** **10 Years (3,650 Days)** calculated from item creation date.
* **Disposal Action:** Strict retention hold (`Keep content for 10 years`), preventing premature deletion by standard operators.

---

## 🧪 Verification & Proof of Concept (PoC)

### Test Case: Unauthorized External Exfiltration Attempt

A synthetic medical record (`Fisa_Medicala_Test.txt`) was generated containing clinical cardiological diagnostic details and patient identifier `MED-481920`.

```text
=============================================================
SPITALUL CLINIC UNIVERSITAR - SECTIA CARDIOLOGIE
DOSAR MEDICAL CONFIDENTIAL

Nume Pacient: Popescu Ion
Identificator Fisa: MED-481920
Diagnostic: Insuficienta cardiaca clasa II NYHA, HTA stadiul 2
Tratament: Ramipril 5mg, Bisoprolol 2.5mg
Observatii clinice: Pacientul a fost inclus in protocolul de cercetare clinica.
=============================================================
```

#### Verification Results:
1. **Sharing Attempt:** The file was uploaded to OneDrive and an external invitation was initiated toward a non-corporate recipient (`*@yahoo.com`).
2. **Preventive Interception:** Purview Cloud DLP immediately intercepted the API call in real time.
3. **User Experience:**
   * Recipient address flagged with an exfiltration warning.
   * Send button disabled (`Grayed Out`).
   * Explicit Policy Tip served:
     > *"This item contains sensitive information. It can't be shared with people outside your organization."*

*(Add your captured screenshots in the `/docs/screenshots` folder)*:
* `docs/screenshots/01_onedrive_dlp_block.png` (Direct external block validation)
* `docs/screenshots/02_retention_policy_active.png` (10-year retention rule confirmation)
* `docs/screenshots/03_activity_explorer_events.png` (Audit logging confirmation)

---

## 🔧 Engineering Challenges & Troubleshooting Runbook

| Challenge Encountered | Root Cause | Engineering Resolution |
| :--- | :--- | :--- |
| **`UnifiedAuditLogDisabledException` on Auto-labeling** | Cloud latency between the Unified Audit Log (UAL) service activation and the simulation policy deployment engine. | Validated tenant audit status via `Audit` module and proceeded with deterministic DLP controls while background indexing synchronized. |
| **`RetentionType` Parameter Error in PowerShell** | Parameter deprecation within the modern Exchange Online / Compliance REST API module. | Stripped legacy parameters and provisioned declarative compliance rules via native cmdlets and portal orchestration. |
| **Exchange `550 5.7.708` SMTP Rejection** | Anti-abuse and anti-spam controls enforced by Microsoft on fresh developer tenant outbound relay IPs. | Validated policy enforcement directly through OneDrive/SharePoint sharing mechanisms, demonstrating protocol-agnostic DLP capabilities. |

---

## 💻 PowerShell Deployment Scripts

### Connect to Management Sessions
```powershell
# Authenticate to Purview Security & Compliance session
Connect-IPPSSession

# Authenticate to Exchange Online v3 REST module
Connect-ExchangeOnline
```

### Deploy Retention Architecture
```powershell
# Create global 10-year retention policy
New-RetentionCompliancePolicy `
    -Name "Healthcare PHI 10-Year Retention" `
    -SharePointLocation All `
    -OneDriveLocation All `
    -ExchangeLocation All

# Bind retention duration (3650 days) with preservation hold
New-RetentionComplianceRule `
    -Name "Retain PHI Records 10 Years" `
    -Policy "Healthcare PHI 10-Year Retention" `
    -RetentionDuration 3650 `
    -RetentionComplianceAction Keep
```

---

## 🎯 Key Skills Demonstrated

* **Security & Compliance Architecture:** Enterprise information protection framework design.
* **Data Loss Prevention (DLP):** Cross-workload policy authoring, Policy Tips, and external boundary isolation.
* **Regulatory Alignment:** Mapping GDPR and HIPAA mandates to concrete cloud security controls.
* **Modern Administration:** Hybrid implementation using PowerShell (REST API) and Purview Compliance Portal.
* **Incident Triage & Validation:** PoC execution, audit log analysis, and defense validation.
