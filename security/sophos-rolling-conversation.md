# Sophos Research — Rolling Conversation Log

**Started:** 2026-04-23  
**Participants:** Dan, Joel  
**Context:** Dan's machines at Fike run Sophos Intercept X. Need to understand what it does, how to work around it, and what admins can see.

---

## Session 1 — Thursday, April 23, 2026 — 7:16 PM CDT

### Initial Request
Dan needs a comprehensive deep dive on Sophos endpoint protection. Machines at Fike are managed by Sophos and it interferes with development work.

### Deliverable
Created `security/sophos-deep-dive.md` in `dan-life-joel-playground-org/reference-docs` — 459-line comprehensive technical reference covering architecture, services, tamper protection, developer exclusions, uninstallation, and community tools.

---

## Session 1 (continued) — Thursday, April 23, 2026 — 8:02 PM CDT

### Question 1: Randy's VM Suggestion — Does It Work?

**Randy (cybersecurity head) recommended using VMs to avoid Sophos interference.**

#### The Short Answer
Randy is right. VMs are an effective workaround. Sophos on the host cannot see inside a VM's guest filesystem, processes, or memory.

#### The Full Picture

**What Sophos CAN'T see from the host:**
- Individual files inside the VM's guest OS filesystem
- Processes running inside the VM
- Memory contents of the VM
- Network traffic details inside the VM (in most configurations)
- What you're building, compiling, or installing inside the VM

Sophos themselves published a blog post (April 2026) about threat actors abusing QEMU VMs, in which they explicitly state that activity within a virtual machine is "essentially invisible to endpoint security controls and leaves little forensic evidence on the host itself." That's Sophos admitting the limitation of their own product.

**What Sophos CAN still affect from the host:**
1. **VM disk file scanning** — Sophos's SophosFileScanner on the host sees your .vmdk, .vhdx, or .qcow2 file as a big binary blob. When the VM writes to its virtual disk, the host file changes, and Sophos may scan it. This can cause I/O performance degradation on the host side. Mitigation: add the VM disk file paths as folder/file exclusions in Sophos policy.
2. **Network traffic interception (NAT mode)** — If your VM's network adapter is set to NAT, traffic routes through the host's network stack. Sophos Network Threat Protection on the host intercepts non-browser network traffic and Web Protection intercepts browser traffic. This means Sophos could flag SSH connections, block URLs, or interfere with package downloads FROM the VM if running through NAT. Mitigation: use **Bridged networking** instead of NAT — this puts the VM directly on the physical network, bypassing the host's network stack and Sophos's traffic interception.
3. **Host resource contention** — Sophos services running on the host still consume CPU and memory, leaving less for your VM. But they won't be scanning your build artifacts.

**One critical exception — Sophos for Virtual Environments (SSVM):**
Sophos offers a product called "Sophos for Virtual Environments" that deploys a Security VM on the hypervisor host and thin agents inside each guest VM. If THIS is deployed, Sophos CAN scan inside VMs. However:
- This requires a separate, deliberate deployment by the admin
- It uses a thin guest agent inside each VM
- It's designed for VMware vSphere and Hyper-V enterprise environments
- If your VM doesn't have a Sophos agent installed inside it, this doesn't apply
- The thin agent is a SCANNING agent only — less policy control than full Intercept X

**Bottom line for Dan's workflow:**
- Spin up a Linux VM (VirtualBox, Hyper-V, or VMware Workstation)
- Use Bridged networking
- Don't install any Sophos agent inside the VM
- Do all your builds, tool installs, Python work, Git operations INSIDE the VM
- Sophos on the host won't interfere
- Ask Randy to exclude your VM disk files from Sophos scanning to avoid I/O slowdowns

---

### Question 2: Can Sophos Admins Remote Into Your Machine or See Your Screen?

#### Screen Viewing / Screen Capture
**No.** Sophos has NO screen viewing, screen sharing, or screen capture capability. Period. It cannot see what's on your display.

#### What They CAN Do

**1. Live Response (Remote Command Line)**
- Available with XDR/EDR license
- Gives a Sophos Central Super Admin a **full elevated CMD session** on your machine
- Requires: Super Admin role + MFA + Live Response enabled in Global Settings
- They can: browse your filesystem, view running processes, read log files, edit registry, install/uninstall software, stop processes, reboot the machine
- They CANNOT: see your screen, interact with your GUI, see what's in your browser, watch you work
- The session runs through Sophos's cloud (Sophos Central), not direct network connection
- Components involved: `Sophos-live-terminal.exe` and `Sophos-winpty-agent.exe` running in Task Manager
- You can check if these are running in Task Manager right now

**2. Live Discover / Live Query (Data Queries)**
- SQL-based querying system — admin writes SQL queries, runs them against your endpoint
- Can query: running processes (with network connections, hashes, paths), OS info, registry entries, installed software, file existence/hashes, open network connections, user accounts, browser settings
- Can expose "shadow IT" — unauthorized software installations
- The data is processed ON YOUR DEVICE and sent to Central
- Pre-built queries exist for things like "Processes with open network connections" — can see what apps are phoning home
- **Data Lake uploads** — your endpoint continuously uploads telemetry data to Sophos's cloud. This includes process execution history, file activity, network connections, etc. This data is queryable EVEN AFTER the fact.

**3. Device Isolation**
- Admin can remotely isolate your machine from the network (allows only Sophos Central communication)
- Used during incident response

**4. Remote Scanning**
- Admin can trigger on-demand scans remotely

**5. What They See in the Dashboard (Always, Passively)**
- Device health status (green/yellow/red)
- Security events and detections (malware found, exploits blocked, etc.)
- Software inventory
- OS version and patch status
- Last communication time
- Policy compliance status

#### What They CANNOT Do
- See your screen
- See your browser history (unless web events are logged — but these are URL-level, not page content)
- Read the content of your files (they can see file names and hashes, not file content)
- See your keystrokes
- Listen through your microphone
- Access your camera
- Take screenshots
- Record your activity in real-time

#### The VM Angle on Privacy
If you're working inside a VM without a Sophos agent: Live Response, Live Discover, and all telemetry collection only apply to the HOST. They cannot reach into your VM. Processes running inside the VM don't appear in the host's process list that Sophos queries. Files inside the VM's virtual disk are opaque to Sophos. This is another strong argument for Randy's VM approach.

---

### Action Items
- [ ] Dan: Talk to Randy about getting VM disk file exclusions added to Sophos policy
- [ ] Dan: Set up development VM (Linux?) with Bridged networking
- [ ] Dan: Verify whether Sophos for Virtual Environments (SSVM) is deployed at Fike
- [ ] Dan: Check Task Manager for Sophos-live-terminal.exe and Sophos-winpty-agent.exe to see if Live Response is active
- [ ] Joel: Continue researching specific Sophos issues as they come up

---

*This is a living document. New sessions will be appended below.*
