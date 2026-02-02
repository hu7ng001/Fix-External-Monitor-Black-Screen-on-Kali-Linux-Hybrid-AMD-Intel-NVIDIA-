# Fixing NVIDIA Drivers on Kali Linux with Kernel 6.x (Hybrid Graphics Laptops)

## Problem Overview

Lenovo Legion 5 laptops with hybrid graphics (AMD CPU with iGPU + NVIDIA dGPU) experience black screens when connecting external monitors on Linux. This occurs because the AMD iGPU driver conflicts with NVIDIA, even when BIOS is set to discrete GPU mode.

## Symptoms

- Black screen on all displays when external monitors are connected
- System boots but hangs at black screen with ACPI and AMD GPU errors
- Display only works after manually running `sudo startx`
- Boot errors showing: `amdgpu ERROR: dp_get_max_link_enc_cap: Max link encoder caps unknown`

## Prerequisites

- Lenovo Legion 5 (or similar hybrid graphics laptop)
- NVIDIA GPU (tested with RTX 4060)
- AMD CPU with integrated graphics
- Kali Linux (or Debian-based distribution)

## The Complete Fix

### Step 1: Configure BIOS

1. Restart and press **F2** (or **Fn+F2**) to enter BIOS
2. Find "Switchable Graphics" or "Hybrid Mode" setting
3. Change to **"Discrete Mode"** (dGPU only)
4. Save and exit (F10)

### Step 2: Install NVIDIA Drivers and Tools

```bash
# Update system
sudo apt update
sudo apt upgrade

# Install NVIDIA drivers and configuration tools
sudo apt install nvidia-driver firmware-misc-nonfree

# Install NVIDIA configuration utilities
sudo apt update
sudo apt install nvidia-xconfig nvidia-settings
```

### Step 3: Generate NVIDIA X Configuration

```bash
# Generate proper Xorg configuration for NVIDIA
sudo nvidia-xconfig --allow-empty-initial-configuration
```

This creates `/etc/X11/xorg.conf` with the correct NVIDIA settings.

### Step 4: Blacklist Integrated GPU Drivers

Even in discrete GPU mode, the integrated GPU driver tries to load and causes conflicts. Blacklist it:

```bash
# Create blacklist file
sudo nano /etc/modprobe.d/blacklist-amdgpu.conf
```

Add these lines:

```
blacklist amdgpu
blacklist radeon
```

Save and exit (**Ctrl+O**, **Enter**, **Ctrl+X**).

### Step 5: Configure Kernel Boot Parameters

Edit GRUB to prevent integrated GPU drivers from loading:

```bash
sudo nano /etc/default/grub
```

Find the line `GRUB_CMDLINE_LINUX_DEFAULT` and change it to:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet modprobe.blacklist=amdgpu,radeon nvidia-drm.modeset=1"
```

Update GRUB:

```bash
sudo update-grub
```

### Step 6: Update Initramfs

Apply the driver blacklist to the boot image:

```bash
sudo update-initramfs -u
```

### Step 7: Install and Enable Display Manager

```bash
# Install LightDM display manager
sudo apt install lightdm

# Enable it to start at boot
sudo systemctl enable lightdm

# Set graphical boot target
sudo systemctl set-default graphical.target
```

### Step 8: Reboot

```bash
sudo reboot
```

Your system should now boot directly to the graphical login screen with full external monitor support.

## Verification

After reboot, verify everything is working:

```bash
# Check NVIDIA driver is loaded
lsmod | grep nvidia

# Verify integrated GPU driver is NOT loaded (should return nothing)
lsmod | grep amdgpu

# Check display manager is running
systemctl status lightdm

# List detected displays
xrandr
```

## What This Fix Does

1. **BIOS Discrete Mode:** Forces laptop to use only NVIDIA GPU
2. **NVIDIA Drivers:** Installs proper drivers for GPU control
3. **nvidia-xconfig:** Generates correct X server configuration for NVIDIA
4. **Blacklist Integrated GPU Drivers:** Prevents AMD iGPU drivers from loading and conflicting
5. **Kernel Parameters:** Ensures integrated GPU drivers stay disabled from boot
6. **Display Manager:** Enables automatic graphical login at boot

## Troubleshooting

### Still Getting Black Screen

If issues persist after reboot:

```bash
# Check X server errors
cat /var/log/Xorg.0.log | grep "(EE)"

