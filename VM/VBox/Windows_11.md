# Windows 11 VM on VirtualBox — Practical Guide (Ubuntu / Windows / macOS hosts)

Short description  
Step‑by‑step instructions to create and run a Windows 11 virtual machine on VirtualBox hosts (Ubuntu 22.04, Windows 10/11 using PowerShell, macOS Ventura). Covers install of VirtualBox, VM creation (GUI + VBoxManage), attaching ISO, Guest Additions, networking, shared folders, automation and troubleshooting. Notes on TPM/Secure Boot for Windows 11 and safe workarounds are included.

Index — Chapters
1. TL;DR & prerequisites  
2. Host preparations & limits (TPM note)  
3. Install VirtualBox (Ubuntu / PowerShell / macOS)  
4. Obtain Windows 11 ISO (Live lookup required)  
5. Create VM (GUI)  
6. Create VM (VBoxManage CLI)  
7. Install Windows 11 (installer tips & TPM note)  
8. Guest Additions (Windows guest)  
9. Networking & SSH / RDP access (NAT, bridged, port forward)  
10. Shared folders & file copy (vboxsf)  
11. Headless start / automation (systemd, Scheduled Task, launchd)  
12. Troubleshooting & diagnostics (per host)  
13. Logs & support bundle  
14. Examples & command summary  
15. References

1. TL;DR & prerequisites
- Use VirtualBox 6.1+ or 7.x; install Extension Pack for USB/RDP features.  
- Windows 11 expects TPM 2.0 + Secure Boot; VirtualBox may not provide vTPM on all versions — prefer Hyper‑V or KVM for full platform compliance. If you must use VirtualBox, enable EFI and follow vendor guidance for TPM or use documented installer workarounds (ADVANCED).  
- Required privileges: sudo / Admin to install VirtualBox, create systemd units, or register Scheduled Tasks. Live download of Windows 11 ISO required.

2. Host preparations & limits (TPM / Secure Boot)
- VirtualBox historically lacks native vTPM on many hosts; VirtualBox 7 introduced experimental vTPM support on some platforms. Check your VirtualBox version and docs (live lookup).  
ADVANCED: If you need fully compliant TPM2, use QEMU + swtpm or Hyper‑V on Windows hosts. Workarounds to bypass TPM checks exist but reduce security — use only in isolated test environments.

3. Install VirtualBox
3.1 Ubuntu 22.04 (Linux host)
Objective: install VirtualBox + Extension Pack.
Commands:
```bash
# WARNING: installs packages and kernel modules
sudo apt update
sudo apt install -y virtualbox virtualbox-ext-pack dkms
```
Verification:
```bash
VBoxManage --version
# Expected: version string, e.g., 6.1.x or 7.x
systemctl status vboxdrv
# Expected: active (running)
```
Troubleshooting:
```bash
# WARNING: reinstalls kernel modules (if vboxdrv fails)
sudo apt install --reinstall virtualbox-dkms
sudo systemctl restart vboxdrv
journalctl -u vboxdrv -n 200 --no-pager
```

3.2 Windows 10 / 11 (PowerShell 7+ as Admin)
Objective: install VirtualBox using winget / MSI.
Commands (PowerShell Admin):
```powershell
# WARNING: installs software (Admin)
winget install --id=Oracle.VirtualBox -e
```
Verification:
```powershell
& "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" --version
# Expected: version string
```
Notes:
- If Hyper‑V is enabled, VirtualBox may be limited. To disable Hyper‑V:
```powershell
# WARNING: changes Windows features; reboot required
bcdedit /set hypervisorlaunchtype off
Disable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V-All -NoRestart
```

3.3 macOS Ventura
Objective: install VirtualBox via Homebrew Cask or Oracle installer.
Commands:
```bash
# WARNING: installs package and may require Security approval
brew install --cask virtualbox
```
Verification:
```bash
VBoxManage --version
# Expected: version string
```
Notes:
- macOS may block kernel extensions; open System Settings → Privacy & Security and allow Oracle kext, then reboot.

4. Obtain Windows 11 ISO (Live lookup required)
- Download official Windows 11 ISO from Microsoft and verify checksums. Save to: ~/Downloads/Win11.iso (Linux/macOS) or C:\Users\<you>\Downloads\Win11.iso (Windows).
WARNING: Do not use unofficial/unverified ISOs for production testing.

