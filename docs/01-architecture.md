# Architecture

## What this lab is

This lab is my own home SOC that I built to learn how security monitoring works in Azure. I used a Windows 11 virtual machine running on my Linux PC and connected it to Microsoft Azure. The idea is to collect logs from Windows, send them to Azure, and then use Microsoft Sentinel to find anything suspicious. I built everything myself so I could understand how each part works instead of just following a demo.

## Data flow

```
┌──────────────────────────────────────┐
│  CachyOS host (Ryzen 7 7800X3D)      │
│                                      │
│  ┌────────────────────────────────┐  │
│  │  KVM/QEMU virtual machine      │  │
│  │  Windows 11 Pro                │  │
│  │                                │  │
│  │  Sysmon (sysmon-modular cfg)   │  │
│  │       ↓ writes to              │  │
│  │  Microsoft-Windows-Sysmon/     │  │
│  │        Operational channel     │  │
│  │       ↓ read by                │  │
│  │  Azure Monitor Agent (AMA)     │  │
│  │       ↑ installed via          │  │
│  │  Azure Connected Machine Agent │  │
│  │        (Azure Arc)             │  │
│  └────────────┬───────────────────┘  │
│    NAT network 192.168.122.0/24      │
└───────────────┼──────────────────────┘
                │ HTTPS 443 outbound only
                ▼
┌──────────────────────────────────────┐
│  Microsoft Azure                     │
│                                      │
│  Azure Arc-enabled server object     │
│       ↓ scoped by                    │
│  Data Collection Rule (DCR)          │
│    XPath filter on Sysmon channel    │
│       ↓ writes to                    │
│  Log Analytics Workspace             │
│    → Event table                     │
│       ↓ queried by                   │
│  Microsoft Sentinel                  │
│    3 scheduled analytics rules (KQL) │
│       ↓ produces                     │
│  Incidents                           │
└──────────────────────────────────────┘
```

## Design decisions

### 1. Why Azure Arc?

My Windows VM isn't actually running in Azure, so Azure Arc lets Azure see it like it is. That means I can install the Azure Monitor Agent and manage it using Azure just like a normal Azure virtual machine.

### 2. Why Azure Monitor Agent (AMA), not the legacy Log Analytics agent (MMA)?

I used Azure Monitor Agent because it's the one Microsoft supports now. The older MMA agent isn't supported anymore, so there wasn't much point using outdated software for this project.

### 3. Why a custom Data Collection Rule?

Sysmon saves its logs in a different place from normal Windows Security logs. Because of that, I had to make my own Data Collection Rule so Azure would collect the right events and send them to my Log Analytics Workspace. Sysmon events land in the generic `Event` table rather than `SecurityEvent`, which matters later because every KQL query in this lab has to target `Event`, not the table most Sentinel tutorials assume.

### 4. Why NAT networking, not bridged?

I wanted to keep the virtual machine separate from the rest of my home network because I'll be running suspicious commands during testing. NAT lets the VM access the internet but stops other devices on my network from connecting straight to it.

## Out of scope

This lab doesn't include:

- NIST SP 800-53 or ISO 27001/27002 because this project is about learning SOC skills, not compliance.
- MITRE D3FEND, OWASP, or CIS Controls because they aren't really needed for what this lab is trying to do.
- A full risk assessment or risk register because that's usually done in GRC roles, not by junior SOC analysts.
- Enterprise network segmentation because one isolated virtual machine is enough for this project.

See `06-lessons-learned.md` for things I would change if I was building this for a real company.