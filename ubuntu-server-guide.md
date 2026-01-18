# Ubuntu Server 24.04.3 LTS with ZFS Setup Guide

**Hardware**: IBM System x3250 M5  
**Storage**: 500GB SSD (OS) + 4x 4TB SATA drives (ZFS RAIDZ1)  
**Author**: Jeff  
**Date**: January 2026

---

## Table of Contents

- [Initial Installation](#initial-installation)
- [ZFS Pool Configuration](#zfs-pool-configuration)
- [File Sharing Setup](#file-sharing-setup)
- [Web Management Interface](#web-management-interface)
- [SSH Security](#ssh-security)
- [Monitoring & Maintenance](#monitoring--maintenance)
- [Maintenance Cheat Sheet](#maintenance-cheat-sheet)

---

## Initial Installation

### 1. Download and Install Ubuntu Server 24.04.3 LTS

1. Download from [ubuntu.com](https://ubuntu.com/download/server)
2. Create bootable USB using Rufus (Windows) or `dd` (Linux)
3. Boot from USB on IBM x3250 M5

### 2. Installation Configuration

- **Storage**: Use 500GB SSD for OS
- **Partitioning**: Accept defaults (LVM)
- **Network**: Configure network interface
- **User**: Create user `jeff` with password
- **SSH**: Enable OpenSSH server during installation
- **Don't select ZFS** during install (configure manually later)

### 3. Initial System Setup

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install essential tools
sudo apt install -y zfsutils-linux vim git curl wget htop net-tools

# Enable password authentication temporarily (for initial setup)
sudo sed -i 's/^PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
sudo systemctl restart ssh
```

---

## ZFS Pool Configuration

### 1. Identify Disks

```bash
# List disks by-id (better than /dev/sdX for ZFS)
ls -l /dev/disk/by-id/ | grep -v part

# Verify disk sizes
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT
```

### 2. Create ZFS Pool 'tank' with RAIDZ1

**Example with actual disk IDs:**

```bash
sudo zpool create -f tank raidz1 \
  /dev/disk/by-id/ata-ST4000NM0024_00FN144_00FN147IBM_Z4F05182 \
  /dev/disk/by-id/ata-WDC_WD40EFRX-68WT0N0_WD-WCC4E7ETD427 \
  /dev/disk/by-id/ata-ST4000NM0024_00FN144_00FN147IBM_Z4F04YYH \
  /dev/disk/by-id/ata-ST4000NM0024_00FN144_00FN147IBM_Z4F052LZ
```

**Note**: Replace with your actual disk IDs from `ls -l /dev/disk/by-id/`

### 3. Configure Pool Properties

```bash
# Set optimal properties
sudo zfs set compression=lz4 tank
sudo zfs set atime=off tank
sudo zfs set xattr=sa tank

# Verify pool status
sudo zpool status
sudo zfs list
```

### 4. Create Datasets

```bash
# Create datasets for different purposes
sudo zfs create tank/data
sudo zfs create tank/backups

# Optional: Create additional datasets
# sudo zfs create tank/docker
# sudo zfs create tank/vms

# Optimize recordsize for different workloads
sudo zfs set recordsize=1M tank/backups    # Large sequential files
sudo zfs set recordsize=128k tank/data     # General purpose

# List all datasets
sudo zfs list -r tank
```

**Expected Result**: ~10.4TB usable space (RAIDZ1 = 3 drives capacity)

---

## File Sharing Setup

### NFS Configuration

```bash
# Install NFS server
sudo apt install -y nfs-kernel-server

# Enable NFS service
sudo systemctl enable --now nfs-server

# Configure ZFS datasets for NFS sharing
# Adjust network range to match your homelab
sudo zfs set sharenfs="rw=@192.168.1.0/24,no_root_squash" tank/data
sudo zfs set sharenfs="rw=@192.168.1.0/24,no_root_squash" tank/backups

# Verify NFS exports
sudo exportfs -v
showmount -e localhost
```

**Client Mount Example:**
```bash
# From Linux client
sudo mount -t nfs server-ip:/tank/data /mnt/data
```

### Samba Configuration

```bash
# Install Samba
sudo apt install -y samba samba-common-bin

# Enable Samba services
sudo systemctl enable --now smbd
sudo systemctl enable --now nmbd

# Backup original config
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.bak

# Edit Samba configuration
sudo nano /etc/samba/smb.conf
```

**Add to `/etc/samba/smb.conf`:**

```ini
[data]
   path = /tank/data
   browseable = yes
   read only = no
   guest ok = no
   valid users = jeff
   create mask = 0775
   directory mask = 0775

[backups]
   path = /tank/backups
   browseable = yes
   read only = no
   guest ok = no
   valid users = jeff
   create mask = 0775
   directory mask = 0775
```

```bash
# Set Samba password
sudo smbpasswd -a jeff

# Test configuration
sudo testparm

# Restart Samba
sudo systemctl restart smbd nmbd
```

**Windows Access:**
```
\\server-ip\data
\\server-ip\backups
```

### Firewall Configuration (if using UFW)

```bash
# Allow NFS and Samba through firewall
# Adjust network range to match your homelab
sudo ufw allow from 192.168.0.0/16 to any port nfs
sudo ufw allow from 192.168.0.0/16 to any port 445
sudo ufw allow from 192.168.0.0/16 to any port 139
sudo ufw allow 9090/tcp  # Cockpit
```

---

## Web Management Interface

### Install Cockpit

```bash
# Install Cockpit and modules
sudo apt install -y cockpit cockpit-pcp cockpit-storaged

# Enable and start
sudo systemctl enable --now cockpit.socket

# Verify it's running
sudo systemctl status cockpit.socket
```

### Install ZFS Manager Plugin (Optional)

```bash
# Install dependencies
sudo apt install -y git make

# Clone and install cockpit-zfs-manager
cd /tmp
git clone https://github.com/optimans/cockpit-zfs-manager.git
cd cockpit-zfs-manager

# Manual installation
sudo mkdir -p /usr/share/cockpit/zfs
sudo cp -r zfs/* /usr/share/cockpit/zfs/

# Restart Cockpit
sudo systemctl restart cockpit.socket
```

**Access Cockpit:**
```
https://server-ip:9090
```

- **Username**: `jeff`
- **Password**: Your user password

### Configure NetworkManager for Cockpit

By default, Ubuntu Server uses `systemd-networkd` for network management, but Cockpit works better with NetworkManager. This allows you to manage network interfaces through Cockpit's GUI.

```bash
# Install NetworkManager
sudo apt install -y network-manager

# Stop and disable systemd-networkd
sudo systemctl stop systemd-networkd
sudo systemctl disable systemd-networkd
```

**Update Netplan Configuration:**

```bash
# Backup current netplan config
sudo cp /etc/netplan/50-cloud-init.yaml /etc/netplan/50-cloud-init.yaml.bak

# Edit the netplan file
sudo nano /etc/netplan/50-cloud-init.yaml
```

**Add `renderer: NetworkManager` to your netplan file:**

```yaml
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    eno1:
      dhcp4: true
    eno2:
      dhcp4: true
    # Add any additional interfaces as needed
```

**Apply the changes:**

```bash
# Apply netplan changes
sudo netplan apply

# Start and enable NetworkManager
sudo systemctl enable --now NetworkManager

# Verify NetworkManager is managing interfaces
nmcli device status

# Check network connectivity
ping -c 4 8.8.8.8
```

**Verify in Cockpit:**
1. Refresh Cockpit in your browser
2. Go to **Networking** tab
3. Interfaces should now be listed as "managed"
4. You can now configure network settings through the GUI

---

## SSH Security

### Add SSH Public Key

**From your local machine:**

```bash
# Copy your public key to the server
ssh-copy-id jeff@server-ip
```

**Or manually on the server:**

```bash
# Create .ssh directory
mkdir -p ~/.ssh
chmod 700 ~/.ssh

# Add your public key
echo "your-public-key-here" >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

### Disable Password Authentication

**Test key authentication first!**

```bash
# Test from local machine
ssh jeff@server-ip
```

**Once confirmed working:**

```bash
# Disable password authentication
sudo sed -i 's/^PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
sudo sed -i 's/^#PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config

# Restart SSH service
sudo systemctl restart ssh
```

**Note**: Ubuntu uses `ssh` service, not `sshd`

---

## Monitoring & Maintenance

### Enable Automatic ZFS Snapshots

```bash
# Install zfs-auto-snapshot
sudo apt install -y zfs-auto-snapshot

# Enable auto-snapshots on datasets
sudo zfs set com.sun:auto-snapshot=true tank
sudo zfs set com.sun:auto-snapshot=true tank/data
sudo zfs set com.sun:auto-snapshot=true tank/backups

# Verify settings
sudo zfs get com.sun:auto-snapshot -r tank
```

**Snapshot Schedule** (via cron):
- **Frequent**: Every 15 minutes (keeps 4)
- **Hourly**: Every hour (keeps 24)
- **Daily**: Every day (keeps 31)
- **Weekly**: Every week (keeps 8)
- **Monthly**: Every month (keeps 12)

### Enable Monthly ZFS Scrubs

```bash
# Enable monthly scrub for pool 'tank'
sudo systemctl enable zfs-scrub-monthly@tank.timer
sudo systemctl start zfs-scrub-monthly@tank.timer

# Verify timer
sudo systemctl list-timers | grep zfs-scrub
```

**Scrubs run on the 1st of each month**

### Install SMART Monitoring

```bash
# Install smartmontools
sudo apt install -y smartmontools

# Enable SMART daemon
sudo systemctl enable smartmontools

# Quick health check
for disk in sda sdb sdc sdd sde; do
  echo "=== /dev/$disk ==="
  sudo smartctl -H /dev/$disk
done
```

### Install fail2ban

```bash
# Install fail2ban for SSH protection
sudo apt install -y fail2ban

# Enable and start
sudo systemctl enable --now fail2ban

# Check SSH jail status
sudo fail2ban-client status sshd
```

---

## Maintenance Cheat Sheet

### ZFS Health Monitoring

```bash
# Check pool status and health
sudo zpool status

# Check pool I/O statistics (live)
sudo zpool iostat -v 2

# Check space usage
sudo zfs list
sudo zfs list -o space

# Manual scrub
sudo zpool scrub tank

# Check scrub progress
sudo zpool status

# View snapshots
sudo zfs list -t snapshot
sudo zfs list -t snapshot -r tank
```

### ZFS Snapshot Management

```bash
# List all snapshots
sudo zfs list -t snapshot

# Create manual snapshot
sudo zfs snapshot tank/data@backup-$(date +%Y%m%d-%H%M)

# Delete specific snapshot
sudo zfs destroy tank/data@snapshot-name

# Restore from snapshot
sudo zfs rollback tank/data@snapshot-name

# View snapshot space usage
sudo zfs list -t snapshot -o name,used,refer
```

### NFS Management

```bash
# View active NFS exports
sudo exportfs -v
showmount -e localhost

# Reload NFS exports
sudo exportfs -ra

# Restart NFS server
sudo systemctl restart nfs-server

# Check NFS status
sudo systemctl status nfs-server
```

### Samba Management

```bash
# Test Samba configuration
sudo testparm

# Restart Samba
sudo systemctl restart smbd nmbd

# Check Samba status
sudo systemctl status smbd
sudo systemctl status nmbd

# List active connections
sudo smbstatus

# Change Samba password
sudo smbpasswd jeff
```

### SMART Disk Health

```bash
# Check all drives health
for disk in sda sdb sdc sdd sde; do
  echo "=== /dev/$disk ==="
  sudo smartctl -H /dev/$disk
done

# Detailed SMART info
sudo smartctl -a /dev/sda

# Run short self-test
sudo smartctl -t short /dev/sda

# Run long self-test (takes hours)
sudo smartctl -t long /dev/sda

# View test results
sudo smartctl -l selftest /dev/sda
```

### fail2ban Management

```bash
# Check SSH jail status
sudo fail2ban-client status sshd

# Unban an IP
sudo fail2ban-client set sshd unbanip 192.168.1.100

# View banned IPs
sudo fail2ban-client status sshd

# Restart fail2ban
sudo systemctl restart fail2ban
```

### System Maintenance

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Clean up packages
sudo apt autoremove -y
sudo apt autoclean

# Check system resources
htop

# View system logs
sudo journalctl -xe
sudo journalctl -u smbd
sudo journalctl -u nfs-server

# Check disk usage
df -h
du -sh /tank/*
```

### Quick System Status Command

```bash
# One-command system overview
echo "=== ZFS Health ===" && sudo zpool status && \
echo -e "\n=== Snapshots ===" && sudo zfs list -t snapshot | tail -5 && \
echo -e "\n=== Services ===" && \
for svc in cockpit.socket nfs-server smbd nmbd fail2ban smartmontools; do \
  printf "%-20s : %s\n" "$svc" "$(systemctl is-active $svc)"; \
done && \
echo -e "\n=== Disk Space ===" && sudo zfs list
```

---

## Emergency Recovery

### Import ZFS Pool

```bash
# If pool doesn't auto-mount after reboot
sudo zpool import tank

# Force import if needed
sudo zpool import -f tank

# Check for errors
sudo zpool status -v
```

### Recovery Mode

1. Reboot server
2. Select "Advanced options" at GRUB menu
3. Select "Recovery mode"
4. Import ZFS pool manually if needed

---

## Service Information

### Access Points

- **Cockpit GUI**: `https://server-ip:9090` (user: jeff)
- **SSH**: `ssh jeff@server-ip` (key-only authentication)
- **NFS Shares**: 
  - `server-ip:/tank/data`
  - `server-ip:/tank/backups`
- **Samba Shares**: 
  - `\\server-ip\data`
  - `\\server-ip\backups`

### Important Ports

- **22**: SSH
- **111**: NFS rpcbind
- **139**: Samba NetBIOS
- **445**: Samba SMB
- **2049**: NFS
- **9090**: Cockpit Web Interface

### Service Status Check

```bash
# Check all critical services
for service in cockpit.socket nfs-server smbd nmbd fail2ban smartmontools; do
  printf "%-20s : %s\n" "$service" "$(systemctl is-active $service)"
done
```

---

## System Specifications

- **OS**: Ubuntu Server 24.04.3 LTS
- **Hardware**: IBM System x3250 M5
- **CPU**: (varies by model)
- **RAM**: 32GB
- **OS Drive**: 500GB Crucial MX500 SSD
- **Data Drives**: 4x 4TB (3x Seagate NM0024 + 1x WD Red)
- **ZFS Configuration**: RAIDZ1 (~10.4TB usable)
- **Compression**: LZ4 (enabled)

---

## Scheduled Tasks

| Task | Frequency | Command/Service |
|------|-----------|----------------|
| ZFS Frequent Snapshots | Every 15 min | `zfs-auto-snapshot` (cron) |
| ZFS Hourly Snapshots | Every hour | `zfs-auto-snapshot` (cron) |
| ZFS Daily Snapshots | Daily | `zfs-auto-snapshot` (cron) |
| ZFS Weekly Snapshots | Weekly | `zfs-auto-snapshot` (cron) |
| ZFS Monthly Snapshots | Monthly | `zfs-auto-snapshot` (cron) |
| ZFS Scrub | Monthly (1st) | `zfs-scrub-monthly@tank.timer` |
| SMART Monitoring | Continuous | `smartd` |
| Security Monitoring | Continuous | `fail2ban` |

---

## Best Practices

### ZFS

- **Regular Scrubs**: Monthly scrubs verify data integrity
- **Snapshots**: Use liberally - they're cheap and instant
- **Compression**: LZ4 is essentially free performance-wise
- **Monitoring**: Check `zpool status` regularly
- **UPS Recommended**: Protect against power loss corruption

### Security

- **SSH Keys Only**: Never enable password authentication in production
- **fail2ban**: Monitors and blocks brute force attempts
- **Firewall**: Restrict access to trusted networks only
- **Updates**: Keep system updated regularly

### Backups

- **3-2-1 Rule**: 3 copies, 2 different media, 1 offsite
- **ZFS Snapshots**: Great for quick recovery, not a backup
- **Test Restores**: Regularly test snapshot rollbacks
- **Offsite**: Consider backing up critical data offsite

### Maintenance

- **Monthly**: Check SMART status, review logs
- **Quarterly**: Test disaster recovery procedures
- **Annually**: Review and update documentation

---

## Troubleshooting

### ZFS Pool Won't Mount

```bash
# Check pool status
sudo zpool status

# Import pool
sudo zpool import tank

# Check for errors
sudo zpool status -v

# Clear errors (if minor)
sudo zpool clear tank
```

### NFS Share Not Accessible

```bash
# Check NFS is running
sudo systemctl status nfs-server

# Verify exports
sudo exportfs -v

# Reload exports
sudo exportfs -ra

# Check firewall
sudo ufw status
```

### Samba Share Not Accessible

```bash
# Check Samba status
sudo systemctl status smbd nmbd

# Test configuration
sudo testparm

# Restart Samba
sudo systemctl restart smbd nmbd

# Check user permissions
ls -la /tank/data
```

### SSH Connection Refused

```bash
# Check SSH service
sudo systemctl status ssh

# Verify SSH is listening
sudo ss -tlnp | grep 22

# Check firewall
sudo ufw status

# Review logs
sudo journalctl -u ssh
```

### High Disk I/O

```bash
# Check ZFS I/O stats
sudo zpool iostat -v 1

# Check running processes
sudo iotop

# Check for scrub/resilver
sudo zpool status
```

---

## Additional Resources

- [Ubuntu Server Documentation](https://ubuntu.com/server/docs)
- [OpenZFS Documentation](https://openzfs.github.io/openzfs-docs/)
- [Cockpit Project](https://cockpit-project.org/)
- [ZFS Best Practices Guide](https://openzfs.github.io/openzfs-docs/Performance%20and%20Tuning/Workload%20Tuning.html)

---

## Changelog

| Date | Change |
|------|--------|
| 2026-01-18 | Initial setup completed |

---

## License

This guide is provided as-is for personal and educational use.

---

**Author**: Jeff  
**Last Updated**: January 18, 2026 