5. Create VM — GUI workflow (all hosts)
Steps (concise):
- VirtualBox → New → Name: Win11-VM → Type: Microsoft Windows → Version: Windows 10 (64-bit) or Windows 11 if listed.  
- RAM: 4096 MB+ (8 GB recommended). CPU: 2+ cores.  
- Create virtual disk VDI (dynamically allocated) 64 GB+.  
- Settings → System → enable EFI (UEFI) (recommended for Win11).  
- Settings → Storage → attach Win11 ISO to optical drive.  
- Settings → Display → Video memory 128 MB; enable 3D acceleration optional.  
- Settings → Network → NAT (default) or Bridged for LAN.  
- Start VM and run installer.

6. Create VM — VBoxManage (CLI) examples
Objective: scriptable & repeatable VM creation.
Commands (Linux/macOS shell; adapt paths on Windows PowerShell):
```bash
# WARNING: creates VM and disk files on host
VM="Win11-VM"
ISO="$HOME/Downloads/Win11.iso"
VBoxManage createvm --name "$VM" --ostype Windows10_64 --register
VBoxManage modifyvm "$VM" --memory 8192 --cpus 4 --vram 128 --nic1 nat
VBoxManage createmedium disk --filename "$HOME/VirtualBox VMs/$VM/$VM.vdi" --size 65536
VBoxManage storagectl "$VM" --name "SATA" --add sata --controller IntelAhci
VBoxManage storageattach "$VM" --storagectl "SATA" --port 0 --device 0 --type hdd --medium "$HOME/VirtualBox VMs/$VM/$VM.vdi"
VBoxManage storageattach "$VM" --storagectl "SATA" --port 1 --device 0 --type dvddrive --medium "$ISO"
# Enable EFI (UEFI)
VBoxManage modifyvm "$VM" --firmware efi
# Optional: enable nested/IO APIC etc.
VBoxManage modifyvm "$VM" --ioapic on
VBoxManage startvm "$VM" --type gui
```
Verification:
```bash
VBoxManage list vms | grep "$VM"
# Expected: "Win11-VM" {UUID}
VBoxManage showvminfo "$VM" | grep -E "State|memory|firmware"
# Expected: State: running and firmware=EFI
```

7. Install Windows 11 (installer tips & TPM note)
7.1 Normal install
- In VM console, proceed with GUI installer: partition the virtual disk (use defaults), create user/account and set regional settings.

7.2 TPM / Secure Boot considerations (ADVANCED)
- Windows 11 setup may refuse install if TPM2 / Secure Boot not detected. Options:
  - Official: use VirtualBox version with vTPM support (check docs) or use Hyper‑V / QEMU + swtpm.  
  - Workaround (test-only): during install press Shift+F10 → regedit, create key:
    HKEY_LOCAL_MACHINE\SYSTEM\Setup\LabConfig
    Add DWORD: BypassTPMCheck = 1, BypassSecureBootCheck = 1, BypassRAMCheck = 1 (or AllowUpgradesWithUnsupportedTPMOrCPU=1)  
    Close regedit and continue installer.  
  WARNING: This weakens platform checks — use only in isolated test environments.
ADVANCED: Use swtpm + QEMU to provide a proper TPM2 device in CI/test labs.

Verification:
- After install, confirm Windows 11 version:
  - Start → Settings → System → About → Windows specifications.
- From host, verify VM is running:
```bash
VBoxManage showvminfo "Win11-VM" | grep -E "State|VMName"
# Expected: State: running
```

8. Install Guest Additions (Windows guest)
Objective: improve integration (drivers, shared folders, better video).
Steps:
- VirtualBox menu: Devices → Insert Guest Additions CD image...
- In Windows guest Explorer, run VBoxWindowsAdditions.exe as Administrator.
Verification:
- Device Manager shows VirtualBox Guest Additions drivers.
- Shared folders mount as network drive or under \\VBOXSVR\<sharename>.
Troubleshooting:
- If installer fails: ensure matching Guest Additions version to VirtualBox host version; reboot guest and retry.

9. Networking & remote access
9.1 NAT + port forwarding (access RDP/SSH from host)
Commands:
```bash
# WARNING: modifies VM NAT rules
VBoxManage modifyvm "Win11-VM" --natpf1 "rdp,tcp,,3389,,3389"
```
Verification:
- From host: `mstsc /v:localhost:3389` (Windows RDP) or `rdesktop localhost:3389` (Linux) and connect to guest's RDP service (enable Remote Desktop inside Windows guest).

9.2 Bridged networking
- Use Bridged when VM needs LAN IP. Verify all NIC settings and ensure host NIC selected correctly.

