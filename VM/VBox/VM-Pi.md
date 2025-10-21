# Create a Raspberry Pi VM — Step‑by‑Step Guide (VirtualBox-focused)

Short description  
This guide explains how to create a Raspberry Pi–like VM on VirtualBox hosts (Ubuntu, Windows PowerShell, macOS). It uses the Raspberry Pi Desktop (x86) ISO for a quick, usable Pi desktop VM in VirtualBox and covers GUI and CLI (VBoxManage) workflows, Guest Additions, shared folders, networking, and troubleshooting per OS.

TL;DR
- VirtualBox cannot emulate ARM CPU; use Raspberry Pi Desktop (x86) ISO for a fast Pi‑like VM on VirtualBox.  
- Steps: install VirtualBox → create VM → attach ISO → install guest → install Guest Additions → configure shared folders and networking.  
- Use VBoxManage for scripted/headless workflows and systemd / Scheduled Task / launchd to auto‑start VMs.  
- WARNING: commands that install packages, write disks or enable kernel modules are marked "WARNING:" — double‑check before running.

Prerequisites
- Host with virtualization enabled in BIOS/UEFI (VT‑x/AMD‑V).  
- Admin / sudo privileges to install VirtualBox and kernel modules.  
- Download Raspberry Pi Desktop (x86) ISO and keep path handy (Live lookup required).

Overview & limitations
- This method runs an x86 port of the Raspberry Pi desktop environment — good for app and UI testing but not for GPIO, camera or hardware-specific tests. For ARM fidelity use QEMU or real Pi hardware.

1. Install VirtualBox (per OS)
1.1 Linux (Ubuntu 22.04+)
Objective: install VirtualBox with kernel module support.
Commands:
```bash
# WARNING: installs packages and kernel modules (requires sudo)
sudo apt update
sudo apt install -y virtualbox virtualbox-ext-pack
# Ensure DKMS rebuilds modules on kernel updates
sudo apt install -y dkms
```
Verification:
```bash
VBoxManage --version
# Expected: version string (e.g., 6.x or 7.x)
systemctl status vboxdrv
# Expected: active (running)
```
Troubleshooting:
- If vboxdrv fails: reinstall dkms modules:
```bash
# WARNING: reinstalls kernel modules
sudo apt install --reinstall virtualbox-dkms
sudo systemctl restart vboxdrv
journalctl -u vboxdrv -n 200 --no-pager
```

1.2 Windows 10 / 11 (PowerShell 7+)
Objective: install VirtualBox via winget or MSI.
Commands (PowerShell Admin):
```powershell
# WARNING: installs software (requires Admin)
winget install --id=Oracle.VirtualBox -e
```
Verification:
```powershell
& "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" --version
# Expected: version string
```
Notes & troubleshooting:
- If Hyper‑V is enabled, VirtualBox performance may be reduced or fail. Disable Hyper‑V for optimal VirtualBox:
```powershell
# WARNING: disables Windows features and requires reboot
Disable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V-All -NoRestart
```
Reboot required.

1.3 macOS Ventura
Objective: install VirtualBox via Homebrew or Oracle installer.
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
- After install, macOS may block kernel extensions. Open System Settings → Privacy & Security and allow Oracle kernel extension, then reboot.

2. Obtain Raspberry Pi Desktop (x86) ISO
Objective: download the x86 ISO (Live lookup required).
Save as: ~/Downloads/raspberry-pi-desktop.iso (Linux/macOS) or C:\Users\<you>\Downloads\raspberry-pi-desktop.iso (Windows).

3. Create VM (GUI and CLI)
3.1 GUI (all OS)
- VirtualBox → New → Name: RaspberryPI‑Desktop  
- Type: Linux, Version: Debian (64‑bit) or choose 32‑bit ISO variant.  
- Memory: 2048 MB+ (4096 MB if host permits).  
- Create VDI (dynamically allocated) 20 GB+.  
- Settings → Storage → attach downloaded ISO to optical drive.  
- Settings → System → enable EFI only if ISO requires it (most Raspberry Pi Desktop ISOs do not).  
- Settings → Network → Adapter 1 NAT (default) or Bridged for LAN.  
- Start VM and follow installer.

