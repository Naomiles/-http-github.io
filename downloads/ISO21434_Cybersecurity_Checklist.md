# ISO 21434 Cybersecurity Engineering Checklist
## TARA Template & Process Guide for Automotive ECUs

**Author:** Naomile  
**Version:** 1.0 | August 2026  
**Scope:** ISO/SAE 21434 cybersecurity engineering for display / instrument cluster ECUs

---

## 1. Overview

This document provides a practical cybersecurity engineering checklist aligned with ISO/SAE 21434 (Road Vehicles — Cybersecurity Engineering). It includes:

- A **TARA (Threat Analysis and Risk Assessment)** template
- A **threat modeling worksheet**
- A **secure boot verification guide**
- **Risk assessment matrices** and mitigation strategy templates

Based on hands-on experience with Renault SSP cybersecurity lead, TÜV Rheinland certified engineer practices, and VW ABT15 secure boot implementation.

---

## 2. ISO 21434 Process Phases Checklist

### Phase 1: Planning

- [ ] Cybersecurity plan document created and approved
- [ ] Cybersecurity case defined (scope, assumptions, out-of-scope items)
- [ ] Roles and responsibilities assigned (cybersecurity lead, assessor, developer)
- [ ] Item definition completed (system boundary, interfaces, data flows)
- [ ] Cybersecurity-relevant assets identified (safety-critical data, keys, firmware)

### Phase 2: Concept

- [ ] Threat scenarios identified (see Section 3 — TARA Template)
- [ ] Damage scenarios assessed (S, F, O severity per ISO 21434 Annex B)
- [ ] Attack feasibility rated (per ISO 21434 Annex D)
- [ ] Risk level determined → cybersecurity goals derived
- [ ] Cybersecurity goals approved by stakeholders

### Phase 3: Development

- [ ] Cybersecurity requirements allocated to HW/SW components
- [ ] Secure coding guidelines enforced (MISRA C:2012 + CWE/SANS Top 25)
- [ ] Cryptographic controls implemented (key management, secure storage)
- [ ] Authentication and access control mechanisms in place
- [ ] Secure boot chain implemented (see Section 5)

### Phase 4: Verification

- [ ] Static code analysis completed (Polyspace, Coverity, or QAC)
- [ ] Penetration testing performed (CAN fuzzing, JTAG access, flash extraction)
- [ ] Cybersecurity verification report written
- [ ] All cybersecurity claims traced to requirements
- [ ] Residual risk documented and accepted

### Phase 5: Production & Operation

- [ ] Key injection process secured (HSM-backed manufacturing)
- [ ] Incident response plan defined
- [ ] Vulnerability monitoring process in place
- [ ] OTA update mechanism secured (authenticated + encrypted)

---

## 3. TARA Template (Threat Analysis & Risk Assessment)

### 3.1 Asset Identification

| Asset ID | Asset Name | Type | Location |
|----------|-----------|------|----------|
| AS-001 | ECU Firmware Image | Software | Internal Flash |
| AS-002 | Secret Keys (AES/RSA) | Cryptographic | Secure Element / HSM |
| AS-003 | Odometer Data | Calibration | NvM Block (Flash) |
| AS-004 | CAN Message Payload | Communication | CAN Bus |
| AS-005 | Diagnostic Session | Service | UDS Protocol |

### 3.2 Threat Scenario Worksheet

```
Format:
  TS-[ID]: [Threat Scenario Description]
  Target Asset: AS-XXX
  Attack Vector: [Remote / Physical / Adjacent / Local]
  Attack Path: [Detailed steps an attacker would take]
```

#### Example Threat Scenarios for Display ECU:

**TS-001: Unauthorized Firmware Replacement**
- Target: AS-001 (ECU Firmware Image)
- Vector: Physical (OBD-II port)
- Path: Attacker connects laptop → enters extended diagnostic session → uploads modified firmware via UDS 0x34/0x36/0x37
- Impact: Malicious code execution on ECU

**TS-002: Secret Key Extraction**
- Target: AS-002 (Secret Keys)
- Vector: Physical (JTAG debug interface)
- Path: Attacker connects JTAG → reads flash memory → extracts key material from plaintext storage
- Impact: Cryptographic operations compromised

**TS-003: Odometer Tampering**
- Target: AS-003 (Odometer Data)
- Vector: Physical (OBD-II port)
- Path: Attacker sends crafted UDS WriteDataByIdentifier → overwrites odometer value in NvM
- Impact: Financial fraud, incorrect maintenance intervals

**TS-004: CAN Bus Message Injection**
- Target: AS-004 (CAN Message Payload)
- Vector: Adjacent (CAN bus access)
- Path: Attacker connects CAN sniffer/injector → sends spoofed speed/RPM messages
- Impact: False instrument readings, potential safety hazard

**TS-005: Unauthorized Diagnostic Session**
- Target: AS-005 (Diagnostic Session)
- Vector: Physical (OBD-II port)
- Path: Attacker bypasses security access seed/key algorithm → gains extended session → full ECU control
- Impact: Complete ECU compromise

---

## 4. Risk Assessment Matrix

### 4.1 Damage Severity Rating (per ISO 21434 Annex B)

| Rating | Safety (S) | Financial (F) | Operational (O) |
|--------|-----------|---------------|-----------------|
| Severe | S3: Life-threatening injuries | F3: Major financial loss | O3: Major operational impact |
| Moderate | S2: Severe injuries | F2: Moderate financial loss | O2: Moderate operational impact |
| Negligible | S1: Light injuries | F1: Minor financial loss | O1: Minor operational impact |

