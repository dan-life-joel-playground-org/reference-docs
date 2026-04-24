# Sophos Endpoint Protection — Deep Dive Reference

**Compiled:** 2026-04-23  
**Purpose:** Comprehensive operational reference for dealing with Sophos-managed machines  
**Context:** Dan's machines at Fike and elsewhere run Sophos Intercept X / Endpoint Protection

---

## 1. What Sophos Actually Is

Sophos is a British cybersecurity company (founded 1985) that provides endpoint protection, network security, cloud security, and encryption products. Their primary endpoint product line is:

- **Sophos Endpoint Protection** — on-premise, traditional AV + HIPS + web filtering
- **Sophos Intercept X** — cloud-managed next-gen endpoint security (the modern product)
- **Sophos Intercept X Advanced with XDR** — adds EDR/XDR capabilities
- **Sophos Intercept X Advanced with MDR Complete** — 24/7 managed detection

In a corporate setting like Fike, you're almost certainly dealing with **Intercept X** managed through **Sophos Central** (their cloud console). The admin controls policies; you as an endpoint user have limited local control.

---

## 2. Architecture — How It Hooks Into the System

### 2.1 Windows Components & Services

Sophos installs a LOT of components. Here's the full inventory of what runs on a Windows endpoint:

**Core Services:**
| Service | Process | Function |
|---------|---------|----------|
| Sophos Endpoint Defense (SED) | SEDService.exe | Core tamper protection, manages all sub-services. KERNEL-MODE DRIVER. Cannot be stopped. |
| Sophos MCS Agent | MCSAgent.exe | Communicates with Sophos Central for policy/reporting |
| Sophos MCS Client | MCSClient.exe | Client-side of central management communication |
| Sophos File Scanner | SophosFileScanner.exe | On-access and on-demand file scanning (MAJOR performance hog) |
| Sophos System Protection Service | SSPService.exe | Behavioral detection engine (removed in agent 2025.2+ for Win10 x64+) |
| Sophos Network Threat Protection | SophosNtpService.exe | Network traffic interception, malicious traffic detection |
| Sophos Health Service | SophosHealth.exe | Reports device health to Central |
| Sophos Live Query | SophosLiveQuery.exe | Enables remote queries from admin console |
| Sophos Live Terminal | SophosLiveTerminal.exe | Enables remote terminal access by admin |
| HitmanPro.Alert | hmpalert.exe | Anti-exploit, anti-ransomware (CryptoGuard). Acquired 2015. |
| Sophos AutoUpdate | ALUpdate.exe | Handles update downloads |
| Sophos Endpoint Firewall Management | — | Local firewall rules management |
| Sophos Endpoint UI | SophosUI.exe | Tray icon and local user interface |
| Sophos Antimalware Scan Interface Protection | — | AMSI integration |
| Sophos Self Help Tool | — | Diagnostic/troubleshooting UI |
| Sophos Diagnostic Utility (SDU) | — | Log collection for support |
| Sophos Management Gateway | — | Communication gateway |
| Machine Learning Engine | — | Deep learning malware detection |
| Threat Detection Engine | — | Signature + heuristic scanning engine |
| Competitor Removal Tool | — | Auto-removes other AV products on install |

