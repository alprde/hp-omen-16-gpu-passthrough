# CachyOS ile Windows 11 GPU Passthrough Klavuzu
## HP OMEN Gaming Laptop 16-wf0015nt (7Q7W7EA) için

> ⚠️ **Bu klavuz deneysel niteliktedir ve henüz test edilmemiştir**

### 🖥️ Sistem Özellikleri
- **Model**: HP OMEN Gaming Laptop 16-wf0015nt (7Q7W7EA)
- **İşlemci**: Intel Core i9-13900HX (5.4 GHz Turbo, 36 MB L3 cache, 24 core, 32 thread)
- **RAM**: 32 GB DDR5-5600 MHz (2x16 GB)
- **GPU**: NVIDIA GeForce RTX 4070 Mobile (8 GB GDDR6, 80W TGP)
- **iGPU**: Intel UHD Graphics (entegre)
- **Ekran**: 16.1" QHD (2560x1440), 240Hz, 3ms, IPS, 300 nits, 100% sRGB
- **MUX Switch**: ✅ VAR (Advanced Optimus/Dynamic Display Switch)
- **Optimus**: Advanced Optimus (Dinamik GPU geçişi)
- **Host OS**: CachyOS (Arch Linux tabanlı)
- **Guest OS**: Windows 11

---

