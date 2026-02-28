# Rhythmic Attestation Network (RAN)
### IoT Trust & Drift Analytics — Hackathon Submission

---

## Executive Summary

Every existing IoT security tool asks the same question: *"Is this device behaving correctly?"*

RAN asks a fundamentally different question: *"Is this actually the device it claims to be?"*

The **Rhythmic Attestation Network** is a hardware identity verification system for IoT fleets. It works by continuously measuring and verifying the invisible physical rhythm of every device — the micro-timing patterns in packet inter-arrival intervals, hardware clock drift, and latency jitter that are determined entirely by the physical chip, oscillator, and environment of the device.

These rhythms cannot be faked. A hacker can copy firmware, replicate behavior, and mirror traffic patterns. He cannot replicate the physics of a different piece of hardware.

**The result:** A security system that detects device substitution, hardware cloning, and supply-chain tampering at the moment of first contact — before any malicious behavior occurs — while also providing continuous drift monitoring to detect gradual compromise over time.

**Market positioning:** RAN is the only IoT security system that operates at the hardware identity layer, not the behavioral layer. It is complementary to, not competitive with, existing SIEM and behavioral analytics tools.

---

## Threat & Failure Model

### Threats RAN Is Designed to Catch

**1. Physical Device Substitution**
An attacker physically swaps a legitimate IoT device (e.g., a smart camera) with an identical-looking compromised unit. The replacement has the same firmware, same credentials, same behavioral patterns. Every existing security tool would trust it completely. RAN detects it immediately because the hardware rhythm of a different physical chip is measurably different.

**2. Hardware Cloning / Counterfeit Devices**
A counterfeit device with matching firmware is inserted into a fleet. Common in supply chains. The hardware rhythm profile of a counterfeit differs from genuine hardware because oscillator tolerances, capacitor aging, and PCB characteristics vary between manufacturers and even between batches.

**3. Firmware-Level Compromise (Silent)**
A device is compromised via a malicious firmware update. Behavioral tools may not catch this if the attacker is careful to preserve normal traffic patterns. However, firmware changes that touch hardware timing (e.g., modifying kernel scheduling, altering interrupt handling) produce measurable rhythm shifts that RAN detects as drift.

**4. Replay Attacks (Defeated by Design)**
An attacker captures and replays a legitimate device's traffic to impersonate it. RAN defeats this with ephemeral one-time challenges — the attacker must produce a live hardware response to an unpredictable request, which requires controlling the actual physical hardware.

**5. Gradual Rhythm Drift (Insider / Long-term Compromise)**
A device is slowly modified over weeks. Standard threshold-based detection misses it. RAN's LSTM drift detector models the expected natural aging curve of each device's rhythm (hardware genuinely drifts slightly over time due to capacitor aging and temperature) and flags deviations from that curve.

### Threats RAN Does NOT Claim to Solve

- Application-layer attacks that do not affect hardware timing (handled by behavioral SIEM tools)
- Attacks on devices whose hardware profile has not yet been enrolled
- Zero-day attacks that complete entirely within a 30-second attestation window
- Fully air-gapped physical tampering with no network connection

### Failure Modes & Mitigations

| Failure Mode | Likelihood | Mitigation |
|---|---|---|
| False positive (legitimate device flagged) | Medium | Adaptive threshold tuning per device class; human-in-loop for first 3 false positives |
| False negative (compromised device passes) | Low | Ephemeral challenges + continuous re-attestation every 30s |
| Mesh node failure | Medium | Profiles replicated to 3 neighbor nodes; automatic failover |
| Profile poisoning attack | Low | Cryptographically signed profiles; TPM-backed edge nodes |
| Natural hardware drift over-triggering | Medium | LSTM models expected aging curve per device class |

---

## Technical & Functional Architecture

### Layer Overview

```
┌─────────────────────────────────────────────────────────┐
│                    DEVICE LAYER                         │
│  Medical IoT │ Cameras │ Sensors │ Guest Devices        │
└──────────────────────┬──────────────────────────────────┘
                       │ hardware rhythm signals
                       ▼
┌─────────────────────────────────────────────────────────┐
│                     EDGE LAYER                          │
│  Passive Rhythm Collector (Zeek/eBPF)                  │
│  Hardware Fingerprint Engine                            │
│  Ephemeral Challenge Generator                          │
└──────────────────────┬──────────────────────────────────┘
                       │ compressed profiles (~4KB each)
                       ▼
┌─────────────────────────────────────────────────────────┐
│               INTELLIGENCE LAYER                        │
│  Rhythm Attestation Engine   │  Drift Detector (LSTM)  │
│  Distributed Profile Mesh    │  Policy Enforcer         │
└──────────────────────┬──────────────────────────────────┘
                       │ alerts + structured evidence
                       ▼
┌─────────────────────────────────────────────────────────┐
│                   OUTPUT LAYER                          │
│  Explainability Engine (LLM) │ Dashboard               │
│  SIEM Integration            │ REST API                │
└─────────────────────────────────────────────────────────┘
```

