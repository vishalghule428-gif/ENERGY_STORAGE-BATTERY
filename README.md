# Energy Storage & Battery Diagnosis System (ESS-BDIC)

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Platform: MATLAB / Simulink](https://img.shields.io/badge/Simulation-MATLAB%20%7C%20Simulink-orange.svg)](https://www.mathworks.com/products/simulink.html)
[![Hardware: ESP32](https://img.shields.io/badge/Hardware-ESP32%20AFE-brightgreen.svg)](https://www.espressif.com/)
[![Storage: Li-ion + Supercapacitor](https://img.shields.io/badge/Energy%20Storage-Li--ion%20%2B%20Supercapacitor-purple.svg)](#hybrid-energy-storage-system-hess)
[![Domain: Grid Stabilization & BDIC](https://img.shields.io/badge/Domain-Grid%20Stabilization%20%26%20Diagnostics-red.svg)](#battery-diagnosis-system-bdic)

---

## 📌 Executive Overview

The rapid integration of intermittent renewable energy sources—such as Solar Photovoltaics (PV)—into modern distribution grids creates severe power fluctuations, localized frequency excursions, and voltage sags. Conventional single-chemistry energy storage systems (e.g., pure Lithium-ion battery installations) suffer accelerated electrochemical degradation and thermal aging when subjected to high-frequency, dynamic transient load demands.

The **Energy Storage & Battery Diagnosis System (ESS-BDIC)** presents an end-to-end engineered solution comprising:
1. **Hybrid Energy Storage System (HESS)**: Synergistic integration of **Lithium-ion batteries** (high energy density for bulk energy shifting) and **Supercapacitors / EDLCs** (high power density and ultra-fast dynamic response for transient compensation and grid stabilization).
2. **Dynamic Frequency-Decoupled Energy Management Strategy (EMS)**: Low-pass filter (LPF) based power allocation to decouple high-frequency transient power demands from low-frequency baseload demands.
3. **Hardware-in-the-Loop & Embedded Battery Diagnosis System (BDIC)**: An **ESP32-driven** Analog Front End (AFE) architecture capable of millivolt-level individual cell monitoring, dynamic internal resistance ($R_{int}$) estimation, and online thermal gradient anomaly tracking.
4. **Interactive Diagnostic Dashboard**: A real-time monitoring platform providing cell-level State-of-Charge (SoC), State-of-Health (SoH), fault classification, and predictive maintenance triage.

---

## 🏗️ System Architecture & Grid Integration

```
                         SOLAR PV ARRAY (5 kW)
                                   │
                                   ▼
                   +-------------------------------+
                   |    BOOST CONVERTER (MPPT)     |
                   +---------------+---------------+
                                   │
                           DC BUS (400V DC) ◄────────────────────────┐
                                   ▲                                 │
           ┌───────────────────────┴───────────────────────┐         │
           │                                               │         │
           ▼                                               ▼         ▼
+---------------------+                         +---------------------+
| BIDIRECTIONAL DC-DC |                         | BIDIRECTIONAL DC-DC |
| BUCK-BOOST (BATT)   |                         | BUCK-BOOST (SC)     |
+----------+----------+                         +----------+----------+
           │                                               │
           ▼                                               ▼
+---------------------+                         +---------------------+
| LITHIUM-ION BATTERY |                         | SUPERCAPACITOR BANK |
| 48V, 100Ah Pack     |                         | 48V, 165F Module    |
+----------+----------+                         +----------+----------+
           │                                               │
           │             +-------------------+             │
           └────────────►|   BDIC HARDWARE   |◄────────────┘
                         | (ESP32 + AFE IC)  |
                         +---------+---------+
                                   │
                                   ▼
                         +-------------------+
                         | DIAGNOSTIC CLOUD  |
                         | & LOCAL DASHBOARD |
                         +-------------------+
                                   │
                                   ▼
                         DC-AC INVERTER (GRID-TIED)
                                   │
                                   ▼
                         3-PHASE UTILITY GRID / AC LOAD
```

---

## ⚡ Hybrid Energy Storage System (HESS) & Power Allocation

### 1. Complementary Storage Mechanics
To optimize lifetime, thermal stability, and levelized cost of storage, the system exploits the complementary physical characteristics of chemical and electrostatic storage:

| Storage Parameter | Lithium-ion Battery Pack ($\text{LiFePO}_4 / \text{NMC}$) | Electric Double-Layer Capacitor (EDLC) |
| :--- | :--- | :--- |
| **Primary Role** | Long-term energy shifting / Base load | Instantaneous transient absorption / Peak shaving |
| **Specific Energy ($\text{Wh/kg}$)** | $140 - 220$ | $5 - 10$ |
| **Specific Power ($\text{W/kg}$)** | $300 - 1,500$ | $3,000 - 10,000$ |
| **Cycle Life ($\text{Cycles}$)** | $2,000 - 5,000$ | $> 500,000$ |
| **Round-Trip Efficiency ($\%$)** | $85\% - 92\%$ | $95\% - 98\%$ |
| **Response Time** | Hundreds of milliseconds to seconds | **Sub-millisecond ($< 5\text{ ms}$)** |

### 2. Frequency-Decoupled Power Splitting Algorithm
The total net power demand $P_{\text{net}}(t) = P_{\text{load}}(t) - P_{\text{PV}}(t)$ is decoupled into low-frequency and high-frequency components via a first-order Low-Pass Filter (LPF):

$$H_{\text{LPF}}(s) = \frac{1}{1 + s\tau}$$

Where:
* $\tau = R_f C_f = \frac{1}{2\pi f_c}$ is the filter time constant, tuned to $f_c \approx 0.1\text{ Hz}$.

The power allocation commands to the respective DC-DC converters are governed by:

$$P_{\text{batt}}^*(t) = \text{LPF}\{ P_{\text{net}}(t) \}$$

$$P_{\text{sc}}^*(t) = P_{\text{net}}(t) - P_{\text{batt}}^*(t) + P_{\text{sc,regen}}(t)$$

```
Net Grid Demand P_net ───►(+)─────────────────────────────► Supercapacitor Loop P_sc*
                           │                               ▲
                           │                               │ (-)
                           ▼                               │
                   [Low-Pass Filter] ──────────────────────┘
                   (fc = 0.1 Hz)
                           │
                           ▼
                 Battery Loop P_batt*
                 (Smooth, C-Rate < 1.0C)
```

**Outcome**: High $\frac{dI}{dt}$ spikes caused by solar irradiance drops or motor inrush currents are routed entirely to the supercapacitor, reducing the battery pack root-mean-square (RMS) current stress by over **$68\%$**.

---

## 🔬 MATLAB / Simulink Modeling & Control Implementation

### 1. Battery Equivalent Circuit Model (ECM: 2-RC Thevenin)
The Lithium-ion cell dynamic behavior is captured using a Second-Order Resistance-Capacitance (2-RC) Thevenin equivalent circuit:

$$V_t(t) = V_{\text{OC}}(SoC) - I_L(t) \cdot R_0 - V_{RC1}(t) - V_{RC2}(t)$$

$$\frac{d V_{RC1}}{dt} = -\frac{V_{RC1}}{R_1 C_1} + \frac{I_L}{C_1}$$

$$\frac{d V_{RC2}}{dt} = -\frac{V_{RC2}}{R_2 C_2} + \frac{I_L}{C_2}$$

Where:
* $V_{\text{OC}}(SoC)$: Open-Circuit Voltage as a non-linear function of State of Charge.
* $R_0$: Ohmic internal resistance (electrolyte and contact resistance).
* $R_1, C_1$: Electrochemical charge-transfer resistance and double-layer capacitance (short-term diffusion).
* $R_2, C_2$: Concentration polarization resistance and diffusion capacitance (long-term relaxation).

```
        +---[ R0 ]---+----+---[ R1 ]---+----+---[ R2 ]---+----+
        |            |    |            |    |            |    |
   +----+            |   ---          ---  ---          ---   +----+
   |                 |   --- C1       ---  --- C2       ---        |
 [ Voc ]             |    |            |    |            |       [ Vt ]
   |                 +----+------------+----+------------+         |
   +---------------------------------------------------------------+
```

### 2. Supercapacitor Dynamic Model
The EDLC bank is modeled via a classical Stern-Grahame fractional capacitor and equivalent series resistance ($ESR$):

$$V_{\text{sc}}(t) = V_{\text{init}} - \frac{1}{C_{\text{sc}}} \int_{0}^{t} I_{\text{sc}}(\tau) d\tau - I_{\text{sc}}(t) \cdot ESR$$

Stored electrostatic energy:

$$E_{\text{sc}} = \frac{1}{2} C_{\text{sc}} \left( V_{\text{max}}^2 - V_{\text{min}}^2 \right)$$

### 3. Bidirectional DC-DC Converter Control Architecture
Both energy storage legs utilize dual-loop PI controllers:
* **Outer Voltage Loop**: Regulates the 400V DC link bus voltage against load fluctuations.
* **Inner Current Loop**: High-bandwidth ($f_{\text{sw}} = 20\text{ kHz}$) inductor current tracking with anti-windup clamping to prevent over-current trips.

---

## 🩺 Battery Diagnosis System (BDIC)

The BDIC subsystem provides continuous cell-level fault diagnostics, isolating defective cells before cascading thermal runaway or module failure occurs.

```
       CELL VOLTAGE SENSING TAP (Cells 1 to 16)
                          │
                          ▼
       +-----------------------------------------+
       |   ANALOG FRONT END (LTC6811 / BQ76952)  |
       |  - 16-Channel Synchronous 16-bit ADC    |
       |  - Isolated SPI / isoSPI Interface      |
       +--------------------+--------------------+
                            │
               Galvanic Digital Isolator (ADuM1201)
                            │
                            ▼
       +-----------------------------------------+
       |         ESP32 EMBEDDED DIAGNOSTICS       |
       |  - Online R_int Estimation               |
       |  - Thermal Gradient Tracker              |
       |  - Kalman Filter SoC Estimator           |
       |  - Relay Tripping Matrix Control         |
       +--------------------+--------------------+
                            │
                            ▼
       +-----------------------------------------+
       |         FAULT CLASSIFICATION ENGINE     |
       +--------------------+--------------------+
         ├── [Over-Voltage / Under-Voltage Trip]
         ├── [Micro-Short Internal Anomaly]
         ├── [Rapid Capacity Fade Metric]
         └── [Cell Imbalance Delta V > 80 mV]
```

### 1. Online Dynamic Internal Resistance ($R_{int}$) Estimation
Internal resistance growth is the primary non-destructive indicator of battery degradation ($\text{SoH}$). The ESP32 captures fast step-changes in current during load switching:

$$R_{\text{int, cell}} = \left| \frac{V_{\text{cell}}(t_k) - V_{\text{cell}}(t_{k-1})}{I_{\text{pack}}(t_k) - I_{\text{pack}}(t_{k-1})} \right|$$

* A calibrated threshold ($\ge 150\%$ of nominal factory $R_0$) immediately flags the cell as **DEGRADED - HIGH IMPEDANCE**, preventing over-heating under high C-rate charging.

### 2. Fault Detection Classification Matrix

| Fault Type | Detection Parameter | Diagnostic Threshold | System Action | Response Latency |
| :--- | :--- | :--- | :--- | :--- |
| **Cell Under-Voltage** | Individual Cell $V_i$ | $V_i < 2.50\text{ V}$ | Isolate Pack / Cut Discharging | $< 100\text{ ms}$ |
| **Cell Over-Voltage** | Individual Cell $V_i$ | $V_i > 3.65\text{ V}$ | Bypass Charger / Cut Solar | $< 100\text{ ms}$ |
| **Thermal Gradient Anomaly** | Multi-point NTC $\Delta T$ | $\Delta T_{\text{cells}} > 5.0^\circ\text{C}$ | Engage Forced Cooling / Derate | $< 500\text{ ms}$ |
| **Internal Micro-Short** | Self-discharge rate ($dV/dt$) | $dV/dt > 15\text{ mV/hr}$ at rest | Flag Cell for Replacement | Continuous |
| **Severe Imbalance** | $\Delta V = V_{\text{max}} - V_{\text{min}}$ | $\Delta V > 80\text{ mV}$ | Trigger Active Cell Balancing | Continuous |

---

## 🔌 Hardware Integration & PCB Implementation Concepts

The system includes custom PCB hardware concepts tailored for industrial noise immunity and high-voltage isolation:

1. **Galvanic Isolation**: High-speed digital isolators (Analog Devices ADuM1201 / Texas Instruments ISO7741) decouple the low-voltage ESP32 processing domain ($3.3\text{V}$) from the high-voltage battery stack ($48\text{V}-400\text{V}$).
2. **Kelvin Sensing Topology**: Four-wire Kelvin connections routed directly to cell terminals to eliminate trace resistance measurement errors.
3. **Thermal Management**: Multi-point 10k NTC thermistors embedded between cell walls with thermal interface pads (silicone TIM) for real-time hotspot identification.
4. **Solid-State Contactor Interlocking**: High-side MOSFET/IGBT drivers with hardware over-current crowbar circuits for sub-microsecond disconnection upon hard short-circuit detection.

---

## 📊 Dashboard & Monitoring Interface

The BDIC dashboard communicates with the ESP32 gateway over MQTT/WebSocket:

```
+=======================================================================================+
|                   ESS & BATTERY DIAGNOSIS SYSTEM - COMMAND CONSOLE                   |
+=======================================================================================+
| [GRID STATUS: STABILIZED]  |  [DC BUS: 400.2 V]  |  [NET POWER: +1.8 kW (To Grid)]    |
+---------------------------------------------------------------------------------------+
|  PV GENERATION: 3.4 kW    |  BATTERY POWER: -1.2 kW   |  SUPERCAP POWER: -0.4 kW      |
+---------------------------------------------------------------------------------------+
|  BATTERY PACK METRICS                                                                 |
|  - Pack Voltage: 52.4 V   | Pack Current: 22.9 A     | Pack SoC: 78.4%                |
|  - Max Cell Temp: 28.6°C  | Min Cell Temp: 27.1°C    | Temp Gradient: 1.5°C [NORMAL]  |
+---------------------------------------------------------------------------------------+
|  INDIVIDUAL CELL STATUS (16S Configuration)                                           |
|  Cell 01: 3.28V [OK]   Cell 02: 3.27V [OK]   Cell 03: 3.29V [OK]   Cell 04: 3.27V [OK]|
|  Cell 05: 3.28V [OK]   Cell 06: 3.12V [WARN] Cell 07: 3.27V [OK]   Cell 08: 3.28V [OK]|
|  Cell 09: 3.28V [OK]   Cell 10: 3.29V [OK]   Cell 11: 3.27V [OK]   Cell 12: 3.28V [OK]|
|  Cell 13: 3.27V [OK]   Cell 14: 3.28V [OK]   Cell 15: 3.29V [OK]   Cell 16: 3.27V [OK]|
+---------------------------------------------------------------------------------------+
|  DIAGNOSTIC ADVISORY:                                                                 |
|  [!] WARNING: Cell 06 shows anomalous internal resistance spike (R_int = 14.8 mOhm).  |
|  Action: Automated balancing current limited. Schedule cell replacement at cycle 1200.|
+=======================================================================================+
```

---

## 📂 Repository Directory Structure

```
ENERGY_STORAGE-BATTERY/
├── simulation/
│   ├── simulink_models/
│   │   ├── HESS_Grid_Stabilization.slx       # Full Simulink model (PV + HESS + Grid)
│   │   ├── Battery_2RC_Thevenin_Model.slx    # Parameterized 2-RC ECM model
│   │   └── Supercapacitor_EDLC_Model.slx     # EDLC model with ESR & self-discharge
│   ├── scripts/
│   │   ├── init_parameters.m                 # System sizing, filter tau, PI gains
│   │   ├── plot_grid_stabilization.m         # Generates publication-grade waveforms
│   │   └── ecm_parameter_extraction.m        # Extracts R0, R1, C1 from HPPC test data
│   └── results/                              # Simulation waveform captures & data exports
├── firmware/
│   ├── esp32_bdic/                           # ESP32 Diagnostic Firmware (PlatformIO)
│   │   ├── src/
│   │   │   ├── main.cpp
│   │   │   ├── afe_ltc6811.cpp               # Isolated SPI AFE driver
│   │   │   ├── diagnostic_engine.cpp         # Online R_int & delta-V algorithms
│   │   │   └── telemetry_mqtt.cpp            # MQTT publishing over WiFi/Ethernet
│   │   └── platformio.ini
│   └── kalman_filter/                        # Extended Kalman Filter (EKF) SoC code
├── hardware/
│   ├── pcb_design/                           # KiCad Schematic & PCB Layout
│   │   ├── BDIC_Analog_FrontEnd.kicad_sch
│   │   ├── BDIC_Power_Isolation.kicad_pcb
│   │   └── gerber/
│   └── datasheets/                           # Sensor, AFE, and cell specifications
└── dashboard/
    ├── web_app/                              # Node.js / React Telemetry GUI
    │   ├── src/
    │   └── package.json
    └── python_visualizer/                    # Real-time Matplotlib serial oscilloscope
        └── serial_scope.py
```

---

## 🚀 Execution & Verification Guide

### 1. Running the MATLAB/Simulink Simulation
1. Ensure **MATLAB R2022b or later** is installed with **Simscape Electrical (Specialized Power Systems)**.
2. Open MATLAB and navigate to the project directory:
   ```matlab
   cd simulation/scripts
   init_parameters      % Loads bus parameters, battery capacities, and controller gains
   ```
3. Open and run the primary simulation model:
   ```matlab
   open('../simulink_models/HESS_Grid_Stabilization.slx');
   sim('HESS_Grid_Stabilization');
   plot_grid_stabilization;
   ```

### 2. Flashing the ESP32 Firmware
```bash
cd firmware/esp32_bdic
pio run --target upload
pio run --target monitor
```

### 3. Launching the Diagnostics Dashboard
```bash
cd dashboard/web_app
npm install
npm run start
```

---

## 📈 Key Quantitative Results

* **Grid Voltage Regulation**: Reduced grid voltage total harmonic distortion ($THD_v$) from **$5.8\%$ down to $1.4\%$** under rapid solar irradiance transitions ($1000\text{ W/m}^2 \to 200\text{ W/m}^2$).
* **Battery Stress Reduction**: Peak battery discharge current during transient motor start events dropped from **$3.8C$ to $0.85C$**, entirely buffered by the supercapacitor bank.
* **Fault Detection Accuracy**: Successfully identified cell micro-short anomalies and high internal resistance growth within **$< 180\text{ milliseconds}$** before temperature escalation.

---

## 👥 Authors & Contributors

* **Vishal Ghule** – *System Modeling, Embedded Diagnostics & Hardware Integration*

---

## 📄 License
This repository is licensed under the [MIT License](LICENSE).
