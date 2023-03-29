#Linux #Arch #Tutos #LVM 

```bash
#!/bin/bash

UUID=$(blkid -s UUID -o value /dev/mapper/crypt--os-cryptroot)
parted /dev/nvme0n1 mklabel gpt && # Initialiser la table de partition (GPT) sur >parted /dev/sdb mklabel gpt
parted /dev/nvme0n2 mklabel gpt &&
parted /dev/nvme0n3 mklabel gpt &&
parted /dev/nvme0n4 mklabel gpt &&

parted /dev/nvme0n1 mkpart primary fat32 1MiB 513MiB && # Creation partition UEFI
parted /dev/nvme0n1 set 1 esp on && # On met le type ESP (EFI System Partition)

parted /dev/nvme0n1 mkpart primary ext4 513MiB 100% && # Creation de la deuxième partition
parted /dev/nvme0n2 mkpart primary ext4 0% 100% &&
parted /dev/nvme0n3 mkpart primary ext4 0% 100% &&
parted /dev/nvme0n4 mkpart primary ext4 0% 100% &&

pvcreate /dev/nvme0n1 /dev/nvme0n2 /dev/nvme0n3 /dev/nvme0n4
read
vgcreate crypt-os /dev/nvme0n1p2 /dev/nvme0n2p1
read
vgcreate crypt-data /dev/nvme0n3p1 /dev/nvme0n4p1
read
lvcreate -L 30G -n cryptroot crypt-os
read
lvcreate -L 8G -n cryptswap crypt-os
read
lvcreate -l +100%FREE -n crypthome crypt-os
read
lvcreate -L 15G -n crypttimeshift crypt-data
read
lvcreate -l +100%FREE -n cryptgames crypt-data
read
cryptsetup luksFormat /dev/crypt-os/cryptroot --type luks2 &&
cryptsetup open /dev/crypt-os/cryptroot root &&

mkfs.ext4 /dev/mapper/root -L root &&
mount /dev/mapper/root /mnt &&
dd if=/dev/zero of=/dev/nvme0n1p1 bs=1M status=progress
read
mkfs.fat -F32 /dev/nvme0n1p1
mount /dev/nvme0n1p1 /mnt/boot --mkdir
read

pacstrap /mnt base base-devel linux linux-firmware sysfsutils usbutils e2fsprogs inetutils netctl nano less which man-db man-pages lvm2 cryptsetup grub efibootmgr &&

genfstab -U /mnt >> /mnt/etc/fstab

```

```bash
sed -i '/^HOOKS=/ s/block filesystems/block lvm2 encrypt filesystems/' /etc/mkinitcpio.conf
```
`nano /etc/mkinitcpio.conf`
```bash
HOOKS=(base udev autodetect keyboard keymap modconf kms consolefont block lvm2 encrypt filesystems fsck)
```

`mkinitcpio -P`
`blkid
```bash
/dev/mapper/crypt--os-cryptroot: UUID="b89b5021-63e1-44ff-9dd5-caf82a610f08" TYPE="crypto_LUKS"
/dev/mapper/root: LABEL="root" UUID="72c78961-48fb-4786-a229-006beee43fce" BLOCK_SIZE="4096" TYPE="ext4"
```

```bash
sed -i "s/GRUB_CMDLINE_LINUX_DEFAULT=\"loglevel=3 quiet\"/GRUB_CMDLINE_LINUX_DEFAULT=\"loglevel=3 quiet cryptdevice=UUID=${UUID}:root root=\/dev\/mapper\/root\"/" /etc/default/grub

sed -i 's/#\(GRUB_DISABLE_OS_PROBER=false\)/\1/' /etc/default/grub
```

`nano /etc/default/grub
```bash
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet cryptdevice=UUID=b89b5021-63e1-44ff-9dd5-caf82a610f08:root root=/dev/mapper/root"
```

#### Swap chiffré
`nano /etc/crypttab`
```bash
swap            /dev/crypt-os/cryptswap         /dev/urandom            swap,cipher=aes-xts-plain64,size=256
```
`nano /etc/fstab`
```bash
/dev/mapper/swap        none            swap            sw              0               0
```

