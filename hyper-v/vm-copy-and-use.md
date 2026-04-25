# Hyper-V VM Copy & Use — Quick Reference

**Rolling doc — updated as we learn things**
**Last updated:** 2026-04-24
**Example VM:** `Win-10-Encrypted`
**Host:** `DAN-PDS-WIN11`
**VM storage root:** `D:\records\vm\`

---

## Option A: Folder Copy (faster, works when conditions are clean)

Use this when the VM is **shut down** and has **no checkpoints**.

### 0. Turn off TPM (if distributing)

In Hyper-V Manager: VM Settings → Security → uncheck "Enable Trusted Platform Module." Do this before anything else.

### 1. Verify the VM is off

```powershell
Get-VM -Name "Win-10-Encrypted" | Select-Object Name, State
```

If running, shut down from inside the guest or:

```powershell
Stop-VM -Name "Win-10-Encrypted"
```

### 2. Verify no checkpoints exist

```powershell
Get-VMCheckpoint -VMName "Win-10-Encrypted"
```

If checkpoints exist, delete them first (this triggers a merge — wait for it to finish):

```powershell
Remove-VMCheckpoint -VMName "Win-10-Encrypted" -Name *
```

### 3. Copy the VM folder

```powershell
Copy-Item -Path "D:\records\vm\win10-encrypted" -Destination "D:\records\vm\win10-encrypted-copy" -Recurse
```

Adjust the source path to wherever your VM folder actually lives. Check in Explorer if unsure — you're looking for the folder that contains `Virtual Hard Disks\` and `Virtual Machines\`.

### 4. Import as a new VM with a new ID

```powershell
$vmcx = Get-ChildItem "D:\records\vm\win10-encrypted-copy" -Recurse -Filter "*.vmcx" | Select-Object -First 1
Import-VM -Path $vmcx.FullName -Copy -GenerateNewId
```

`-Copy` makes it independent. `-GenerateNewId` gives it a new identity so both VMs can coexist.

### 5. Rename the copy

Check what Hyper-V named it:

```powershell
Get-VM
```

Then rename:

```powershell
Rename-VM -Name "Win-10-Encrypted" -NewName "Win-10-Encrypted-COPY"
```

(Use whichever name is the copy — the import may keep the original name.)

### 6. Detach any ISO from the copy

```powershell
Get-VMDvdDrive -VMName "Win-10-Encrypted-COPY" | Set-VMDvdDrive -Path $null
```

### 7. Start it

```powershell
Start-VM -Name "Win-10-Encrypted-COPY"
```

---

## Option B: Export/Import (slower, but handles edge cases automatically)

Use this when you're not sure about checkpoints, or when you want a guaranteed clean portable package.

### 0. Turn off TPM (if distributing)

Same as Option A — VM Settings → Security → uncheck TPM. Do this before export.

### 1. Shut down the VM (same as above)

### 2. Merge checkpoints (same as above)

### 3. Turn off automatic checkpoints

```powershell
Set-VM -Name "Win-10-Encrypted" -AutomaticCheckpointsEnabled $false
```

### 4. Export

```powershell
Export-VM -Name "Win-10-Encrypted" -Path "D:\records\vm\exports"
```

**This takes a while.** It copies the entire VHDX. Go get coffee.

### 5. Import as copy with new ID

```powershell
$vmcx = Get-ChildItem "D:\records\vm\exports\Win-10-Encrypted" -Recurse -Filter "*.vmcx" | Select-Object -First 1
Import-VM -Path $vmcx.FullName -Copy -GenerateNewId -VhdDestinationPath "D:\records\vm\Win-10-Encrypted-COPY\Virtual Hard Disks" -VirtualMachinePath "D:\records\vm\Win-10-Encrypted-COPY"
```

### 6. Rename, detach ISO, start (same as Option A steps 5-7)

### 7. Clean up the export staging folder

```powershell
Remove-Item -Path "D:\records\vm\exports\Win-10-Encrypted" -Recurse -Force
```

---

## Full Distribution Workflow

This is the end-to-end process for making a distributable copy of a VM. Your source VM is never modified by this process (except temporarily turning off TPM, which you turn back on when done).

### Prepare the source VM (do this once, before any copies)

Boot the source VM and do the following inside the guest:

**1. Install all Windows updates.** Settings → Update & Security → Check for updates. Install everything. Reboot. Repeat until there are no more updates. This can take a long time, but do it once on the source and every copy inherits a fully patched OS — you never have to do it again per copy.

**2. Disable reserved storage.** From an elevated PowerShell inside the guest:

```powershell
DISM /Online /Set-ReservedStorageState /State:Disabled
```

Verify:

```powershell
DISM /Online /Get-ReservedStorageState
```

Reserved storage holds disk space for future updates. If it's active when sysprep runs, sysprep fails with `0x800F0975`. Disabling it on the source means every copy is already clean.

**3. Shut down the source VM.**

### On the source host, before copying

**4. Turn off TPM on the source VM.**
VM Settings → Security → uncheck "Enable Trusted Platform Module." A vTPM is cryptographically tied to the host it was created on. A recipient on a different host cannot unwrap the key protector and the VM will not boot. This is a required step for any VM you intend to distribute.

**5. Copy the VM** using Option A (folder copy) or Option B (export/import) from above.

**6. Turn TPM back on your source** if you want it. Your source is done — it goes back to normal.

### On the copy (the one going out the door)

**7. Boot the copy.**

**8. Inside the copy, run sysprep.** Open an elevated PowerShell or cmd inside the guest:

```
C:\Windows\System32\Sysprep\sysprep.exe /generalize /oobe /shutdown
```

This strips the machine SID (security identifier), resets Windows activation, and shuts the VM down automatically. Without this step, every copy has the same SID — two machines with the same SID on the same network cause authentication collisions and unpredictable security behavior.

**If sysprep fails**, check the log:

```powershell
Get-Content C:\Windows\System32\Sysprep\Panther\setupact.log | Select-String "SYSPRP" | Select-Object -Last 30
```

Common blockers:
- **`0x800F0975` — reserved storage in use.** A pending update or servicing operation is in the way. Install all updates, reboot, then disable reserved storage with `DISM /Online /Set-ReservedStorageState /State:Disabled`. This is why we do it on the source first.
- **Appx package blocking generalization.** The log will name the package. Remove it with `Get-AppxPackage -Name "*package.name*" | Remove-AppxPackage`, retry.

**9. Do not boot the copy again.** After sysprep shuts it down, the image is sealed. The next boot will run the Windows out-of-box experience (OOBE) — pick language, create user account, etc. That next boot is for the recipient, not you.

**7. (Optional) Compact the VHDX.** With the copy shut down:

```powershell
Optimize-VHD -Path "D:\records\vm\<copy>\Virtual Hard Disks\<disk>.vhdx" -Mode Full
```

This reclaims free space inside the VHDX and shrinks the file.

### What the recipient does

**8. Import the VM** on their Hyper-V host (folder copy or export, same as above).

**9. Attach a fresh vTPM** if they want encryption. VM Settings → Security → check "Enable Trusted Platform Module." Hyper-V generates a brand new vTPM tied to their host.

**10. Boot the VM.** Windows OOBE runs — they set up their own user account, language, etc. They can then enable BitLocker if desired.

### Summary

| Step | Where | What |
|---|---|---|
| Install all updates | Inside source guest | Do once, every copy inherits |
| Disable reserved storage | Inside source guest | Prevents sysprep `0x800F0975` failure |
| Shut down source | Source guest | Clean shutdown |
| Turn off TPM | Source host, VM Settings | Required for portability |
| Copy the VM | Source host | Folder copy or export |
| Turn TPM back on | Source host | Restore your own VM |
| Boot the copy, run sysprep | Inside copy guest | Strips SID, resets activation, shuts down |
| Don't boot again | — | Image is sealed |
| Recipient imports | Recipient host | Standard import |
| Recipient adds fresh vTPM | Recipient host | If they want encryption |
| Recipient boots | Recipient host | OOBE runs, fresh identity |

---

## Folder copy vs Export — when to use which

| Condition | Use |
|---|---|
| VM is off, no checkpoints, same host | **Folder copy** — faster |
| Unsure about checkpoint state | **Export** — it resolves chains for you |
| Moving to a different host | **Export** — produces a portable package |
| VM is huge and you're impatient | **Folder copy** — avoids the export staging step |

---

## Processor Settings — What Every Field Means

**Host CPU:** Intel Xeon Gold 6208U — 16 cores, 32 threads (Hyper-Threading). Hyper-V sees 32 logical processors (LPs).

### Number of virtual processors

How many of the host's 32 LPs the VM can use. The guest OS sees exactly this many CPUs. Set this to match what you want the VM to have. With SMT/Hyper-Threading enabled, use even numbers.

**Recommendation for single-VM-on-dedicated-box:** 28. Leaves 4 LPs for the host OS so it stays responsive.

### Virtual machine reserve (percentage)

Guaranteed minimum CPU. The host promises this VM at least this percentage of its assigned vCPUs' capacity, no matter what. If the host can't honor the reserve at VM startup, the VM refuses to start. Default is 0 (no guarantee).

**Recommendation for single VM:** Leave at 0. Nothing to compete with.

### Percent of total system resources (under reserve)

**Read-only. You cannot edit this.** Calculated: (vCPUs / total LPs) × reserve %. Shows what the reserve means in terms of the whole host. Example: 28 vCPUs, 50% reserve → 28/32 × 50% = 43.75%.

### Virtual machine limit (percentage)

Ceiling — the maximum percentage of its assigned vCPUs the VM can consume. At 100%, no cap. At 50%, each vCPU could only use half its capacity even if the host CPU is idle. This prevents a runaway VM from starving others.

**Recommendation for single VM:** Leave at 100.

### Percent of total system resources (under limit)

**Read-only. You cannot edit this.** Calculated: (vCPUs / total LPs) × limit %. With 1 vCPU at 100% limit, this shows 3% — meaning the VM can use 3% of the whole host. With 28 vCPUs at 100%, this shows 87.5%.

### Relative weight

Priority number from 1 to 10,000. Only matters when multiple VMs fight for CPU simultaneously. Higher weight = more CPU time during contention. When there's no contention, this does nothing. Default is 100.

**Recommendation for single VM:** Leave at 100. Irrelevant with one VM.

---

## RAM Configuration

The VM currently has 8192 MB (8 GB). For a single VM on a dedicated host, give it most of the host's physical RAM minus 2-4 GB for the host OS. Check total installed RAM on the host:

```powershell
(Get-CimInstance Win32_PhysicalMemory | Measure-Object Capacity -Sum).Sum / 1GB
```

**General guidance:**
- Host OS needs ~2-4 GB to stay healthy
- If host has 64 GB installed → give VM 60 GB
- If host has 128 GB installed → give VM 124 GB
- If host has 32 GB installed → give VM 28 GB

Change in VM Settings → Memory, or:

```powershell
Set-VMMemory -VMName "Win-10-Encrypted" -StartupBytes 60GB
```

(VM must be off to change this.)

---

## Windows Activation — What Happens Without It

Activation verifies your copy of Windows is genuine. It does NOT downgrade your edition. A Windows 10 Pro install without activation is still Pro — it has BitLocker, Remote Desktop, Group Policy, Hyper-V, domain join. Those are edition features, not activation features.

### What stops working (after 30-day grace period)

Personalization only. You can't change desktop wallpaper (goes to black/default), can't change colors or themes, can't change lock screen background, can't customize the taskbar or Start menu. A permanent "Activate Windows" watermark appears in the bottom-right corner of the screen, on top of everything including fullscreen apps.

### What keeps working

Everything functional. All apps, files, programs. Security and quality updates continue. All Pro features (BitLocker, RDP, gpedit, Hyper-V, domain join) remain available. The OS runs indefinitely in this state.

### What this means for distributing a VM

Practically nothing. Sysprep with `/generalize` resets activation by design — the recipient was always going to need their own license. They boot the VM, get OOBE, set up their account, and enter their own product key whenever they want. Everything works in the meantime. The watermark is cosmetic annoyance, not a functional limitation.

---

## Notes & lessons learned

- 2026-04-24: Win-10-Encrypted — TPM turned off before copy. Required because this VM will be distributed.
- 2026-04-24: Sysprep failed with `0x800F0975` — reserved storage was active and updates were pending. Fix: install all updates on source, disable reserved storage with DISM, THEN copy. Do this once on source so every copy is clean.
- Export-VM copies the full VHDX to a staging folder, which is why it takes so long on big disks. Folder copy skips that intermediate step.

---

*Full pipeline reference (sealing, repo management, distribution): see `felsenthal-logic/virtual-machine-management` repo.*