# Check display manager logs
journalctl -b -u lightdm

# Verify AMD driver is actually blacklisted
lsmod | grep amdgpu
# Should return nothing
```

### Manual X Start Still Required

If you need to manually run `sudo startx`:

```bash
# Check if display manager is enabled
systemctl is-enabled lightdm

# Re-enable if needed
sudo systemctl enable lightdm

# Check boot target
systemctl get-default
# Should show: graphical.target
```

### External Monitors Not Detected

```bash
# Force detection
xrandr --output HDMI-1 --auto

# Open NVIDIA settings for configuration
nvidia-settings
```

## Using Multiple Monitors

Configure multiple displays with:

```bash
# Use NVIDIA Settings GUI
nvidia-settings

# Or use xrandr command line
xrandr --output HDMI-1 --auto --right-of eDP-1
```

## Important Notes

- **Battery Life:** Discrete mode significantly reduces battery life (30-50%) since the NVIDIA GPU is always active
- **Updates:** After kernel updates, you may need to reinstall `linux-headers-$(uname -r)` and run `sudo update-initramfs -u`
- **Port Usage:** Try different ports (USB-C, HDMI) if one doesn't work - they may be wired differently

## Why This Works

The root cause is the integrated GPU driver (amdgpu) trying to initialize during boot, even when BIOS is set to discrete GPU mode. This causes:

1. Display port initialization errors
2. Conflicts with NVIDIA driver
3. X server failing to start properly

By blacklisting the integrated GPU driver at both the kernel level and boot parameters, we ensure only the NVIDIA driver loads, eliminating all conflicts.

## Summary of Commands

Here's the complete sequence of commands that fixed the issue:

```bash
# Install NVIDIA drivers and tools
sudo apt update
sudo apt upgrade
sudo apt install nvidia-driver firmware-misc-nonfree
sudo apt update
sudo apt install nvidia-xconfig nvidia-settings

# Generate X configuration
sudo nvidia-xconfig --allow-empty-initial-configuration

# Blacklist AMD drivers
sudo nano /etc/modprobe.d/blacklist-amdgpu.conf
# Add: blacklist amdgpu
# Add: blacklist radeon

# Update GRUB
sudo nano /etc/default/grub
# Modify: GRUB_CMDLINE_LINUX_DEFAULT="quiet modprobe.blacklist=amdgpu,radeon nvidia-drm.modeset=1"
sudo update-grub

# Update initramfs
sudo update-initramfs -u

# Setup display manager
sudo apt install lightdm
sudo systemctl enable lightdm
sudo systemctl set-default graphical.target

# Reboot
sudo reboot
```

## Tested Configuration

- **Laptop:** Lenovo Legion 5
- **GPU:** NVIDIA GeForce RTX 4060 Laptop
- **CPU:** AMD with integrated graphics
- **OS:** Kali Linux 2024.4
- **Kernel:** 6.17.10-kali-amd64
- **NVIDIA Driver:** 550.163.01

This solution provides a stable, working graphical environment with full external monitor support on Kali Linux for both AMD and Intel CPU-based laptops.

---

## For Intel CPU Users

If you have an Intel CPU with integrated graphics instead of AMD, follow the same steps above but make these modifications:

**Step 4 - Blacklist Intel drivers instead:**

```bash
# Create blacklist file
sudo nano /etc/modprobe.d/blacklist-intel.conf
```

Add these lines:

```
blacklist i915
blacklist intel_agp
```

Save and exit.

**Step 5 - Use Intel-specific kernel parameters:**

```bash
sudo nano /etc/default/grub
```

Change to:

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet modprobe.blacklist=i915,intel_agp nvidia-drm.modeset=1"
```

Then continue:

```bash
sudo update-grub
sudo update-initramfs -u
```

**Verification:**

```bash
# Verify Intel driver is NOT loaded (should return nothing)
lsmod | grep i915
```

All other steps remain identical to the AMD instructions above.