3.2 VBoxManage (scriptable, headless)
Example (Linux/macOS shell or PowerShell with adapted paths):
```bash
# WARNING: creates VM and disk files on host
VM="rpi-desktop"
ISO="$HOME/Downloads/raspberry-pi-desktop.iso"
VBoxManage createvm --name "$VM" --ostype Debian_64 --register
VBoxManage modifyvm "$VM" --memory 4096 --cpus 2 --nictype1 82540EM --nic1 nat
VBoxManage createhd --filename "$HOME/VirtualBox VMs/$VM/$VM.vdi" --size 20000
VBoxManage storagectl "$VM" --name "SATA" --add sata --controller IntelAhci
VBoxManage storageattach "$VM" --storagectl "SATA" --port 0 --device 0 --type hdd --medium "$HOME/VirtualBox VMs/$VM/$VM.vdi"
VBoxManage storageattach "$VM" --storagectl "SATA" --port 1 --device 0 --type dvddrive --medium "$ISO"
VBoxManage sharedfolder add "$VM" --name "shared" --hostpath "$HOME/Shared" --automount
VBoxManage startvm "$VM" --type gui
```
Verification:
```bash
VBoxManage list vms | grep rpi-desktop
# Expected: "rpi-desktop" {UUID}
VBoxManage showvminfo rpi-desktop | grep -E "State|memory"
# Expected: State: running
```

4. Install the OS and Guest Additions
4.1 Install guest through installer UI
- Boot VM from ISO, follow installer: language, keyboard, partition (use defaults), create user, finish and reboot. After first boot detach ISO from virtual drive.

4.2 Install VirtualBox Guest Additions (improves graphics, shared folders, clipboard)
- In VirtualBox menu: Devices → Insert Guest Additions CD image...
- Inside guest (terminal):
```bash
# Mount and run installer (Debian-based)
sudo apt update
sudo apt install -y build-essential dkms linux-headers-$(uname -r)
sudo mount /dev/cdrom /mnt
cd /mnt
sudo sh ./VBoxLinuxAdditions.run
# Reboot guest
sudo reboot
```
Verification inside guest:
```bash
# Shared folder mount
ls /media/sf_shared   # if automount placed at /media/sf_shared
# Guest Additions kernel module
lsmod | grep vboxguest
# Expected: vboxguest present
```
Troubleshooting:
- If Guest Additions fail: check kernel headers installed and run the installer log at /var/log/vboxadd-setup.log for errors.

5. Networking tips & SSH access
5.1 NAT with port forwarding (make guest SSH accessible from host)
Set up port forward via VBoxManage:
```bash
# Map host port 5022 -> guest port 22
VBoxManage modifyvm "rpi-desktop" --natpf1 "guestssh,tcp,,5022,,22"
```
Verify:
```bash
VBoxManage showvminfo "rpi-desktop" --details | grep guestssh
# Expected: NAT rule visible
# Then from host:
ssh -p 5022 user@localhost
```
5.2 Bridged mode
- Use bridged when guest must be on the same LAN; ensure host network interface chosen is that of the desired NIC.

6. Shared folders & file copy
- Create host folder and add as shared folder (see VBoxManage sharedfolder add above). In guest, add your user to vboxsf group:
```bash
# inside guest
sudo usermod -aG vboxsf youruser
# Log out/in to apply group change
```
Verify:
```bash
ls /media/sf_shared
# Expected: host files visible
```

7. Start/Stop headless and automation
7.1 systemd unit (Linux host) to auto-start VM on boot
```ini
// filepath: /etc/systemd/system/vbox-rpi.service
[Unit]
Description=Start rpi-desktop VM
After=network.target vboxdrv.service

[Service]
Type=forking
User=youruser
ExecStart=/usr/bin/VBoxManage startvm "rpi-desktop" --type headless
ExecStop=/usr/bin/VBoxManage controlvm "rpi-desktop" acpipowerbutton
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```
Enable:
```bash
# WARNING: enables systemd unit
sudo systemctl daemon-reload
sudo systemctl enable --now vbox-rpi.service
```
Verification:
```bash
systemctl status vbox-rpi.service
# Expected: active (exited) or running depending on VM state
```