## 📋 İçindekiler
1. [Ön Gereksinimler](#ön-gereksinimler)
2. [BIOS/UEFI Ayarları](#biosuefi-ayarları)
3. [CachyOS Kurulumu ve Yapılandırması](#cachyos-kurulumu-ve-yapılandırması)
4. [VFIO Kurulumu](#vfio-kurulumu)
5. [QEMU/KVM Kurulumu](#qemukvm-kurulumu)
6. [Windows 11 VM Oluşturma](#windows-11-vm-oluşturma)
7. [GPU Passthrough Yapılandırması](#gpu-passthrough-yapılandırması)
8. [MUX Switch ve Advanced Optimus Ayarları](#mux-switch-ve-advanced-optimus-ayarları)
9. [Çoklu Ekran ve Hibrit Kullanım](#çoklu-ekran-ve-hibrit-kullanım)
10. [Performans Optimizasyonları](#performans-optimizasyonları)
11. [Sorun Giderme](#sorun-giderme)

---

## 🔧 Ön Gereksinimler

### Donanım Gereksinimleri
- ✅ Intel VT-x/VT-d desteği (i9-13900HX destekler)
- ✅ IOMMU desteği
- ✅ MUX Switch (Advanced Optimus ile)
- ✅ İki GPU (Intel UHD + RTX 4070)
- ✅ Yeterli RAM (32 GB ideal)
- ✅ SSD depolama (performans için önemli)

### Yazılım Gereksinimleri
- CachyOS (güncel sürüm)
- QEMU/KVM
- VFIO modülleri
- Windows 11 ISO dosyası
- VirtIO sürücüleri

### Önemli Notlar
- **MUX Switch**: Bu laptop Advanced Optimus teknolojisine sahip, bu da GPU passthrough için idealdir
- **Dynamic Display Switch (DDS)**: Ekran otomatik olarak GPU'lar arasında geçiş yapabilir
- **80W TGP RTX 4070**: Yüksek performanslı GPU passthrough için mükemmel

---

## ⚙️ BIOS/UEFI Ayarları

### 1. BIOS'a Giriş
- Bilgisayarı yeniden başlatın
- **F10** tuşuna basarak BIOS'a girin (HP OMEN sistemlerde)
- Alternatif: **ESC** tuşuna basıp **F10** seçin

### 2. Gerekli Ayarlar
```
Advanced/Security:
├── Intel VT-x: Enabled
├── Intel VT-d: Enabled
├── Secure Boot: Disabled
├── Fast Boot: Disabled
└── CSM Support: Disabled (UEFI only)

Advanced/Device Options:
├── Primary Display: Auto (hibrit kullanım için)
├── Hybrid Graphics: Enabled (hibrit kullanım için)
├── Discrete Graphics: Enabled
└── Advanced Optimus: Enabled (varsa)

Power Management:
├── USB Wake Support: Disabled
└── Network Boot: Disabled
```

### 3. MUX Switch Ayarları (Hibrit Kullanım için)
```
Advanced/Graphics Configuration:
├── Graphics Mode: Hybrid (hibrit mod için)
├── Dynamic Display Switch: Enabled
├── GPU Power Saving: Auto
└── Optimus: Advanced/Dynamic
```

---

## 🐧 CachyOS Kurulumu ve Yapılandırması

### 1. CachyOS Kurulumu
```bash
# CachyOS ISO'yu indirin ve USB'ye yazın
# Normal kurulum yapın, UEFI modunda kurun
```

### 2. Sistem Güncellemesi
```bash
sudo pacman -Syu
```

### 3. Gerekli Paketlerin Kurulumu
```bash
# Temel paketler
sudo pacman -S qemu-full virt-manager virt-viewer dnsmasq vde2 bridge-utils openbsd-netcat

# VFIO paketleri
sudo pacman -S vfio-pci vfio-pci-override-vga

# Ek araçlar
sudo pacman -S edk2-ovmf swtpm dmidecode

# NVIDIA araçları (host için)
sudo pacman -S nvidia-utils nvidia-settings
```

---

## 🔗 VFIO Kurulumu

### 1. IOMMU Kontrolü
```bash
# IOMMU gruplarını kontrol edin
sudo dmesg | grep -i iommu

# IOMMU gruplarını listeleyin
for d in /sys/kernel/iommu_groups/*/devices/*; do 
    n=${d#*/iommu_groups/*}; n=${n%%/*}
    printf 'IOMMU Group %s ' "$n"
    lspci -nns "${d##*/}"
done
```

### 2. GPU PCI ID'lerini Bulma
```bash
# NVIDIA RTX 4070 Mobile PCI ID'lerini bulun
lspci -nn | grep -i nvidia

# Beklenen çıktı (sizin sisteminize göre değişebilir):
# 01:00.0 VGA compatible controller [0300]: NVIDIA Corporation AD104M [GeForce RTX 4070 Mobile] [10de:2786] (rev a1)
# 01:00.1 Audio device [0403]: NVIDIA Corporation AD104M High Definition Audio Controller [10de:22bc] (rev a1)
```

### 3. GRUB Yapılandırması
**Dosya Yolu**: `/etc/default/grub`
```bash
# GRUB yapılandırmasını düzenleyin
sudo nano /etc/default/grub

# GRUB_CMDLINE_LINUX_DEFAULT satırını bulun ve şu parametreleri ekleyin:
# (PCI ID'leri kendi sisteminize göre güncelleyin)
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt vfio-pci.ids=10de:2786,10de:22bc rd.driver.pre=vfio-pci video=efifb:off nouveau.modeset=0 nvidia-drm.modeset=0"

# GRUB'u güncelleyin
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

**Parametre Açıklamaları**:
- `nouveau.modeset=0`: Nouveau sürücüsünün KMS (Kernel Mode Setting) özelliğini devre dışı bırakır
- `nvidia-drm.modeset=0`: NVIDIA DRM sürücüsünün modesetting özelliğini devre dışı bırakır
- Bu yöntem sürücüleri tamamen blacklist etmez, sadece display kontrolünü engeller
- Hibrit kullanım için daha uygun (CachyOS + VM senaryonuz için ideal)

### 4. VFIO Modüllerini Yükleme
**Dosya Yolu**: `/etc/mkinitcpio.conf`
```bash
# /etc/mkinitcpio.conf dosyasını düzenleyin
sudo nano /etc/mkinitcpio.conf

# MODULES satırını bulun ve şu modülleri ekleyin:
MODULES=(vfio_pci vfio vfio_iommu_type1 vfio_virqfd)

# HOOKS satırında modconf'un varlığını kontrol edin
HOOKS=(base udev autodetect modconf block filesystems keyboard fsck)

# initramfs'i yeniden oluşturun
sudo mkinitcpio -P
```

### 5. VFIO Yapılandırma Dosyası
**Dosya Yolu**: `/etc/modprobe.d/vfio.conf`
```bash
# VFIO yapılandırma dosyası oluşturun
sudo nano /etc/modprobe.d/vfio.conf

# Aşağıdaki içeriği ekleyin (PCI ID'lerinizi kullanın):
options vfio-pci ids=10de:2786,10de:22bc
softdep nouveau pre: vfio-pci
softdep nvidia* pre: vfio-pci
```

**ÖNEMLİ**: Hibrit kullanım için blacklist dosyası oluşturmayın! GRUB'daki `nouveau.modeset=0 nvidia-drm.modeset=0` parametreleri yeterlidir.

### 6. Blacklist Dosyası (Opsiyonel)
**Dosya Yolu**: `/etc/modprobe.d/blacklist-nvidia.conf`
```bash
# NVIDIA sürücülerini blacklist edin
sudo nano /etc/modprobe.d/blacklist-nvidia.conf

# İçerik:
blacklist nouveau
blacklist nvidia
blacklist nvidia_drm
blacklist nvidia_modeset
```

---

## 🖥️ QEMU/KVM Kurulumu

### 1. Libvirt Servislerini Başlatma
```bash
# Libvirt servislerini etkinleştirin ve başlatın
sudo systemctl enable libvirtd
sudo systemctl start libvirtd

# Kullanıcıyı libvirt grubuna ekleyin
sudo usermod -a -G libvirt $(whoami)

# Oturumu yeniden açın veya şu komutu çalıştırın:
newgrp libvirt
```

### 2. Libvirt Yapılandırması
**Dosya Yolu**: `/etc/libvirt/libvirtd.conf`
```bash
# Libvirt yapılandırmasını düzenleyin
sudo nano /etc/libvirt/libvirtd.conf

# Şu satırları bulun ve düzenleyin:
unix_sock_group = "libvirt"
unix_sock_rw_perms = "0770"
auth_unix_ro = "none"
auth_unix_rw = "none"

# Servisi yeniden başlatın
sudo systemctl restart libvirtd
```

### 3. QEMU Yapılandırması
**Dosya Yolu**: `/etc/libvirt/qemu.conf`
```bash
# QEMU yapılandırmasını düzenleyin
sudo nano /etc/libvirt/qemu.conf

# Şu satırları bulun ve düzenleyin (kullanıcı adınızı yazın):
user = "kullanici_adiniz"
group = "libvirt"

# Örnek:
# user = "musa"
# group = "libvirt"
```

---

## 💿 Windows 11 VM Oluşturma

### 1. Windows 11 ISO ve VirtIO Sürücülerini İndirme
```bash
# Windows 11 ISO'yu Microsoft'tan indirin
# VirtIO sürücülerini indirin:
wget https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso
```

### 2. VM Oluşturma (virt-manager ile)
```bash
# virt-manager'ı başlatın
virt-manager
```

#### VM Ayarları (i9-13900HX için optimize):
- **İşletim Sistemi**: Windows 11
- **RAM**: 24 GB (32 GB'den 24 GB ayırın)
- **CPU**: 16 çekirdek (24 çekirdekten 16 ayırın)
- **Depolama**: 120 GB (SSD'de)
- **Ağ**: NAT veya Bridge

### 3. XML Yapılandırması
**VM XML Dosyası**: `/etc/libvirt/qemu/windows11.xml`
```bash
# VM'i oluşturduktan sonra XML'i düzenleyin
virsh edit windows11

# Aşağıdaki özellikleri ekleyin/düzenleyin:
```

```xml
<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
  <name>windows11</name>
  <memory unit='KiB'>25165824</memory>
  <currentMemory unit='KiB'>25165824</currentMemory>
  <vcpu placement='static'>16</vcpu>
  
  <features>
    <acpi/>
    <apic/>
    <hyperv>
      <relaxed state='on'/>
      <vapic state='on'/>
      <spinlocks state='on' retries='8191'/>
      <vendor_id state='on' value='kvm hyperv'/>
      <frequencies state='on'/>
    </hyperv>
    <kvm>
      <hidden state='on'/>
    </kvm>
    <vmport state='off'/>
  </features>
  
  <cpu mode='host-passthrough' check='none' migratable='on'>
    <topology sockets='1' dies='1' cores='8' threads='2'/>
    <feature policy='require' name='topoext'/>
  </cpu>
  
  <clock offset='localtime'>
    <timer name='rtc' tickpolicy='catchup'/>
    <timer name='pit' tickpolicy='delay'/>
    <timer name='hpet' present='no'/>
    <timer name='hypervclock' present='yes'/>
    <timer name='tsc' present='yes' mode='native'/>
  </clock>

  <cputune>
    <vcpupin vcpu='0' cpuset='4'/>
    <vcpupin vcpu='1' cpuset='5'/>
    <vcpupin vcpu='2' cpuset='6'/>
    <vcpupin vcpu='3' cpuset='7'/>
    <vcpupin vcpu='4' cpuset='8'/>
    <vcpupin vcpu='5' cpuset='9'/>
    <vcpupin vcpu='6' cpuset='10'/>
    <vcpupin vcpu='7' cpuset='11'/>
    <vcpupin vcpu='8' cpuset='12'/>
    <vcpupin vcpu='9' cpuset='13'/>
    <vcpupin vcpu='10' cpuset='14'/>
    <vcpupin vcpu='11' cpuset='15'/>
    <vcpupin vcpu='12' cpuset='16'/>
    <vcpupin vcpu='13' cpuset='17'/>
    <vcpupin vcpu='14' cpuset='18'/>
    <vcpupin vcpu='15' cpuset='19'/>
    <emulatorpin cpuset='0-3'/>
    <iothreadpin iothread='1' cpuset='20-23'/>
  </cputune>
</domain>
```

---

## 🎮 GPU Passthrough Yapılandırması

### 1. GPU'yu VM'e Ekleme
```bash
# VM'in XML yapılandırmasını düzenleyin
virsh edit windows11
```

```xml
<!-- RTX 4070 Mobile (Video) -->
<hostdev mode='subsystem' type='pci' managed='yes'>
  <source>
    <address domain='0x0000' bus='0x01' slot='0x00' function='0x0'/>
  </source>
  <address type='pci' domain='0x0000' bus='0x05' slot='0x00' function='0x0'/>
</hostdev>

<!-- RTX 4070 Mobile (Audio) -->
<hostdev mode='subsystem' type='pci' managed='yes'>
  <source>
    <address domain='0x0000' bus='0x01' slot='0x00' function='0x1'/>
  </source>
  <address type='pci' domain='0x0000' bus='0x06' slot='0x00' function='0x0'/>
</hostdev>
```

### 2. VBIOS Dumping (RTX 4070 Mobile için)
```bash
# VBIOS dizinini oluşturun
sudo mkdir -p /var/lib/libvirt/vbios

# GPU VBIOS'unu dump edin (gerekirse)
sudo echo 1 > /sys/bus/pci/devices/0000:01:00.0/rom
sudo cat /sys/bus/pci/devices/0000:01:00.0/rom > /var/lib/libvirt/vbios/rtx4070_vbios.rom
sudo echo 0 > /sys/bus/pci/devices/0000:01:00.0/rom

# İzinleri ayarlayın
sudo chown libvirt-qemu:kvm /var/lib/libvirt/vbios/rtx4070_vbios.rom
sudo chmod 644 /var/lib/libvirt/vbios/rtx4070_vbios.rom

# VBIOS dosyasının yolu: /var/lib/libvirt/vbios/rtx4070_vbios.rom
```

### 3. VBIOS'u VM'e Ekleme (XML'de)
```xml
<hostdev mode='subsystem' type='pci' managed='yes'>
  <source>
    <address domain='0x0000' bus='0x01' slot='0x00' function='0x0'/>
  </source>
  <rom file='/var/lib/libvirt/vbios/rtx4070_vbios.rom'/>
  <address type='pci' domain='0x0000' bus='0x05' slot='0x00' function='0x0'/>
</hostdev>
```

---

## 🔄 MUX Switch ve Advanced Optimus Ayarları

### 1. MUX Switch Kontrolü
```bash
# MUX switch durumunu kontrol edin
sudo nvidia-smi --query-gpu=name,driver_version,pstate --format=csv

# Advanced Optimus durumunu kontrol edin
cat /sys/class/drm/card*/device/power_dpm_state 2>/dev/null || echo "Advanced Optimus aktif"
```

### 2. GPU Geçiş Scriptleri
**Dosya Yolu**: `/usr/local/bin/gpu-switch-discrete.sh`
```bash
# Discrete GPU'ya geçiş scripti oluşturun
sudo nano /usr/local/bin/gpu-switch-discrete.sh

#!/bin/bash
# RTX 4070'e geçiş
echo "Discrete GPU'ya geçiliyor..."
sudo tee /sys/kernel/debug/vgaswitcheroo/switch <<< "DDIS"
sudo systemctl restart display-manager
```

**Dosya Yolu**: `/usr/local/bin/gpu-switch-integrated.sh`
```bash
# Integrated GPU'ya geçiş scripti oluşturun
sudo nano /usr/local/bin/gpu-switch-integrated.sh

#!/bin/bash
# Intel iGPU'ya geçiş
echo "Integrated GPU'ya geçiliyor..."
sudo tee /sys/kernel/debug/vgaswitcheroo/switch <<< "IGD"
sudo systemctl restart display-manager
```

```bash
# Scriptleri çalıştırılabilir yapın
sudo chmod +x /usr/local/bin/gpu-switch-*.sh
```

### 3. Udev Kuralları (Otomatik GPU Geçişi)
**Dosya Yolu**: `/etc/udev/rules.d/99-gpu-switch.rules`
```bash
# Udev kuralları oluşturun
sudo nano /etc/udev/rules.d/99-gpu-switch.rules

# VM başladığında discrete GPU'ya geç
SUBSYSTEM=="vfio", ACTION=="add", RUN+="/usr/local/bin/gpu-switch-discrete.sh"

# VM kapandığında integrated GPU'ya geç
SUBSYSTEM=="vfio", ACTION=="remove", RUN+="/usr/local/bin/gpu-switch-integrated.sh"
```

---

## 🖥️ Çoklu Ekran ve Hibrit Kullanım

Bu bölüm, Windows VM'i hem dahili hem harici ekranda kullanabilmenizi ve CachyOS'u engellemeden çalışabilmenizi sağlar.

### 1. Hibrit Mod Yapılandırması

#### BIOS Ayarları (Hibrit Mod için)
```
Advanced/Graphics Configuration:
├── Graphics Mode: Hybrid (Hibrit mod için)
├── Primary Display: Auto
├── Discrete Graphics: Enabled
├── Advanced Optimus: Enabled
└── Multi-Display: Enabled
```

#### CachyOS Hibrit Sürücü Kurulumu
```bash
# Hibrit NVIDIA sürücülerini kurun
sudo pacman -S nvidia nvidia-utils nvidia-settings
sudo pacman -S optimus-manager optimus-manager-qt

# Optimus Manager'ı etkinleştirin
sudo systemctl enable optimus-manager.service
```

### 2. Çoklu Ekran Yapılandırması

#### Xorg Yapılandırması
**Dosya Yolu**: `/etc/X11/xorg.conf.d/10-nvidia.conf`
```bash
# NVIDIA hibrit yapılandırması oluşturun
sudo nano /etc/X11/xorg.conf.d/10-nvidia.conf

# İçerik:
Section "Module"
    Load "modesetting"
EndSection

Section "Device"
    Identifier "nvidia"
    Driver "nvidia"
    BusID "PCI:1:0:0"
    Option "AllowEmptyInitialConfiguration"
EndSection

Section "Device"
    Identifier "intel"
    Driver "modesetting"
    BusID "PCI:0:2:0"
EndSection

Section "Screen"
    Identifier "nvidia"
    Device "nvidia"
    Option "AllowEmptyInitialConfiguration"
EndSection
```

#### Display Manager Ayarları
**Dosya Yolu**: `/etc/sddm.conf.d/optimus.conf`
```bash
# SDDM için optimus ayarları
sudo nano /etc/sddm.conf.d/optimus.conf

[General]
DisplayCommand=/usr/share/sddm/scripts/Xsetup
DisplayStopCommand=/usr/share/sddm/scripts/Xstop

[X11]
ServerArguments=-nolisten tcp -dpi 96
```

### 3. Dinamik Ekran Geçişi

#### Ekran Geçiş Scriptleri
**Dosya Yolu**: `/usr/local/bin/display-switch-internal.sh`
```bash
# Dahili ekrana geçiş scripti
sudo nano /usr/local/bin/display-switch-internal.sh

#!/bin/bash
# Dahili ekrana geçiş
echo "Dahili ekrana geçiliyor..."
xrandr --output eDP-1 --primary --mode 2560x1440 --rate 240
xrandr --output HDMI-1 --off
xrandr --output DP-1 --off
```

**Dosya Yolu**: `/usr/local/bin/display-switch-external.sh`
```bash
# Harici ekrana geçiş scripti
sudo nano /usr/local/bin/display-switch-external.sh

#!/bin/bash
# Harici ekrana geçiş
echo "Harici ekrana geçiliyor..."
xrandr --output HDMI-1 --primary --mode 1920x1080 --rate 144
xrandr --output eDP-1 --off
```

**Dosya Yolu**: `/usr/local/bin/display-switch-dual.sh`
```bash
# Çift ekran modu scripti
sudo nano /usr/local/bin/display-switch-dual.sh

#!/bin/bash
# Çift ekran modu
echo "Çift ekran moduna geçiliyor..."
xrandr --output eDP-1 --mode 2560x1440 --rate 240 --primary
xrandr --output HDMI-1 --mode 1920x1080 --rate 144 --right-of eDP-1
```

```bash
# Scriptleri çalıştırılabilir yapın
sudo chmod +x /usr/local/bin/display-switch-*.sh
```

### 4. VM Ekran Yapılandırması

#### Looking Glass Kurulumu (Dahili Ekran için)
```bash
# Looking Glass kurun (dahili ekranda VM görüntüsü için)
sudo pacman -S looking-glass

# Shared memory ayarları
sudo nano /etc/tmpfiles.d/10-looking-glass.conf

# İçerik:
f /dev/shm/looking-glass 0660 user kvm -

# Systemd tmpfiles'ı yeniden yükleyin
sudo systemd-tmpfiles --create /etc/tmpfiles.d/10-looking-glass.conf
```

#### VM XML'de Çoklu Ekran Desteği
```xml
<!-- Looking Glass için IVSHMEM -->
<shmem name='looking-glass'>
  <model type='ivshmem-plain'/>
  <size unit='M'>32</size>
</shmem>

<!-- Spice display (dahili ekran için) -->
<graphics type='spice' autoport='yes'>
  <listen type='address'/>
  <image compression='off'/>
</graphics>

<!-- Video device (harici ekran için passthrough) -->
<video>
  <model type='none'/>
</video>
```

### 5. Otomatik Ekran Yönetimi

#### Udev Kuralları (Otomatik Ekran Geçişi)
**Dosya Yolu**: `/etc/udev/rules.d/99-display-switch.rules`
```bash
# Otomatik ekran geçiş kuralları
sudo nano /etc/udev/rules.d/99-display-switch.rules

# HDMI bağlandığında harici ekrana geç
ACTION=="add", SUBSYSTEM=="drm", KERNEL=="card0-HDMI-A-1", RUN+="/usr/local/bin/display-switch-external.sh"

# HDMI çıkarıldığında dahili ekrana geç
ACTION=="remove", SUBSYSTEM=="drm", KERNEL=="card0-HDMI-A-1", RUN+="/usr/local/bin/display-switch-internal.sh"

# VM başladığında uygun ekran moduna geç
SUBSYSTEM=="vfio", ACTION=="add", RUN+="/usr/local/bin/vm-display-setup.sh"
```

#### VM Display Setup Scripti
**Dosya Yolu**: `/usr/local/bin/vm-display-setup.sh`
```bash
# VM ekran kurulum scripti
sudo nano /usr/local/bin/vm-display-setup.sh

#!/bin/bash
# VM başladığında ekran ayarları

# HDMI bağlı mı kontrol et
if xrandr | grep "HDMI-1 connected"; then
    # Harici ekran varsa, VM'i harici ekrana yönlendir
    echo "VM harici ekrana yönlendiriliyor..."
    /usr/local/bin/display-switch-external.sh
    
    # Looking Glass'ı dahili ekranda başlat
    su - $SUDO_USER -c "looking-glass-client -F &"
else
    # Sadece dahili ekran varsa, Looking Glass kullan
    echo "VM dahili ekranda Looking Glass ile çalışacak..."
    /usr/local/bin/display-switch-internal.sh
    
    # Looking Glass'ı tam ekranda başlat
    su - $SUDO_USER -c "looking-glass-client -F &"
fi

# Scriptleri çalıştırılabilir yapın
sudo chmod +x /usr/local/bin/vm-display-setup.sh
```

### 6. Klavye Kısayolları

#### i3/Sway Kısayolları (Opsiyonel)
**Dosya Yolu**: `~/.config/i3/config` veya `~/.config/sway/config`
```bash
# Ekran geçiş kısayolları ekleyin
bindsym $mod+F1 exec /usr/local/bin/display-switch-internal.sh
bindsym $mod+F2 exec /usr/local/bin/display-switch-external.sh
bindsym $mod+F3 exec /usr/local/bin/display-switch-dual.sh

# VM kontrol kısayolları
bindsym $mod+F9 exec virsh start windows11
bindsym $mod+F10 exec virsh shutdown windows11
bindsym $mod+F11 exec looking-glass-client -F
```

### 7. Hibrit Mod Optimizasyonları

#### Power Management
**Dosya Yolu**: `/etc/udev/rules.d/50-nvidia-power.rules`
```bash
# NVIDIA güç yönetimi kuralları
sudo nano /etc/udev/rules.d/50-nvidia-power.rules

# NVIDIA kartı VM kullanmadığında uyku moduna al
ACTION=="add", SUBSYSTEM=="pci", ATTR{vendor}=="0x10de", ATTR{class}=="0x030000", TEST=="power/control", ATTR{power/control}="auto"
```

#### Optimus Manager Profilleri
```bash
# Hibrit profil oluşturun
optimus-manager --switch hybrid --no-confirm

# Intel profil (güç tasarrufu için)
optimus-manager --switch intel --no-confirm

# NVIDIA profil (performans için)
optimus-manager --switch nvidia --no-confirm
```

### 8. Kullanım Senaryoları

#### Senaryo 1: Dahili Ekranda VM + CachyOS
```bash
# 1. VM'i başlatın
virsh start windows11

# 2. Looking Glass'ı başlatın
looking-glass-client -F

# 3. CachyOS dahili ekranda çalışmaya devam eder
# 4. VM Looking Glass penceresi içinde görünür
```

#### Senaryo 2: Harici Ekranda VM + Dahili Ekranda CachyOS
```bash
# 1. Harici monitörü bağlayın
# 2. VM'i başlatın (otomatik olarak harici ekrana yönlendirilir)
virsh start windows11

# 3. CachyOS dahili ekranda çalışmaya devam eder
# 4. VM harici ekranda tam ekran çalışır
```

#### Senaryo 3: Çift Ekran Modu
```bash
# 1. Çift ekran modunu etkinleştirin
/usr/local/bin/display-switch-dual.sh

# 2. VM'i başlatın
virsh start windows11

# 3. Looking Glass'ı ikinci ekranda açın
looking-glass-client -F --spice-host 127.0.0.1
```

### 9. Sorun Giderme (Çoklu Ekran)

#### Ekran Tanınmama Problemi
```bash
# Ekran durumunu kontrol edin
xrandr --listproviders
xrandr --verbose

# NVIDIA ekran çıkışlarını kontrol edin
nvidia-settings --query all
```

#### Looking Glass Problemi
```bash
# Looking Glass loglarını kontrol edin
journalctl -u looking-glass

# Shared memory'yi kontrol edin
ls -la /dev/shm/looking-glass

# İzinleri düzeltin
sudo chown user:kvm /dev/shm/looking-glass
sudo chmod 660 /dev/shm/looking-glass
```

#### Hibrit Mod Problemi
```bash
# Optimus durumunu kontrol edin
optimus-manager --status

# GPU'ları kontrol edin
lspci | grep VGA
nvidia-smi

# Sürücü durumunu kontrol edin
lsmod | grep nvidia
lsmod | grep i915
```

---

## ⚡ Performans Optimizasyonları

### 1. CPU Pinning (i9-13900HX için)
```xml
<!-- VM XML'inde CPU pinning (24 core için optimize) -->
<vcpu placement='static'>16</vcpu>
<cputune>
  <vcpupin vcpu='0' cpuset='4'/>
  <vcpupin vcpu='1' cpuset='5'/>
  <vcpupin vcpu='2' cpuset='6'/>
  <vcpupin vcpu='3' cpuset='7'/>
  <vcpupin vcpu='4' cpuset='8'/>
  <vcpupin vcpu='5' cpuset='9'/>
  <vcpupin vcpu='6' cpuset='10'/>
  <vcpupin vcpu='7' cpuset='11'/>
  <vcpupin vcpu='8' cpuset='12'/>
  <vcpupin vcpu='9' cpuset='13'/>
  <vcpupin vcpu='10' cpuset='14'/>
  <vcpupin vcpu='11' cpuset='15'/>
  <vcpupin vcpu='12' cpuset='16'/>
  <vcpupin vcpu='13' cpuset='17'/>
  <vcpupin vcpu='14' cpuset='18'/>
  <vcpupin vcpu='15' cpuset='19'/>
  <emulatorpin cpuset='0-3'/>
  <iothreadpin iothread='1' cpuset='20-23'/>
</cputune>
```

### 2. Hugepages
**Dosya Yolu**: `/etc/default/grub`
```bash
# Hugepages ayarları
sudo nano /etc/default/grub

# GRUB_CMDLINE_LINUX_DEFAULT'a ekleyin:
hugepagesz=1G default_hugepagesz=1G hugepages=26

# GRUB'u güncelleyin
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

### 3. VM XML Optimizasyonları
```xml
<!-- Bellek optimizasyonları -->
<memoryBacking>
  <hugepages/>
  <locked/>
  <source type='memfd'/>
  <access mode='shared'/>
</memoryBacking>

<!-- Disk optimizasyonları -->
<disk type='file' device='disk'>
  <driver name='qemu' type='qcow2' cache='none' io='native' discard='unmap'/>
  <!-- ... -->
</disk>

<!-- Ağ optimizasyonları -->
<interface type='network'>
  <model type='virtio'/>
  <driver name='vhost' queues='4'/>
  <!-- ... -->
</interface>
```

### 4. Governor Ayarları
**Dosya Yolu**: `/etc/systemd/system/cpu-performance.service`
```bash
# CPU performance service oluşturun
sudo nano /etc/systemd/system/cpu-performance.service

[Unit]
Description=Set CPU governor to performance
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/bin/bash -c 'echo performance | tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor'
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target

# Servisi etkinleştirin
sudo systemctl enable cpu-performance.service
```

---

## 🔧 Sorun Giderme

### 1. Yaygın Sorunlar

#### IOMMU Grupları Problemi
```bash
# IOMMU gruplarını kontrol edin
for d in /sys/kernel/iommu_groups/*/devices/*; do 
    n=${d#*/iommu_groups/*}; n=${n%%/*}
    printf 'IOMMU Group %s ' "$n"
    lspci -nns "${d##*/}"
done

# RTX 4070 ayrı grupta olmalı
```

#### GPU Bind Edilmeme
```bash
# GPU'nun VFIO'ya bind edilip edilmediğini kontrol edin
lspci -nnk -d 10de:2786

# Driver: vfio-pci olmalı
# Eğer nvidia ise, blacklist çalışmamış demektir
```

#### MUX Switch Problemi
```bash
# MUX switch durumunu kontrol edin
cat /sys/kernel/debug/vgaswitcheroo/switch

# Advanced Optimus loglarını kontrol edin
dmesg | grep -i optimus
dmesg | grep -i mux
```

#### VM Başlamama
```bash
# Libvirt loglarını kontrol edin
sudo journalctl -u libvirtd

# QEMU loglarını kontrol edin
sudo tail -f /var/log/libvirt/qemu/windows11.log

# VFIO durumunu kontrol edin
dmesg | grep -i vfio
```

### 2. Performans Sorunları

#### Düşük FPS (RTX 4070 için)
- CPU pinning yapıldığından emin olun
- Hugepages aktif olduğunu kontrol edin
- MSI interrupts etkinleştirin
- GPU boost clock ayarlarını kontrol edin

#### Ses Problemi
- GPU'nun ses kartını da passthrough yaptığınızdan emin olun
- PulseAudio/PipeWire yapılandırmasını kontrol edin
- HDMI audio çıkışını kontrol edin

#### Ekran Problemi (MUX Switch ile ilgili)
- BIOS'ta MUX switch ayarlarını kontrol edin
- Advanced Optimus'un doğru çalıştığından emin olun
- Display output ayarlarını kontrol edin

### 3. Faydalı Komutlar
```bash
# VFIO durumunu kontrol etme
dmesg | grep -i vfio

# PCI cihazları listeleme
lspci -nnv

# GPU durumunu kontrol etme
nvidia-smi
sudo nvidia-smi -q

# VM durumunu kontrol etme
virsh list --all

# VM'i başlatma/durdurma
virsh start windows11
virsh shutdown windows11

# MUX switch durumu
cat /sys/kernel/debug/vgaswitcheroo/switch

# CPU durumu
cat /proc/cpuinfo | grep "model name" | head -1
lscpu
```

### 4. Log Dosyaları
- **Libvirt Logları**: `/var/log/libvirt/qemu/windows11.log`
- **QEMU Logları**: `/var/log/libvirt/qemu/`
- **Sistem Logları**: `journalctl -u libvirtd`
- **VFIO Logları**: `dmesg | grep vfio`
- **GPU Logları**: `dmesg | grep nvidia`

---

## 📚 Ek Kaynaklar

- [Arch Linux VFIO Wiki](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF)
- [VFIO Discord Sunucusu](https://discord.gg/f63cXwH)
- [r/VFIO Subreddit](https://www.reddit.com/r/VFIO/)
- [Looking Glass Dokümantasyonu](https://looking-glass.io/docs/)
- [HP OMEN Support](https://support.hp.com/)
- [NVIDIA Advanced Optimus](https://www.nvidia.com/en-us/geforce/technologies/optimus/)

---

## ⚠️ Önemli Notlar

1. **Yedekleme**: Kuruluma başlamadan önce önemli verilerinizi yedekleyin
2. **BIOS Güncellemesi**: En son BIOS sürümünü kullanın (HP Support'tan)
3. **Sürücüler**: Windows'ta en son NVIDIA sürücülerini kurun
4. **MUX Switch**: Advanced Optimus teknolojisi sayesinde GPU geçişi otomatik olabilir
5. **Performans**: RTX 4070 Mobile 80W TGP ile mükemmel performans bekleyebilirsiniz
6. **Güvenlik**: Secure Boot'u devre dışı bırakmanız gerekebilir
7. **Sıcaklık**: OMEN Tempest Cooling sistemi sayesinde sıcaklık problemi yaşamazsınız

---

## 🎯 Sonuç

Bu klavuz, HP OMEN Gaming Laptop 16-wf0015nt (7Q7W7EA) modeliniz için özel olarak hazırlanmıştır. Advanced Optimus ve MUX Switch teknolojileri sayesinde GPU passthrough kurulumu daha kolay ve verimli olacaktır.

**Sisteminizin Avantajları:**
- ✅ MUX Switch ile donanım seviyesinde GPU geçişi
- ✅ Advanced Optimus ile otomatik optimizasyon
- ✅ 80W TGP RTX 4070 ile yüksek performans
- ✅ i9-13900HX ile 24 çekirdek güç
- ✅ 32 GB DDR5 RAM ile yeterli bellek
- ✅ QHD 240Hz ekran ile mükemmel görüntü

Herhangi bir sorunla karşılaştığınızda, sorun giderme bölümünü kontrol edin ve gerekirse topluluk kaynaklarından yardım alın.

**İyi şanslar! 🚀**