10. Shared folders & file copy (vboxsf)
10.1 Create shared folder (host)
Commands:
```bash
# WARNING: config changes
VBoxManage sharedfolder add "Win11-VM" --name "shared" --hostpath "/home/you/Shared" --automount
```
In Windows guest:
- Shared folder appears as a network location: \\VBOXSVR\shared or mounted drive letter.
Verification:
- Explorer shows the shared folder; copy files to/from host.

11. Headless start / automation
11.1 systemd unit (Linux host)
Create unit:
```ini
// filepath: /etc/systemd/system/vbox-win11.service
[Unit]
Description=Start Win11-VM headless
After=network.target vboxdrv.service

[Service]
Type=forking
User=youruser
ExecStart=/usr/bin/VBoxManage startvm "Win11-VM" --type headless
ExecStop=/usr/bin/VBoxManage controlvm "Win11-VM" acpipowerbutton
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```
Enable:
```bash
# WARNING: enables systemd unit
sudo systemctl daemon-reload
sudo systemctl enable --now vbox-win11.service
```
Verification:
```bash
systemctl status vbox-win11.service
# Expected: active (exited) and VM running (VBoxManage list runningvms)
```

11.2 Windows Scheduled Task (boot)
PowerShell (Admin):
```powershell
# WARNING: creates Scheduled Task
$action = New-ScheduledTaskAction -Execute "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" -Argument 'startvm "Win11-VM" --type headless'
$trigger = New-ScheduledTaskTrigger -AtStartup
Register-ScheduledTask -TaskName "StartWin11VM" -Action $action -Trigger $trigger -RunLevel Highest -User "SYSTEM"
```

11.3 macOS launchd
- Use a LaunchAgent/LaunchDaemon to run VBoxManage startvm at user login or system boot (requires correct plist and permissions).

12. Troubleshooting & diagnostics (per host)
12.1 Common host errors
- Linux: VERR_VM_DRIVER_ERROR / vboxdrv not loaded — reinstall virtualbox-dkms and restart vboxdrv (see 3.1).  
- Windows: Hyper‑V conflict — disable Hyper‑V and reboot (see 3.2).  
- macOS: Kernel extension blocked — approve in Privacy & Security and reboot.

12.2 Installer / VM errors
- Windows 11 blocks install for TPM/CPU: either use supported host/hypervisor or use isolated test workaround (see 7.2).  
- Guest Additions fail: ensure Windows has latest updates and drivers; run installer as Administrator.

12.3 Diagnose network issues
Commands:
```bash
# Host: inspect NAT rules
VBoxManage showvminfo "Win11-VM" --details | grep -i nat
# Guest: ipconfig /all (in Windows)
```

12.4 Collect logs & support bundle
Commands:
```bash
mkdir -p ~/vbox-support
VBoxManage list vms > ~/vbox-support/vms.txt
VBoxManage showvminfo "Win11-VM" --machinereadable > ~/vbox-support/win11.info
journalctl -u vboxdrv -n 500 > ~/vbox-support/vboxdrv.log   # Linux host only
tar czf ~/vbox-support.tgz -C ~/vbox-support .
```

13. Examples & command summary
- Create VM & attach ISO (headless): see section 6.  
- Enable EFI:
```bash
VBoxManage modifyvm "Win11-VM" --firmware efi
```
- Export VM:
```bash
VBoxManage export "Win11-VM" -o ~/win11-vm.ova
VBoxManage import ~/win11-vm.ova
```
- NAT port forward for RDP:
```bash
VBoxManage modifyvm "Win11-VM" --natpf1 "rdp,tcp,,3389,,3389"
```

14. ADVANCED explainers
- ADVANCED: vTPM — virtual TPM provides TPM interfaces to guests. VirtualBox experimental vTPM support depends on host & VB version. For production TPM tests use swtpm + QEMU or Hyper‑V.  
- ADVANCED: Secure Boot within VM requires UEFI firmware and signed bootloader/keys; enabling Secure Boot in VirtualBox is limited and host dependent.

15. References
- VirtualBox official: https://www.virtualbox.org/ — downloads, manual, Extension Pack.  
- Microsoft Windows 11 requirements: https://www.microsoft.com/windows/windows-11-specifications — TPM/Secure Boot guidance.  
- VirtualBox Guest Additions: VirtualBox manual → Guest Additions chapter.  
- DKMS guide (Ubuntu): https://wiki.ubuntu.com/Kernel/DKMS — manage kernel modules.

Final notes / Safety
- WARNING: Disabling Hyper‑V or using registry workarounds for Windows 11 weakens security — use only in isolated test environments.  
- Always download ISOs from official sources and verify checksums. Export VMs (OVA) before major changes and test restores regularly.
