# 基础

------
## 1. 概念

### 1.1 AHB/APB  
- AHB主要用于高性能模块(如CPU, DMA和DSP等)之间的连接，是SOC的片上系统总线     
- APB主要用于低带宽的周围外设之间的连接，如UART等

### 1.2 BIOS基本输入输出系统  
- 固化在计算机内主板上一个ROM芯片上的程序，保存计算机最重要的基本输入输出的程序，如系统设置程序，开机自检程序和系统自启动程序；提供最底层，最直接的硬件设置和控制

------
## 2. 命令
### 2.1 go  
- 跳转到指定地址执行

### 2.2 help
```
=> help
?       - alias for 'help'
base    - print or set address offset
bdinfo  - print Board Info structure
bmode   - sd2|sd3|emmc|normal|usb|sata|ecspi1:0|ecspi1:1|ecspi1:2|ecspi1:3|esdhc1|esdhc2|esdhc3|esdhc4 [noreset]
boot    - boot default, i.e., run 'bootcmd'
bootd   - boot default, i.e., run 'bootcmd'
bootelf - Boot from an ELF image in memory
bootm   - boot application image from memory
bootp   - boot image via network using BOOTP/TFTP protocol
bootvx  - Boot vxWorks from an ELF image
bootz   - boot Linux zImage image from memory
clocks  - display clocks
clrlogo - fill the boot logo area with black
cmp     - memory compare
coninfo - print console devices and information
cp      - memory copy
cpu     - Multiprocessor CPU boot manipulation and release
crc32   - checksum calculation
dcache  - enable or disable data cache
dfu     - Device Firmware Upgrade
dhcp    - boot image via network using DHCP/TFTP protocol
dm      - Driver model low level access
echo    - echo args to console
editenv - edit environment variable
env     - environment handling commands
erase   - erase FLASH memory
exit    - exit script
ext2load- load binary file from a Ext2 filesystem
ext2ls  - list files in a directory (default /)
ext4load- load binary file from a Ext4 filesystem
ext4ls  - list files in a directory (default /)
ext4size- determine a file's size
ext4write- create a file in the root directory
false   - do nothing, unsuccessfully
fatinfo - print information about filesystem
fatload - load binary file from a dos filesystem
fatls   - list files in a directory (default /)
fatsize - determine a file's size
fdt     - flattened device tree utility commands
flinfo  - print FLASH memory information
fstype  - Look up a filesystem type
fuse    - Fuse sub-system
go      - start application at address 'addr'
gpio    - query and control gpio pins
help    - print command description/usage
i2c     - I2C sub-system
icache  - enable or disable instruction cache
iminfo  - print header information for application image
imxtract- extract a part of a multi-image
itest   - return true/false on integer compare
load    - load binary file from a filesystem
loadb   - load binary file over serial line (kermit mode)
loads   - load S-Record file over serial line
loadx   - load binary file over serial line (xmodem mode)
loady   - load binary file over serial line (ymodem mode)
loop    - infinite loop on address range
ls      - list files in a directory (default /)
md      - memory display
mdio    - MDIO utility commands
mii     - MII utility commands
mm      - memory modify (auto-incrementing address)
mmc     - MMC sub system
mmcinfo - display MMC info
mtest   - simple RAM read/write test
mw      - memory write (fill)
nfs     - boot image via network using NFS protocol
nm      - memory modify (constant address)
ping    - send ICMP ECHO_REQUEST to network host
pmic    - PMIC
printenv- print environment variables
protect - enable or disable FLASH write protection
reset   - Perform RESET of the CPU
run     - run commands in an environment variable
save    - save file to a filesystem
saveenv - save environment variables to persistent storage
setenv  - set environment variables
setexpr - set environment variable as the result of eval expression
sf      - SPI flash sub-system
showvar - print local hushshell variables
size    - determine a file's size
sleep   - delay execution for some time
source  - run script from memory
test    - minimal test like /bin/sh
tftpboot- boot image via network using TFTP protocol
true    - do nothing, successfully
ums     - Use the UMS [USB Mass Storage]
usb     - USB sub-system
usbboot - boot from USB device
version - print monitor, compiler and linker version
```

