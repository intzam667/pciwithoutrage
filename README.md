# GRUB Configuration
```bash
# Edit GRUB configuration
sudo nano /etc/default/grub
```
```plaintext
# Add the following line or modify if it already exists (VFIO IDs might be different):
GRUB_CMDLINE_LINUX_DEFAULT="intel_iommu=on iommu=pt loglevel=3 vfio-pci.ids=10de:25a2,10de:2291"
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

# Virt-Manager Configuration
1. **Create a New VM**
2. **Select UEFI Firmware**
3. **Add GTX PCI Devices**:
   - Video Controller
   - Audio Controller
4. **Use Bridged Network** (NAT is not recommended).
5. **Enable TPM for Windows 11**:
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
nmcli connection add type bridge-slave ifname enp0s20f0u4 master br0

# Bring connections up and down
nmcli connection down "<Connection Name>"
nmcli connection up bridge-br0
nmcli connection up bridge-slave-enp30s0
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
f	/dev/shm/looking-glass	0660	kurultaysokak	kvm
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
3. Enjoy a seamless Windows experience inside Linux!