### Component Descriptions

**Passive Rhythm Collector**
Deployed as a network tap or inline sensor. Uses Zeek for deep packet inspection and eBPF for nanosecond-precision kernel-level timing. Requires zero modification to IoT devices. Captures: packet inter-arrival times (IAT), TCP/UDP response latencies, DNS query timing, TLS handshake timing, and power-correlated burst signatures.

**Hardware Fingerprint Engine**
Computes the statistical signature of a device's rhythm: mean IAT (μ), variance (σ²), skewness, kurtosis, clock drift coefficient, and oscillator jitter profile. These 12–16 statistical features constitute the rhythm fingerprint — stored as ~4KB per device, not raw data logs.

**Ephemeral Challenge Generator**
When attestation is required, issues a one-time challenge instructing the target device to perform a specific sequence of communication actions (e.g., ping 3 specific endpoints in order, then wait 2 seconds, then query DNS). The device's hardware rhythm during this challenge is compared against its enrolled profile. The challenge is unpredictable and cannot be pre-computed.

**Distributed Profile Mesh**
Each edge node stores rhythm profiles only for its 5–10 nearest network neighbors. There is no central database. Profiles are replicated to 3 nodes for redundancy. Profiles are cryptographically signed using libsodium — tampering is detected immediately. This design means there is no single attack surface that gives access to all profiles.

**Rhythm Attestation Engine**
Core comparison engine. Uses Mahalanobis distance to compare live rhythm observations against the enrolled statistical profile. Mahalanobis distance is chosen because it accounts for correlations between rhythm features and is not fooled by scaling. Outputs an attestation confidence score from 0–100.

**LSTM Drift Detector**
A Long Short-Term Memory neural network trained per device class to model the expected natural evolution of hardware rhythm over time. Hardware genuinely drifts as components age — a 2-year-old camera's oscillator runs slightly differently than when new. The LSTM learns this expected aging curve. Deviations from the learned curve (sudden shifts, wrong direction, excessive magnitude) are flagged as anomalies.

**Explainability Engine**
Structured evidence from the attestation engine (which features deviated, by how much, with what confidence) is passed to Claude API. The LLM generates a plain-English incident narrative explaining what happened, when it started, why it matters, and what action to take. Output is structured JSON with a human-readable summary field for dashboard display and PDF report export.

---

## Telemetry Plan & Dataset Strategy

### Telemetry Features Collected Per Device

| Feature | Collection Method | Frequency | Storage |
|---|---|---|---|
| Packet inter-arrival time (IAT) | eBPF | Per packet | Aggregated to stats every 30s |
| TCP response latency | Zeek | Per connection | Aggregated to stats every 30s |
| Clock drift coefficient | NTP timing analysis | Every 60s | Rolling 24hr window |
| DNS query timing | Passive DNS tap | Per query | Aggregated hourly |
| TLS handshake timing | Zeek TLS analyzer | Per session | Aggregated daily |
| Payload size distribution | Zeek | Per connection | Stats only, no content |
| Burst pattern signature | eBPF | Per burst | Compressed to 128-dim vector |

### Dataset Strategy

**Synthetic Datasets (for development and evaluation)**
Generate synthetic rhythm profiles for 10 device classes (medical, camera, thermostat, industrial sensor, etc.) using realistic hardware specifications. Simulate: normal operation, gradual drift, sudden substitution, firmware modification, replay attacks.

**Real Hardware Lab Datasets**
Procure 3–5 units each of 5 IoT device models. Capture genuine rhythm profiles over 2 weeks. Introduce controlled anomalies: swap identical devices, modify firmware, physically move devices to different temperature environments. Use these as ground truth for model evaluation.

**Public Datasets (baseline validation)**
- N-BaIoT dataset (Meidan et al., 2018): Real IoT traffic from 9 commercial devices
- UNSW-NB15 dataset: Network-level features for anomaly detection baseline
- IoT-23 dataset: Labeled IoT device behavior logs

### Storage Estimation

With 10,000 devices and statistical aggregation: 4KB profile × 10,000 = 40MB for all profiles. 30-second aggregated telemetry: ~200 bytes × 10,000 devices × 2,880 windows/day = ~5.8GB/day. This is 50–100× less than raw packet logging.

