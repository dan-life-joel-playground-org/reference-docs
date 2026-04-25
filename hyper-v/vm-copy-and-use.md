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

## TPM — Turn It Off Before You Copy

**If you will distribute this VM (give it to someone else, move it to a different host), turn off TPM before copying.** A vTPM is cryptographically tied to the host it was created on. A recipient on a different host cannot unwrap the key protector and the VM will not boot.

**How to turn it off:** VM Settings → Security → uncheck "Enable Trusted Platform Module."

Do this BEFORE you copy or export. It's a required step for any VM you intend to distribute.

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

## Notes & lessons learned

- 2026-04-24: Win-10-Encrypted — TPM turned off before copy. Required because this VM will be distributed.
- Export-VM copies the full VHDX to a staging folder, which is why it takes so long on big disks. Folder copy skips that intermediate step.

---

*Full pipeline reference (sealing, repo management, distribution): see `felsenthal-logic/virtual-machine-management` repo.*
