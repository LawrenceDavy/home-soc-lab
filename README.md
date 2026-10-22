# Home SOC Lab — Windows Sysmon and Azure Monitoring

**Status: in progress.** Started August 2026.

A personal lab to understand how endpoint events can be collected, investigated and eventually used for detections in Microsoft Sentinel. This is a learning environment, not an enterprise SOC implementation.

## Documented progress

- Set up a Windows 11 virtual machine using KVM/QEMU on my home computer.
- Installed Sysmon and a sysmon-modular configuration.
- Verified local events, including process creation and registry changes.
- Recorded installation, networking and Defender-related troubleshooting.

The local Sysmon work is documented in [the build guide](docs/02-build-guide.md) and [troubleshooting notes](docs/05-troubleshooting.md).

## Planned monitoring workflow

Windows VM → Sysmon → Azure Arc / Azure Monitor Agent → Data Collection Rule → Log Analytics → Microsoft Sentinel.

See [architecture and design notes](docs/01-architecture.md). This describes the intended end-to-end design; the current repository does **not yet provide complete reproducible evidence** for Sentinel ingestion, three working KQL analytics rules or incident triage. I will add and validate those stages as I progress.

## Repository contents

- `configs/` — Sysmon configuration adapted from the external sysmon-modular project, with attribution in the file
- `docs/` — build and troubleshooting notes
- `detections/` — reserved for KQL rule files
- `screenshots/` — reserved for validation images

## Next milestone

Confirm logs arrive in the intended Azure table, save a working KQL query with its output, then develop and test one detection rule. Document costs and clean up cloud resources after testing.
