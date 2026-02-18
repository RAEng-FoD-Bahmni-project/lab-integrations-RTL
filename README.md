# 🧬 Lab Machine Integrations for Bahmni RTL

Plug-and-play integrations for connecting laboratory analyzers and instruments to the Bahmni RTL hospital system via OpenELIS.

## Overview

This repository contains middleware, drivers, and configuration templates for integrating common lab machines into the Bahmni RTL stack. Results flow automatically from the analyzer into OpenELIS and sync to the patient record in OpenMRS.

## Supported Protocols

| Protocol | Description | Status |
|----------|-------------|--------|
| HL7 v2.x | Health Level 7 messaging for lab results | 🔜 Coming soon |
| ASTM (LIS2-A2) | Standard protocol for clinical lab instruments | 🔜 Coming soon |
| Serial/RS-232 | Direct serial connection to legacy analyzers | 🔜 Coming soon |

## Planned Integrations

- **Hematology analyzers** (CBC machines)
- **Chemistry analyzers** (liver, kidney, metabolic panels)
- **Urinalysis analyzers**
- **Coagulation analyzers**
- **Immunoassay analyzers**

## Architecture

```
┌──────────────┐     ┌──────────────────┐     ┌──────────────┐
│  Lab Machine │────▶│  Middleware /     │────▶│  OpenELIS    │
│  (Analyzer)  │     │  Interface Layer  │     │  (RTL)       │
│              │◀────│                   │◀────│              │
└──────────────┘     └──────────────────┘     └──────┬───────┘
                                                      │
                                                      ▼ Atomfeed
                                              ┌──────────────┐
                                              │  OpenMRS      │
                                              │  (Patient EMR)│
                                              └──────────────┘
```

## Contributing

We welcome contributions — especially from teams who have integrated specific lab machines with Bahmni or OpenELIS. Please open an issue or PR with your integration.

## License

AGPL-3.0 — consistent with the Bahmni RTL ecosystem.