**Key directories:**
- `C:\Program Files\Sophos\` — main installation
- `C:\Program Files\Sophos\Endpoint Defense\` — SED, SEDcli.exe, sophosinterceptxcli.exe
- `C:\Program Files\Sophos\Sophos Endpoint Agent\` — SophosUninstall.exe
- `C:\ProgramData\Sophos\` — config, logs, caches
- `C:\ProgramData\Sophos\Endpoint Defense\logs\` — SSP.log and other debug logs

**Key registry locations:**
```
HKLM\SYSTEM\CurrentControlSet\Services\Sophos Endpoint Defense\
HKLM\SYSTEM\CurrentControlSet\Services\Sophos Endpoint Defense\TamperProtection\Config
HKLM\SYSTEM\CurrentControlSet\Services\Sophos Endpoint Defense\TamperProtection\Services
HKLM\SYSTEM\CurrentControlSet\Services\Sophos Endpoint Defense\Scanning\Config
HKLM\SYSTEM\CurrentControlSet\Services\Sophos MCS Agent
HKLM\SOFTWARE\WOW6432Node\Sophos\SAVService\TamperProtection
```

**Windows user groups created:** SophosUser, SophosPowerUser, Sophos Administrator — DO NOT DELETE.

### 2.2 Linux Components

On Linux, Sophos installs as "Sophos Protection for Linux" (server-class license):

- Default install path: `/opt/sophos-spl/`
- AV scanner process: `/opt/sophos-spl/plugins/av/sbin/soapd`
- Log location: `/opt/sophos-spl/plugins/av/log/av.log`
- Status check: `systemctl status sophos-spl`
- On-demand scanner: `avscanner` (command-line tool)
- Installation via: `SophosSetup.sh` (thin installer, downloads components)
- Custom install dir: `sudo ./SophosSetup.sh --install-dir=/path`
- Debug install: `sudo OVERRIDE_INSTALLER_CLEANUP=1 DEBUG_THIN_INSTALLER=1 bash -x ./SophosSetup.sh 2>&1 | tee install.log`

Linux endpoint installers are found under "Server" in Sophos Central, NOT under "Endpoint" (which is Windows/Mac only).

---

## 3. Protection Layers — What It Actually Does

### 3.1 File Scanning (On-Access)
- Every file read/write is intercepted by `SophosFileScanner.exe`
- Uses signature matching + machine learning (deep learning model trained on millions of malware samples)
- **THIS IS THE #1 PERFORMANCE KILLER FOR DEVELOPERS**
- Scans every `.exe`, `.dll`, `.py`, `.js`, archive, etc. as they're touched
- During builds, compiles, and `npm install`, it scans EVERY intermediate file

### 3.2 Behavioral Detection (HIPS/SSP)
- `SSPService.exe` monitors process behavior in real-time
- Watches for: unusual child processes, registry modifications, file encryption patterns
- Can flag legitimate developer tools as suspicious (Python scripts, PowerShell, custom builds)
- Generates events sent to SSPService.exe that consume CPU even with file exclusions

### 3.3 Web Interception & HTTPS Decryption
- Intercepts ALL outbound browser connections
- Checks URLs against SophosLabs reputation database
- **SSL/TLS decryption**: Can decrypt HTTPS traffic for deeper inspection
- This can break: client certificate auth, sites with unusual encryption, websockets
- Controlled via Sophos Central: Global Settings → "SSL/TLS decryption of HTTPS websites"
- Firefox users: enable `security.enterprise_roots.enabled` to work with Sophos's CA cert

### 3.4 Malicious Traffic Detection (MTD)
- Intercepts traffic from NON-BROWSER processes
- Analyzes whether traffic is destined for C2 servers
- Can interfere with: SSH connections, Git over SSH, custom network tools, API calls

### 3.5 Exploit Prevention (HitmanPro.Alert)
- Dynamic shellcode protection — detects covert remote access agents
- CryptoGuard — monitors for file encryption patterns (ransomware)
- Safe Browsing — protects browser internals
- Credential theft prevention — blocks LSASS memory access
- Code cave detection — flags injected code in legitimate processes
- APC violation prevention
- Privilege escalation prevention
- **False positives common with**: Python scripts via VBA, custom executables, development tools

### 3.6 Application Control
- Can block applications by category or name
- Blocks "potentially unwanted applications" (PUAs) which can include developer tools

### 3.7 Device Control
- Controls USB/removable media access
- Can block external drives, USB devices

### 3.8 Data Loss Prevention (DLP)
- Restricts unauthorized data flow via pre-built or custom rules

---

## 4. Tamper Protection — The Iron Cage

This is the core "aggressive" feature that makes Sophos hard to deal with:

### What It Does
- Prevents modification or stopping of ANY Sophos service
- Prevents uninstalling Sophos without the password
- Prevents changing scan settings locally
- Enforced by a **kernel-mode driver** (Sophos Endpoint Defense)
- **ON by default**, and Sophos recommends keeping it on

### Who Controls It
- ONLY a Sophos Central administrator with MFA can:
  - View the tamper protection password
  - Generate a new password
  - Temporarily disable it for a specific device
- Local admins CANNOT override this by design
- Domain admins CANNOT override this by design

### The Password
- Auto-generated per-device by Sophos Central
- Required to disable tamper protection locally
- Access: Sophos Central → device details → Tamper Protection → View password
- Passwords stored 120 days after device deletion

### Local Disable (If You Have the Password)
```cmd
cd "C:\Program Files\Sophos\Endpoint Defense"
SEDcli.exe -OverrideTPoff <password>
```
This gives you a 4-hour window.

### Emergency Recovery (No Password, No Central Access)
Must be done in **Safe Mode** or **WinPE**:

1. Boot into Safe Mode
2. Stop Sophos Anti-Virus service (services.msc → Disabled)
3. Registry edits:
```
HKLM\SYSTEM\CurrentControlSet\Services\Sophos Endpoint Defense\TamperProtection\Config
  → SAVEnabled = 0
  → SEDEnabled = 0

