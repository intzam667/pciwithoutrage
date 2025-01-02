
# GRUB configuration
- sudo nano /etc/default/grub: <br>
GRUB_CMDLINE_LINUX_DEFAULT="intel_iommu=on iommu=pt loglevel=3 vfio-pci.ids=10de:25a2,10de:2291"<br>

- sudo grub-mkconfig -o /boot/grub/grub.cfg<br>

- (reboot)<br>

# VFIO configuration
- sudo nano /etc/modprobe.d/vfio.conf:<br>
options vfio-pci ids=10de:25a2,10de:2291<br>
softdep nvidia pre: vfio-pci<br>
- sudo mkinitcpio -P linux<br>
- (reboot)<br>

# Packages (There might be some other dependencies)
- sudo pacman -S qemu libvirt ovmf virt-manager 
- sudo systemctl enable --now libvirtd

# Create empty "Microsoft basic data" NVME partition
- sudo cfdisk /dev/nvme0n1

# Virt-Manager Configuration
- Create a new VM
- Use UEFI firmware
- Select GTX PCI devices, Video and Audio Controller
- Use bridged network since NAT sucks.
- Use Emulated TIS TPM 2.0 Module for Windows 11
- If you get an error, try this: <br>
   sudo pacman -Sy swtpm

# Bridge
- ip a
- nmcli connection add type bridge-slave ifname enp0s20f0u4 (This might change. Type "ip a" and find the line starting with enpXs) master br0
- nmcli connection down "Connection name" (press tab if you need to)
- nmcli connection up bridge-br0
- nmcli connection up bridge-slave-enp30s0

# Looking Glass
- sudo virsh edit vmname: <br>
- Refer to https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#Adding_IVSHMEM_Device_to_virtual_machines
  
- sudo mkdir -p /etc/tempfiles.d
- sudo nvim /etc/tempfiles.d/10-looking-glass.conf:<br>
  f	/dev/shm/looking-glass	0660	kurultaysokak	kvm<br>
- sudo systemd-tmpfiles --create /etc/tempfiles.d/10-looking-glass.conf<br>
- yay looking-glass (Brc7-1)<br>

# Windows Installation
- (Basic Win11 installation)
- Install VirtIO drivers
- Install HP drivers (High Definition Audio, Both Intel and NVIDIA GPU)
- Install NVIDIA app
- Install Game Ready Drivers
- <a href="https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/upstream-virtio/">Install this</a>
- Unzip and use the upstream folder, open Device Manager, under "System Devices" find "PCI Standard RAM Controller"
- Click update, browse from the files and select path: upstream - Win10 - amd64
- Install Looking Glass Host of corresponding Linux client version.
- Use Video Virtio to get the audio done
- Sound should be default: ICH9 or the like

# Virtual Display Drivers (NEEDED.)
- <a href="https://github.com/VirtualDisplay/Virtual-Display-Driver">Here</a>
(You might not need to do this if setup automatically configures it for you)
- Drivers are hard-coded to C:\IddSampleDriver
- After basic setup, open Device Manager
- Click "Action"
- Click "Add Legacy Hardware"
- Select "...manually select from list"
- Select "Show All Devices"
- Click "Have Disk"
- Find IddSampleDriver folder and click it
- Select the only shown file

# Adjusting Display Settings
- Make the virtual display your default
- Choose resolution, if bigger than 1080: Change VM Looking Glass shmem unit to "64"
- In the advanced display sections, choose your desired refresh rate
- Enjoy fresh native Windows inside Linux
