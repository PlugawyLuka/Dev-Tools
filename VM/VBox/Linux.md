# Linux VM on VirtualBox — Practical Guide (Ubuntu / Windows / macOS hosts)

Short description  
Step‑by‑step instructions to create and run a Linux virtual machine (Ubuntu example) on VirtualBox hosts (Ubuntu 22.04, Windows 10/11 via PowerShell, macOS Ventura). Covers VirtualBox install on each host, VM creation (GUI + VBoxManage), attaching ISO, Guest Additions, networking, SSH, shared folders, headless automation and troubleshooting with commands and examples.

TL;DR
- Use VirtualBox + official Linux ISO (e.g., Ubuntu Desktop/Server) for portable Linux VMs across hosts.  
- Steps: install VirtualBox on your host → download Linux ISO → create VM (GUI or VBoxManage) → install guest OS → install Guest Additions (for improved integration) → configure networking and shared folders → automate start/stop with systemd / Scheduled Task / launchd.  
- WARNING: commands that install packages, write disks, or modify system services are marked "WARNING:" — double‑check before running.

Prerequisites
- Host with virtualization enabled in BIOS/UEFI (VT‑x/AMD‑V).  
- Admin / sudo privileges to install VirtualBox and kernel modules.  
- Download desired Linux ISO (Ubuntu, Debian, CentOS, etc.). Live lookup required for latest images.

1. Host: Install VirtualBox (per OS)
1.1 Linux host (Ubuntu 22.04+)
Objective: install VirtualBox and required kernel integration.
Commands:
```bash
# WARNING: installs packages and kernel modules (requires sudo)
sudo apt update
sudo apt install -y virtualbox virtualbox-ext-pack dkms
```
Verification:
```bash
VBoxManage --version
systemctl status vboxdrv
# Expected: vboxdrv active (running)
```
Troubleshooting:
```bash
# WARNING: reinstalls kernel modules if vboxdrv failed
sudo apt install --reinstall virtualbox-dkms
sudo systemctl restart vboxdrv
journalctl -u vboxdrv -n 200 --no-pager
```

1.2 Windows 10 / 11 (PowerShell 7+)
Objective: install VirtualBox using winget or MSI.
Commands (PowerShell Admin):
```powershell
# WARNING: installs software (requires Admin)
winget install --id=Oracle.VirtualBox -e
```
Verification:
```powershell
& "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" --version
```
Notes / Troubleshooting:
- If Hyper‑V is enabled, VirtualBox performance/compatibility may be limited. To disable Hyper‑V:
```powershell
# WARNING: changes Windows features; reboot required
bcdedit /set hypervisorlaunchtype off
Disable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V-All -NoRestart
```
Reboot required.

1.3 macOS Ventura
Objective: install VirtualBox via Homebrew Cask or Oracle installer.
Commands:
```bash
# WARNING: installs package and may require Security approval
brew install --cask virtualbox
```
Verification:
```bash
VBoxManage --version
```
Notes:
- macOS may block kernel extensions; open System Settings → Privacy & Security and allow Oracle kext, then reboot.

2. Obtain Linux ISO
Objective: download an official ISO (e.g., Ubuntu Desktop/Server).
Save as: ~/Downloads/ubuntu-22.04.iso (Linux/macOS) or C:\Users\<you>\Downloads\ubuntu-22.04.iso (Windows). Verify checksums where provided.

3. Create VM — GUI workflow (all hosts)
Steps:
- VirtualBox → New → Name: linux-vm → Type: Linux → Version: Ubuntu (64‑bit)  
- Memory: 2048 MB+ (4096+ recommended for desktop).  
- Create VDI (dynamically allocated) 20 GB+.  
- Settings → Storage → attach downloaded ISO to optical drive.  
- Settings → System → leave EFI off for BIOS boot (enable EFI only if required).  
- Settings → Network → Adapter1 NAT (default) or Bridged for direct LAN access.  
- Start VM and follow the installer.

4. Create VM — CLI (VBoxManage) examples
Objective: scriptable and repeatable creation (Linux/macOS shell shown).
```bash
# WARNING: creates VM and disk files on host
VM="linux-vm"
ISO="$HOME/Downloads/ubuntu-22.04.iso"
VBoxManage createvm --name "$VM" --ostype Ubuntu_64 --register
VBoxManage modifyvm "$VM" --memory 4096 --cpus 2 --nic1 nat
VBoxManage createhd --filename "$HOME/VirtualBox VMs/$VM/$VM.vdi" --size 30000
VBoxManage storagectl "$VM" --name "SATA" --add sata --controller IntelAhci
VBoxManage storageattach "$VM" --storagectl "SATA" --port 0 --device 0 --type hdd --medium "$HOME/VirtualBox VMs/$VM/$VM.vdi"
VBoxManage storageattach "$VM" --storagectl "SATA" --port 1 --device 0 --type dvddrive --medium "$ISO"
VBoxManage startvm "$VM" --type gui
```
Verification:
```bash
VBoxManage list vms | grep "$VM"
VBoxManage showvminfo "$VM" | grep -E "State|memory"
```

5. Install the Linux OS & Guest Additions
5.1 Install via installer UI
- Boot VM, follow installation: language, keyboard, partitioning (use guided defaults), create user, finish and reboot. Detach ISO after first reboot.

