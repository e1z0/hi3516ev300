# China's most nerve-wracking camera

# ORIGINAL ENVIRONMENT INFO

```bash
hisilicon # printenv
arch=arm
baudrate=115200
board=hi3516ev300
board_name=hi3516ev300
bootargs=mem=40M console=ttyAMA0,115200 root=/dev/mtdblock2 rootfstype=squashfs mtdparts=hi_sfc:320K(boot),1856K(kernel),1024K(rootfs),384K(config),4608K(data)
bootcmd=sf probe 0;sf read 0x42000000 0x50000 0x1d0000;bootm 0x42000000
bootdelay=0
cpu=armv7
ethact=eth0
ethaddr=72:bd:88:4a:24:05
fileaddr=42000000
filesize=fe000
ipaddr=192.168.1.70
serverip=192.168.1.20
soc=hi3516ev300
stderr=serial
stdin=serial
stdout=serial
vendor=hisilicon
verify=n

Environment size: 557/65532 bytes
```


# Serial UART

 ![](/camera_raspberrypi_gpio_ev300-2.png)

The RaspberryPI needs to disable internal UART serial debugging support, [manual here](https://www.raspberrypi.org/documentation/configuration/uart.md).

```bash
screen -L -Logfile cam.log /dev/ttyS0 115200
```


 ![](/Screenshot%202021-05-11%20at%2021.18.15.png)

# PTZ problem solution (fix)

Out of the box PTZ does not work on camera it has unpowered section on the board and also badly connected pelco d txd cable which need to be placed in the right pin. From the long testing it’s obvious that camera was not designed to be powered using external DC connector but over PoE. In the deeper testing it was clear that it’s not enough too, it needs to be changed some cable connections that comes from camera module to the ptz control board, because the main serial connection was connected to the wrong ping and not reaching the stm8 microcontroller which is responsible for the camera movement controls. As shown in this original picture


 ![](/IMG_2648.jpeg)
 
 )Here are detailed trace how things should be and what are the real pins to stm8 rx/tx

 ![](/IMG_2669.jpeg)

# Root

Create hash on another linux machine

```bash
openssl passwd -1  -salt 5RPVAd simplepass
$1$5RPVAd$vI.6i8JejrecPYrWLybRl1
```


Keep pressing (tapping) esc key while powering on the device

```
setenv bootargs ${bootargs} "init=/bin/busybox sh"
boot
/bin/mount -a
/etc/init.d/S00devs
/etc/init.d/S01udev
/etc/init.d/S02moutfs
/etc/init.d/S80network
# We will create new file with new password hash
echo 'root:$1$5RPVAd$vI.6i8JejrecPYrWLybRl1:0:0::/root:/bin/sh' > /mnt/flash/passwd
# When we will override unused nfs mount file to override the system and execute additional file in the background
echo -e "#!/bin/sh\n/mnt/flash/hack&" > /mnt/flash/mountnfs
# We create additional file that binds our generated password with system (overrides it)
echo -e "#!/bin/sh\nsleep 15\nmount --bind /mnt/flash/passwd /etc/passwd\necho 'rooted!'" > /mnt/flash/hack
# we give execute +x permissions to these files
chmod +x /mnt/flash/mountnfs /mnt/flash/hack
reboot
```


# Backup

> Chris Walker, [2021-05-26 22:35:53]:
>
> 0x40000000 is the base address - that is RAM at 0x0 = 0x40000000
>
> on CV500, the ram base address is 0x80000000
>
> sf read 0x82000000 0x0 0x800000 literally means read 0x800000 bytes from SPI flash starting at 0x0 (in SPI flash) and write them to 0x82000000 in ram (or RAM_ADDR + 32M)
>
> on EV200, the ram base address is 0x40000000
>
> u-boot is loaded in ram at RAM_BASE_ADDR + 32K, so if you wrote to 0x40000000 instead of 0x42000000, you might accidentally overwrite your currently running uboot program - making you need to power-cycle/restart your camera
>

```javascript
setenv ipaddr 192.168.13.127
setenv serverip 192.168.13.1
sf probe 0
mw.b 0x42000000 ff 1000000
sf read 0x42000000 0x0 0x800000
tftp 0x42000000 fullflash.img 0x800000
tftpput 0x42000000 0x800000 192.168.13.1:/fullflash.img
```

# Tricks

## Pelco (ptz commands)

```bash
echo -e -n "\xFF\x01\x00\x04\x3F\x00" > /dev/ttyAMA1
```

<https://www.serialporttool.com/sptblog/?p=4830>

## Nfs mount (from original os)

```bash
mount -t nfs -o nolock 192.168.13.1:/root/cam_dev /mnt/nfs
```

## Secret urls

<http://192.168.13.127/browse/settings/Intelligent.asp>

<http://192.168.13.127/browse/settings/calibrate.htm>

<http://192.168.13.127/browse/settings/update.asp>


# Load OpenIPC

Download [this](https://github.com/OpenIPC/openipc-2.1/releases/download/latest/openipc.hi3516ev300-br.tgz) to tftp server root and extract tar xzvf archive.tgz

Boot the camera with connected uart and keep pressing esc on bootup, it should prompt uboot #hisilicon command line where you can put these commands to load rootfs and kernel properly

too boot openipc root filesystem you first need to extract it’s rootfs from squashfs image.

```javascript
unsquashfs -f -d ev300/ rootfs.squashfs.hi3516ev300
```

## Boot over tftp/nfs

```bash
# EV300
setenv totalmem 128M;setenv bootargs mem=34M console=ttyAMA0,115200 panic=20 root=/dev/ram0 ro init=/linuxrc root=/dev/nfs nfsroot=192.168.13.1:/root/cam_dev/ev300,tcp,v3 ip=192.168.13.127:192.168.13.1:192.168.13.1:255.255.255.0:camera1::off\;setenv bootcmd "";setenv ipaddr 192.168.13.127;setenv serverip 192.168.13.1;tftp 0x42000000 uImage.hi3516ev300;bootm 0x42000000

# EV200
setenv totalmem=128M;setenv bootargs mem=34M console=ttyAMA0,115200 panic=20 root=/dev/ram0 ro rdinitrd=0x82000000,5193728 init=/linuxrc root=/dev/nfs nfsroot=192.168.13.1:/root/cam_dev/ev200,tcp,v3 ip=192.168.13.127:192.168.13.1:192.168.13.1:255.255.255.0:camera1::off\;setenv bootcmd "";setenv ipaddr 192.168.13.127;setenv serverip 192.168.13.1;tftp 0x42000000 uImage.hi3516ev200;bootm 0x42000000
```


# Other materials

## Flash OpenIPC

* <https://github.com/OpenIPC/openipc-2.1/wiki/install_hisi>


## Compile firmware

```bash
git clone --depth=1 https://github.com/OpenIPC/chaos_calmer.git OpenIPC
cd OpenIPC
./Project_OpenIPC.sh update
./Project_OpenIPC.sh 16ev300_DEFAULT
```

## Golang

* <https://github.com/OpenHisiIpCam/wrt-hisicam#hardware-support>


# Thanks

Big thanks to guys from OpenIPC community for helping manage on that china nerve wrecking thing.