#### Install GRUB
```bash
 grub-install --efi-directory=/boot --bootloader-id=GRUB
 grub-mkconfig -o /boot/grub/grub.cfg
```

#### Encrypt other device
```bash
mkdir -m 700 /etc/luks-keys # Cette commande crée un répertoire nommé "luks-keys" dans le répertoire "/etc/", et lui assigne les autorisations de lecture, d'écriture et d'exécution (700) à l'utilisateur qui exécute la commande.

dd if=/dev/random of=/etc/luks-keys/home bs=1 count=256 status=progress # Cette commande copie 256 octets aléatoires à partir du périphérique "/dev/random" dans le fichier "/etc/luks-keys/home".
dd if=/dev/random of=/etc/luks-keys/timeshift bs=1 count=256 status=progress
dd if=/dev/random of=/etc/luks-keys/games bs=1 count=256 status=progress
```

```bash
cryptsetup luksFormat -v /dev/crypt-os/crypthome /etc/luks-keys/home --type luks2
cryptsetup luksFormat -v /dev/crypt-data/crypttimeshift /etc/luks-keys/timeshift --type luks2
cryptsetup luksFormat -v /dev/crypt-data/cryptgames /etc/luks-keys/games --type luks2 
```

```bash
cryptsetup -d  /etc/luks-keys/home open /dev/crypt-os/crypthome home
cryptsetup -d  /etc/luks-keys/timeshift open /dev/crypt-data/crypttimeshift timeshift
cryptsetup -d  /etc/luks-keys/games open /dev/crypt-data/cryptgames games
```

```bash
mkfs.ext4 -L home /dev/mapper/home
mkfs.ext4 -L timeshift /dev/mapper/timeshift
mkfs.ext4 -L games /dev/mapper/games
```

```bash
mount /dev/mapper/home /mnt/home --mkdir
mount /dev/mapper/timeshift /mnt/timeshift --mkdir
mount /dev/mapper/games /mnt/games --mkdir
```

#### Setting up device for mounting and unencrypting
`nano /etc/crypttab`
```bash
home            /dev/crypt-os/crypthome         /etc/luks-keys/home
timeshift       /dev/crypt-data/crypttimeshift  /etc/luks-keys/timeshift
```
`nano /etc/fstab`
```bash
/dev/mapper/home        /home           ext4    defaults        0       2
/dev/mapper/timeshift   /mnt/timeshift  ext4    defaults        0       0
/dev/mapper/games       /mnt/games      ext4    defaults        0       0
```