5.2 Install Guest Additions (improves video, shared folders, clipboard)
Inside Linux guest (Debian/Ubuntu example):
```bash
# WARNING: installs packages inside guest
sudo apt update
sudo apt install -y build-essential dkms linux-headers-$(uname -r)
# In VirtualBox menu: Devices -> Insert Guest Additions CD image...
sudo mount /dev/cdrom /mnt
cd /mnt
sudo sh ./VBoxLinuxAdditions.run
sudo reboot
```
Verification inside guest:
```bash
lsmod | grep vboxguest
ls /media  # check for mounted shared folders like /media/sf_<name>
```
Troubleshooting:
- If build/install fails, check /var/log/vboxadd-setup.log and ensure kernel headers/version match.

6. Networking & SSH access
6.1 NAT + port forwarding (access guest SSH from host)
Set NAT port forward on host:
```bash
# Map host port 5022 -> guest port 22
VBoxManage modifyvm "linux-vm" --natpf1 "guestssh,tcp,,5022,,22"
```
Then from host:
```bash
ssh -p 5022 youruser@localhost
```
6.2 Bridged networking
- Use bridged when VM should appear on the same LAN (guest receives LAN IP). Choose correct host interface in VM settings.

7. Shared folders & file transfer
Host: add shared folder:
```bash
VBoxManage sharedfolder add "linux-vm" --name "shared" --hostpath "$HOME/Shared" --automount
```
Guest:
```bash
# inside guest
sudo usermod -aG vboxsf youruser
# Log out/in to apply
ls /media  # or /media/sf_shared depending on automount location
```
Troubleshooting:
- If permission denied, ensure vboxsf group membership and Guest Additions installed.

8. Headless start / automation
8.1 systemd unit (Linux host) to auto-start VM
```ini
// filepath: /etc/systemd/system/vbox-linux-vm.service
[Unit]
Description=Start linux-vm headless
After=network.target vboxdrv.service

[Service]
Type=forking
User=youruser
ExecStart=/usr/bin/VBoxManage startvm "linux-vm" --type headless
ExecStop=/usr/bin/VBoxManage controlvm "linux-vm" acpipowerbutton
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```
Enable:
```bash
# WARNING: enables systemd unit
sudo systemctl daemon-reload
sudo systemctl enable --now vbox-linux-vm.service
```
8.2 Windows Scheduled Task (boot)
PowerShell (Admin):
```powershell
# WARNING: creates Scheduled Task
$action = New-ScheduledTaskAction -Execute "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" -Argument 'startvm "linux-vm" --type headless'
$trigger = New-ScheduledTaskTrigger -AtStartup
Register-ScheduledTask -TaskName "StartLinuxVM" -Action $action -Trigger $trigger -RunLevel Highest -User "SYSTEM"
```
8.3 macOS launchd
- Create LaunchAgent/Daemon plist to run VBoxManage startvm at login/boot; control via launchctl.

9. Common troubleshooting (host & guest)
9.1 VM won't start / VERR_VM_DRIVER_ERROR (host kernel module)
Linux host:
```bash
systemctl status vboxdrv
journalctl -u vboxdrv -n 200
# WARNING: reinstall kernel modules if necessary
sudo apt install --reinstall virtualbox-dkms
sudo systemctl restart vboxdrv
```
Windows host:
- Hyper‑V conflicts: disable Hyper‑V, reboot (see 1.2). Check Event Viewer for errors.

macOS host:
- Kernel extension blocked — approve in System Settings → Privacy & Security and reboot.

9.2 Guest Additions or modules fail to build
- Ensure linux-headers and build-essential installed; check /var/log/vboxadd-setup.log for root cause.

9.3 SSH/Network issues
- Verify NAT rule: `VBoxManage showvminfo "linux-vm" --details | grep -i nat`  
- Inside guest: `ip a`, `ss -tln`, `sudo systemctl status ssh` (or `sshd`).

9.4 Shared folder access denied
- Add user to vboxsf group and relogin. Confirm Guest Additions installed and matching host version.

10. Logs & support bundle
Collect this on host and guest for debugging:
```bash
mkdir -p ~/vbox-support
VBoxManage list vms > ~/vbox-support/vms.txt
VBoxManage showvminfo "linux-vm" --machinereadable > ~/vbox-support/linux-vm.info
journalctl -u vboxdrv -n 500 > ~/vbox-support/vboxdrv.log   # Linux host
# From guest, copy /var/log/vboxadd-setup.log and dmesg output into support dir
tar czf ~/vbox-support.tgz -C ~/vbox-support .
```

11. Useful examples & commands summary
- Create VM (headless): see section 4.  
- NAT SSH forwarding:
```bash
VBoxManage modifyvm "linux-vm" --natpf1 "guestssh,tcp,,5022,,22"
```
- Add shared folder:
```bash
VBoxManage sharedfolder add "linux-vm" --name "shared" --hostpath "/home/you/Shared" --automount
```
- Export / import:
```bash
VBoxManage export "linux-vm" -o ~/linux-vm.ova
VBoxManage import ~/linux-vm.ova
```

12. Advanced notes & limitations
- For hardware-specific tests (GPIO, Pi camera, Bluetooth), use real hardware or QEMU ARM emulation.  
- Keep VirtualBox host and Guest Additions versions aligned. After host kernel upgrades reinstall DKMS modules. For CI, use self‑hosted runners with VirtualBox installed.

13. References
- VirtualBox: https://www.virtualbox.org/  
- Ubuntu downloads: https://ubuntu.com/download (or your chosen distro site)  
- DKMS on Ubuntu: https://wiki.ubuntu.com/Kernel/DKMS
