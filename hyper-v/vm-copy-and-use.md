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

## Notes & lessons learned

- 2026-04-24: Win-10-Encrypted — TPM turned off before copy. Required because this VM will be distributed.
- Export-VM copies the full VHDX to a staging folder, which is why it takes so long on big disks. Folder copy skips that intermediate step.

---

*Full pipeline reference (sealing, repo management, distribution): see `felsenthal-logic/virtual-machine-management` repo.*
