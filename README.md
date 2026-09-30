```markdown
# Edge Predictive Maintenance & Vibration Intelligence System

An edge-native condition monitoring and prognostic microservice for rotating industrial machinery (induction motors, bearing assemblies, and drivetrains). 

The service ingests high-rate acceleration waveforms (10 kHz), computes time- and frequency-domain digital signal processing (DSP) metrics, isolates kinematic defect harmonics (BPFO/BPFI), detects multivariate anomalies via unsupervised Isolation Forests, and extrapolates Remaining Useful Life (RUL) against ISO 10816 vibration severity standards.

---

## Architecture & Industrial Context (ISA-95)

```text
Level 3: Manufacturing Operations Management (MES / SAP PM Work Orders)
   ▲
   │ Advisory Diagnostics & Work-Order Triggers
Level 2: Supervisory Control & Data Acquisition (SCADA / Ignition / Wonderware)
   ▲
   │ Telemetry Aggregates (REST / Server-Sent Events / MQTT)
┌──┴────────────────────────────────────────────────────────────────────────┐
│ Edge Node: Predictive Maintenance Microservice (This Service)            │
│                                                                           │
│  [10 kHz Telemetry] ──> [DSP Engine: Hann Windowing & Real FFT]           │
│                                │                                          │
│                                ▼                                          │
│                     [Kinematic Defect Matcher]                            │
│                     - 1X Shaft Rotation (~29.5 Hz)                        │
│                     - 2X Angular Misalignment (~59.0 Hz)                  │
│                     - BPFO Outer-Race Defect (~105.7 Hz)                  │
│                                │                                          │
│                                ▼                                          │
│                     [ISO 10816-3 Severity Machine]                        │
│                     - Zone A: Good (<= 1.12 g)                            │
│                     - Zone B: Acceptable (<= 2.80 g)                      │
│                     - Zone C: Alert (<= 4.50 g)                           │
│                     - Zone D: Danger / Trip (> 4.50 g)                    │
│                                │                                          │
│                                ▼                                          │
│                     [Multivariate Isolation Forest]                       │
│                     - Inputs: [RMS, Crest Factor, Kurtosis, Temperature]  │
│                                │                                          │
│                                ▼                                          │
│                     [Exponential Prognostic RUL Model]                    │
│                     - Curve: y(t) = a * exp(b * t)                        │
│                     - Predicts time-to-breach for Zone D                  │
└────────────────────────────────┬──────────────────────────────────────────┘
                                 ▲
Level 0: Physical Machine Assets │ Raw Accelerometer & Thermocouple Channels

```

---

## Core Capabilities

* **High-Frequency Vibration DSP:** Computes Root Mean Square (RMS) energy, crest factor, kurtosis, skewness, and peak-to-peak amplitudes on detrended 10 kHz time-series data.
* **FFT Spectral Kinematics:** Computes single-sided Hann-windowed amplitude spectra to isolate machinery fault frequencies calculated from bearing geometry (SKF 6205 outer race multiplier $3.584 \times 1\text{X}$).
* **ISO 10816-3 Condition Standards:** Automatically maps current vibration severity against standardized zones for Class II industrial machinery.
* **Unsupervised Anomaly Scoring:** Uses an Isolation Forest baseline trained on healthy machine run-in cycles to detect multidimensional drift across vibration and thermal metrics.
* **Exponential RUL Prognostics:** Fits progressive operational degradation trends to project remaining hours until maintenance intervention limits are reached.
* **Zero-Dependency Edge Dashboard:** Real-time diagnostics HUD running via Server-Sent Events (SSE) and native HTML5 Canvas without external CDN dependencies.

---

## Project Structure

```text
predictive-maintenance-system/
├── config/
│   ├── equipment_specs.yaml       # Kinematic bearing geometry & motor parameters
│   └── alarm_thresholds.yaml     # ISO 10816-3 vibration severity limits
├── src/
│   ├── api/
│   │   ├── app.py                 # FastAPI application with SSE streaming
│   │   └── index.html             # Pure Canvas edge monitoring HUD
│   ├── core/
│   │   ├── anomaly_engine.py      # ISO severity classifier & Isolation Forest
│   │   ├── rul_estimator.py       # Exponential wear trajectory prognosticator
│   │   └── signal_processor.py    # DSP feature extraction & Hann-windowed FFT
│   └── ingestion/
│       └── sensor_stream.py       # Multi-harmonic 10 kHz telemetry synthesizer
├── tests/
│   └── test_pdm.py                # Automated pytest suite
├── test_dsp_runner.py             # Standalone verification runner for DSP
├── test_pipeline_e2e.py           # End-to-end degradation & RUL verification
├── main.py                        # Service entrypoint
├── requirements.txt               # Dependency definitions
└── README.md

```

---

## Equipment Specifications (`config/equipment_specs.yaml`)

```yaml
equipment:
  id: "MTR-2040-A"
  type: "Three-Phase Induction Motor"
  rated_power_kw: 15.0
  rated_rpm: 1770.0               # Running frequency 1X = ~29.5 Hz
  bearing:
    model: "SKF 6205"
    pitch_diameter_mm: 38.5
    ball_diameter_mm: 7.94
    num_balls: 9
    contact_angle_deg: 0.0
    bpfo_multiplier: 3.584        # Outer race defect factor (~105.7 Hz at 1770 RPM)
    bpfi_multiplier: 5.416        # Inner race defect factor (~159.8 Hz at 1770 RPM)

```

---

## Getting Started

### 1. Environment Setup

```powershell
# Clone the repository
git clone [https://github.com/Bamstep/predictive-maintenance-system.git](https://github.com/Bamstep/predictive-maintenance-system.git)
cd predictive-maintenance-system

# Create and activate virtual environment
python -m venv venv
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\venv\Scripts\Activate.ps1

# Install dependencies
pip install -r requirements.txt

```

### 2. Run Diagnostic Test Suite

```powershell
python -m pytest tests/ -v

```

### 3. Verify End-to-End Pipeline Headless

```powershell
python test_pipeline_e2e.py

```

### 4. Launch the Edge Service

```powershell
python main.py

```

Open your browser to **http://127.0.0.1:8000** to view the live dashboard.

---

## API Endpoints

| Method | Endpoint | Description |
| --- | --- | --- |
| `GET` | `/` | Serves the HTML5 Canvas telemetry dashboard |
| `GET` | `/stream` | Server-Sent Events (SSE) telemetry data stream |
| `POST` | `/api/degrade?rate=0.05` | Injects synthetic accelerated mechanical wear |
| `POST` | `/api/reset` | Resets machinery state to nominal zero-wear baseline |

---

## Verification & Mathematical Validation

* **Root Mean Square (RMS):**

$$x_{\text{rms}} = \sqrt{\frac{1}{N} \sum_{n=1}^{N} [x(n) - \bar{x}]^2}$$


* **Crest Factor:**

$$C = \frac{\max \vert{}x(n)\vert{}}{x_{\text{rms}}}$$


* **Degradation Trajectory Fit (RUL):**

$$\ln(y) = \ln(a) + b \cdot t \implies t_{\text{fail}} = \frac{\ln(\text{threshold}) - \ln(a)}{b}$$



---

## License

MIT License. Developed as part of the Industrial AI Suite.

```

```
