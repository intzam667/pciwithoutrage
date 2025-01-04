A detailed guide for setting up PCI passthrough virtualization, focusing on isolating GPUs for virtual machines. It includes steps for configuring GRUB and VFIO, installing required virtualization packages, creating NVMe partitions, and setting up virtual machines in Virt-Manager. Additionally, bridge networking and features like Looking Glass for better VM performance.



## Table of Contents
- 0
- [GRUB Configuration](#grub-configuration)
- [VFIO Configuration](#vfio-configuration)
- 1
- [Install Required Packages](#install-required-packages)
- [Create NVME Partition](#create-nvme-partition)
- [Virt-Manager Configuration](#virt-manager-configuration)
- [Bridge Network Setup](#bridge-network-setup)
- [Looking Glass Setup](#looking-glass-setup)
- [Windows Installation Steps](#windows-installation-steps)
- [Virtual Display Drivers](#virtual-display-drivers)
- [Adjusting Display Settings](#adjusting-display-settings)

## Additional
- [CPU Pinning](#cpu-pinning)

# 0: Find and Isolate GPU VFIO IDs
```bash
# I'm using NVIDIA, might be different for you. Copy the IDs responsibly.
lspci -nnk | grep -i nvidia -A2
```
Output should be something like this:
```
01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GA107M [GeForce RTX 3050 Mobile] [10de:25a2] (rev a1)
01:00.1 Audio device [0403]: NVIDIA Corporation GA107 High Definition Audio Controller [10de:2291] (rev a1)
```

# GRUB Configuration
```bash
# Edit GRUB configuration
sudo nano /etc/default/grub
```
```plaintext
# Add the following line or modify if it already exists (VFIO IDs might be different):
GRUB_CMDLINE_LINUX_DEFAULT="intel_iommu=on iommu=pt [rest of already existing content] vfio-pci.ids=10de:25a2,10de:2291"
```
```bash
# Update GRUB
sudo grub-mkconfig -o /boot/grub/grub.cfg

# Reboot the system
sudo reboot
```

---

# VFIO Configuration
```bash
# Create or edit VFIO configuration
sudo nano /etc/modprobe.d/vfio.conf
```
```plaintext
# Add the following lines (VFIO IDs might be different):
options vfio-pci ids=10de:25a2,10de:2291
softdep nvidia pre: vfio-pci
```
```bash
# Regenerate initramfs
sudo mkinitcpio -P linux

# Reboot the system
sudo reboot
```

---

# 1: Check if the GPU has been isolated.
```bash
lspci -k | grep -i nvidia -A2
```
Output should be something like this:
```
01:00.0 VGA compatible controller: NVIDIA Corporation GA107M [GeForce RTX 3050 Mobile] (rev a1)
	Subsystem: Hewlett-Packard Company Device 8a4f
	Kernel driver in use: vfio-pci
	Kernel modules: nouveau, nvidia_drm, nvidia
01:00.1 Audio device: NVIDIA Corporation GA107 High Definition Audio Controller (rev a1)
	Subsystem: Hewlett-Packard Company Device 8a4f
	Kernel driver in use: vfio-pci
```
Be sure you see the kernel driver "vfio-pci"

# Install Required Packages
```bash
# Install virtualization packages and enable libvirt
sudo pacman -S qemu libvirt ovmf virt-manager
sudo systemctl enable --now libvirtd
```

---

# Create NVME Partition
```bash
# Open cfdisk to create a "Microsoft basic data" partition
sudo cfdisk /dev/nvme0n1
```

---

# Virt Manager Configuration
1. **Create a New VM** <br>
1.1 **In the storage ".qcow" section, use custom storage path as /dev/nvme0n1pX or /dev/sdX** <br>
1.2 **In CPU configuration, do 1-8-1 topology to get native performance as threads are needed the most.** <br>
3. **Select UEFI Firmware**<br>
4. **Add GTX PCI Devices**:
   - Video Controller
   - Audio Controller
5. **Use Bridged Network** (NAT is not recommended).<br>
6. **Enable TPM for Windows 11**:<br>
   - Use Emulated TIS TPM 2.0 Module.
   - If you encounter an error:
     ```bash
     sudo pacman -Sy swtpm
     ```

---

# Bridge Network Setup
```bash
# Check network interfaces
ip a

# Add a bridge slave (replace `enpXs` with your interface)
nmcli connection add type bridge-slave ifname enpX master br0

# Bring connections up and down
nmcli connection down "<Connection Name>"
nmcli connection up bridge-br0
nmcli connection up bridge-slave-enpX
```

---

# Looking Glass Setup
```bash
# Edit VM configuration for IVSHMEM
sudo virsh edit <vmname>
```
- Refer to: [Adding IVSHMEM Device](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#Adding_IVSHMEM_Device_to_virtual_machines)

```bash
# Create temporary files for Looking Glass
sudo mkdir -p /etc/tempfiles.d
sudo nvim /etc/tempfiles.d/10-looking-glass.conf
```
```plaintext
# Add the following line:
f	/dev/shm/looking-glass	0660	user	kvm
```
```bash
# Apply the configuration
sudo systemd-tmpfiles --create /etc/tempfiles.d/10-looking-glass.conf

# Install Looking Glass using yay
yay looking-glass (Brc7-1)
```

---

# Windows Installation Steps
1. Install **Windows 11**.
2. Install **VirtIO Drivers**.
3. Install **Hardware-Specific Drivers**:
   - High Definition Audio
   - NVIDIA GPU
4. Install **NVIDIA Application and Game Ready Drivers**.
5. Install VirtIO drivers:
   - Download from [VirtIO Drivers](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/upstream-virtio/).
   - Unzip and use the `upstream` folder.
   - Update `PCI Standard RAM Controller`:
     ```plaintext
     Device Manager > System Devices > PCI Standard RAM Controller > Update Driver
     Browse to: upstream\Win10\amd64
     ```
6. Install Looking Glass Host (matching your Linux client version).
7. Set **Video Virtio** for audio.
8. Default Sound Device:
   - Set to `ICH9` or similar.

---

# Virtual Display Drivers
1. Download from: [Virtual Display Driver](https://github.com/VirtualDisplay/Virtual-Display-Driver).
2. Install drivers (default path: `C:\IddSampleDriver`):
   ```plaintext
   Device Manager > Action > Add Legacy Hardware
   > Manually Select from List > Show All Devices > Have Disk
   Browse to: IddSampleDriver folder > Select driver file
   ```

---

# Adjusting Display Settings
1. Set the virtual display as **default**.
2. Adjust resolution and refresh rate:
   - For resolutions larger than 1080p, change VM Looking Glass shmem unit to `64`.

# CPU Pinning
First, make a hooks folder inside /etc/libvirt <br>
1. Recreate this tree: <br>
└── qemu.d <br>
    └── vm_name (not custom!) <br>
        ├── prepare <br>
        │   └── begin <br>
        │       └── isolatecpu.sh <br>
        └── release <br>
            └── end <br>
                └── revert.sh <br>
		
(As seen on SomeOrdinaryGamers' video :D) <br>

2. Check your CPU topology.
```bash
lscpu -e
```
3. Note the CPU sets that both use the same CPU (e.g, 0-1 or 4-8)
4. Edit isolstart.sh to include this lines: <br>
```plaintext
systemctl set-property --runtime -- user.slice AllowedCPUs={change CPU set, like 0,1} <br>
systemctl set-property --runtime -- system.slice AllowedCPUs={change CPU set, like 0,1} <br>
systemctl set-property --runtime -- init.scope AllowedCPUs={change CPU set, like 0,1} <br>
```
Make sure to chmod<br>
5. Edit revert.sh to include this lines: <br>
```plaintext
systemctl set-property --runtime -- user.slice AllowedCPUs={all of your CPUs, like 0-11}
systemctl set-property --runtime -- system.slice AllowedCPUs={all of your CPUs, like 0-11}
systemctl set-property --runtime -- init.scope AllowedCPUs={all of your CPUs, like 0-11}
```
Make sure to chmod<br>
6. Create a qemu executable dispatcher under /etc/libvirt/hooks folder: <br>
```plaintext
#!/bin/bash
DOMAIN_NAME="$1"
ACTION="$2"
PHASE="$3"

if [ "$DOMAIN_NAME" = "vm_name" ]; then
  case "$ACTION" in
  "prepare")
    if [ "$PHASE" = "begin" ]; then
      /etc/libvirt/hooks/qemu.d/win11/prepare/begin/isolatecpu.sh
    fi
    ;;
  "release")
    if [ "$PHASE" = "end" ]; then
      /etc/libvirt/hooks/qemu.d/win11/release/end/revert.sh
    fi
    ;;
  esac
fi
```
Now when you open the VM, hooks should launch by the dispatcher script and allow you to tune CPUs to gain more performance inside VM. <br>
After you exit the VM, dispatcher reverts the CPU isolation to default state. <br>
Enjoy pure native Gaming experience in Windows, better than IN Windows. <br>