#### Historique
```bash
    1  passwd
    2  blkid
    3  blkid -s UUID -o value /dev/mapper/crypt--os-cryptroot
    4  sed -i 's/HOOKS=(base udev autodetect keyboard keymap modconf kms consolefont block filesystems fsck)/HOOKS=(base udev autodetect keyboard keymap modconf kms consolefont block lvm2 encrypt filesystems fsck)/' /etc/mkinitcpio.conf
    5  nano /etc/mkinitcpio.conf
    6  sed -i '/^HOOKS=/ s/block filesystems/block lvm2 encrypt filesystems/' /etc/mkinitcpio.conf
    7  nano /etc/mkinitcpio.conf
    8  blkid -s UUID -o value /dev/mapper/crypt--os-cryptroot
    9  UUID=$(blkid -s UUID -o value /dev/mapper/crypt--os-cryptroot)
   10  sed -i "s/GRUB_CMDLINE_LINUX_DEFAULT=\"loglevel=3 quiet\"/GRUB_CMDLINE_LINUX_DEFAULT=\"loglevel=3 quiet cryptdevice=UUID=${UUID}:root root=\/dev\/mapper\/root\"/" /etc/default/grub
   11  nano /etc/default/grub
   12  mkinitcpio -P
   13  nano /etc/default/grub
   14  nano fstab.sh
   15  chmod +x fstab.sh
   16  ./fstab.sh
   17  cat /etc/fstab
   18  cat /etc/crypttab
   19   grub-install --efi-directory=/boot --bootloader-id=GRUB
   20   grub-mkconfig -o /boot/grub/grub.cfg
   21  mkdir -m 700 /etc/luks-keys # Cette commande crée un répertoire nommé "luks-keys" dans le répertoire "/etc/", et lui assigne les autorisations de lecture, d'écriture et d'exécution (700) à l'utilisateur qui exécute la commande.
   22  dd if=/dev/random of=/etc/luks-keys/home bs=1 count=256 status=progress # Cette commande copie 256 octets aléatoires à partir du périphérique "/dev/random" dans le fichier "/etc/luks-keys/home".
   23  dd if=/dev/random of=/etc/luks-keys/timeshift bs=1 count=256 status=progress
   24  dd if=/dev/random of=/etc/luks-keys/games bs=1 count=256 status=progress
   25  cryptsetup luksFormat -v /dev/crypt-os/crypthome /etc/luks-keys/home --type luks2
   26  cryptsetup luksFormat -v /dev/crypt-data/crypttimeshift /etc/luks-keys/timeshift --type luks2
   27  cryptsetup luksFormat -v /dev/crypt-data/cryptgames /etc/luks-keys/games --type luks2
   28  cryptsetup -d  /etc/luks-keys/home open /dev/crypt-os/crypthome home
   29  cryptsetup -d  /etc/luks-keys/timeshift open /dev/crypt-data/crypttimeshift timeshift
   30  cryptsetup -d  /etc/luks-keys/games open /dev/crypt-data/cryptgames games
   31  mkfs.ext4 -L home /dev/mapper/home
   32  mkfs.ext4 -L timeshift /dev/mapper/timeshift
   33  mkfs.ext4 -L games /dev/mapper/games
   34  mount /dev/mapper/home /mnt/home --mkdir
   35  mount /dev/mapper/timeshift /mnt/timeshift --mkdir
   36  mount /dev/mapper/games /mnt/games --mkdir
   37  echo arch-crypt > /etc/hostname
   38  nano /etc/hosts
   39  nano /etc/locale.gen
   40  locale-gen
   41  echo LANG=en_US.UTF-8 > /etc/locale.conf
   42  export LANG=en_US.UTF-8
   43  echo KEYMAP=fr-latin1 > /etc/vconsole.conf
   44  ln -sf /usr/share/zoneinfo/Europe/Paris /etc/localtime
   45  nano /etc/pacman.conf
   46  pacman -Syu
   47  useradd -m -G audio,video,input,wheel,sys,log,rfkill,lp,adm -s /bin/bash ivar
   48  passwd ivar
   49  nano visudo
   50  pacman -S sudo
   51  nano visudo
   52  EDITOR=nano visudo
   53  pacman -S xorg-server xorg-xinit xorg-xrandr xorg-xfontsel xorg-xlsfonts xorg-xkill xorg-xinput xorg-xwininfo plasma kdialog packagekit-qt5 kcalc icoutils libappimage konsole dolphin kdegraphics-thumbnailers svgpart ffmpegthumbs kdenetwork-filesharing gwenview kimageformats ark kate okular kcron kdf filelight print-manager kdeconnect sshfs sddm
   54  systemctl enable sddm
   55  systemctl enable dhcpcd
   56  pacman -S dhcpcd
   57  pacman -S dhcpcd
   58  systemctl enable dhcpcd
   59  pacman -S jshon expac git wget acpid avahi net-tools xdg-user-dirs dkms
   60  pacman -S networkmanager networkmanager-openvpn networkmanager-pptp networkmanager-vpnc
   61  systemctl enable NetworkManager
```
#### fstab.sh
```bash
#!/bin/bash

cat << EOF >> /etc/fstab
/dev/mapper/swap        none            swap    sw              0       0
/dev/mapper/home        /home           ext4    defaults        0       2
/dev/mapper/timeshift   /mnt/timeshift  ext4    defaults        0       0
/dev/mapper/games       /mnt/games      ext4    defaults        0       0
EOF

cat << EOF >> /etc/crypttab
swap            /dev/crypt-os/cryptswap         /dev/urandom            swap,cipher=aes-xts-plain64,size=256
home            /dev/crypt-os/crypthome         /etc/luks-keys/home
timeshift       /dev/crypt-data/crypttimeshift  /etc/luks-keys/timeshift
games           /dev/crypt-data/cryptgames      /etc/luks-keys/games
```