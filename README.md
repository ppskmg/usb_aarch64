# Установка aarch64 на SSD USB c файловой системой btrfs, шифрованием и удалённой разблакировкой по ssh

## Необходимо:
1. RPI 4
2. micro SD
3. flash USB
4. SSD USB (NVME m2)
5. Устройство для доступа по SSH (В моём случае iPad PRO + blink)
6. Устройство для записи образа (В моём случае Android)
7. Клавиатура + мыш
8. ethernet
9. Монитор 

## Запись образа через телефон:
1. Вставить кардридер или флешку в телефон
2. Установить приложение Raspi Card Imager:
3. Setting > Format SD
4. Setting > Choice OS > Raspberry Pi OS light
5. Выбрать SD
6. Write to SD

## Установка:
1. Извлечь SD из телефона
2. Вставить SD в Raspberry PI
3. Подключить переферию (мыш, клавиатура, ethernet, монитор)
4. Включить питание RPI
> Если при включении на экране RGB цвета и не чего не происходит, то отключить монитор и подключить снова
5. Пройти процедуру установки

## Включение SSH:
```
sudo raspi-config
```
В настрйках найти SSH включить и перезагрузить RPI
Дальше можно перейти к работе через SSH (iPad pro + Blink + ssh/mosh)

## Настрйка коннекта с iPad:
Blink >
```
config
```
в конфиге > Host > add
name: rpi
login: pi
Ip: смотреть назанченный ip провайдером 
> Зайти в настрйоки провайдера (скорее всего по адресу 192.168.1.1) в настройках > lan > подключенные > raspberypi
password: raspberry

Blink > 
```
ssh rpi
```
При первом подключении запросит доверие к сертификату: Y


Обновить EEPROM Pi, чтобы он поддерживал полную загрузку с USB.
```
sudo apt update
sudo apt full-upgrade
```

Отредактируйте файл конфигурации, определяющий, какие обновления встроенного ПО вы получаете:
```
sudo nano /etc/default/rpi-eeprom-update
```
изменить строку на:
FIRMWARE_RELEASE_STATUS="stable"

Обновить прошивку Pi:
```
sudo rpi-eeprom-update -d -a
```

Вставить usb флешку

```
fdisk /dev/sdX
```

```
o
p

n
p
1
enter
+500M
t
c

n
p
t
0

w
```

```
mkfs.vfat /dev/sdX1
mkdir boot
mount /dev/sdX1 boot
```

```
mkfs.ext4 /dev/sdX2
mkdir root
mount /dev/sdX2 root
```

Загрузите и извлеките корневую файловую систему (как root, а не через sudo):

```
wget http://os.archlinuxarm.org/os/ArchLinuxARM-rpi-aarch64-latest.tar.gz
bsdtar -xpf ArchLinuxARM-rpi-aarch64-latest.tar.gz -C root
sync
```

```
mv root/boot/* boot
```

```
umount boot root
```

```
nano root/etc/fstab
```

### Чтобы при перезагрузке без SD, загрузится с flash USB
```
..mmcblk0.. boot
```
на
```
/dev/sda1 boot
```

```
sudo poweroff
```
1. извлечь SD
2. оставить USB
3. включить питание

## Настройка dropbear для раннего доступа по SSH во время загрузки 

На хосте, сгенерировать ключ где alarm@192.168.1._ пользователь и ip текущей установки на flash USB 
```
ssh-keygen -t rsa -b 4096 -a 100 -f ~/.ssh/pi_unlock_key
rsync ~/.ssh/pi_unlock_key.pub alarm@192.168.1._:/home/alarm/
```

### Для разблакировки
#### Mac
```
nvim ~/.ssh/config
--------------------------------------------------------------------------------
Host pi-unlock
  HostName pi-locked
  User root
  IdentityFile ~/.ssh/pi_unlock_key
```

#### iPad
```
blink > config > add host
```

#### На RPI
```
sudo mv ~/pi_unlock_key.pub /etc/dropbear/root_key
sudo rm -v /etc/ssh/ssh_host_*
sudo ssh-keygen -t rsa -b 4096 -f ssh_host_rsa_key -N "" -m PEM < /dev/null
```
```
sudo nvim /etc/mkinitcpio.conf
```

Для следующего шага нужно установить пакеты
```
sudo pacman -S mkinitcpio-dropbear mkinitcpio-utils mkinitcpio-netconf
```

```
# ...
  MODULES=(btrfs pcie_brcmstb broadcom)

# ...
  BINARIES=(/usr/lib/libgcc_s.so.1)

# ...
  FILES=()

# ...
  HOOKS=(base udev autodetect keyboard keymap modconf block sleep netconf dropbear encryptssh filesystems fsck)
```

