## **第7章-U-boot移植步骤**
> 参考自**[链接](http://www.imx6rex.com/open-rex/software/yocto-uboot-how-to-add-support-for-a-custom-board/)**

### **1. 复制现有板子目录，创建新板子**
```
// 板级目录
xingyanl@yocto:uboot$ cd board/freescale/
// 创建新板子
xingyanl@yocto:uboot$ cp -R mx6sabresd mx6qsensorgw
```
### **2. 修改相关文件**

#### **2.1 修改Makefile文件**
```
// 修改mx6qsensorgw目录下得到Makefile文件
xingyanl@yocto:mx6qsensorgw$ vi Makefile
xingyanl@yocto:mx6qsensorgw$ cat Makefile
obj-y  := mx6qsensorgw.o  // 更新内容
```

#### **2.2 重命名.c文件**
```
// 更改板级c文件
xingyanl@yocto:mx6qsensorgw$ cp  mx6sabresd.c  mx6qsensorgw.c
```

#### **2.3 修改Kconfig文件**
**a) 修改/board/freescale/mx6qsensorgw目录下的Kconfig**
```
xingyanl@yocto:mx6qsensorgw$ vi Kconfig
xingyanl@yocto:mx6qsensorgw$ cat Kconfig
if TARGET_MX6QSENSORGW  // 新板子名字

config SYS_BOARD
        default "mx6qsensorgw"  // 新板子名字

config SYS_VENDOR
        default "freescale"

config SYS_CONFIG_NAME
        default "mx6qsensorgw"  // 新板子名字

endif

```
**b) 修改/arch/arm/cpu/armv7/mx6目录下的Kconfig**   
```
// 添加如下内容
xingyanl@yocto:mx6$ vi Kconfig
......
// 添加新板类型
config TARGET_MX6QSENSORGW
        bool "Support mx6qsensorgw"
        select BOARD_LATE_INIT
        select SUPPORT_SPL
        select DM
        select DM_THERMAL
        select BOARD_EARLY_INIT_F
......
// source新板子文件Kconfig文件
source "board/freescale/mx6qsensorgw/Kconfig"
......
```

#### **2.4 更新include/configs/xxx.h文件**
```
xingyanl@yocto:uboot$ cd include/configs
// copy mx6sabresd.h文件
xingyanl@yocto:configs$ cp mx6sabresd.h mx6qsensorgw.h

// 修改#include "mx6sabre_common.h"为 #include "mx6qsensorgw_common.h"
xingyanl@yocto:configs$ vi mx6qsensorgw.h
#include "mx6qsensorgw_common.h"

// copy mx6sabre_common.h文件
xingyanl@yocto:configs$ cp mx6sabre_common.h mx6qsensorgw_common.h

// 一些宏定义
CONFIG_LOADADDR         // ZImage内核会load到这个地址引导
CONFIG_SYS_MALLOC_LEN   // Heap内存大小
CONFIG_STACKSIZE        // stack的大小
CONFIG_NR_DRAM_BANKS    // DDR banks的数量
PHYS_SDRAM_SIZE         // DDR的大小，以MB为单位
PHYS_SDRAM             // DDR的物理地址
fdt_file               // 配置宏定义

"#define CONFIG_DEFAULT_FDT_FILE <customer>.dtb"

// 或者直接修改
"fdt_file=<customer>.dtb"

// Config文件对U-Boot很重要，它决定了u-boot.bin的大小，功能和性能
```

### **3. 为新板创建配置文件**

```
xingyanl@yocto:uboot$ cd configs/
xingyanl@yocto:configs$ cp mx6qsabresd_defconfig mx6qsensorgw_defconfig
xingyanl@yocto:configs$ vi mx6qsensorgw_defconfig
// 替换TARGET
# CONFIG_TARGET_MX6SABRESD=y  // 删除
CONFIG_TARGET_MX6QSENSORGW=y // 添加

// 更新cfg文件目录，和内存相关配置文件
# CONFIG_SYS_EXTRA_OPTIONS="IMX_CONFIG=board/freescale/mx6sabresd/mx6q_4x_mt41j128.cfg,MX6Q"
CONFIG_SYS_EXTRA_OPTIONS="IMX_CONFIG=board/freescale/mx6qsensorgw/mx6q_4x_mt41j128.cfg,MX6Q"

// 更新设备树文件
#CONFIG_DEFAULT_DEVICE_TREE="imx6q-sabresd"
CONFIG_DEFAULT_DEVICE_TREE="imx6q-sensorgw"
```

### **4. 为新板子指定设备树文件**
```
// 添加设备树文件
xingyanl@yocto:uboot$ cd arch/arm/dts/
xingyanl@yocto:dts$ cp imx6q-sabresd.dts imx6q-sensorgw.dts
```

### **5. 创建u-boot的build脚本**
**5.1 创建u-boot编译脚本**
```
xingyanl@yocto:uboot$ vi u-boot.sh
#!/bin/bash
export ARCH=arm
# 指定交叉编译器目录---重要
export CROSS_COMPILE=/opt/fsl-imx-x11/4.1.15-2.0.0/sysroots/x86_64-pokysdk-linux/usr/bin/arm-poky-linux-gnueabi/arm-poky-linux-gnueabi-
make distclean;
make mx6qsensorgw_defconfig
make                                                                                                            
```
**5.2 编译u-boot**
```
// 添加权限
xingyanl@yocto:uboot$ chmod a+x u-boot.sh

// 编译
xingyanl@yocto:uboot$ ./u-boot.sh
```
