# Home-Network

# Complete OPNsense on Proxmox Installation Plan for Beelink EQ14

## Prerequisites and Hardware Setup

### Hardware Requirements
- **Beelink EQ14** with at least 2 Ethernet ports
- **Minimum 8GB RAM** (16GB recommended for IDS/IPS)
- **128GB+ storage** (SSD recommended)
- **USB drive** for Proxmox installer (8GB+)
- **Temporary PC/laptop** for configuration
- **Ethernet cables**

### Pre-Installation Preparation

#### 1. Download Required Software
- **Proxmox VE ISO**: Download from https://www.proxmox.com/en/downloads
- **OPNsense DVD Image**: Download from https://opnsense.org/download/
- **Balena Etcher** or similar for USB imaging

#### 2. BIOS Configuration (Critical Step)
1. Boot Beelink EQ14 and enter BIOS (usually F2 or Delete during boot)
2. **Enable virtualization features**:
   - Intel VT-x (Virtualization Technology)
   - Intel VT-d (if available)
   - IOMMU (if available)
3. **Set boot priority** to USB first
4. **Disable Secure Boot** if enabled
5. Save and exit

#### 3. Network Planning
- **Port 1 (eth0)**: Proxmox management + OPNsense LAN
- **Port 2 (eth1)**: OPNsense WAN
- **IP Scheme**: 192.168.1.0/24 for LAN (adjustable)

## Phase 1: Proxmox Installation

### Step 1: Create Proxmox Installer
1. Use Balena Etcher to flash Proxmox ISO to USB drive
2. Insert USB into Beelink EQ14
3. Boot from USB

### Step 2: Install Proxmox
1. Select "Install Proxmox VE"
2. Accept EULA
3. **Filesystem Selection**:
   - Choose **ext4** for single disk setup (ZFS if you want advanced features)
   - Select your primary storage disk
4. **Location Settings**:
   - Set country, timezone, and keyboard layout
5. **Administrator Setup**:
   - Set strong root password
   - Enter valid email address
6. **Network Configuration**:
   - **Management Interface**: Select first Ethernet port (usually enp1s0)
   - **Hostname**: `proxmox-router` (or your preference)
   - **IP Address**: `192.168.1.100/24` (static IP for management)
   - **Gateway**: `192.168.1.1` (your current router)
   - **DNS**: `192.168.1.1` or `8.8.8.8`

### Step 3: Initial Proxmox Configuration
1. Reboot and remove USB drive
2. Connect **Port 1 to your current router/switch**
3. From another computer, browse to `https://192.168.1.100:8006`
4. Login with root and your password

### Step 4: Configure Proxmox Repositories (Essential)
**Option A: Using Helper Script (Recommended)**
1. In Proxmox web UI, go to your node > Shell
2. Run: `bash -c "$(wget -qLO - https://github.com/tteck/Proxmox/raw/main/misc/post-pve-install.sh)"`
3. Answer "Yes" to all prompts except disable high availability (unless you need clustering)
4. System will update and reboot

**Option B: Manual Method**
1. SSH to Proxmox or use Shell in web UI
2. Edit sources: `nano /etc/apt/sources.list`
3. Add community repo: `deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription`
4. Comment out enterprise repo in `/etc/apt/sources.list.d/pve-enterprise.list`
5. Update: `apt update && apt upgrade -y`

## Phase 2: Network Bridge Configuration

### Step 1: Identify Network Interfaces
1. In Proxmox web UI: Go to Node > Network
2. Note interface names (usually enp1s0, enp2s0, etc.)
3. **Document which physical port corresponds to which interface name**

### Step 2: Create Network Bridges

#### Configure LAN Bridge (vmbr0)
- **Purpose**: Proxmox management + OPNsense LAN
- **Already created during installation** - verify configuration:
  - Name: `vmbr0`
  - Bridge ports: `enp1s0` (first physical port)
  - IP: `192.168.1.100/24`

#### Create WAN Bridge (vmbr1)
1. Click "Create" > "Linux Bridge"
2. **Name**: `vmbr1`
3. **Bridge ports**: `enp2s0` (second physical port - verify this matches your intended WAN port)
4. **Comment**: `OPNsense WAN Interface`
5. **No IP configuration needed** (will be handled by OPNsense)
6. Click "Create"

### Step 3: Apply Network Configuration
1. Click "Apply Configuration" button
2. **Important**: Network will briefly disconnect - this is normal

## Phase 3: Storage Configuration

### Step 1: Configure VM Storage
If you have a second drive or want to separate VM storage:

1. Go to Node > Disks
2. If second disk needs wiping: use "Wipe Disk" button
3. Go to Node > Disks > Directory (for ext4) or ZFS (for ZFS)
4. Create new storage called `vm-storage`
5. Go to Datacenter > Storage
6. Edit `vm-storage` and enable "Disk image" and "Container"

### Step 2: Disable Default VM Storage (Recommended)
1. Go to Datacenter > Storage
2. Select `local-lvm` (or `local-zfs` if using ZFS)
3. Edit and **uncheck "Enable"** to prevent accidentally filling boot disk

