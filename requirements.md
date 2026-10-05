---
title: "System Requirements Specification - RMG-100 Remote Monitoring Gateway"
doc_id: SRS-RMG-100
status: "Draft 1.0 - external review"
owner: Systems Engineering
---

# System Requirements Specification — RMG-100 Remote Monitoring Gateway

## 1. Introduction

### 1.1 Purpose

This document specifies the functional and non-functional requirements for the RMG-100 Remote Monitoring Gateway ("the gateway"), a device that connects rooftop HVAC units (RTUs) to the FleetCloud telemetry platform. It is the source of record for requirements that flow into Azure DevOps work items and test plans.

### 1.2 Scope

The requirements cover gateway firmware and its connectivity behaviour. Installation, commissioning, and field service are out of scope and are specified in the Installation Manual [1].

### 1.3 Intended audience

Systems engineering, firmware development, QA, and product management. External stakeholders take part in scheduled reviews of this document; they comment but are not authors of record.

### 1.4 Definitions and acronyms

| Term | Definition |
| --- | --- |
| RTU | Rooftop unit (HVAC equipment) |
| RMG-100 | Remote Monitoring Gateway, hardware revision C |
| FleetCloud | The vendor telemetry and fleet-management platform |
| OTA | Over-the-air firmware update |
| WAN | Wide area network |
| Alarm | A condition requiring operator attention, raised by the gateway |
| Heartbeat | A periodic status message sent to FleetCloud to signal that the gateway is alive |

### 1.5 References