### 4.2 Attack Feasibility Rating (per ISO 21434 Annex D)

| Rating | Description | Elapsed Time | Expertise |
|--------|------------|-------------|-----------|
| Very High | Trivially exploitable | < 1 day | Layman |
| High | Easy to exploit | < 1 month | Proficient |
| Medium | Feasible with effort | < 6 months | Expert |
| Low | Difficult to exploit | < 1 year | Multiple experts |
| Very Low | Practically infeasible | > 1 year | Specialized team |

### 4.3 Risk Level Determination

| Feasibility ↓ \ Damage → | Negligible | Moderate | Severe |
|--------------------------|-----------|----------|--------|
| **Very High** | Medium | High | **Critical** |
| **High** | Low | Medium | High |
| **Medium** | Negligible | Low | Medium |
| **Low** | Negligible | Negligible | Low |
| **Very Low** | Negligible | Negligible | Negligible |

### 4.4 Risk Treatment for Display ECU TARA Results

| Threat Scenario | Severity | Feasibility | Risk Level | Treatment |
|----------------|----------|-------------|-----------|-----------|
| TS-001: Firmware Replace | Severe | High | **Critical** | Secure Boot + dual-bank |
| TS-002: Key Extraction | Severe | Medium | Medium | Secure element + JTAG lock |
| TS-003: Odometer Tamper | Moderate | High | Medium | CRC + signed write |
| TS-004: CAN Injection | Moderate | Very High | High | SecOC + message auth |
| TS-005: Diag Session Bypass | Severe | Medium | Medium | Strong seed/key (AES-128) |

---

## 5. Secure Boot Verification Guide

### 5.1 Architecture

```
ROM Bootloader (immutable)
    │
    ├── Verify Stage-1 Bootloader (RSA-2048 / ECDSA-P256)
    │       │
    │       └── Verify Application Image (RSA-2048 / ECDSA-P256)
    │               │
    │               └── Execute Application
    │
    └── FAIL → Halt / Enter recovery mode
```

### 5.2 Verification Steps (Infineon Traveo 2 Example)

```c
// Step 1: Initialize CSM
Csm_VerifyStart(jobId, &verifyConfig);

// Step 2: Feed image chunks
for (uint32 offset = 0; offset < imageSize; offset += CHUNK_SIZE) {
    Csm_VerifyUpdate(jobId, imageBase + offset, CHUNK_SIZE);
    while (Csm_GetJobStatus(jobId) != CSMAPIACCEPTED) {
        // Wait for processing
    }
}

// Step 3: Finalize
Csm_VerifyFinish(jobId);
Csm_VerificationStatusType result;
do {
    Csm_GetVerifyResult(jobId, &result);
} while (result.CsmJobStatus == CSMAPIACCEPTED);

if (result.CsmJobStatus == CSMAPIFINISHED) {
    // ✅ Signature valid — boot application
} else {
    // ❌ Signature invalid — halt or enter recovery
}
```

### 5.3 Secure Boot Checklist

- [ ] Root of trust stored in ROM (immutable)
- [ ] Public key provisioned during manufacturing (HSM-backed)
- [ ] Image signing tool integrated into CI/CD pipeline
- [ ] Hash algorithm: SHA-256 minimum
- [ ] Signature algorithm: RSA-2048 or ECDSA-P256
- [ ] Anti-rollback counter implemented (if applicable)
- [ ] Boot failure → recovery mode (not silent skip)
- [ ] Verification time within 500ms (typical target)

---

## 6. Mitigation Strategy Templates

### 6.1 Defense-in-Depth Layers

| Layer | Mechanism | Protects Against |
|-------|-----------|-----------------|
| Physical | JTAG lock, epoxy coating, secure element | Hardware attacks |
| Boot | Secure boot chain, anti-rollback | Firmware tampering |
| Communication | SecOC, TLS, MAC verification | Bus injection/replay |
| Diagnostic | Seed/key security access (0x27) | Unauthorized diagnostics |
| Data | NvM CRC, signed writes, encryption | Data tampering |
| Monitoring | WDGM supervision, anomaly detection | Runtime attacks |

### 6.2 Incident Response Plan (Template)

```
1. Detection
   - Automated: security monitoring alerts from fleet data
   - Manual: customer complaint, security researcher report

2. Triage (within 24h)
   - Assess severity using ISO 21434 damage scale
   - Determine affected vehicle population
   - Notify cybersecurity incident response team

3. Containment (within 72h)
   - Issue temporary workaround to affected vehicles
   - Disable vulnerable attack vector if possible
   - Preserve forensic evidence

4. Remediation
   - Develop and validate fix
   - Deploy via OTA update
   - Verify fix effectiveness in field

5. Post-Incident
   - Root cause analysis
   - Update TARA (new threat scenarios if needed)
   - Lessons learned report
   - Update cybersecurity case document
```

---

## 7. Reference Documents

- ISO/SAE 21434:2021 — Road Vehicles, Cybersecurity Engineering
- ISO 26262:2018 — Road Vehicles, Functional Safety
- UNECE R155 — Uniform Provisions on Cyber Security
- UNECE R156 — Uniform Provisions on Software Update
- BSI K-444 / K-445 — German Cybersecurity Standards
- TISAX VDA ISA — Information Security Assessment

---

*© 2026 Naomile — Free for personal and educational use*
