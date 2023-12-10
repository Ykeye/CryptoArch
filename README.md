
# Очень сильно зашифрованная инсталляция Арч Линукс с btrfs и снэпшотами!
------


Версия для BTRFS пока не тестировалась, основана на гайде https://wiki.archlinux.org/title/User:ZachHilman/Installation_-_Btrfs_%2B_LUKS2_%2B_Secure_Boot

<details>
 
```
Create the filesystem
mkfs.btrfs --label system /dev/mapper/system
Mount at root
mount -t btrfs LABEL=system /mnt
Create the root subvolume (this will be '/' on the final system)
 btrfs subvolume create /mnt/@root
Create the home directory subvolume (this will hold all user data)
 btrfs subvolume create /mnt/@home
Create a subvolume for snapshot storage
 btrfs subvolume create /mnt/@snapshots
Unmount root so we can change mount options
 umount -R /mnt
Remount the root subvolume with options. The compression here is optional, zstd offers the best storage but you may prefer a different algorithm for speed, or omit entirely. Only use ssd if you are on an SSD. You can also enable atime if desired, but it comes with overhead.
 mount -t btrfs -o defaults,x-mount.mkdir,compress=zstd,ssd,noatime,subvol=@root LABEL=system /mnt
Mount the home subvolume (same deal with options as previous step)
 mount -t btrfs -o defaults,x-mount.mkdir,compress=zstd,ssd,noatime,subvol=@home LABEL=system /mnt/home
Mount the snapshots volume to '/.snapshots' (same deal with options as previous step)
 mount -t btrfs -o defaults,x-mount.mkdir,compress=zstd,ssd,noatime,subvol=@snapshots LABEL=system /mnt/.snapshots

=== EFI System Partition ===
Format the partition as FAT-32, with label 'EFI'
 mkfs.fat -F32 -n EFI /dev/disk/by-partlabel/EFI
Make the mount directory
 mkdir /mnt/efi
Mount the ESP
 mount LABEL=EFI /mnt/efi

```

</details>


