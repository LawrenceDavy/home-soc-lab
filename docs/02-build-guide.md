## Milestone 2 — Sysmon and sysmon-modular (4 August 2026)

**Installed:** I installed **Sysmon v15.21** from Microsoft Sysinternals (released **17 June 2026**). Before installing it, I checked the digital signature with `Get-AuthenticodeSignature` to make sure the file was genuine. The signature was valid and signed by **Microsoft Corporation**.

**Configuration:** I used the default balanced configuration from **sysmon-modular**, downloaded from the `olafhartong/sysmon-modular` GitHub repository on **4 August 2026**. Since the project doesn't have official releases, I recorded the SHA256 hash so I know exactly which version of the configuration I used.

**SHA256:**

```
4516404fa30ee87cea558567820cdc78863cc4ab07889519e49eac3cca92e0d2
```

**Install command:**

```powershell
.\Sysmon64.exe -accepteula -i C:\Sysmon\sysmonconfig.xml
```

The `-accepteula` option automatically accepts the Sysinternals licence, and `-i` installs Sysmon using my configuration file straight away. After the installation, I confirmed that both the **Sysmon64** service and the **SysmonDrv** driver were running correctly.

**Operational log size:** Before moving on to Azure, I increased the Sysmon Operational log to **1 GiB** using `wevtutil`. The default log size fills up quite quickly with the sysmon-modular configuration, and I didn't want older events being overwritten before I had the Azure Monitor Agent collecting them.

**Local testing:** Before connecting anything to Azure, I made sure Sysmon was logging events correctly.

- **Event ID 1 (Process Creation)** was created when I ran my PowerShell test. The full command line, including the Base64 encoded command, was recorded correctly. Sysmon also tagged it with **MITRE ATT&CK T1059.001**.
- **Event ID 13 (Registry Value Set)** was created when I added my test Run key. The event showed the correct registry path ending in `CurrentVersion\Run\M2LabTest` and was tagged with **MITRE ATT&CK T1547.001**.
- My `certutil` attack simulation was blocked by Microsoft Defender, so that specific test couldn't run. However, running `certutil.exe -?` still created an Event ID 1, confirming that Sysmon was monitoring the executable correctly. More details about this are in `docs/05-troubleshooting.md`.

**Why I tested locally first:** I wanted to make sure Sysmon was working before introducing Azure into the setup. That way, if logs don't appear in Microsoft Sentinel later, I'll already know the problem isn't with Sysmon itself and can focus on troubleshooting the Azure side of the pipeline instead.