1. RMG-100 Installation Manual, IM-4412, Rev C
2. FleetCloud Platform Integration Specification, FCS-INT-017
3. RFC 2119 — [Key words for use in RFCs to Indicate Requirement Levels](https://www.rfc-editor.org/rfc/rfc2119)

## 2. System overview

The RMG-100 connects to up to four RTUs over Modbus RS-485, samples operating data, and forwards telemetry to FleetCloud over the customer's network. Day-to-day configuration is done through the on-device dashboard or the FleetCloud API.

> **Constraint C-1:** The gateway ships into customer networks that the vendor does not control. Connectivity requirements must tolerate customer firewalls, captive portals, and TLS-terminating proxies unless a requirement explicitly states otherwise.

## 3. Functional requirements

The keywords **shall**, **should**, and **may** are used as defined in RFC 2119 [3].

### 3.1 Connectivity

- **REQ-CON-001.** The gateway shall connect to FleetCloud over Wi-Fi (IEEE 802.11 b/g/n, 2.4 GHz, WPA2) or 10/100 Ethernet.
  - *Acceptance:* Verified by test T-CON-001. The gateway joins a WPA2 network and completes registration with FleetCloud.
- **REQ-CON-002.** If both Wi-Fi and Ethernet are available, the gateway shall prefer Ethernet and shall establish a Wi-Fi session within 60 seconds of losing the Ethernet link.
  - *Acceptance:* Verified by test T-CON-002. Unplugging Ethernet results in an active Wi-Fi session within 60 s.
- **REQ-CON-003.** The gateway shall establish a connection to FleetCloud within 15 seconds of power-up.
  - *Acceptance:* Verified by test T-CON-003. Median of 20 power-up cycles.
- **REQ-CON-004.** While connected, the gateway shall send a heartbeat status message to FleetCloud every 30 seconds.
  - *Acceptance:* Verified by test T-CON-004. Interval measured over a 1-hour soak.

### 3.2 Telemetry and data retention

- **REQ-DAT-001.** The gateway shall publish telemetry for each connected RTU every 60 seconds.
  - *Acceptance:* Verified by test T-DAT-001. Publishing interval measured over a 1-hour soak.
- **REQ-DAT-002.** The gateway shall retain telemetry locally for at least 30 days while disconnected from FleetCloud.
  - *Acceptance:* Verified by test T-DAT-002. Retention window measured at maximum write rate.
- **REQ-DAT-003.** Telemetry messages shall use the following JSON payload schema:

  ```json
  {
    "ts": "2026-10-02T14:03:11Z",
    "rtu": 2,
    "measurements": {
      "supply_air_temp_c": 12.4,
      "return_air_temp_c": 21.1,
      "supply_current_a": 8.7,
      "operating_mode": "cooling"
    }
  }
  ```

- **REQ-DAT-004.** After a disconnection, the gateway shall forward all retained telemetry to FleetCloud once connectivity is restored (store-and-forward).
  - *Acceptance:* Verified by test T-DAT-004. No gaps in FleetCloud data after a simulated 24-hour outage.

### 3.3 Alarms

- **REQ-ALM-001.** The gateway shall raise an alarm when a connected RTU reports a supply current above the configured threshold for longer than the configured dwell time.
  - *Acceptance:* Verified by test T-ALM-001.
- **REQ-ALM-002.** The gateway shall send alarms to FleetCloud promptly.
  - *Acceptance:* Verified by inspection.
- **REQ-ALM-003.** An alarm shall be cleared only after the gateway recieves an acknowledgment from FleetCloud.
  - *Acceptance:* Verified by test T-ALM-003.

### 3.4 Dashboard and API

- **REQ-API-001.** The gateway shall expose a REST API over HTTPS for configuration and status.
  - *Acceptance:* Verified by test T-API-001.
- **REQ-API-002.** The API shall allow an authorized user to read current telemetry and change alarm thresholds.
  - *Acceptance:* Verified by test T-API-002.
- **REQ-API-003.** The API shall be versioned (`/v1`) and shall maintain backward compatibility within a major version.
  - *Acceptance:* Verified by inspection.
- **REQ-API-004.** The on-device dashboard shall be intuitive and user-friendly.
  - *Acceptance:* Verified by demonstration D-API-004.

### 3.5 Over-the-air updates

- **REQ-OTA-001.** The gateway must accept only firmware images signed with the vendor signing key.
  - *Acceptance:* Verified by test T-OTA-001.
- **REQ-OTA-002.** An OTA update shall complete without disrupting normal monitoring operation.
  - *Acceptance:* TBD.
- **REQ-OTA-003.** If an installed update fails validation at boot, the gateway shall roll back to the previous firmware image within five minutes.
  - *Acceptance:* Verified by test T-OTA-003.

## 4. Non-functional requirements

### 4.1 Performance

- **NFR-PERF-001.** End-to-end latency from RTU sample to FleetCloud receipt shall not exceed 5 seconds under normal operating conditions.
- **NFR-PERF-002.** The gateway shall support 50 concurrent API sessions.
- **NFR-PERF-003.** The gateway shall synchronize its clock against NTP and shall keep drift within 2 seconds.

### 4.2 Security

- **NFR-SEC-001.** All cloud connections shall use TLS 1.3.
- **NFR-SEC-002.** Third-party integrations accessing the FleetCloud API shall be accepted with TLS 1.2 or higher.
- **NFR-SEC-003.** The gateway shall pin the FleetCloud server certificate.
- **NFR-SEC-004.** The gateway shall verify firmware image signatures before boot (secure boot).

### 4.3 Environmental and reliability

The gateway shall tolerate the enviromental conditions below.

- **NFR-REL-001.** The gateway shall operate continuously in ambient temperatures from -30 °C to +60 °C.
- **NFR-REL-002.** The gateway shall have a demonstrated MTBF of at least 100,000 hours.
- **NFR-REL-003.** The gateway shall lose no telemetry data during a WAN outage of up to 90 days.

## 5. Verification methods

| Requirement group | Verification method |
| --- | --- |
| REQ-CON-001 … 004 | Test |
| REQ-DAT-001 … 004 | Test |
| REQ-ALM-001 … 003 | Test |
| REQ-API-001 … 003 | Test and demonstration |
| REQ-OTA-001 … 003 | Test |
| NFR-PERF-001 … 003 | Test |
| NFR-SEC-001 … 004 | Inspection and test |
| NFR-REL-001 … 003 | Analysis |

## 6. Traceability

| Requirement | Work item | Test plan |
| --- | --- | --- |
| REQ-CON-001 | AB#1201 | TP-CON |
| REQ-CON-002 | AB#1202 | TP-CON |
| REQ-CON-003 | AB#1203 | TP-CON |
| REQ-CON-004 | AB#1204 | TP-CON |
| REQ-DAT-001 | AB#1311 | TP-DAT |
| REQ-DAT-002 | AB#1312 | TP-DAT |
| REQ-DAT-003 | AB#1313 | TP-DAT |
| REQ-DAT-004 | AB#1314 | TP-DAT |
| REQ-DAT-005 | AB#1319 | — |
| REQ-ALM-001 | AB#1331 | TP-ALM |
| REQ-ALM-002 | AB#1332 | TP-ALM |
| REQ-ALM-003 | AB#1333 | TP-ALM |
| REQ-API-001 | AB#1341 | TP-API |
| REQ-API-002 | AB#1342 | TP-API |
| REQ-API-003 | AB#1343 | TP-API |
| REQ-OTA-001 | AB#1401 | TP-OTA |
| REQ-OTA-002 | AB#1402 | TP-OTA |
| REQ-OTA-003 | AB#1403 | TP-OTA |

## 7. Open issues

- [ ] Confirm default alarm thresholds with product management.
- [ ] Schedule external review round 1.

## 8. Revision history

| Version | Date | Author | Notes |
| --- | --- | --- | --- |
| 0.9 | 2026-08-21 | Systems Engineering | Initial internal draft |
| 1.0 | 2026-10-02 | Systems Engineering | Draft for external review: adds OTA requirements, tightens connection and heartbeat timing, removes USB CSV export |