## Phase 4: OPNsense VM Creation

### Step 1: Upload OPNsense ISO
1. Go to Node > local (storage) > ISO Images
2. Click "Upload"
3. Select your downloaded OPNsense ISO file
4. Wait for upload to complete

### Step 2: Create OPNsense VM
1. Click "Create VM" (top right)
2. **General Tab**:
   - VM ID: `100`
   - Name: `OPNsense`
   - Advanced: Check "Start at boot"
   - Start/Shutdown order: `1`

3. **OS Tab**:
   - Storage: `local`
   - ISO image: Select OPNsense ISO
   - Guest OS: Linux (default is fine)

4. **System Tab** (Important for Performance):
   - Machine: `q35`
   - BIOS: `OVMF (UEFI)` - **Important for better performance**
   - Add EFI Disk: Checked
   - EFI Storage: `vm-storage` or `local-lvm`
   - Pre-Enroll keys: **Unchecked** (OPNsense doesn't support Secure Boot)
   - Qemu Agent: Checked (recommended)

5. **Disks Tab**:
   - Storage: `vm-storage` or `local-lvm`
   - Disk size: `32 GB` (64GB if you plan heavy logging/packages)
   - Cache: `Write back` (for better performance)
   - SSD emulation: Check if using SSD storage
   - Discard: Check if using SSD storage

6. **CPU Tab**:
   - Cores: `4` (minimum 2, but 4 recommended)
   - CPU Type: `host` (for best performance)
   - Extra CPU Flags: Enable `AES` (for VPN performance)

7. **Memory Tab**:
   - Memory: `4096 MB` (8192 MB if using IDS/IPS like Suricata/Sensei)
   - Ballooning Device: Enabled

8. **Network Tab** (Only configure one for now):
   - Bridge: `vmbr1` (WAN interface)
   - Model: `VirtIO (paravirtualized)`
   - Multiqueue: `4` (match CPU core count)
   - Firewall: Unchecked

9. **Confirm Tab**:
   - **Uncheck "Start after created"** - we need to add another network interface first

### Step 3: Add LAN Network Interface
1. Select the OPNsense VM
2. Go to Hardware tab
3. Click "Add" > "Network Device"
4. **Bridge**: `vmbr0` (LAN interface)
5. **Model**: `VirtIO (paravirtualized)`
6. **Multiqueue**: `4`
7. Click "Add"

### Step 4: Verify Network Interface Order
In Hardware tab, you should see:
- **net0**: vmbr1 (WAN)
- **net1**: vmbr0 (LAN)

**Note the order** - this affects interface assignment in OPNsense.

## Phase 5: OPNsense Installation

### Step 1: Start VM and Install
1. Start the OPNsense VM
2. Click "Console" to access VM console
3. At "Press any key to start the configuration importer": **Wait and let it timeout**
4. At "Press any key to start the manual interface assignment": **Press any key**

### Step 2: Interface Assignment (Critical)
1. **Configure LAGGs?**: Press `n`
2. **Configure VLANs?**: Press `n`
3. **Interface Assignment**:
   - **WAN interface**: `vtnet0` (first interface added to VM)
   - **LAN interface**: `vtnet1` (second interface added to VM)
   - **OPT interfaces**: Just press Enter to skip
4. **Proceed with configuration?**: Press `y`

### Step 3: Install OPNsense
1. At login prompt:
   - Username: `installer`
   - Password: `opnsense`
2. **Keymap**: Select appropriate (US is default)
3. **Partitioning**: 
   - Select "Auto (UFS)" - **Don't use ZFS in VM**
   - Select the virtual disk (da0)
4. **Swap**: Accept default
5. **Root Password**: Set a strong password
6. Wait for installation to complete
7. Remove installation medium: Go to Hardware > CD/DVD Drive > Edit > "Do not use any media"
8. Reboot VM

## Phase 6: Initial OPNsense Configuration

### Step 1: Connect for Configuration
**Important**: You need to temporarily connect your configuration PC directly to the Beelink's first Ethernet port (same as Proxmox management) or configure your PC with a static IP in the 192.168.1.x range.

1. **Set static IP on your PC**:
   - IP: `192.168.1.10`
   - Subnet: `255.255.255.0` (or /24)
   - Gateway: `192.168.1.1`
   - DNS: `192.168.1.1`

2. **Access OPNsense Web Interface**:
   - Browse to `https://192.168.1.1`
   - Username: `root`
   - Password: (what you set during installation)

### Step 2: Initial Setup Wizard
1. **Welcome Screen**: Click "Next"
2. **General Information**:
   - Hostname: `opnsense`
   - Domain: Your domain or `local`
   - Primary/Secondary DNS: `8.8.8.8` and `8.8.4.4`
3. **Time Server**: Leave default or use `pool.ntp.org`
4. **Configure WAN Interface**:
   - Type: DHCP (or Static if you know your ISP settings)
   - If static, enter your ISP provided IP, gateway, and DNS
5. **Configure LAN Interface**:
   - IP: `192.168.1.1`
   - Subnet: `24`
6. **Set Admin Password**: Change from default
7. **Reload Configuration**

## Phase 7: Deployment and Testing

### Step 1: Disconnect from Current Network
1. **Important**: Shut down the OPNsense VM first
2. Disconnect Beelink from your current network
3. Connect your modem/ONT to **Port 2** (WAN) of Beelink
4. Connect your main switch to **Port 1** (LAN) of Beelink
5. Start OPNsense VM

### Step 2: Verify Connectivity
1. Your PC should get an IP from OPNsense DHCP (192.168.1.x range)
2. Test internet connectivity
3. Access OPNsense at `https://192.168.1.1`
4. Check WAN status in Status > Interfaces

### Step 3: Configure Firewall Rules (if needed)
Basic LAN to WAN rule should exist by default, but verify:
1. Firewall > Rules > LAN
2. Should see "Default allow LAN to any rule"

## Phase 8: Advanced Configuration and Optimization

### Step 1: Install Qemu Guest Agent (Recommended)
1. In OPNsense: System > Firmware > Plugins
2. Install `os-qemu-guest-agent`
3. Enable in Services > Qemu Guest Agent

### Step 2: Performance Optimization
1. **System > Settings > Miscellaneous**:
   - Thermal sensors: Intel Core
   - Hardware crypto: Enable AES-NI CPU Crypto & Thermal Sensors
2. **System > Settings > Tunables**:
   - Add `net.inet.ip.fastforwarding` = 1 (for better routing performance)

### Step 3: Update and Backup
1. **Update OPNsense**: System > Firmware > Updates
2. **Create Configuration Backup**: System > Configuration > Backups

## Phase 9: VLAN Setup and Common Pitfalls (Optional)

If you plan to implement VLANs later, be aware of these critical considerations:

### VLAN Planning Considerations
1. **Multicast Traffic (mDNS)**:
   - Smart home devices, AirPlay, Chromecast rely on mDNS for discovery
   - Enable **mDNS Repeater** in Services > mDNS Repeater
   - Enable **IGMP Snooping** on switches
   - Create firewall rules allowing UDP port 5353 between VLANs

2. **Switch Compatibility**:
   - **Unmanaged switches strip VLAN tags** - they collapse everything to VLAN 1
   - Use **managed switches only** in VLAN paths
   - Alternative: Use PVID (Port VLAN ID) on managed switch ports connected to unmanaged switches

3. **DHCP and IP Assignment Issues**:
   - If clients get 169.254.x.x addresses, check:
     - VLAN tags on trunk ports to router
     - DHCP pool size (expand range or reduce lease time to 15 minutes for testing)
     - Proxy-ARP conflicts from Proxmox/Docker hosts

### OPNsense VLAN Firewall Behavior
**Critical**: OPNsense blocks all inter-VLAN traffic by default (unlike pfSense/UniFi)

Required firewall rules for each VLAN:
1. **Allow VLAN to WAN**: Interfaces > [VLAN] > Add rule allowing LAN net to any (for internet access)
2. **Allow DNS**: Create rule allowing VLAN to router IP on port 53
3. **Allow mDNS**: Create rule allowing UDP 5353 to other VLANs if device discovery needed
4. **NAT Rules**: Should be automatically created, but verify in Firewall > NAT > Outbound

### DNS Configuration for VLANs
1. **Services > Unbound DNS > General Settings**:
   - Add each VLAN interface to "Network Interfaces" list
   - Ensure "Register DHCP leases" is enabled for local name resolution

### Common VLAN Troubleshooting Steps
1. **No Internet on VLAN**:
   - Check firewall rules (OPNsense blocks by default)
   - Verify NAT outbound rules exist
   - Test DNS resolution (`nslookup 8.8.8.8`)

2. **Device Discovery Issues**:
   - Enable mDNS Repeater
   - Check UDP 5353 firewall rules
   - Verify IGMP Snooping on managed switches

3. **DHCP Assignment Problems**:
   - Verify VLAN tagged on all trunk ports
   - Check DHCP pool size vs. connected devices
   - Look for IP conflicts with virtualization hosts

## Troubleshooting Common Issues

### Network Interface Problems
- **Wrong interface assignment**: Use console option 1 to reassign interfaces
- **No WAN connectivity**: Check physical cable connections and VM network settings
- **Can't access web interface**: Verify PC is on same network (192.168.1.x) and firewall allows access

### Performance Issues
- **Slow throughput**: Ensure VirtIO drivers are used and multiqueue is enabled
- **High CPU usage**: Increase CPU cores allocated to VM
- **Memory issues**: Increase RAM allocation, especially if using IDS/IPS

### VM Boot Issues
- **Won't boot**: Check UEFI settings and ensure EFI disk is present
- **Interface assignment problems**: Use VM console to manually assign interfaces

## Security Considerations

1. **Change default passwords** on both Proxmox and OPNsense
2. **Enable automatic updates** for both systems
3. **Configure proper firewall rules** in OPNsense
4. **Consider VPN access** for remote management
5. **Regular configuration backups** of both Proxmox and OPNsense

This plan should give you a complete, production-ready virtualized router setup on your Beelink EQ14!