HKLM\SOFTWARE\WOW6432Node\Sophos\SAVService\TamperProtection
  → Enabled = 0

HKLM\SYSTEM\CurrentControlSet\Services\Sophos MCS Agent
  → Start = 4 (disabled)
```
4. Under `\TamperProtection\Services\` — set `Protected = 0` for every subkey
5. Reboot normally

### SCCM/Mass Recovery (for fleet operations)
One proven approach: Use SCCM task sequence to:
1. Suspend BitLocker
2. Reboot into WinPE
3. Mount installed OS registry hives
4. Apply registry changes
5. Disable MCS service to prevent re-enabling
6. Reboot to normal mode

### GitHub Tool: No_Sophos_TamperProtection
`github.com/thomasbad/No_Sophos_TamperProtection` — GUI/CLI tool that automates the WinPE registry approach. Run from WinPE or Safe Mode environment.

---

## 5. Developer Survival Guide — Exclusions & Performance

### 5.1 The Problem

Developers consistently report:
- Build times 2x+ longer with Sophos active
- `SophosFileScanner.exe` consuming 20%+ CPU during builds
- `SSPService.exe` consuming CPU for behavioral events even with file exclusions
- HitmanPro service causing significant CPU load independent of exclusion settings
- Python scripts getting blocked by exploit prevention
- Git operations blocked by web protection or malicious traffic detection
- SSH connections flagged or dropped

### 5.2 Exclusion Types (from Sophos Central)

Three types of exclusions, each with different scope:

**1. File/Folder Exclusions** — Exclude paths from on-access scanning
- Example: `C:\dev\`, `C:\Users\*\node_modules\`
- Risky: anything placed in excluded folders is unscanned

**2. Process Exclusions** — Exclude files accessed BY a specific process
- Example: `C:\Program Files\Microsoft Visual Studio\...\devenv.exe`
- Better: still scans the process itself, just not files it touches
- Behavioral scanning still occurs
- Also excludes network communication by that process

**3. Detection ID / SHA Exclusions** — Allow specific detected items
- Best for false positives on specific known-good files
- Uses SHA-256 hash or Sophos Detection ID
- Won't match if the file changes

### 5.3 Recommended Developer Exclusions

**Process exclusions (prefer these):**
- `devenv.exe` (Visual Studio)
- `MSBuild.exe`
- `dotnet.exe`
- `node.exe`
- `python.exe` / `python3.exe`
- `javac.exe` / `java.exe`
- `gcc.exe` / `g++.exe` / `make.exe`
- `cmake.exe`
- `cargo.exe` (Rust)
- `go.exe`
- `git.exe`
- `ssh.exe`
- IDE-specific processes

**Folder exclusions (use sparingly):**
- Build output directories
- Package manager caches (`node_modules`, `.gradle`, `.cargo`, `.pip-cache`)
- Source code directories (if you trust them)
- Docker volumes
- Virtual machine images

### 5.4 How to Verify Exclusions Are Applied

1. **Registry check:** `HKLM\SYSTEM\CurrentControlSet\Services\Sophos Endpoint Defense\Scanning\Config`
2. **Endpoint Self Help Tool:** Shows policy application timestamps under "Management Communication" and "Policy" tabs
3. **EICAR test:** Place EICAR test file in excluded directory — if it survives, exclusion works
4. **Process Monitor:** Use Sysinternals ProcMon to see if Sophos is still scanning your files

### 5.5 The HitmanPro Problem

HitmanPro's CPU usage is NOT reduced by file/folder/process exclusions in Sophos Central. The behavioral monitoring operates at a different layer. If HitmanPro is hammering CPU:
- Option 1: Ask Sophos admin to disable specific exploit mitigations for your policy
- Option 2: As a last resort, stop the HitmanPro.Alert service (requires tamper protection off)
- Option 3: Accept the overhead and resize expectations

### 5.6 The 4-Hour Override

From the local Sophos UI (Settings tab):
- Check "Override Sophos Central Policy for up to 4 hours to troubleshoot"
- This lets you temporarily toggle: real-time scanning, website blocking, exploit detection
- Changes persist across reboots but expire after 4 hours
- Useful for running a critical build

---

## 6. Command-Line Tools

### sophosinterceptxcli.exe
Location: `C:\Program Files\Sophos\Endpoint Defense\`
```cmd
sophosinterceptxcli.exe query configuration on_access_scan_enabled
sophosinterceptxcli.exe query configuration on_access_scan_enabled --json
```
Currently only outputs `on_access_scan_enabled` setting.

### SEDcli.exe
Location: `C:\Program Files\Sophos\Endpoint Defense\`
```cmd
SEDcli.exe -OverrideTPoff <password>    # Disable tamper protection for 4h
```

### SophosUninstall.exe
Location: `C:\Program Files\Sophos\Sophos Endpoint Agent\`
```cmd
SophosUninstall.exe    # Interactive uninstall (requires tamper protection off)
```

### SophosZap.exe
Last-resort cleanup tool. Download from Sophos support.
- Requires tamper protection to be OFF first
- Uses heuristics to find and remove all Sophos components
- Run from admin command prompt

### Linux: avscanner
```bash
/opt/sophos-spl/plugins/av/sbin/avscanner /path/to/scan    # On-demand scan
systemctl status sophos-spl                                  # Check service status
```

---

## 7. Sophos Central — What the Admin Sees

If you need to request changes from your Sophos admin, here's what they can do:

### Global Settings
- **Tamper Protection:** On/off globally, per-device password management
- **SSL/TLS decryption:** On/off, hostname/IP exclusions
- **Global Exclusions:** File, folder, process, website exclusions applied to all devices

### Threat Protection Policy
- On-access scanning: on/off, file types
- Scheduled scans: timing, scope
- Deep learning: on/off
- Behavioral detection (HIPS): settings
- Web protection: on/off
- Malicious traffic detection: on/off
- Exploit mitigations: per-technique enable/disable
- CryptoGuard: on/off, response action
- Application control: categories and names
- Device control: peripheral rules
- DLP: rules and actions

### Per-User/Group Policies
- Policies can be applied to specific users, groups, or devices
- Ask admin to create a "Developer" policy with relaxed scanning
- This is the RIGHT way to handle it — don't fight the tool, get proper policy

---

## 8. Uninstallation

### Standard (from Programs and Features)
Requires tamper protection password. Single unified uninstaller removes all components.

### Command Line
```cmd
cd "C:\Program Files\Sophos\Endpoint Defense"
SEDcli.exe -OverrideTPoff <password>
cd "C:\Program Files\Sophos\Sophos Endpoint Agent"
SophosUninstall.exe
```

### SophosZap (Nuclear Option)
1. Disable tamper protection first
2. Download SophosZap from Sophos support
3. Run from admin command prompt
4. May require reboot
5. On Win 10+ / Server 2016+, you may NOT need the tamper protection password for deleted/expired devices

### PowerShell Removal Script
Community tool: `github.com/ayeskatalas/Sophos-Removal-Tool`
- Disables tamper protection via registry
- Finds all Sophos uninstall strings via registry
- Silently removes each component
- Cleans up services, directories (Program Files, ProgramData)
- Must run as admin with Sophos Admin or Local Admin rights

### Post-Uninstall
- Windows Defender may not restart automatically (especially Win 8.1)
- Start Defender manually if needed
- Check for residual services in services.msc
- Check for residual registry entries under `HKLM\SOFTWARE\Sophos\` and `HKLM\SYSTEM\CurrentControlSet\Services\Sophos*`

---

## 9. Common Pain Points & Solutions

### Git operations blocked
- Sophos web protection can intercept Git HTTPS traffic
- Sophos MTD can flag Git SSH traffic
- Solution: Process exclusion for `git.exe` and `ssh.exe`
- For firewall-level Sophos (UTM/XG): may need firewall rule exceptions

### Python scripts blocked
- Exploit prevention flags Python-via-VBA or unusual Python behavior
- Solution: Exclude by Detection ID if consistent, or add Python to exploit mitigation exclusions in policy

### SSL certificate errors in browsers
- Sophos HTTPS decryption acts as MITM, installs its own CA cert
- Firefox: enable `security.enterprise_roots.enabled` in about:config
- Or: ask admin to add hostname exclusions for problematic sites
- Client certificate sites: will NOT work without exclusion

### Build performance
- Add process exclusions for compiler, linker, IDE processes
- Add folder exclusions for build output directories
- Move scheduled scans to off-hours (admin controls this)
- Check if `SophosFileScanner.exe` or `SSPService.exe` is the bottleneck
- Enable debug logging: Endpoint Self Help → SophosFileScanner.exe → Scan Summaries → Debug

### Custom executables flagged
- Machine learning model flags unfamiliar executables
- Solution: Allow by certificate (recommended) or SHA-256 hash in Sophos Central
- Admin path: My Environment → Computers → device → Events → detection → Details → Allow

---

## 10. Sophos Update Architecture

Sophos deliberately does NOT use MSI for updates. Their proprietary update system:
- Keeps tamper protection active during ALL updates
- Protection remains active during upgrades and downgrades
- Cannot be interrupted by standard Windows installer tools
- Update caches and message relays can be configured for network efficiency

---

## 11. Key URLs

- Sophos Central Admin: https://central.sophos.com
- Sophos Docs: https://docs.sophos.com
- Community Forum: https://community.sophos.com
- Release Notes (Endpoint): https://docs.sophos.com/releasenotes/output/en-us/esg/sesc_core_rn.html
- Release Notes (Server Core): https://docs.sophos.com/releasenotes/output/en-us/esg/sesc_servercore_rn.html
- Tamper Protection Docs: https://docs.sophos.com/central/customer/help/en-us/ManageYourProducts/GlobalSettings/TamperProtection/
- Exclusions Guide: https://docs.sophos.com/central/customer/help/en-us/ManageYourProducts/GlobalSettings/GlobalExclusions/UsingExclusions/
- False Positives: https://docs.sophos.com/central/customer/help/en-us/ManageYourProducts/Alerts/ThreatAdvice/FalsePositive/
- Allowed Applications: https://docs.sophos.com/central/customer/help/en-us/ManageYourProducts/GlobalSettings/AllowedApplications/
- Linux Agent: https://docs.sophos.com/central/customer/help/en-us/PeopleAndDevices/ProtectDevices/ServerProtection/SophosProtectionLinux/
- Tamper Protection Recovery (community): https://martinsblog.dk/sophos-endpoint-defense-how-to-recover-a-tamper-protected-system/
- Removal Tool (community): https://github.com/ayeskatalas/Sophos-Removal-Tool
- Tamper Protection Bypass (community): https://github.com/thomasbad/No_Sophos_TamperProtection

---

## 12. TL;DR for a Developer Under Sophos

1. **You can't uninstall it** without the admin password.
2. **You can't stop its services** without disabling tamper protection.
3. **Your best bet** is getting the admin to create a developer-specific policy with:
   - Process exclusions for your IDE, compiler, Git, SSH, Python, Node
   - Folder exclusions for build output and package caches
   - Scheduled scans moved to off-hours
   - Optionally: specific exploit mitigations relaxed for dev tools
4. **4-hour override** from local UI lets you temporarily toggle features for troubleshooting.
5. **If builds are slow**, check whether `SophosFileScanner.exe` or `SSPService.exe` is the culprit via Task Manager. File scanner = need more exclusions. SSP = behavioral monitoring, harder to fix.
6. **HitmanPro** CPU overhead is NOT fixable via exclusions — it operates at a different layer.
7. **HTTPS issues** = Sophos is decrypting your traffic. Ask admin for hostname exclusions or disable SSL/TLS decryption for your device.
8. **Git/SSH blocked** = process exclusion for git.exe and ssh.exe.