---

## Trust Score Model

### RAN replaces traditional trust scores with Attestation Confidence

Rather than a 0–100 trust score that can be gamed by gradually shifting behavior, RAN produces an **Attestation Confidence** value derived from hardware physics comparison.

**Attestation Confidence = f(Mahalanobis Distance, Challenge Response Match, Drift Delta)**

```
AC = w1 × (1 - normalize(MD)) + w2 × ChallengeMatch + w3 × (1 - DriftDelta)

Where:
  MD = Mahalanobis distance between live rhythm and enrolled profile
  ChallengeMatch = cosine similarity of challenge response rhythm to expected
  DriftDelta = magnitude of deviation from LSTM-predicted drift curve
  w1, w2, w3 = class-specific weights (configurable per device type)
```

**Threshold configuration:**

| Device Class | Block Threshold | Alert Threshold | Notes |
|---|---|---|---|
| Medical / Critical | AC < 95% | AC < 97% | Zero tolerance, human review required |
| Industrial OT | AC < 88% | AC < 92% | High confidence needed |
| Office IoT | AC < 75% | AC < 82% | Standard threshold |
| Guest / Unmanaged | AC < 60% | AC < 70% | Permissive, high monitoring |

---

## Policy Model & Drift Logic

### Policy Definitions

Policies are defined per device class in YAML and specify:

```yaml
device_class: medical_pump
attestation:
  continuous_interval_seconds: 30
  block_threshold: 0.95
  alert_threshold: 0.97
  challenge_required: true
  challenge_interval_minutes: 10
drift:
  max_daily_drift_coefficient: 0.02
  max_weekly_drift_coefficient: 0.08
  lstm_anomaly_threshold: 2.5  # standard deviations from expected aging curve
communication:
  allowed_peers: [nurse_station, pharmacy_system]
  max_trust_gap_for_connection: 25  # AC points difference
actions:
  on_block: [quarantine, alert_siem, generate_narrative]
  on_alert: [flag_dashboard, increase_monitoring_frequency]
  on_drift: [alert_it, schedule_physical_inspection]
```

### Drift Logic

**Short-term drift (minutes to hours):** Sudden rhythm shift — indicates active compromise, device swap, or hardware failure. Treated as high severity. Immediate block and alert.

**Medium-term drift (days):** Rhythm moving steadily in unexpected direction — may indicate gradual firmware modification or environmental change. LSTM confidence score decreases. Alert at threshold, schedule inspection.

**Long-term drift (weeks to months):** Rhythm slowly evolving — expected for aging hardware. LSTM learns this curve and updates the expected profile accordingly. If drift rate exceeds expected aging rate for device class, flag for review.

**Fleet-wide drift:** If 15%+ of devices in a class show similar rhythm shifts simultaneously, this may indicate a supply-chain level attack or coordinated firmware compromise. Fleet-level alert with elevated severity.

---

## AI / ML Components

### Component 1: LSTM Drift Predictor

**Architecture:** 2-layer LSTM with 128 hidden units, trained per device class.

**Input:** Rolling 7-day window of daily rhythm statistics (12 features per day = 84-dimensional input).

**Output:** Predicted rhythm statistics for next 24 hours + confidence interval.

**Training:** Supervised on 90 days of normal device operation per class. Anomaly scoring uses the prediction error (actual vs predicted) normalized by confidence interval width.

**Why LSTM over simpler models:** Rhythm drift has temporal dependencies — yesterday's drift affects tomorrow's expected state. LSTM captures these dependencies. A simpler moving average would not distinguish between expected aging drift and anomalous drift.

### Component 2: Isolation Forest Anomaly Scorer

**Purpose:** Initial screening before full LSTM inference. Computationally cheap — runs on every 30-second window.

**Input:** 12-feature rhythm snapshot.

**Output:** Anomaly score 0–1. Scores above 0.7 trigger full LSTM + Attestation pipeline.

**Why this combination:** Isolation Forest is fast and handles high-dimensional outliers well. LSTM is slower but temporally aware. Two-stage pipeline balances speed and accuracy.

### Component 3: LLM Narrative Generator

**Model:** Claude API (claude-sonnet-4-6).

**Input:** Structured JSON evidence object:

```json
{
  "device_id": "cam-floor3-07",
  "device_class": "surveillance_camera",
  "event_type": "rhythm_mismatch",
  "attestation_confidence": 23.4,
  "enrolled_profile_date": "2025-09-15",
  "features_deviated": [
    {"feature": "mean_iat", "enrolled": 12.4, "observed": 31.7, "z_score": 4.2},
    {"feature": "clock_drift_coeff", "enrolled": 0.0003, "observed": 0.0019, "z_score": 8.1}
  ],
  "drift_timeline": [...],
  "challenge_response_match": 0.18
}
```

