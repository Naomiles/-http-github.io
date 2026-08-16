# AUTOSAR BSW Integration Guide
## A Step-by-Step Checklist for Automotive Display ECU Development

**Author:** Naomile  
**Version:** 1.0 | August 2026  
**Scope:** AUTOSAR 4.x Basic Software module integration for instrument cluster / display ECUs

---

## 1. Overview

This guide provides a practical integration checklist and reference workflow for integrating AUTOSAR Basic Software (BSW) modules into automotive ECU projects, specifically targeting instrument cluster and display applications. It draws from real-world production experience with VW, PSA, Chery and Isuzu programs.

---

## 2. Pre-Integration Checklist

### 2.1 Tool Chain Setup

- [ ] AUTOSAR configurator installed (ISOLAR, EB tresos, or DaVinci Configurator)
- [ ] Compiler toolchain configured (GHS Green Hills, TASKING, or GCC for ARM)
- [ ] ECU abstraction files from SoC vendor acquired (e.g., Infineon, NXP, Renesas)
- [ ] CAN/LIN/ETH network database (.dbc / .ldf / .arxml) received from OEM
- [ ] Flash tool and debugger JTAG connection verified

### 2.2 BSW Module List

| Module | Category | Required for Display ECU |
|--------|----------|--------------------------|
| WDGM   | Watchdog | Yes (ASIL-B+)           |
| NvM    | Memory   | Yes (persistent data)    |
| Fee    | Flash    | Yes (NvM backend)        |
| Fls    | Flash    | Yes (Flash driver)       |
| CSM    | Crypto   | Yes (Secure Boot)        |
| FuSa   | Safety   | Yes (ASIL rated)         |
| Com    | Comm     | Yes (CAN/LIN)           |
| PduR   | PDU Router | Yes                   |
| CanIf  | CAN      | Yes                     |
| Dcm    | Diag     | Yes (UDS)              |
| Dem    | Diag     | Yes (DTC mgmt)          |

---

## 3. Module Integration Steps

### 3.1 WDGM (Watchdog Manager)

```
1. Configure WDGM in arxml:
   - WdgMConfig.WdgMTrackedWatchdog.WdgMTriggerMode = TRIGGER_MODE_WINDOW
   - Set window timing based on main task period (typically 10ms or 20ms)

2. Add supervision entities:
   - WdgMSupervisedEntity → one per SWC that needs supervision
   - Link WdgMCheckpointRefs to SWC checkpoints

3. Integrate into Os_Application:
   - Call WdgM_MainFunction() in your 10ms task
   - Call WdgM_TriggerCheckpoint() at each SWC entry point

4. Common Pitfalls:
   - ❌ Forgetting to start the global watchdog (WdgM_StartWatchdog)
   - ❌ Trigger checkpoint called but entity not yet activated
   - ✅ Always call WdgM_ActivateSupervisedEntity() in SWC Init()
```

### 3.2 NvM (NVRAM Manager)

```
1. Define NvM Blocks:
   - NvMBlockDescriptor → one per data block
   - Set NvMNvBlockNeeds (ReadDataSize, WriteDataSize, etc.)
   - Configure CRC usage (CRC16 recommended for safety)

2. Configure Fee/Ea backend:
   - NvM → Fee (Flash) for persistent calibration/odometer data
   - NvM → Ea (EEPROM) for high-frequency small data

3. Runtime Usage:
   - NvM_ReadBlock()  → triggers async read, check status with NvM_GetErrorStatus()
   - NvM_WriteBlock() → triggers async write
   - NvM_WriteAll()   → write all dirty blocks (call before key-off)

4. Common Pitfalls:
   - ❌ Not calling NvM_Init() before any read/write
   - ❌ Blocking on NvM_ReadBlock (it's async!)
   - ✅ Use callback notification pattern with NvM_JobEndNotification
```

### 3.3 CSM (Crypto Stack Manager)

```
1. Configure CSM:
   - Define CsmJob → CsmAlgRef (AES-128, SHA-256, etc.)
   - Set key references from Key Manager (KeyM)

2. Secure Boot Integration:
   - Configure CSM for signature verification (RSA-2048 or ECDSA-P256)
   - Link CSM to bootloader verification chain
   - Verify root of trust is established in ROM bootloader

3. Runtime Pattern:
   Csm_VerifyStart() → Csm_VerifyUpdate() (loop) → Csm_VerifyFinish()
   // Poll Csm_GetJobStatus() for CSMAPIACCEPTED → CSMAPIFINISHED
```

---

## 4. Flash Bootloader (FBL) Integration

### 4.1 VW 80126/80128 Compliance

- [ ] Implement dual-bank flash architecture (Bank A / Bank B)
- [ ] Secure boot verification on every reset (CSM + OEM public key)
- [ ] UDS services: 0x27 (Security Access), 0x31 (Routine Control), 0x34/0x36/0x37 (Transfer)
- [ ] Support CAN and UDS diagnostic session management (0x10)

### 4.2 Dual-Bank Architecture

```
┌──────────────┐    ┌──────────────┐
│   Bank A     │    │   Bank B     │
│  (Active)    │    │  (Staging)   │
│  Application │    │  New Image   │
│  Image       │    │  (Flashing)  │
└──────────────┘    └──────────────┘
        │                   │
        ▼                   ▼
   Bootloader → Verify → Swap → Reset
```

### 4.3 UNECE R155/R156 Requirements

- Software version identification (UDS 0x1A / ReadDataByIdentifier)
- Software modification audit trail
- Verification that only authenticated software runs on the ECU

---

## 5. Integration Testing Checklist

| Test Item | Expected Result | Status |
|-----------|----------------|--------|
| WDGM triggers correctly | No watchdog resets in normal operation | [ ] |
| WDGM detects deadlock | ECU resets within safety timeout | [ ] |
| NvM read after power cycle | All persistent data retained | [ ] |
| NvM write + power cut | Data integrity (CRC valid) | [ ] |
| CSM verify valid image | Signature check passes | [ ] |
| CSM verify tampered image | Signature check fails, boot blocked | [ ] |
| FBL normal flash | New image boots successfully | [ ] |
| FBL interrupted flash | Rollback to previous image | [ ] |
| CAN message routing | All PDU routed correctly via CanIf/PduR | [ ] |
| UDS diagnostic session | Extended session + security access working | [ ] |

---

## 6. Debug Tips

1. **Watchdog Reset Loop**
   - Check if WdgM_StartWatchdog() is called too early
   - Verify all supervised entities are activated
   - Use debugger to trace WdgM state machine

2. **NvM Read Failure**
   - Ensure Fee/Ea drivers are initialized before NvM_Init()
   - Check flash sector is not write-protected
   - Verify NvM block descriptor matches runtime data size

3. **CSM Signature Fail**
   - Confirm public key is correctly flashed in secure storage
   - Check image hash computation matches signing tool configuration
   - Verify no padding/alignment issues in the image binary

---

## 7. Reference Documents

- AUTOSAR Classic Platform 4.x Specification (autosar.org)
- VW 80126 / VW 80128 (Volkswagen Flash Tool Specification)
- UNECE R155 / R156 (Cybersecurity & Software Update)
- ISO 26262 Part 6 (Software Level Development)

---

*© 2026 Naomile — Free for personal and educational use*