Это перевод на русский с небольшими дополнениями гайда по установке Arch Linux с шифрованием разделов на LVM с применением LUKS и GRUB для систем на базе UEFI от от [huntrar](https://www.github.com/huntrar).
Дополнительно без перевода доступен раздел по укреплению системы от атак на Secure Boot типа Evil Maid 



Основано на гайде от [huntrar](https://www.github.com/huntrar)
[Смотри Оригинал Тут](https://gist.github.com/huntrar/e42aee630bee3295b2c671d098c81268)
+ дефолтный гайд на [Arch Wiki](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#Encrypted_boot_partition_(GRUB))

**Note: Данный гайд основан на конфигуации с NVMe если у вас  SSD или HDD, замените ```/dev/nvme0nX``` with ```/dev/sdX``` соответсвенно.**



-----
## ПОМНИ, ЮЗЕРНЕЙМ, ЕСЛИ ТЫ НАКОСЯЧИШЬ, ЗАБУДЕШЬ ПАРОЛЬ, ПОБЪЕШЬ ДИСК ИЛИ ЕЩЕ ЧТО, ОЧЕНЬ ВЕРОЯТНО, ЧТО ВСЁ СТАНЕТ ТЫКВОЙ! ДЕЛАЙ [БЭКАПЫ](https://wiki.archlinux.org/title/Synchronization_and_backup_programs)
-----

# ЧТО ДЕЛАТЬ? Я всё ЗАШИФРОВАЛ И СЛОМАЛ!
Не ссы, Петруха, если ты уже разметил диски и тебе надо зайти обратно, то открывай контейнеры руками: 
```
cryptsetup luksOpen /dev/sdX1 luks
mount /dev/mapper/vgcrypt-root /mnt
arch-chroot /mnt /bin/bash
```

--------------

## До установки
### Подключитесь к интернету
[Arch Wiki](https://wiki.archlinux.org/index.php/installation_guide#Connect_to_the_internet).

### Обновите системное время
```
timedatectl set-ntp true
```

### Подготовьте диск

#### (Не обязательно) Забейте весь ваш жесткий диск каким-то рандомным спамом

>*ВНИМАНИЕ* ДАННАЯ ОПЦИЯ УНИЧТОЖИТ ВСЕ ДАННЫЕ /dev/sdХ*Й! БЕЗ ВОЗМОЖНОСТИ ВОССТАНОВЛЕНИЯ! ИСПОЛЬЗУЙ fdisk -l и ВНИМАТЕЛЬНО УБЕДИСЬ ЧТО ПОНИМАЕШЬ, ЧТО ДЕЛАЕШЬ!
--------------------------
--------------
>*И ЕЩЕ РАЗ УБЕДИСЬ ЧТО ПОНИМАЕШЬ*
--------------------------
```
cat /dev/urandom > /dev/sdХ*Й
```

#### Создаем партиции EFI и Linux LUKS
##### В начале делаем 1 мб партицию биос на всякий случай

Number | Start (sector) | End (sector) |    Size    | Code |        Name         |
-------|----------------|--------------|------------|------|---------------------|
   1   |   2048         |   4095       | 1024.0 KiB | EF02 | BIOS boot partition |
   2   |   4096         |   1130495    | 550.0 MiB  | EF00 | EFI System          |
   3   |   1130496      |   976773134  | 465.2 GiB  | 8309 | Linux LUKS          |

```gdisk /dev/nvme0n1```
```
o
n
[Enter]
0
+1M 
ef02 # BIOS партиция 1
n
[Enter]
[Enter]
+550M  
ef00 # EFI партиция 2
n
[Enter]
[Enter]
[Enter]
8309 # Остальное место
w
```

####  Создаем LUKS1 шифрованный контейнер (GRUB не поддерживает LUKS2 с Мая 2019)
```cryptsetup luksFormat --type luks1 --use-random -S 1 -s 512 -h sha512 -i 5000 /dev/nvme0n1p3```

#### Открываем контейнер (расшифровыватся и становится доступным по /dev/mapper/crytlvm)
```
cryptsetup open /dev/nvme0n1p3 cryptlvm
```

### Готовим логические разделы
#### Создаем физический раздел поверх открытого LUKS контейнера
```
pvcreate /dev/mapper/cryptlvm
```

#### Создаем группу разделов и добавляем туда физический раздел 
```
vgcreate vg /dev/mapper/cryptlvm
```

#### Создаем логические разделы для swap, root и home на разделе 
```
lvcreate -L 8G vg -n swap
lvcreate -l 100%FREE vg -n system
```
Рарзмеры свопа и рута стоит поменять в соответсвии со своими нуждами. 

#### Форматируем файловые системы
```
mkfs.btrfs --label system /dev/vg/system
mkswap /dev/vg/swap
```


####

Монтируем btrfs систему

```
  mount -t btrfs LABEL=system /mnt
```

#### Создаем BTRFS subvolume и монтируем их и монтируем swap
```
btrfs subvolume create /mnt/@root
btrfs subvolume create /mnt/@home
btrfs subvolume create /mnt/@snapshots

swapon /dev/vg/swap
```

потом делаем `umount -R /mnt`

и млнтируем их обратно с такими вот параметрами
```
mount -t btrfs -o defaults,x-mount.mkdir,compress=zstd,ssd,noatime,subvol=@root LABEL=system /mnt
```

```
mount -t btrfs -o defaults,x-mount.mkdir,compress=zstd,ssd,noatime,subvol=@root LABEL=system /mnt
```
```
mount -t btrfs -o defaults,x-mount.mkdir,compress=zstd,ssd,noatime,subvol=@home LABEL=system /mnt/home
```

```
mount -t btrfs -o defaults,x-mount.mkdir,compress=zstd,ssd,noatime,subvol=@snapshots LABEL=system /mnt/.snapshots
```


### Готовим EFI партицию
#### Создаем fat32 файловую систему.
```
mkfs.fat -F32 /dev/nvme0n1p2
```

#### Создаем точку монтирования для /efi. Нужно для совместимости c grub-install. Монтируем.
```
mkdir /mnt/efi
mount /dev/nvme0n1p2 /mnt/efi
```

## Установка
### Устанвливаем самый нужный софт -
```base linux linux-firmware mkinitcpio lvm2 vi wpa_supplicant btrfs-progs grub``` ≈ обязательный минимум для ноутбуков ( wpa_supplicant можно убрать если не нужен wi-fi)

```
pacstrap /mnt base linux linux-firmware mkinitcpio lvm2 vi vim tmux sudo dhcpcd wpa_supplicant grub openssh networkmanager network-manager-applet
```

## Если ставим i3

```
i3-wm xorg xorg-xinit i3status-rust rofi xorg-server xorg-apps
```
##### Обязательно добавим к списку какой-то из эмуляторов терминала, например:

- alacritty
- rxvt-unicode
- terminator
------

## Настраиваем систему
### Генерим фстаб!
```
genfstab -U /mnt >> /mnt/etc/fstab
```

#### (Доп!) Изменяем `relatime` на `noatime` (может вызывать косяки)

```/mnt/etc/fstab```

### Меняем рут
```
arch-chroot /mnt
```

#### На этом моменте при введении ```lsblk``` вы должны видеть что-то вроде:

NAME            MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
sda               8:0    0   100G  0 disk
|-sda1            8:1    0     1M  0 part
|-sda2            8:2    0   550M  0 part  /efi
`-sda3            8:3    0  99.5G  0 part
  `-cryptlvm    254:0    0  99.5G  0 crypt
    |-vg-swap   254:1    0     8G  0 lvm   [SWAP]
    `-vg-system 254:2    0  91.5G  0 lvm   /home
                                           /.snapshots
                                           /
### Время
#### Установим часовой пояс
Заменим /Europe/Moscow/ на ваш часовой пояс из `/usr/share/zoneinfo`
```
ln -sf /usr/share/zoneinfo/Europe/Moscow /etc/localtime
```

#### Запустим `hwclock` чтоб сгенерировать ```/etc/adjtime```
```
hwclock --systohc
```

### Локализация
#### Для англ версии раскомментим ```en_US.UTF-8 UTF-8``` в файое ```/etc/locale.gen``` и сгенерируй локаль
```
locale-gen
```

#### Создем ```locale.conf``` и зададим переменную ```LANG``` 
```/etc/locale.conf```
```
LANG=en_US.UTF-8
```
запусти locale-gen еще раз на всякий случай

### Настройка сети
#### Создадим файл с хостнеймом
```/etc/hostname```
```
myhostname
```
myhostname - произвольное сетевое имя устройства.

#### Заполним /etc/hosts
```/etc/hosts```
```
127.0.0.1 localhost
::1 localhost
127.0.1.1 myhostname.localdomain myhostname
```

### Initramfs
#### Добавим ```keyboard```, ```encrypt```,  ```lvm2``` и ```consolefont``` ```btrfs``` хуки в ```/etc/mkinitcpio.conf```

```
HOOKS=(base udev autodetect modconf block keymap encrypt lvm2 consolefont filesystems btrfs keyboard fsck shutdown)

```
----
#### *ЗАМЕТКО:* Внимание, тут важен порядок! Ориентируемся на вариант выше - самый актуальный.
----
*user.append brain* 

```
HOOKS=(base udev autodetect keyboard modconf block encrypt lvm2 filesystems fsck)
```

#### Обновим initramfs образ
```
mkinitcpio -p linux
```

### Зададим рут пароль
```
passwd
```

### Загрузчик

#### Настроим  GRUB чтоб бутился с /boot зашифрованной LUKS партиции

```vim /etc/default/grub```

```
GRUB_ENABLE_CRYPTODISK=y
```

#### Настроим параметры GRUB так, чтобы LVM диск расшифровывался при загрузке

##### UUID можно найти так:
```blkid```
```
/dev/nvme0n1p3: UUID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" TYPE="crypto_LUKS" PARTLABEL="Linux LUKS" PARTUUID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

```vim /etc/default/grub```

##### Вставим вместо xxxxxx UUID из предыдущего шага
```
GRUB_CMDLINE_LINUX="cryptdevice=UUID=9c726b8d-a693-47f0-934a-e2f7cbaf1989:cryptlvm root=/dev/mapper/vg-system subvol=/@root loglevel=3 quiet cryptkey=rootfs:/root/secrets/crypto_keyfile.bin cryptkey=rootfs:/root/secrets/crypto_keyfile.bin"

```


"cryptkey=" расщифровывает 2ой раунд сохраненным ключем. Нужно чтобы не вводить пароль 2 раза подряд и похоже это не работает, надо подумать


#### Установим и настроим efiboot manager для GRUB загрузки UEFI
```
pacman -S efibootmgr
grub-install --target=x86_64-efi --efi-directory=/efi
```

#### Разрешим микрокод апдейты 

__grub-mkconfig автоматически обнаружит эти апдейты и всё сделает заебись(хз чё он делает, есчесн, игнорируй, если знаешь, что делаешь)__
Для интел используем intel-ucode а для amd - amd-ucode
```
pacman -S intel-ucode
```

#### Генерим конфиг файл GRUB
```
grub-mkconfig -o /boot/grub/grub.cfg
```

#### Это кусок, который решает проблему с тем, что пароль вводить надо 2 раза для расшифровки .
<details> <summary>  (Не обязательно) </summary>


#### Создаем keyfile и добавим его как ключ для LUKS
```
mkdir /root/secrets && chmod 700 /root/secrets
head -c 64 /dev/urandom > /root/secrets/crypto_keyfile.bin && chmod 600 /root/secrets/crypto_keyfile.bin
cryptsetup -v luksAddKey -i 1 /dev/nvme0n1p3 /root/secrets/crypto_keyfile.bin
```

#### Добавим keyfile в образ initramfsт
```/etc/mkinitcpio.conf```
```
FILES=(/root/secrets/crypto_keyfile.bin)
```


#### Пересоздадим образ
```
mkinitcpio -p linux
```


#### Ограничим доступ до /boot
```
chmod 700 /boot
```
#### Пользователя добавим
```
useradd --create-home example_user
passwd example_user
usermod --append --groups wheel example_user
visudo
```
Раскомментим ```%wheel ALL=(ALL) ALL``` чтоб разрешить wheel


Сохраним и выйдем из visudo

### Установим всякую нужную дрянь:

   - git
   - base-devel package
   - yay ( AUR помощник)

```
pacman -S --needed git base-devel && git clone https://aur.archlinux.org/yay.git && cd yay && makepkg -si
```
Занимет около  300MB + 70mb скачивает

### Установка завершена.

```
exit
reboot
```
-----
### После ребута
-----
```
pacman -S 
```
#### Ставим легкий display manager
```
yay -S ly
```
#### Конфигурим его 

редактируем соответственно - ```/etc/ly/config.ini```

----
Рекомендуется включить animation для максимального doom hellfire
----

#### Не забудь про звук
```
pacman -S pulseaudio alsa-utils alsa-plugins pavucontrol 
```
and utils - fonts

```
pacman -S wget ttf-font-awesome firefox
```
## После инсталла
### Настраиваем драйвера для xorg
```
lspci -v | grep -A1 -e VGA -e 3D
```
Например так:
```
-Ss xf86-video-vmware
```
xf86-video-vmware заменяется на xf86-video и там уже как пойдет. Дефолтный ```-Ss xf86-video```


Включим нужных демонов 
```
systemctl enable sshd
systemctl enable dhcpcd
systemctl enable NetworkManager
systemctl enable ly
````
## Настраиваем i3 и i3-status-rust


редактируем .config/i3/config
   - включаем rofi
   - изменяем i3status на i3status-rs

скопируем пример конфига status-bar-rust

```
cp /usr/share/doc/i3status-rust/examples/config.toml ~.config/i3status-rust/config.toml
```

## Добавляем Black Arch

Переходим в нужную дирекорию и 

```
curl -O https://blackarch.org/strap.sh
chmod +x strap.sh
sudo ./strap.sh
```

--------------------
--------------------
--------------



### ( Рекомендовано к исполнению) Hardening against Evil Maid attacks
With an encrypted boot partition, nobody can see or modify your kernel image or initramfs, but you would be still vulnerable to [Evil Maid](https://www.schneier.com/blog/archives/2009/10/evil_maid_attac.html) attacks.

One possible solution is to use UEFI Secure Boot. Get rid of preloaded Secure Boot keys (you really don't want to trust Microsoft and OEM), enroll [your own Secure Boot keys](https://wiki.archlinux.org/index.php/Secure_Boot#Using_your_own_keys) and sign the GRUB boot loader with your keys. Evil Maid would be unable to boot modified boot loader (not signed by your keys) and the attack is prevented.

#### Creating keys
The following steps should be performed as the `root` user, with accompanying files stored in the `/root` directory.

##### Install `efitools`
```
pacman -S efitools
```

##### Create a GUID for owner identification
```
uuidgen --random > GUID.txt
```

##### Platform key
CN is a Common Name, which can be written as anything.

```
openssl req -newkey rsa:4096 -nodes -keyout PK.key -new -x509 -sha256 -days 3650 -subj "/CN=my Platform Key/" -out PK.crt
openssl x509 -outform DER -in PK.crt -out PK.cer
cert-to-efi-sig-list -g "$(< GUID.txt)" PK.crt PK.esl
sign-efi-sig-list -g "$(< GUID.txt)" -k PK.key -c PK.crt PK PK.esl PK.auth
```

##### Sign an empty file to allow removing Platform Key when in "User Mode"
```
sign-efi-sig-list -g "$(< GUID.txt)" -c PK.crt -k PK.key PK /dev/null rm_PK.auth
```

##### Key Exchange Key
```
openssl req -newkey rsa:4096 -nodes -keyout KEK.key -new -x509 -sha256 -days 3650 -subj "/CN=my Key Exchange Key/" -out KEK.crt
openssl x509 -outform DER -in KEK.crt -out KEK.cer
cert-to-efi-sig-list -g "$(< GUID.txt)" KEK.crt KEK.esl
sign-efi-sig-list -g "$(< GUID.txt)" -k PK.key -c PK.crt KEK KEK.esl KEK.auth
```

##### Signature Database key
```
openssl req -newkey rsa:4096 -nodes -keyout db.key -new -x509 -sha256 -days 3650 -subj "/CN=my Signature Database key/" -out db.crt
openssl x509 -outform DER -in db.crt -out db.cer
cert-to-efi-sig-list -g "$(< GUID.txt)" db.crt db.esl
sign-efi-sig-list -g "$(< GUID.txt)" -k KEK.key -c KEK.crt db db.esl db.auth
```

#### Signing bootloader and kernel
When Secure Boot is active (i.e. in "User Mode") you will only be able to launch signed binaries, so you need to sign your kernel and boot loader.

Install `sbsigntools`
```
pacman -S sbsigntools
```
```
sbsign --key db.key --cert db.crt --output /boot/vmlinuz-linux /boot/vmlinuz-linux
sbsign --key db.key --cert db.crt --output /efi/EFI/arch/grubx64.efi /efi/EFI/arch/grubx64.efi
```

##### Automatically sign bootloader and kernel on install and updates
It is necessary to sign GRUB with your UEFI Secure Boot keys every time the system is updated via `pacman`. This can be accomplished with a [pacman hook](https://jlk.fjfi.cvut.cz/arch/manpages/man/alpm-hooks.5).

Create the hooks directory
```
mkdir -p /etc/pacman.d/hooks
```

Create hooks for both the `linux` and `grub` packages

```/etc/pacman.d/hooks/99-secureboot-linux.hook```
```
[Trigger]
Operation = Install
Operation = Upgrade
Type = Package
Target = linux

[Action]
Description = Signing Kernel for SecureBoot
When = PostTransaction
Exec = /usr/bin/find /boot/ -maxdepth 1 -name 'vmlinuz-*' -exec /usr/bin/sh -c 'if ! /usr/bin/sbverify --list {} 2>/dev/null | /usr/bin/grep -q "signature certificates"; then /usr/bin/sbsign --key /root/db.key --cert /root/db.crt --output {} {}; fi' \ ;
Depends = sbsigntools
Depends = findutils
Depends = grep
```

```/etc/pacman.d/hooks/98-secureboot-grub.hook```
```
[Trigger]
Operation = Install
Operation = Upgrade
Type = Package
Target = grub

[Action]
Description = Signing GRUB for SecureBoot
When = PostTransaction
Exec = /usr/bin/find /efi/ -name 'grubx64*' -exec /usr/bin/sh -c 'if ! /usr/bin/sbverify --list {} 2>/dev/null | /usr/bin/grep -q "signature certificates"; then /usr/bin/sbsign --key /root/db.key --cert /root/db.crt --output {} {}; fi' \ ;
Depends = sbsigntools
Depends = findutils
Depends = grep
```

#### Enroll keys in firmware
##### Copy all `*.cer`, `*.esl`, `*.auth` to the EFI system partition
```
cp /root/*.cer /root/*.esl /root/*.auth /efi/
```

##### Boot into UEFI firmware setup utility (frequently but incorrectly referred to as "BIOS")
```
systemctl reboot --firmware
```

Firmwares have various different interfaces, see [Replacing Keys Using Your Firmware's Setup Utility](http://www.rodsbooks.com/efi-bootloaders/controlling-sb.html#setuputil) if the following instructions are unclear or unsuccessful.

##### Set OS Type to `Windows UEFI mode`
Find the Secure Boot options and set OS Type to `Windows UEFI mode` (yes, even if we're not on Windows.) This may be necessary for Secure Boot to function.

##### Clear preloaded Secure Boot keys


Using Key Management, clear all preloaded Secure Boot keys (Microsoft and OEM).

By clearing all Secure Boot keys, you will enter into Setup Mode (so you can enroll your own Secure Boot keys).

##### Set or append the new keys
The keys must be set in the following order:

```db => KEK => PK```

This is due to some systems exiting setup mode as soon as a `PK` is entered.

Do not load the factory defaults, instead navigate the available filesystems in search of the files previously copied to the EFI System partition.

Choose any of the formats. The firmware should prompt you to enter the type (*Note:* type names may differ slightly.)
```
*.cer is a Public Key Certificate
*.esl is a UEFI Secure Variable
*.auth is an Authenticated Variable
```

Certain firmware (such as my own) require you use the *.auth files. Try various ones until they work.

##### Set UEFI supervisor (administrator) password
You must also set your UEFI firmware supervisor (administrator) password in the Security settings, so nobody can simply boot into UEFI setup utility and turn off Secure Boot.

You should never use the same UEFI firmware supervisor password as your encryption password, because on some old laptops, the supervisor password could be recovered as plaintext from the EEPROM chip.

##### Exit and save changes
Once you've loaded all three keys and set your supervisor password, hit F10 to exit and save your changes.

If everything was done properly, your boot loader should appear on reboot.

#### Check if Secure Boot was enabled
```
od -An -t u1 /sys/firmware/efi/efivars/SecureBoot-XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX
```
The characters denoted by XXXX differ from machine to machine. To help with this, you can use tab completion or list the EFI variables.

If Secure Boot is enabled, this command returns 1 as the final integer in a list of five, for example:

```6  0  0  0  1```

If Secure Boot was enabled and your UEFI supervisor password set, you may now consider yourself protected against Evil Maid attacks.