**Output:** Plain-English narrative + severity + recommended action.

**Prompt strategy:** Chain-of-thought reasoning with structured output format. The LLM is instructed to explain the evidence as if talking to a non-technical hospital IT manager — no jargon, clear action items.

---

## Baseline Protection Strategy

### What RAN does even without AI components active

1. **Static profile comparison:** Even without LSTM inference, comparing live Mahalanobis distance against enrolled profile catches clear substitution attacks (distance >> threshold).

2. **Challenge-response attestation:** Ephemeral challenges catch replay attacks at the cryptographic layer, independent of any ML model.

3. **Anomaly rate limiting:** Any device requesting connections more than 2 standard deviations faster than its historical rate is flagged and rate-limited.

4. **Neighbor-based sanity check:** A device whose rhythm profile does not match any known profile in the neighbor mesh is quarantined pending enrollment review.

5. **Fleet-level anomaly detection:** Simple statistical comparison — if a device's rhythm is more than 4σ from the mean of its device class, immediate alert regardless of ML model state.

---

## Explainability Output Specification

### Alert Narrative Format

Every alert produces a structured output with the following fields:

```json
{
  "incident_id": "RAN-2025-001847",
  "timestamp": "2025-11-14T03:42:17Z",
  "severity": "CRITICAL",
  "confidence": 0.94,
  "device": {
    "id": "cam-floor3-07",
    "class": "surveillance_camera",
    "location": "Hospital Floor 3, East Wing",
    "enrolled_since": "2025-09-15"
  },
  "summary": "Camera on Floor 3 East Wing failed hardware attestation at 3:42 AM. The device's physical timing signature does not match its enrolled profile, suggesting the physical hardware has been replaced or tampered with. The mismatch was detected within 30 seconds of the change.",
  "evidence": {
    "attestation_confidence": 23.4,
    "key_deviations": [
      "Clock oscillator drift is 6x higher than enrolled value — strongly suggests different physical hardware",
      "Packet inter-arrival time distribution has shifted by 4.2 standard deviations",
      "Challenge response produced rhythm inconsistent with enrolled profile"
    ],
    "drift_history": "Device rhythm was stable for 60 days prior to this event. No gradual drift — the change was sudden, occurring between 3:41 AM and 3:42 AM."
  },
  "recommended_action": "Dispatch security personnel to physically inspect Camera Floor 3 East Wing immediately. Do not restore network access until physical inspection confirms the device is the original enrolled unit. If device appears intact, consider firmware forensic analysis.",
  "technical_detail": {
    "mahalanobis_distance": 14.7,
    "normal_range": "0.8 - 2.1",
    "lstm_anomaly_score": 0.97,
    "features_violated": ["mean_iat", "clock_drift_coeff", "burst_pattern_vector"],
    "shap_top_features": ["clock_drift_coeff (71%)", "mean_iat (19%)", "burst_signature (10%)"]
  }
}
```

### Dashboard Display

The dashboard renders the above as:
- A one-paragraph plain-English summary (suitable for non-technical staff)
- A severity badge (CRITICAL / HIGH / MEDIUM / LOW)
- A drift timeline showing the last 7 days of attestation confidence scores
- A feature deviation chart (bar chart of which rhythm features deviated most)
- A one-click recommended action button

---

## Evaluation Plan

### Metrics

| Metric | Target | Measurement Method |
|---|---|---|
| True Positive Rate (substitution detection) | > 99% | Controlled lab swap tests |
| False Positive Rate (legitimate device flagged) | < 0.5% | 30-day monitoring of known-good fleet |
| Detection Latency | < 60 seconds | Time from device swap to alert generation |
| Profile Storage per device | < 10KB | Direct measurement |
| Attestation throughput | > 1,000 devices/second | Load test on reference hardware |
| Replay attack detection | 100% | Controlled replay attack battery |

### Test Scenarios

**Scenario A — Physical Substitution Test**
Replace 10% of devices in test fleet with identical-model units. Measure: detection rate, time to detection, false positives on unreplaced devices.

**Scenario B — Gradual Drift Test**
Introduce controlled firmware modifications that affect timing by 1%, 5%, 10%, 20% incrementally over 30 days. Measure: at what drift level does LSTM detect anomaly vs expected aging.

**Scenario C — Replay Attack Battery**
Capture 24 hours of rhythm telemetry from 10 devices. Attempt replay attacks. Measure: challenge-response mechanism effectiveness.

