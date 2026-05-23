# 📖 Dokumentasi Instalasi Arch Linux — CIS Compliant
> **Referensi:** CIS Benchmark for Linux | Arch Linux Wiki  
> **Target:** Arch Linux dengan enkripsi penuh, partisi CIS, dan paket keamanan lengkap

---

## 📋 Daftar Isi

1. [Persiapan Instalasi](#1-persiapan-instalasi)
2. [Partisi Disk — CIS Layout](#2-partisi-disk--cis-layout)
3. [Enkripsi LUKS + LVM (System)](#3-enkripsi-luks--lvm-system)
4. [Enkripsi LUKS + LVM (Data)](#4-enkripsi-luks--lvm-data)
5. [Partisi Keys & Boot](#5-partisi-keys--boot)
6. [Mount Semua Partisi](#6-mount-semua-partisi)
7. [Instalasi Base System](#7-instalasi-base-system)
8. [Konfigurasi Sistem (arch-chroot)](#8-konfigurasi-sistem-arch-chroot)
9. [📦 [BOOT] Booster — initramfs](#9--boot-booster--initramfs)
10. [🔒 [SECUREBOOT] Setup Secure Boot](#10--secureboot-setup-secure-boot)
11. [Reboot ke Sistem](#11-reboot-ke-sistem)
12. [📦 [DESKTOP] Instalasi XFCE](#12--desktop-instalasi-xfce)
13. [📦 [FILEMANAGER] Instalasi Superfile](#13--filemanager-instalasi-superfile)
14. [📦 [FIREWALL] Konfigurasi iptables](#14--firewall-konfigurasi-iptables)
15. [📦 [CONTAINER] Instalasi Podman](#15--container-instalasi-podman)
16. [📦 [MULTIMEDIA] MPD + MPC + MPV](#16--multimedia-mpd--mpc--mpv)
17. [📦 [PASSWORD] KeePassXC](#17--password-keepassxc)
18. [📦 [KEY] libsecret (secret-tool)](#18--key-libsecret-secret-tool)
19. [📦 [ACCESS] OpenSSH](#19--access-openssh)
20. [📦 [CONTAINER-APP] Omeka di Podman](#20--container-app-omeka-di-podman)
21. [Verifikasi Akhir](#21-verifikasi-akhir)

---

## Legenda Tanda

| Tanda | Keterangan |
|-------|-----------|
| 📦 | Package yang akan diinstall |
| ⚠️ | Peringatan penting |
| ✅ | Verifikasi/pengecekan |
| 🔴 | Wajib dilakukan saat instalasi |
| 🟢 | Bisa dilakukan pasca instalasi |
| 💡 | Tips & catatan |

---

# FASE 1 — SAAT INSTALASI (Live ISO)

> 🔴 Semua langkah di fase ini **wajib dilakukan sebelum reboot pertama**

---

## 1. Persiapan Instalasi

### 1.1 Boot dari Arch Linux ISO

```bash
# Verifikasi mode UEFI
ls /sys/firmware/efi/efivars
# Jika direktori ada → UEFI mode ✅
```

### 1.2 Koneksi Internet

```bash
# Cek koneksi
ping -c 3 archlinux.org

# Jika menggunakan WiFi
iwctl
  device list
  station wlan0 scan
  station wlan0 get-networks
  station wlan0 connect "NamaWiFi"
  exit
```

### 1.3 Sinkronisasi Waktu

```bash
timedatectl set-ntp true
timedatectl status
```

### 1.4 Identifikasi Disk

```bash
lsblk
# Contoh output:
# sda      — disk utama (misal 300GB)
# sdb      — disk kedua (opsional)

# Catat nama disk, contoh: /dev/sda
fdisk -l /dev/sda
```

---

## 2. Partisi Disk — CIS Layout

> 🔴 Struktur partisi **tidak bisa diubah** setelah instalasi selesai

### 2.1 Struktur Partisi yang Dituju

```
/dev/sda
├── sda1  1.5GB   EFI System      → /boot
├── sda2  2GB     Linux (LUKS)    → keys (penyimpanan kunci)
├── sda3  35GB    Linux (LUKS)    → system (root, var, tmp, dll)
└── sda4  200GB+  Linux (LUKS)    → data (docker, http, libvirt, home)
```

### 2.2 Membuat Partisi dengan cfdisk

```bash
cfdisk /dev/sda
```

Ikuti langkah berikut di dalam cfdisk:

```
1. Pilih: gpt  (untuk UEFI)

2. Buat partisi 1 — BOOT/EFI:
   → New → Size: 1.5G
   → Type: EFI System

3. Buat partisi 2 — KEYS:
   → New → Size: 2G
   → Type: Linux filesystem

4. Buat partisi 3 — SYSTEM:
   → New → Size: 35G
   → Type: Linux filesystem

5. Buat partisi 4 — DATA:
   → New → Size: (sisa disk, misal 200G)
   → Type: Linux filesystem

6. Write → yes → Quit
```

### 2.3 Verifikasi Partisi

```bash
lsblk /dev/sda
# Output yang diharapkan:
# sda
# ├─sda1   1.5G   (boot)
# ├─sda2   2G     (keys)
# ├─sda3   35G    (system)
# └─sda4   200G   (data)
```

---

## 3. Enkripsi LUKS + LVM (System)

> 🔴 Partisi system (sda3) — berisi root, var, tmp, log, audit

### 3.1 Setup LUKS pada Partisi System

```bash
# ⚠️ PENTING: luksFormat DULU sebelum apapun
cryptsetup luksFormat --sector-size 4096 /dev/sda3
# Ketik: YES (kapital)
# Masukkan passphrase yang kuat

# Buka container LUKS
cryptsetup luksOpen /dev/sda3 proc
# Masukkan passphrase yang sama
```

### 3.2 Setup LVM di Atas LUKS

```bash
# Buat Physical Volume
pvcreate /dev/mapper/proc

# ⚠️ Gunakan nama "vgsys", BUKAN "proc" (konflik dengan /proc)
vgcreate vgsys /dev/mapper/proc

# Verifikasi
pvdisplay
vgdisplay
```

### 3.3 Buat Logical Volumes — System

```bash
# ROOT — partisi utama sistem
lvcreate -L 20G vgsys -n root

# TEMP — /tmp (noexec untuk keamanan CIS)
lvcreate -L 2G vgsys -n vtmp

# VAR — /var
lvcreate -L 5G vgsys -n vars

# VLOG — /var/log
lvcreate -L 1.5G vgsys -n vlog

# VAUD — /var/log/audit
lvcreate -L 512M vgsys -n vaud

# VTMP2 — /var/tmp
lvcreate -L 2G vgsys -n vtmp2

# HOME — /home
lvcreate -L 10G vgsys -n home

# SWAP
lvcreate -L 4G vgsys -n swap

# Verifikasi semua LV terbuat
lvdisplay vgsys
```

### 3.4 Format Logical Volumes

```bash
# Format semua LV dengan ext4
mkfs.ext4 -b 4096 /dev/vgsys/root
mkfs.ext4 -q -b 4096 /dev/vgsys/vtmp
mkfs.ext4 -q -b 4096 /dev/vgsys/vars
mkfs.ext4 -q -b 4096 /dev/vgsys/vlog
mkfs.ext4 -q -b 4096 /dev/vgsys/vaud
mkfs.ext4 -q -b 4096 /dev/vgsys/vtmp2
mkfs.ext4 -q -b 4096 /dev/vgsys/home

# Format swap
mkswap /dev/vgsys/swap
```

---

## 4. Enkripsi LUKS + LVM (Data)

> 🔴 Partisi data (sda4) — berisi docker, http, libvirt, home user

### 4.1 Setup LUKS pada Partisi Data

```bash
cryptsetup luksFormat --sector-size 4096 /dev/sda4
# Ketik: YES
# Masukkan passphrase (boleh sama atau berbeda dengan system)

cryptsetup luksOpen /dev/sda4 data
```

### 4.2 Setup LVM Data

```bash
pvcreate /dev/mapper/data
vgcreate vgdata /dev/mapper/data
```

### 4.3 Buat Logical Volumes — Data

```bash
# DOCK — untuk Docker/Podman container storage
lvcreate -L 20G vgdata -n dock

# SRVC — untuk web server (/srv/http)
lvcreate -L 30G vgdata -n srvc

# HOST — untuk libvirt/VM images
lvcreate -L 50G vgdata -n host

# NETS — untuk home user
lvcreate -L 50G vgdata -n nets

# Verifikasi
lvdisplay vgdata
```

### 4.4 Format LV Data

```bash
mkfs.ext4 -q -b 4096 /dev/vgdata/dock
mkfs.ext4 -q -b 4096 /dev/vgdata/srvc
mkfs.ext4 -q -b 4096 /dev/vgdata/host
mkfs.ext4 -q -b 4096 /dev/vgdata/nets
```

---

## 5. Partisi Keys & Boot

### 5.1 Setup LUKS pada Partisi Keys

```bash
# Enkripsi partisi keys
cryptsetup luksFormat --sector-size 4096 /dev/sda2
# Ketik: YES
# Masukkan passphrase

# Buka partisi keys
cryptsetup luksOpen /dev/sda2 keys

# Format
mkfs.ext4 /dev/mapper/keys
```

### 5.2 Format Partisi Boot (EFI)

```bash
# Format sebagai FAT32 untuk EFI
mkfs.vfat -F32 -S 4096 -n BOOT /dev/sda1
```

---

## 6. Mount Semua Partisi

> 🔴 Urutan mount **sangat penting** — root dulu, baru yang lain

### 6.1 Mount Root

```bash
mount /dev/vgsys/root /mnt
```

### 6.2 Buat Direktori Mount

```bash
# Direktori untuk system
mkdir -p /mnt/{boot,tmp,home}
mkdir -p /mnt/var/{log,tmp}
mkdir -p /mnt/var/log/audit

# Direktori untuk data
mkdir -p /mnt/var/lib/docker
mkdir -p /mnt/srv/http
mkdir -p /mnt/var/lib/libvirt/images
mkdir -p /mnt/home/user

# Direktori untuk keys
mkdir -p /mnt/keys
```

### 6.3 Mount Semua Logical Volumes

```bash
# System LVs
mount /dev/vgsys/vtmp   /mnt/tmp
mount /dev/vgsys/vars   /mnt/var
mount /dev/vgsys/vlog   /mnt/var/log
mount /dev/vgsys/vaud   /mnt/var/log/audit
mount /dev/vgsys/vtmp2  /mnt/var/tmp
mount /dev/vgsys/home   /mnt/home

# Aktifkan swap
swapon /dev/vgsys/swap

# Data LVs
mount /dev/vgdata/dock  /mnt/var/lib/docker
mount /dev/vgdata/srvc  /mnt/srv/http
mount /dev/vgdata/host  /mnt/var/lib/libvirt/images
mount /dev/vgdata/nets  /mnt/home/user

# Keys
mount /dev/mapper/keys  /mnt/keys
chmod 700 /mnt/keys

# Boot/EFI
mount -o uid=0,gid=0,fmask=0077,dmask=0077 /dev/sda1 /mnt/boot
```

### 6.4 Verifikasi Mount

```bash
lsblk
# Semua partisi harus terlihat ter-mount
df -h | grep /mnt
```

---

## 7. Instalasi Base System

### 7.1 Pacstrap

```bash
pacstrap /mnt \
  base base-devel \
  linux linux-firmware \
  lvm2 cryptsetup \
  networkmanager \
  vim nano \
  sudo \
  efibootmgr \
  sbctl
```

### 7.2 Generate fstab

```bash
genfstab -U /mnt >> /mnt/etc/fstab

# ⚠️ WAJIB: Edit fstab untuk tambah mount options CIS
vim /mnt/etc/fstab
```

Edit fstab, tambahkan `nodev,nosuid,noexec` sesuai partisi:

```
# /tmp
UUID=xxx  /tmp      ext4  defaults,noatime,nodev,nosuid,noexec  0 2

# /var/tmp
UUID=xxx  /var/tmp  ext4  defaults,noatime,nodev,nosuid,noexec  0 2

# /var/log
UUID=xxx  /var/log  ext4  defaults,noatime,nodev,nosuid,noexec  0 2

# /var/log/audit
UUID=xxx  /var/log/audit  ext4  defaults,noatime,nodev,nosuid,noexec  0 2

# /home
UUID=xxx  /home     ext4  defaults,noatime,nodev,nosuid         0 2

# /boot (EFI — sudah ada dari genfstab, pastikan ada uid=0,gid=0,fmask=0077)
UUID=xxx  /boot     vfat  uid=0,gid=0,fmask=0077,dmask=0077    0 2
```

---

## 8. Konfigurasi Sistem (arch-chroot)

```bash
arch-chroot /mnt
```

### 8.1 Timezone & Locale

```bash
# Set timezone (sesuaikan)
ln -sf /usr/share/zoneinfo/Asia/Jakarta /etc/localtime
hwclock --systohc

# Set locale
echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
echo "id_ID.UTF-8 UTF-8" >> /etc/locale.gen
locale-gen

echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

### 8.2 Hostname & Hosts

```bash
echo "archlinux" > /etc/hostname

cat > /etc/hosts << EOF
127.0.0.1   localhost
::1         localhost
127.0.1.1   archlinux.localdomain archlinux
EOF
```

### 8.3 Root Password & User

```bash
# Set password root
passwd

# Buat user
useradd -m -G wheel,audio,video,storage -s /bin/bash namauser
passwd namauser

# Aktifkan sudo untuk wheel group
EDITOR=vim visudo
# Uncomment: %wheel ALL=(ALL:ALL) ALL
```

### 8.4 Konfigurasi mkinitcpio untuk LUKS+LVM

```bash
vim /etc/mkinitcpio.conf
```

Edit baris HOOKS:
```
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block encrypt lvm2 filesystems fsck)
```

---

## 9. 📦 [BOOT] Booster — initramfs

> 🔴 **Harus dikonfigurasi di dalam arch-chroot sebelum reboot**

### Install

```bash
pacman -S booster
```

### Konfigurasi

```bash
# Buat konfigurasi booster
cat > /etc/booster.yaml << EOF
universal: true
modules_force_load: dm-crypt,dm-mod
EOF

# Generate initramfs
booster build /boot/booster-linux.img
```

### Setup Bootloader (systemd-boot)

```bash
# Install systemd-boot ke EFI partition
bootctl install

# Konfigurasi loader
cat > /boot/loader/loader.conf << EOF
default arch.conf
timeout 5
console-mode max
editor no
EOF

# Dapatkan UUID partisi system
blkid /dev/sda3
# Catat UUID-nya

# Buat entry boot
cat > /boot/loader/entries/arch.conf << EOF
title   Arch Linux
linux   /vmlinuz-linux
initrd  /booster-linux.img
options cryptdevice=UUID=<UUID-sda3>:proc:allow-discards
options root=/dev/vgsys/root rw quiet
EOF
```

### Verifikasi ✅

```bash
ls -la /boot/booster-linux.img
bootctl list
```

---

## 10. 🔒 [SECUREBOOT] Setup Secure Boot

> 🔴 **Harus dilakukan sebelum reboot pertama**
> ⚠️ UEFI harus dalam mode Setup/Clear untuk enroll keys

### Setup Keys

```bash
# Buat secure boot keys
sbctl create-keys

# Enroll keys (termasuk Microsoft keys untuk kompatibilitas)
sbctl enroll-keys --microsoft

# Sign binary yang diperlukan
sbctl sign -s /boot/vmlinuz-linux
sbctl sign -s /boot/EFI/BOOT/BOOTX64.EFI
sbctl sign -s /boot/EFI/systemd/systemd-bootx64.efi

# Simpan list file yang perlu di-sign
sbctl verify
```

### Aktifkan Secure Boot

```bash
# Verifikasi status
sbctl status

# ⚠️ Reboot ke UEFI dan aktifkan Secure Boot setelah installasi selesai
```

---

## 11. Reboot ke Sistem

```bash
# Keluar dari chroot
exit

# Unmount semua
umount -R /mnt
swapoff /dev/vgsys/swap

# Tutup LUKS containers
cryptsetup luksClose data
cryptsetup luksClose keys
cryptsetup luksClose proc

# Reboot
reboot
```

> 💡 **Setelah reboot:** Login dengan user yang dibuat, lanjutkan instalasi package pasca-instalasi

---

---

# FASE 2 — PASCA INSTALASI

> 🟢 Semua langkah di fase ini dilakukan **setelah sistem berhasil booting**

---

## Persiapan Pasca-Instalasi

```bash
# Aktifkan NetworkManager
sudo systemctl enable --now NetworkManager

# Update sistem
sudo pacman -Syu

# Install AUR helper (yay)
sudo pacman -S git base-devel
git clone https://aur.archlinux.org/yay.git
cd yay && makepkg -si
cd .. && rm -rf yay
```

---

## 12. 📦 [DESKTOP] Instalasi XFCE

> 🟢 Desktop environment ringan dan stabil

### Install

```bash
sudo pacman -S \
  xfce4 \
  xfce4-goodies \
  lightdm \
  lightdm-gtk-greeter \
  lightdm-gtk-greeter-settings
```

### Aktifkan Display Manager

```bash
sudo systemctl enable lightdm
```

### Konfigurasi

```bash
# Set greeter
sudo vim /etc/lightdm/lightdm.conf
# Cari [Seat:*] dan set:
# greeter-session=lightdm-gtk-greeter
```

### Verifikasi ✅

```bash
sudo systemctl status lightdm
# Atau start manual
sudo systemctl start lightdm
```

---

## 13. 📦 [FILEMANAGER] Instalasi Superfile

> 🟢 File manager TUI modern berbasis terminal

### Install via AUR

```bash
yay -S superfile-bin
```

### Penggunaan Dasar

```bash
# Jalankan
spf

# Navigasi:
# Arrow keys  — gerak antar file
# Enter       — buka file/folder
# q           — keluar
# ?           — bantuan
# Tab         — switch panel
# d           — hapus file
# c           — copy
# v           — paste
```

### Verifikasi ✅

```bash
spf --version
```

---

## 14. 📦 [FIREWALL] Konfigurasi iptables

> 🟢 Firewall kernel Linux — buka port sesuai aplikasi yang berjalan

### Install

```bash
sudo pacman -S iptables
```

### Konfigurasi Rules

```bash
# Flush rules lama
sudo iptables -F

# Policy default — DROP semua
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT

# Izinkan loopback
sudo iptables -A INPUT -i lo -j ACCEPT

# Izinkan koneksi yang sudah established
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# ===== Buka port sesuai aplikasi =====

# SSH (OpenSSH)
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# MPD (Music Player Daemon) — hanya lokal
sudo iptables -A INPUT -p tcp --dport 6600 -s 127.0.0.1 -j ACCEPT

# Omeka/HTTP via Podman
sudo iptables -A INPUT -p tcp --dport 8080 -j ACCEPT

# =====================================

# Simpan rules
sudo iptables-save | sudo tee /etc/iptables/iptables.rules

# Aktifkan service
sudo systemctl enable --now iptables
```

### Verifikasi ✅

```bash
sudo iptables -L -n -v
sudo systemctl status iptables
```

---

## 15. 📦 [CONTAINER] Instalasi Podman

> 🟢 Container runtime rootless — lebih aman dari Docker

### Install Podman

```bash
sudo pacman -S podman podman-compose
```

### Install Podman Desktop (GUI)

```bash
yay -S podman-desktop
```

### Konfigurasi Rootless

```bash
# Aktifkan socket untuk user
systemctl --user enable --now podman.socket

# Setup subuid/subgid untuk user
sudo usermod --add-subuids 100000-165535 $(whoami)
sudo usermod --add-subgids 100000-165535 $(whoami)

# Konfigurasi storage ke LV yang sudah dibuat
mkdir -p ~/.config/containers

cat > ~/.config/containers/storage.conf << EOF
[storage]
driver = "overlay"
graphRoot = "/var/lib/docker"

[storage.options.overlay]
mount_program = "/usr/bin/fuse-overlayfs"
EOF

# Install fuse-overlayfs
sudo pacman -S fuse-overlayfs
```

### Verifikasi ✅

```bash
podman info
podman run --rm hello-world
systemctl --user status podman.socket
```

---

## 16. 📦 [MULTIMEDIA] MPD + MPC + MPV

> 🟢 Stack multimedia CLI — daemon musik + kontrol + video player

### Install

```bash
sudo pacman -S mpd mpc mpv
```

### Konfigurasi MPD

```bash
# Buat direktori yang diperlukan
mkdir -p ~/.config/mpd
mkdir -p ~/.local/share/mpd/playlists
mkdir -p ~/Music

# Buat konfigurasi MPD
cat > ~/.config/mpd/mpd.conf << EOF
music_directory    "~/Music"
playlist_directory "~/.local/share/mpd/playlists"
db_file            "~/.local/share/mpd/database"
log_file           "~/.local/share/mpd/log"
pid_file           "~/.local/share/mpd/pid"
state_file         "~/.local/share/mpd/state"
sticker_file       "~/.local/share/mpd/sticker.sql"

bind_to_address    "127.0.0.1"
port               "6600"

audio_output {
    type  "pipewire"
    name  "PipeWire Output"
}
EOF

# Aktifkan MPD sebagai user service
systemctl --user enable --now mpd
```

### Penggunaan Dasar MPC

```bash
mpc update           # update database musik
mpc ls               # list semua musik
mpc add "lagu.mp3"   # tambah ke queue
mpc play             # putar musik
mpc pause            # pause
mpc next             # lagu berikutnya
mpc prev             # lagu sebelumnya
mpc volume 80        # set volume 80%
mpc status           # lihat status
```

### Penggunaan Dasar MPV

```bash
mpv video.mp4                    # putar video
mpv --no-video audio.mp3         # putar audio saja
mpv --volume=80 video.mp4        # dengan volume
mpv https://url-video.com/...    # putar dari URL
# Shortcut: Space=pause, q=quit, f=fullscreen, ←/→=seek
```

### Verifikasi ✅

```bash
systemctl --user status mpd
mpc status
mpv --version
```

---

## 17. 📦 [PASSWORD] KeePassXC

> 🟢 Password manager lokal — enkripsi AES-256

### Install

```bash
sudo pacman -S keepassxc
```

### Setup Awal

```bash
# Jalankan KeePassXC
keepassxc &

# Di GUI:
# 1. Database → New Database
# 2. Beri nama database
# 3. Set encryption: AES-256 (default sudah bagus)
# 4. Set Key Derivation: Argon2id
# 5. Set master password yang kuat
# 6. Simpan file .kdbx di lokasi aman
```

### Aktifkan Integrasi SSH Agent

```bash
# Di KeePassXC: Tools → Settings → SSH Agent
# ✓ Enable SSH Agent Integration
# ✓ Use OpenSSH for Windows instead of Pageant

# Simpan SSH private key di entry KeePassXC
# Attachment → Add → pilih file private key
```

### Aktifkan Secret Service (untuk integrasi sistem)

```bash
# Di KeePassXC: Tools → Settings → Secret Service Integration
# ✓ Enable KeePassXC Secret Service Integration
```

### Verifikasi ✅

```bash
keepassxc --version
# Pastikan bisa membuka database yang dibuat
```

---

## 18. 📦 [KEY] libsecret (secret-tool)

> 🟢 Manajemen secrets/credentials via keyring sistem

### Install

```bash
sudo pacman -S libsecret
```

### Penggunaan Dasar

```bash
# Simpan secret
secret-tool store \
  --label="Database Password" \
  service omeka \
  username admin

# Masukkan password saat diminta

# Ambil secret
secret-tool lookup service omeka username admin

# Hapus secret
secret-tool clear service omeka username admin

# List semua secrets
secret-tool search --all service omeka
```

### Integrasi dengan KeePassXC

```bash
# Pastikan KeePassXC berjalan dengan Secret Service diaktifkan (langkah 17)
# Maka secret-tool otomatis menggunakan KeePassXC sebagai backend

# Test integrasi
secret-tool store --label="Test" service test username test
secret-tool lookup service test username test
```

### Verifikasi ✅

```bash
secret-tool --version
```

---

## 19. 📦 [ACCESS] OpenSSH

> 🟢 Remote access aman menggunakan protokol SSH

### Install

```bash
sudo pacman -S openssh
```

### Konfigurasi Keamanan SSH Server

```bash
sudo vim /etc/ssh/sshd_config
```

Ubah/tambahkan konfigurasi berikut:

```
# Keamanan dasar
Port 22
PermitRootLogin no
MaxAuthTries 3
MaxSessions 5

# Autentikasi
PasswordAuthentication no
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys
PermitEmptyPasswords no

# Fitur yang dinonaktifkan
X11Forwarding no
AllowAgentForwarding no
AllowTcpForwarding no
```

### Generate SSH Key Pair

```bash
# Buat key pair dengan algoritma ed25519 (modern & aman)
ssh-keygen -t ed25519 -C "namauser@archlinux"
# Simpan di ~/.ssh/id_ed25519
# Set passphrase (disarankan)

# Lihat public key
cat ~/.ssh/id_ed25519.pub
```

### Aktifkan SSH Server

```bash
sudo systemctl enable --now sshd

# Tambahkan public key untuk akses
mkdir -p ~/.ssh
chmod 700 ~/.ssh
touch ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

# Paste public key ke authorized_keys
echo "ssh-ed25519 AAAA... namauser@archlinux" >> ~/.ssh/authorized_keys
```

### Simpan SSH Key di KeePassXC

```bash
# Di KeePassXC:
# 1. Buat entry baru untuk SSH key
# 2. Beri nama "SSH Key - archlinux"
# 3. Di tab Advanced → Add Attachment → pilih ~/.ssh/id_ed25519
# 4. Di tab SSH Agent → ✓ Add key to agent when database is opened
```

### Verifikasi ✅

```bash
sudo systemctl status sshd
ssh -T namauser@localhost
# Atau dari mesin lain:
ssh namauser@<ip-server>
```

---

## 20. 📦 [CONTAINER-APP] Omeka di Podman

> 🟢 Jalankan aplikasi Omeka-S sebagai container rootless

### Persiapan

```bash
# Pastikan direktori storage ada
ls /var/lib/docker  # → dari LV dock yang sudah dimount

# Buat network khusus untuk omeka
podman network create omeka-net
```

### Jalankan MariaDB (Database)

```bash
podman run -d \
  --name omeka-db \
  --network omeka-net \
  -e MYSQL_ROOT_PASSWORD=rootpassword123 \
  -e MYSQL_DATABASE=omeka \
  -e MYSQL_USER=omeka \
  -e MYSQL_PASSWORD=omekapassword123 \
  -v omeka-db-data:/var/lib/mysql \
  docker.io/mariadb:latest

# Verifikasi container berjalan
podman ps
```

### Jalankan Omeka-S

```bash
podman run -d \
  --name omeka \
  --network omeka-net \
  -p 8080:80 \
  -e OMEKA_DB_HOST=omeka-db \
  -e OMEKA_DB_NAME=omeka \
  -e OMEKA_DB_USER=omeka \
  -e OMEKA_DB_PASSWORD=omekapassword123 \
  -v omeka-data:/var/www/html \
  docker.io/omeka/omeka-s:latest

# Verifikasi
podman ps
```

### Simpan Password di KeePassXC & secret-tool

```bash
# Simpan credentials omeka di keyring
secret-tool store \
  --label="Omeka DB Password" \
  service omeka-db \
  username omeka

# Masukkan: omekapassword123
```

### Buat Service Systemd untuk Auto-start

```bash
# Generate unit file
mkdir -p ~/.config/systemd/user
podman generate systemd --new --name omeka-db > ~/.config/systemd/user/omeka-db.service
podman generate systemd --new --name omeka > ~/.config/systemd/user/omeka.service

# Aktifkan
systemctl --user enable --now omeka-db
systemctl --user daemon-reload
systemctl --user enable --now omeka
```

### Akses Omeka

```
Browser: http://localhost:8080
Setup awal akan muncul untuk konfigurasi admin Omeka
```

### Verifikasi ✅

```bash
podman ps
podman logs omeka
curl -I http://localhost:8080
# Harus mendapat response HTTP 200 atau 302
```

---

## 21. Verifikasi Akhir

### 21.1 Cek Semua Service Berjalan

```bash
# System services
sudo systemctl status lightdm
sudo systemctl status sshd
sudo systemctl status iptables

# User services
systemctl --user status mpd
systemctl --user status podman.socket
systemctl --user status omeka
systemctl --user status omeka-db
```

### 21.2 Cek Partisi & Mount Options

```bash
# Verifikasi semua partisi ter-mount
lsblk
df -h

# Verifikasi mount options CIS
findmnt /tmp      # harus ada noexec,nosuid,nodev
findmnt /var/tmp  # harus ada noexec,nosuid,nodev
findmnt /var/log  # harus ada noexec,nosuid,nodev
findmnt /home     # harus ada nosuid,nodev
```

### 21.3 Cek LUKS & LVM

```bash
# Status enkripsi
sudo cryptsetup status proc
sudo cryptsetup status data

# Status LVM
sudo pvdisplay
sudo vgdisplay
sudo lvdisplay
```

### 21.4 Cek Secure Boot

```bash
sbctl status
# Output: Secure Boot: ✓ enabled
```

### 21.5 Audit Keamanan dengan Lynis

```bash
# Install lynis
sudo pacman -S lynis

# Jalankan audit CIS
sudo lynis audit system

# Lihat skor — target minimal 70/100
# Laporan tersimpan di /var/log/lynis.log
```

### 21.6 Cek Firewall

```bash
# Lihat rules aktif
sudo iptables -L -n -v

# Port yang harus terbuka:
# TCP 22   — SSH
# TCP 6600 — MPD (lokal saja)
# TCP 8080 — Omeka
```

---

## 📊 Ringkasan Package yang Diinstall

| No | Tanda | Package | Fase | Status |
|----|-------|---------|------|--------|
| 1 | 📦 [BOOT] | booster | Instalasi | Wajib saat chroot |
| 2 | 📦 [SECUREBOOT] | sbctl | Instalasi | Wajib saat chroot |
| 3 | 📦 [DESKTOP] | xfce4, lightdm | Pasca | Setelah reboot |
| 4 | 📦 [FILEMANAGER] | superfile | Pasca | Setelah reboot |
| 5 | 📦 [FIREWALL] | iptables | Pasca | Setelah reboot |
| 6 | 📦 [CONTAINER] | podman, podman-desktop | Pasca | Setelah reboot |
| 7 | 📦 [MULTIMEDIA] | mpd, mpc, mpv | Pasca | Setelah reboot |
| 8 | 📦 [PASSWORD] | keepassxc | Pasca | Setelah reboot |
| 9 | 📦 [KEY] | libsecret | Pasca | Setelah reboot |
| 10 | 📦 [ACCESS] | openssh | Pasca | Setelah reboot |
| 11 | 📦 [CONTAINER-APP] | omeka (di podman) | Pasca | Setelah podman |

---

*Dokumentasi ini dibuat berdasarkan CIS Benchmark for Linux dan Arch Linux Wiki.*  
*Selalu sesuaikan konfigurasi dengan kebutuhan dan lingkungan spesifik Anda.*
