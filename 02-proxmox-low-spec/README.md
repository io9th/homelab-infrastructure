# Proxmox on Constrained Hardware: Deploying a Cybersecurity Homelab

## Context and Objective

This project started with a discarded Dell OptiPlex 9020. The hardware specifications were heavily constrained for virtualization purposes:

**CPU:** Intel Core i5-4590

**RAM:** 4GB

**Storage:** 500GB HDD

The objective was to transform this machine into a Proxmox Virtual Environment (PVE) to simulate corporate server environments, test databases, and establish a secure laboratory for cybersecurity studies. After a bare-metal installation of Proxmox and configuring the network structure (FQDN: pve.local), the challenges of operating within extreme resource limits began.

# Phase 1: Virtual Machine Resource Starvation

### Predict
For the first deployment, I allocated 1GB (1024MB) of RAM to a new Ubuntu Server VM. The hypothesis was that 1GB would be sufficient to run a headless Linux installation process smoothly on the hypervisor.

### Observe
The installation process quickly hung and froze completely. Monitoring the Proxmox dashboard revealed that the RAM usage was maxing out.
I recreated the VM with 2GB of RAM and provisioned a new 50GB virtual disk to avoid corrupted files from the frozen install. However, the system crashed into the initramfs emergency shell with mount errors. Furthermore, checking memory allocation (`free -h`) showed 0B of Swap space available.

### Explain
The system crashed because the installer exhausted the available memory, and without a Swap partition to offload inactive pages to the disk, the kernel halted the processes. To recover the corrupted file system and permanently fix the memory constraint, I applied two engineering solutions:

## 1. File System Recovery:

Used the emergency shell to check and repair the root partition.

```bash
fsck /dev/pve/root -y
```

## 2. Swap File Provisioning:

Created and enabled a 2GB Swap file, ensuring it persisted across reboots by mapping it into the File System Table (`/etc/fstab`).

```Bash
# Allocate 2GB for the swap file
sudo fallocate -l 2GB /swapfile 

# Set strict permissions (root only)
sudo chmod 600 /swapfile

# Format as swap and enable it
sudo mkswap /swapfile
sudo swapon /swapfile

# Make the configuration persistent
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

# Phase 2: The "Time Travel" SSL Certificate Issue

### Predict
To manage the server remotely without exposing it directly to the internet, I attempted to install Tailscale using a standard `curl` command. The expectation was a straightforward package download.

### Observe
The command failed with the error: *SSL certificate problem: certificate is not yet valid.*
Network connectivity was functional (ping to 8.8.8.8 succeeded), but domain resolution for secure connections dropped.

### Explain
The physical CMOS battery on the motherboard had died, causing the hardware clock to reset to a default factory date (June, months in the past). Because SSL certificates rely on strict timestamp validation to ensure cryptographic security, the Tailscale server's valid certificate was interpreted by my machine as being from the "future" and therefore rejected.

Attempts to force NTP (Network Time Protocol) synchronization via `timedatectl` failed because the `systemd-timesyncd` service was unavailable. The solution was a manual override of the system clock to unblock `curl` and `apt`, allowing the system to install the necessary NTP daemons for automated syncing on future boots.

```Bash
# Manually overriding the system clock
date -s "2025-11-14 02:12:00"
```

# Phase 3: Package Manager Repositories (The 401 Error)

### Predict
With the clock synchronized, I ran apt-get update to refresh the package lists and proceed with the Tailscale deployment.

### Observe
The package manager returned a *401 Unauthorized error*, and the repository was flagged as not signed. The update process was completely blocked.

### Explain
By default, Proxmox routes apt requests to its paid Enterprise repository. Since I did not have an active subscription, the server rejected the connection.
The goal was to switch to the free pve-no-subscription repository. Modern Proxmox versions migrated from the legacy `.list` files to `.sources` files. I used grep to locate the active enterprise configurations, disabled them, and manually wrote the new repository route.

## 1. Locating and disabling the Enterprise repositories:

```Bash
grep -r "enterprise.proxmox.com" /etc/apt/

mv /etc/apt/sources.list.d/ceph.sources /etc/apt/sources.list.d/ceph.sources.disabled
mv /etc/apt/sources.list.d/pve-enterprise.sources /etc/apt/sources.list.d/pve-enterprise.sources.disabled
```

## 2. Provisioning the No-Subscription repository:

```Bash
nano /etc/apt/sources.list.d/pve-no-subscription.list
```

Added the following routing rule:

```Plaintext
deb http://download.proxmox.com/debian/pve trixie pve-no-subscription
```
After updating the sources, apt-get update ran successfully, clearing the path for the rest of the infrastructure deployment.
