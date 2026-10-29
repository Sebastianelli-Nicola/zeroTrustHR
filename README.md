# ZeroTrustHR 

[![Architecture: Zero Trust](https://img.shields.io/badge/Architecture-Zero%20Trust-blue.svg)](#-system-architecture)
[![Containerized: Docker](https://img.shields.io/badge/Deployment-Docker%20Compose-2496ED.svg)](#-setup--deployment)
[![Security: mTLS | OPA | Snort3](https://img.shields.io/badge/Security-mTLS%20%7C%20OPA%20%7C%20Snort3-brightgreen.svg)](#-system-architecture)
[![SIEM & ML: Splunk AITK](https://img.shields.io/badge/SIEM_%26_ML-Splunk_Random_Forest-orange.svg)](#-dynamic-risk-calculation-model)
[![Report: PDF](https://img.shields.io/badge/Project_Report-PDF_(ITA)-red.svg)](./Tesina_ADV.pdf)

A state-of-the-art **Zero Trust Architecture** designed for secure Human Resources (HR) and enterprise operations. The system implements **real-time dynamic risk assessment** driven by Machine Learning (**Splunk AI Toolkit**), deep network traffic inspection (**Snort 3**), hardware-bound device identity (**TPM & Secure Enclave**), and fine-grained policy enforcement (**OPA & Envoy**).

>  **Full Technical Documentation:** Read the complete 81-page master's course report (in Italian), including architecture diagrams, Rego policies, TPM/Secure Enclave enrollment flows, and test results: **[Read Complete Technical Report (PDF)](./Tesina_ADV_.pdf)**.

---

##  System Architecture

The Zero Trust infrastructure relies on decoupled security components collaborating in real time to evaluate and enforce trust on every single request:

<pre>
                  [ Client / Device Agent ]
                            │
                            ▼
            ┌───────────────────────────────┐
            │   nftables (L3/L4 Firewall)   │
            └───────────────┬───────────────┘
                            │ (mTLS verified by Internal CA)
                            ▼
           ┌─────────────────────────────────┐
           │   PEP: Envoy Proxy (Gateway)    │ ◄───┐
           └───────────────┬─────────────────┘     │
                           │ Checks Policy         │ Real-time
                           ▼                       │ Risk Score
            ┌──────────────────────────────┐       │
            │   PDP: Open Policy Agent     │ ──────┤
            └───────────────┬──────────────┘       │
                           │ Evaluates             │
                           ▼                       │
            ┌──────────────────────────────┐       │
            │   Backend: Node.js / TS      │       │
            └───────────────┬──────────────┘       │
                           │                       │
                           ▼                       │
            ┌──────────────────────────────┐       │
            │   Database: MongoDB (Data)   │       │
            └──────────────────────────────┘       │
                                                   │
  ────────────────── Telemetry & Risk ─────────────┼─
                                                   │
   [ Snort 3 IDS ] ──> Network Anomalies (Pnet) ───┤
                                                   │
   [ Splunk SIEM ] ──> ML Risk Engine (Papp)  ─────┘
                       (Random Forest via HEC)
</pre>

### Core Components
* **PEP (Policy Enforcement Point) — Envoy Proxy:** Perimeter edge gateway enforcing **mutual TLS (mTLS)** authentication, rejecting unauthorized requests at L4/L7 before reaching the backend.
* **PDP (Policy Decision Point) — OPA (Open Policy Agent):** Evaluates every API request against fine-grained dynamic Rego policies, computing resource impact and querying the live risk engine.
* **Risk Engine & SIEM — Splunk:** Collects centralized telemetry logs via HTTP Event Collector (HEC). Hosts the **Random Forest** Machine Learning model (*Splunk AI Toolkit*) to compute Application Probability ($P_{app}$).
* **IDS (Intrusion Detection System) — Snort 3:** Monitors raw network traffic via deep packet inspection. Detects infrastructural anomalies (e.g., Port Scanning) and escalates Network Probability ($P_{net}$).
* **L3/L4 Firewall — nftables:** Manages baseline network-level packet filtering, isolation, and kernel audit logging.
* **Internal CA & Device Agents:** Issues X.509 certificates (`ca-service`) and manages hardware-bound device identity verification (`TPM` for Windows, `Secure Enclave` for macOS, plus software fallback with JA3 fingerprinting).
* **Application Layer (`Node.js/TS` & `React/Vite`):** Handles HR business logic, device enrollment flows, and provides a secure, responsive interface backed by **MongoDB**.

---

##  Dynamic Risk Calculation Model

Continuous mathematical threat evaluation computed for every incoming request:

$$\text{Total Risk} = \max(P_{app}, P_{net}) \times \text{Impact}$$

* **$P_{app}$ (Application Probability $\in [0, 1]$):** Computed in real time by the **Random Forest** classifier on Splunk based on historical and live access patterns (`total_denies`, `time_denies`, `priv_denies`, `auth_denies`, `distinct_ips`, `deny_ratio`).
* **$P_{net}$ (Network Probability $\in [0, 1]$):** Triggered by direct network exploitation (e.g., Nmap SYN Port Scanning) and automatically escalated by **Snort 3** (e.g., $P_{net} = 0.95$).
* **$\text{Impact}$:** Quantifies potential damage via decoupled Splunk lookup tables (`resource_impact.csv` and `asset_weights.csv`) based on resource sensitivity (`public_products`, `company_policies`, `employee_records`) and user privileges.
* **BARAC Adaptive Blocking & Fail-Closed:** Integrates Benefit-Aware Risk Access Control (BARAC) via Splunk saved searches and webhooks (`/api/splunk-webhook`) to dynamically block compromised identities, defaulting to a `Fail-Closed` posture under volumetric DoS attacks.

---

##  Tech Stack & Tools

| Layer | Technologies |
| :--- | :--- |
| **Containerization** | Docker & Docker Compose (`zerotrust_net` & `wan_net` isolation) |
| **Edge Proxy & PKI** | Envoy Proxy (`ext_authz` + mTLS), Custom Internal CA (`Node.js/TS`), X.509 Certificates |
| **Hardware Identity** | Windows TPM Agent (`.NET 8`), macOS Secure Enclave Agent (`Swift/Xcode`), JA3 Fallback |
| **Policy Engine** | Open Policy Agent (OPA) — Declarative Rego Policies (`access`, `user`, `device`, `risk`) |
| **SIEM & ML** | Splunk Enterprise, HTTP Event Collector (HEC), Custom `zerotrust_app` (CaC), Splunk AI Toolkit (AITK) |
| **Network Security** | Snort 3 (IDS), nftables (L3/L4 Host/Bridge Firewall on WSL2) |
| **Application Stack** | Node.js, TypeScript, Express, React, Vite, MongoDB, JWT |
| **Adversary Emulation** | Kali Linux, Nmap, THC-Hydra, Postman, Custom Python Attack Scripts |

---

##  Setup & Deployment

### Prerequisites
* **Docker** & **Docker Compose** installed.
* *(Optional for ML live demo)* Download from [Splunkbase](https://splunkbase.splunk.com/):
  * **Splunk AI Toolkit (AITK)**
  * **Python for Scientific Computing (Linux 64-bit)**

### 1. Initial Configuration
Create a `.env` file in the root directory:

```env
MONGO_ROOT_USER=admin
MONGO_ROOT_PASSWORD=ZeroTrust2026!
SPLUNK_PASSWORD=ZeroTrust2026!
SPLUNK_HEC_TOKEN=a1b2c3d4-e5f6-7890-abcd-ef1234567890
SPLUNK_OPA_TOKEN=b2c3d4e5-f6a1-8901-bcde-f12345678901
JWT_SECRET=dev_jwt_secret_zerotrust_2026
SPLUNK_WEBHOOK_SECRET=dev_webhook_secret_zerotrust_2026
```

### 2. Start Services
```bash
docker-compose down -v
docker-compose up --build -d
```
*(Wait a few minutes for Splunk to initialize at `http://localhost:8000`)*

### 3. Machine Learning Apps Setup (Splunk CLI)
Install the downloaded `.tgz` packages directly into the container to avoid web UI timeouts:

```bash
docker cp "C:\Path\To\python-for-scientific-computing.tgz" zerotrust-splunk:/tmp/
docker cp "C:\Path\To\splunk-ai-toolkit.tgz" zerotrust-splunk:/tmp/

docker exec -u root zerotrust-splunk /opt/splunk/bin/splunk install app /tmp/python-for-scientific-computing.tgz -update 1 -auth admin:ZeroTrust2026!
docker exec -u root zerotrust-splunk /opt/splunk/bin/splunk install app /tmp/splunk-ai-toolkit.tgz -update 1 -auth admin:ZeroTrust2026!

docker-compose restart splunk
```

---

##  Attack Simulations & Testing Guides

### Phase 1: Model Training (Behavioral Analysis)
Generate mixed legitimate (80%) and hostile (20%) traffic to train the Random Forest classifier (`app_risk_model`) on Splunk:

```bash
python scripts/simulate_auth_attacks.py
```

### Phase 2: Predictive Dynamic Blocking (Live Demo)
Run the real-time attack simulation to observe dynamic risk escalation and OPA blocking (`risk_score_too_high`):

```bash
python scripts/demo_live_attacks.py
```

### Phase 3: Infrastructural & Perimeter Attacks (Network & IDS)
* **Port Scanning (Nmap):** Intercepted by **Snort 3**, raising the attacker's Network Probability ($P_{net}$):
  ```bash
  docker run --rm --network zerotrusthr_zerotrust_net instrumentisto/nmap -sS -p- -T4 zerotrust-envoy
  ```
* **Brute Force over mTLS (Hydra):** Natively rejected at the TLS layer by **Envoy Proxy** due to missing client certificates:
  ```bash
  docker run --rm -v "${PWD}/passwords.txt:/passwords.txt" --network zerotrusthr_zerotrust_net secsi/hydra -l admin -P /passwords.txt -s 10000 zerotrust-envoy http-post-form "/api/auth/login:username=^USER^&password=^PASS^:F=Unauthorized"
  ```

---

> [!TIP]
> **Extended Documentation**
> For step-by-step walkthroughs and SPL queries, refer to:
> *  **[Full Academic Report — Tesina_ADV.pdf](./Tesina_ADV.pdf)**
> *  **[Live Demo Guide (demo_guide.md)](./demo_guide.md)**
> *  **[Network Attacks Guide (scripts/network_attacks_guide.md)](./scripts/network_attacks_guide.md)**
> *  **[nftables Firewall Guide (nftables/README.md)](./nftables/README.md)**

---

##  Authors & Academic Context

Developed as a Master's Degree project for the **Advanced Cybersecurity for IT** course (9 CFU, A.Y. 2025/2026) — M.Sc. in **Computer and Automation Engineering** at **Università Politecnica delle Marche (UNIVPM)**, under the supervision of **Prof. Luca Spalazzi**.

* **Nicola Sebastianelli** — [GitHub](https://github.com/Sebastianelli-Nicola)
* **Antonio Di Placido**
* **Pietro Salvatore** — [Original Repository](https://github.com/Pi3tro02/zeroTrustHR)
* **Daniel Diocis**
* **Daniele Giuliani**