7.2 Windows Scheduled Task (start VM at login/boot)
PowerShell example to create a task that runs VBoxManage:
```powershell
# WARNING: creates Scheduled Task (Admin)
$action = New-ScheduledTaskAction -Execute "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" -Argument 'startvm "rpi-desktop" --type headless'
$trigger = New-ScheduledTaskTrigger -AtStartup
Register-ScheduledTask -TaskName "StartRpiVM" -Action $action -Trigger $trigger -RunLevel Highest -User "SYSTEM"
```

7.3 macOS launchd plist (headless)
Create plist in ~/Library/LaunchAgents/com.example.vbox.rpi.plist to run VBoxManage startvm at login. Use launchctl load/unload for control.

8. Common troubleshooting (per OS)
8.1 VM won't start / VERR_VM_DRIVER_ERROR (host kernel module)
Linux:
```bash
# Check vboxdrv
systemctl status vboxdrv
journalctl -u vboxdrv -n 200
# Rebuild DKMS modules
# WARNING: reinstalls kernel modules
sudo apt install --reinstall virtualbox-dkms
sudo systemctl restart vboxdrv
```
Windows:
- Error about Hyper‑V: disable Hyper‑V (requires reboot) or enable "Virtualization Based Security" off. Check Windows Features and run bcdedit /set hypervisorlaunchtype off then reboot.

macOS:
- Kernel extension blocked — open System Settings → Privacy & Security and allow Oracle extension, then reboot.

8.2 Guest Additions build failures
- Ensure guest has linux-headers and build-essential installed. Inspect /var/log/vboxadd-setup.log for details and fix missing packages.

8.3 Shared folder access denied
- Add guest user to vboxsf group and log out/in. Ensure Guest Additions installed.

8.4 Network unreachable between host and guest
- Check NAT port forwarding rules:
```bash
VBoxManage showvminfo "rpi-desktop" --details | grep NAT
```
- For bridged: make sure host NIC supports promiscuous mode and you're using correct adapter.

8.5 Graphics/Mouse capture issues
- Install Guest Additions and enable bi-directional clipboard/drag-and-drop in VM settings.

9. Logs & support bundle
Collect useful info for debugging:
```bash
# Host
VBoxManage list vms > ~/vbox-support/vms.txt
VBoxManage showvminfo "rpi-desktop" --machinereadable > ~/vbox-support/rpi.info
journalctl -u vboxdrv -n 500 > ~/vbox-support/vboxdrv.log   # Linux host
# Guest: copy /var/log/vboxadd-setup.log and dmesg output to support bundle
tar czf ~/vbox-support.tgz -C ~/vbox-support .
```

10. Useful examples & commands summary
- Create VM and attach ISO (headless): see section 3.2.  
- Add NAT SSH forwarding:
```bash
VBoxManage modifyvm "rpi-desktop" --natpf1 "guestssh,tcp,,5022,,22"
```
- Add shared folder:
```bash
VBoxManage sharedfolder add "rpi-desktop" --name "shared" --hostpath "/home/you/Shared" --automount
```
- Export VM:
```bash
VBoxManage export "rpi-desktop" -o ~/rpi-desktop.ova
VBoxManage import ~/rpi-desktop.ova
```

ADVANCED notes
- For GPIO or hardware tests, use a real Raspberry Pi or QEMU ARM emulation (advanced, slower).  
- If using VirtualBox in CI, use self-hosted runners where VirtualBox/Extension Pack is supported.

References
- VirtualBox official: https://www.virtualbox.org/  
- Raspberry Pi Desktop (x86) downloads: Raspberry Pi Foundation site (Live lookup required)  
- Guest Additions manual: VirtualBox User Manual → Guest Additions Chapter  
- DKMS on Ubuntu: https://wiki.ubuntu.com/Kernel/DKMS