### 2.3 u-boot环境变量
```
Normal Boot
Hit any key to stop autoboot:  0
=> print
baudrate=115200
board_name=SABRESD
board_rev=MX6Q
boot_fdt=try
bootcmd=run findfdt;mmc dev ${mmcdev};if mmc rescan; then if run loadbootscript; then run bootscript; else if run loadimage; then run mmcboot; else run netboot; fi; fi; else run netboot; fi
bootcmd_mfg=run mfgtool_args;bootz ${loadaddr} ${initrd_addr} ${fdt_addr};
bootdelay=3
bootscript=echo Running bootscript from mmc ...; source
console=ttymxc0
dfu_alt_info=spl raw 0x400
dfu_alt_info_img=u-boot raw 0x10000
dfu_alt_info_spl=spl raw 0x400
dfuspi=dfu 0 sf 0:0:10000000:0
emmcdev=2
epdc_waveform=epdc_splash.bin
ethact=FEC
ethprime=FEC
fdt_addr=0x18000000
fdt_file=undefined
fdt_high=0xffffffff
findfdt=if test $fdt_file = undefined; then if test $board_name = SABREAUTO && test $board_rev = MX6QP; then setenv fdt_file imx6qp-sabreauto.dtb; fi; if test $board_name = SABREAUTO && test $board_rev = MX6Q; then setenv fdt_file imx6q-sabreauto.dtb; fi; if test $board_name = SABREAUTO && test $board_rev = MX6DL; then setenv fdt_file imx6dl-sabreauto.dtb; fi; if test $board_name = SABRESD && test $board_rev = MX6QP; then setenv fdt_file imx6qp-sabresd.dtb; fi; if test $board_name = SABRESD && test $board_rev = MX6Q; then setenv fdt_file imx6q-sabresd.dtb; fi; if test $board_name = SABRESD && test $board_rev = MX6DL; then setenv fdt_file imx6dl-sabresd.dtb; fi; if test $fdt_file = undefined; then echo WARNING: Could not determine dtb to use; fi; fi;
image=zImage
initrd_addr=0x12C00000
initrd_high=0xffffffff
ip_dyn=yes
loadaddr=0x12000000
loadbootscript=fatload mmc ${mmcdev}:${mmcpart} ${loadaddr} ${script};
loadfdt=fatload mmc ${mmcdev}:${mmcpart} ${fdt_addr} ${fdt_file}
loadimage=fatload mmc ${mmcdev}:${mmcpart} ${loadaddr} ${image}
mfgtool_args=setenv bootargs console=ttymxc0,115200 rdinit=/linuxrc g_mass_storage.stall=0 g_mass_storage.removable=1 g_mass_storage.file=/fat g_mass_storage.ro=1 g_mass_storage.idVendor=0x066F g_mass_storage.idProduct=0x37FF g_mass_storage.iSerialNumber="" enable_wait_mode=off
mmcargs=setenv bootargs console=${console},${baudrate} ${smp} root=${mmcroot}
mmcautodetect=yes
mmcboot=echo Booting from mmc ...; run mmcargs; if test ${boot_fdt} = yes || test ${boot_fdt} = try; then if run loadfdt; then bootz ${loadaddr} - ${fdt_addr}; else if test ${boot_fdt} = try; then bootz; else echo WARN: Cannot load the DT; fi; fi; else bootz; fi;
mmcdev=2
mmcpart=1
mmcroot=/dev/mmcblk3p2 rootwait rw
netargs=setenv bootargs console=${console},${baudrate} ${smp} root=/dev/nfs ip=dhcp nfsroot=${serverip}:${nfsroot},v3,tcp
netboot=echo Booting from net ...; run netargs; if test ${ip_dyn} = yes; then setenv get_cmd dhcp; else setenv get_cmd tftp; fi; ${get_cmd} ${image}; if test ${boot_fdt} = yes || test ${boot_fdt} = try; then if ${get_cmd} ${fdt_addr} ${fdt_file}; then bootz ${loadaddr} - ${fdt_addr}; else if test ${boot_fdt} = try; then bootz; else echo WARN: Cannot load the DT; fi; fi; else bootz; fi;
script=boot.scr
update_emmc_firmware=if test ${ip_dyn} = yes; then setenv get_cmd dhcp; else setenv get_cmd tftp; fi; if ${get_cmd} ${update_sd_firmware_filename}; then if mmc dev ${emmcdev} 1; then setexpr fw_sz ${filesize} / 0x200; setexpr fw_sz ${fw_sz} + 1; mmc write ${loadaddr} 0x2 ${fw_sz}; fi; fi
update_sd_firmware=if test ${ip_dyn} = yes; then setenv get_cmd dhcp; else setenv get_cmd tftp; fi; if mmc dev ${mmcdev}; then if ${get_cmd} ${update_sd_firmware_filename}; then setexpr fw_sz ${filesize} / 0x200; setexpr fw_sz ${fw_sz} + 1; mmc write ${loadaddr} 0x2 ${fw_sz}; fi; fi

Environment size: 3587/8188 bytes
=> 
```