```
sudo mkinitcpio -P
```

## Подготовка SSD USB:
Из расчёта что 
> flash USB = /dev/sda (активный с OS)
> ssd USB = /dev/sdb

### Почистить диск
```
sudo dd if=/dev/zero of=/dev/sdb bs=512  status=progress
```
> Подробней:
> https://wiki.archlinux.org/title/Dm-crypt/Drive_preparation

### Подготовка разделов:
```
sudo fdisk -l
```

```
sudo fdisk /dev/sdb
```

```
o
n
p
1
enter
+500M
t
c

n
p
enter
enter
enter

p

w
```

### Шифрование:
```
sudo mkfs.vfat /dev/sdb1
sudo cryptsetup luksFormat -c aes-xts-plain64 -s 512 -h sha512 --use-random -i 1000 /dev/sdb2
```
```
sudo cryptsetup luksOpen /dev/sda2 root
```
> Если есть ошибки, перезагрузить систему и повторить предыдущую команду
> связано с добавленным модулем encrypt в mkinitcpio 

```
sudo mkfs.btrfs /dev/mapper/root
```

### Настройка подтома btrfs
```
sudo mount /dev/mapper/root /mnt
sudo btrfs subvolume create /mnt/rootfs
sudo umount /mnt
sudo cryptsetup close root
```

### Создание копии конфига загрузчика на flash USB:
```
sudo cp /boot/boot.txt /boot/boot.txt.new
sudo vim /boot/boot.txt.new
```

Закомментировать строку:
```
part uuid ...
```
В строке 
```
setenv
```

Заменить часть 
```
root=PARTUUID=${uuid}
```
на

```
ip=::::pi-locked:eth0:dhcp cryptdevice=/dev/sda2:root root=/dev/mapper/root rootflags=subvol=rootfs
```

для тонкой настройки IP
```
ip=<client-ip>:<server-ip>:<gateway-ip>:<netmask>:<hostname>:<device>:<autoconf>:<dns0-ip>:<dns1-ip>
```
> Подробнее
> https://wiki.archlinux.org/index.php/Mkinitcpio#Using_net

### Итоговый файл boot.txt:
```
# After modifying, run ./mkscr

# Set root partition to the second partition of boot device
#part uuid ${devtype} ${devnum}:2 uuid

setenv bootargs console=ttyS1,115200 console=tty0 ip=::::pi-locked:eth0:dhcp cryptdevice=/dev/sda2:root root=/dev/mapper/root rootflags=subvol=rootfs rw rootwait smsc95xx.macaddr="${usbethaddr}"

# ...
```
#### Не запускать ./mkscr до переноса системы на SSD USB

## Клонирование системы
```
mkdir ~/pi-setup
cd ~/pi-setup
mkdir usb-boot
mkdir usb-root
sudo fdisk -l
sudo cryptsetup luksOpen /dev/sda2 root
sudo mount /dev/sda1 usb-boot/
sudo mount -t btrfs -o subvol=rootfs /dev/mapper/root usb-root/
```

```
sudo rsync --info=progress2 -axHAX /boot/ usb-boot/
sudo rsync --info=progress2 -axHAX / usb-root/
sudo sync
```

### в случае, если система установка была на SD карту
```

sudo vim usb-root/etc/fstab
```

```
заменив /dev/mmcblk1p1 на /dev/sda1
```

> Но так как Raspbian установлена на SD карту и извлечена

> aarch64 установлена на flash USB то диску по дефолту выдаётся sda
> и при клонировании SSD USB будет вставлен как первый диск, и так же получит /dev/sda

> flash USB будет бэкапом установки, на случай, если что-то пошло не так и из неё можно всегда загрузиться и что-то исправить


### Содержимое fstab
```
/dev/sda1 /boot vfat  defaults  0 0
```

```
cd ~/pi-setup/usb-boot
sudo mv boot.txt.new boot.txt
sudo ./mkscr
```

## Финал
```
cd ~/pi-setup
sudo umount usb-boot usb-root
sudo cryptsetup close root
sudo poweroff

```

1. Извлечь flash USB
2. Включить питание
3. Подождать 20 сек

```
ssh pi-unlock
```

## Источники по теме:
1. https://archlinuxarm.org/about/downloads
2. https://archlinuxarm.org/platforms/armv8/broadcom/raspberry-pi-4
3. https://wiki.archlinux.org/title/Dm-crypt/Drive_preparation
4. https://gist.github.com/XSystem252/d274cd0af836a72ff42d590d59647928