**Scenario D — Environmental Stress Test**
Expose devices to temperature extremes (0°C, 40°C) and measure natural rhythm variation. Ensure this does not trigger false positives. Tune LSTM environmental compensation.

**Scenario E — Scale Test**
Simulate 10,000 device fleet. Measure: attestation latency, mesh lookup time, storage utilization, Kafka pipeline throughput.

### Evaluation Timeline

- Week 1–2: Synthetic dataset generation + baseline model training
- Week 3–4: Lab hardware tests (Scenarios A, B, C)
- Week 5: Environmental and scale testing (Scenarios D, E)
- Week 6: Metric compilation + threshold tuning
- Week 7: Final evaluation report

---

## Tools & Technology Stack

### Edge / Collection Layer
- **Zeek** — Network security monitoring, deep packet inspection
- **eBPF** — Kernel-level packet timing, nanosecond precision
- **MQTT / CoAP** — IoT protocol ingestion
- **Raspberry Pi 5** — Edge node hardware for distributed mesh

### Streaming & Storage
- **Apache Kafka** — Real-time telemetry pipeline
- **TimescaleDB** — Time-series database optimized for IoT metrics
- **Redis** — Ultra-low-latency neighbor profile cache

### Core Engine
- **Python 3.11** — Core attestation and fingerprint engine
- **NumPy / SciPy** — Statistical fingerprinting, Mahalanobis distance
- **Scikit-learn** — Isolation Forest anomaly detection

### Machine Learning
- **PyTorch** — LSTM drift prediction model
- **SHAP** — Explainability for feature importance
- **MLflow** — Model versioning and experiment tracking

### Security & Cryptography
- **libsodium** — Cryptographic signing of profiles and challenge tokens
- **TPM 2.0** — Hardware root-of-trust for edge collector nodes
- **mTLS** — Mutual authentication between mesh nodes
- **BLAKE3** — High-speed hashing for rhythm profile fingerprints

### AI / Explainability
- **Claude API (claude-sonnet-4-6)** — LLM-generated incident narratives
- **Structured prompting** — Chain-of-thought with JSON output schema

### API & Dashboard
- **FastAPI** — REST API, webhook integration
- **React + D3.js** — Real-time dashboard with rhythm heatmaps
- **Kubernetes** — Orchestration of distributed mesh nodes

### Integrations
- **Splunk / Azure Sentinel** — SIEM webhook integration
- **PagerDuty** — Alerting and on-call routing
- **OpenTelemetry** — Observability and distributed tracing

---

## Expected Benefits

### Security Benefits

**Catches what behavioral tools miss.** A perfectly mimicked device with identical behavioral patterns will still fail RAN attestation because its hardware physics are different. This is a fundamentally new detection layer, not an improvement to existing ones.

**Zero-configuration per-device rules.** RAN doesn't require security teams to define what a device is allowed to do. It just enrolls, learns the hardware signature, and flags deviations. This eliminates the "we forgot to write a rule for that" failure mode.

**Detects supply-chain attacks pre-behavior.** A compromised device is flagged the moment it connects — before it can perform any malicious actions — if its hardware doesn't match the enrolled profile.

### Operational Benefits

**Agentless deployment.** Zero modification to IoT devices. No firmware updates, no certificates to install, no SDK to integrate. Works with devices that physically cannot run agents (old PLCs, medical devices, etc.).

**Lightweight storage.** ~4KB per device profile vs gigabytes of behavioral log storage. A 10,000-device fleet requires 40MB for all profiles — fits in memory.

**Plain-English alerts.** Non-security staff can understand and act on alerts. Reduces alert fatigue and time-to-response by eliminating the interpretation step.

### Business Benefits

**Regulatory alignment.** EU Cyber Resilience Act (2025) requires hardware-level security assurance for connected devices. RAN is one of very few tools that can provide attestable hardware identity verification at scale.

**Insurance and compliance.** IoT device identity verification is increasingly required for cyber insurance policies. RAN produces audit-ready evidence logs for every attestation event.

**Complementary, not competitive.** RAN integrates alongside Claroty, Armis, and Forescout — it adds a hardware identity layer that none of them have. This makes it saleable to customers of those platforms, not just as a replacement.

**Network effect moat.** Hardware rhythm profiles for device classes (e.g., "Hikvision DS-2CD2143G2 camera") can be shared across customers as anonymized baselines. The more customers, the more accurate the baseline profiles, the better the product works. Classic network-effect SaaS moat.

---

*Rhythmic Attestation Network — v1.0 Hackathon Submission*
*"You can fake what a device does. You cannot fake how it breathes."*
