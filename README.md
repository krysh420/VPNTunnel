# VPNTunnel With IPS/IDS Deployment
A private tunnel to securely access your devices at home and the wider web even on your public networks.

# VPN Server Setup Guide with Arch Linux, WireGuard, and Snort IDS/IPS

This README guides you through installing Arch Linux, setting up a secure VPN server using WireGuard, and configuring Snort 3 for intrusion detection and prevention. It includes troubleshooting tips for common issues.

---

## Table of Contents

1. [System Requirements](#system-requirements)
2. [Arch Linux Installation](#arch-linux-installation)
3. [Post-Install Configuration](#post-install-configuration)
4. [WireGuard Setup](#wireguard-setup)
5. [Snort 3 Setup (IPS Mode)](#snort-3-setup-ips-mode)
6. [Logging and Monitoring](#logging-and-monitoring)
7. [Firewall/NAT Configuration](#firewallnat-configuration)
8. [Common Troubleshooting](#common-troubleshooting)

---

## System Requirements

* x86\_64 device with UEFI or BIOS
* At least 10GB disk space (BTRFS preferred)
* 2 network interfaces (e.g. wlan1 for internet, wg0 for VPN)
* Root access

---

## Arch Linux Installation

1. Boot from Arch ISO.
2. Partition disk (replace `/dev/sdX` accordingly):

   ```bash
   cgdisk /dev/sdX
   # Create EFI (512M), root, and optionally /home partitions
   ```
3. Format partitions:

   ```bash
   mkfs.fat -F32 /dev/sdX1  # EFI
   mkfs.btrfs /dev/sdX2     # root
   mount /dev/sdX2 /mnt
   btrfs subvolume create /mnt/@
   btrfs subvolume create /mnt/@home
   umount /mnt

   mount -o compress=zstd,subvol=@ /dev/sdX2 /mnt
   mkdir /mnt/home
   mount -o compress=zstd,subvol=@home /dev/sdX2 /mnt/home
   mkdir /mnt/boot
   mount /dev/sdX1 /mnt/boot
   ```
4. Install base system:

   ```bash
   pacstrap -K /mnt base linux linux-firmware btrfs-progs
   genfstab -U /mnt >> /mnt/etc/fstab
   arch-chroot /mnt
   ```
5. Basic config:

   ```bash
   ln -sf /usr/share/zoneinfo/Region/City /etc/localtime
   hwclock --systohc
   echo "LANG=en_US.UTF-8" > /etc/locale.conf
   locale-gen
   echo myhostname > /etc/hostname
   passwd
   ```
6. Install bootloader:

   ```bash
   pacman -S grub efibootmgr
   grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
   grub-mkconfig -o /boot/grub/grub.cfg
   ```

---

## Post-Install Configuration

* Add user:

  ```bash
  useradd -mG wheel myuser
  passwd myuser
  echo '%wheel ALL=(ALL) ALL' >> /etc/sudoers
  ```
* Enable network:

  ```bash
  pacman -S networkmanager
  systemctl enable NetworkManager
  ```
* Reboot.

---

## WireGuard Setup

1. Install:

   ```bash
   sudo pacman -S wireguard-tools
   ```
2. Create config:

   ```bash
   sudo nano /etc/wireguard/wg0.conf
   ```

   Example:

   ```ini
   [Interface]
   Address = 10.66.66.1/24
   ListenPort = <Port>
   PrivateKey = <ServerPrivateKey>
   SaveConfig = true

   [Peer]
   PublicKey = <ClientPublicKey>
   AllowedIPs = 10.66.66.2/32
   ```
3. Enable forwarding:

   ```bash
   echo 'net.ipv4.ip_forward=1' >> /etc/sysctl.d/99-sysctl.conf
   sysctl --system
   ```
4. Start WireGuard:

   ```bash
   systemctl enable wg-quick@wg0
   systemctl start wg-quick@wg0
   reboot now
   ```

---

## Snort 3 Setup (IPS Mode)

1. Install dependencies:

   ```bash
   sudo pacman -S cmake make libpcap daq3 luajit pcre2 zlib
   ```
2. Compile Snort 3 (or use prebuilt AUR):

   ```bash
   # Download source and follow official guide
   ```
3. Create config `/etc/snort/snort.lua`:

   ```lua
   HOME_NET = '10.66.66.0/24'
   EXTERNAL_NET = '!$HOME_NET'

   ips = {
     enable_builtin_rules = true,
     rules = {
       [[ alert tcp any any -> $HOME_NET 22 (msg:"SSH attempt"; sid:1000001;) ]],
       [[ alert udp any any -> any <port assigned on wireguard> (msg:"WG Traffic"; sid:1000002;) ]]
     }
   }
   ```
4. Run Snort inline:

   ```bash
   sudo snort -c /etc/snort/snort.lua -i wlan1 -Q --daq afpacket --daq-mode inline -l /var/log/snort
   ```

---

## Logging and Monitoring

Ensure logs are generated:

```bash
ls /var/log/snort/
```

Use `journalctl`, `tmux`, or `logrotate` to manage logs.

---

## Firewall/NAT Configuration

```bash
iptables -t nat -A POSTROUTING -o wlan1 -j MASQUERADE
iptables -A FORWARD -i wg0 -j ACCEPT
iptables -A FORWARD -o wg0 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

Persist with `iptables-save > /etc/iptables.rules` and load on boot.

---

## Common Troubleshooting

### 1. WireGuard connects but no internet

* Verify NAT rules
* Check if `wg0` is up
* Make sure DNS is set on the client

### 2. Snort not logging

* Check permissions on `/var/log/snort`
* Ensure rules are firing
* Validate config: `snort -T -c /etc/snort/snort.lua`

### 3. No IP update from DDNS (e.g. noip)

* Ensure internet works
* Restart `noip2`

### 4. Dual Boot Safe Boot

* Use `sbctl` or `shim-signed` to enroll keys
* Ensure UEFI Secure Boot is OFF if not signed

---

## Final Notes

* Regularly back up WireGuard keys and config.
* Keep Snort rules updated from community.
* Use monitoring tools like `htop`, `iftop`, or `netstat` for network checks.

---

