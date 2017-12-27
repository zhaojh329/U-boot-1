## **第6章-U-boot启动内核**
> 整理自[uboot启动kernel篇（一）——Legacy-uImage & FIT-uImage](http://blog.csdn.net/ooonebook/article/details/53495002)

### **1.uImage**
> 内核编译后会生成Image或者压缩的zImage，但这两种格式缺少足够信息供u-boot进行load、jump或验证操作；因此u-boot提供mkimage工具，用以生成u-boot可以识别的格式，称为uImage

#### **1.1 Legacy-uImage类型**
**在kernel镜像的基础上，附加64Byte的信息在头部供u-boot使用**
> **优点是**实现简单，长度较小；  
**缺点是**使用麻烦，需要在启动kernel命令中额外添加fdt、ramdisk等信息

#### **1.2 FIT-uImage类型**
**类似FDT模式，把kernel，fdt，ramdisk等镜像打包到一个image file文件中，外加一些额外信息；u-boot获得这个image以后，就可以得到其中各镜像的具体信息和内容**
>  **优点是**使用方便，兼容性好  
**缺点是**长度较大，需要额外信息较长  

### **2.Legacy-uImage**
#### **2.1 配置宏，使能**
```
// 自动生成autoconf.mk时会自动使能，不需要额外配置
CONFIG_IMAGE_FORMAT_LEGACY=y
```

#### **2.2 制作和使用**
**a) 制作**   
```
// mkimage工具目录
xingyanl@yocto:~$ ls -lrt uboot/tools/mkimage
-rwxrwxr-x 1 xingyanl xingyanl 164650 Dec 26 15:14 uboot/tools/mkimage
xingyanl@yocto:~$

xingyanl@yocto:~$ mkimage -A arm -O linux -C none -T kernel -a 0x20008000 -e 0x20008040 -n Linux_Image -d zImage uImage

参数含义：
Usage: mkimage -l image
          -l ==> list image header information
       mkimage [-x] -A arch -O os -T type -C comp -a addr -e ep -n name -d data_file[:data_file...] image
          -A ==> set architecture to 'arch'  // 体系
          -O ==> set operating system to 'os' // 操作系统
          -T ==> set image type to 'type' // 镜像类型
          -C ==> set compression type 'comp' // 压缩类型
          -a ==> set load address to 'addr' (hex) // 加载地址
          -e ==> set entry point to 'ep' (hex) // 入口地址
          -n ==> set image name to 'name' // 镜像名称，注意不能超过32B
          -d ==> use image data from 'datafile' // 输入文件
          -x ==> set XIP (execute in place)
```

**b) 使用**   
```
// 使用bootm命令下载Legacy-uImage到指定位置
bootm Legacy-uImage加载地址 ramdisk加载地址 dtb加载地址
```

#### **2.3 和zImage的区别**
```
// 可得uImage比zImage多了64Byte数据
-rw-rw-rw-  1 hlos hlos 1560120 Dec  5 14:46 uImage
-rwxrwxrwx  1 hlos hlos 1560056 Nov 30 15:42 zImage*

// 通过od命令查看这64Byte的数据，并且加在了zImage的头部
xingyanl@yocto:~$ od -tx1 -tc -Ax -N64 uImage
000000  27  05  19  56  5a  f3  f7  8e  58  45  0d  3d  00  17  cd  f8
         ' 005 031   V   Z 363 367 216   X   E  \r   =  \0 027 315 370
000010  20  00  80  00  20  00  80  40  e2  4b  43  b6  05  02  02  00
            \0 200  \0      \0 200   @ 342   K   C 266 005 002 002  \0
000020  4c  69  6e  75  78  5f  49  6d  61  67  65  00  00  00  00  00
         L   i   n   u   x   _   I   m   a   g   e  \0  \0  \0  \0  \0
000030  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00  00
        \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0  \0
```

#### **2.4 格式头说明**

```
// u-boot中使用image_header表示legacy-uImage的头部
typedef struct image_header {
    __be32      ih_magic;     // 魔数头27 05 19 56，用来校验是否是Legacy-uImage
    __be32      ih_hcrc;      // 头部的CRC校验值
    __be32      ih_time;      // 镜像创建的时间戳
    __be32      ih_size;      // 镜像数据长度
    __be32      ih_load;      // 加载地址
    __be32      ih_ep;        // 入口地址
    __be32      ih_dcrc;      // 镜像的CRC校验
    uint8_t     ih_os;        // 操作系统类型
    uint8_t     ih_arch;      // 体系
    uint8_t     ih_type;      // 镜像类型
    uint8_t     ih_comp;       // 压缩类型
    uint8_t     ih_name[IH_NMLEN];   // 镜像名
} image_header_t;
#define IH_NMLEN        32 // 镜像名最大长度
```

### **3. FIT-uImage**
**FIT：**flattened image tree，类似于**FDT(flattened device tree)**的一种实现机制，它使用一定语法和格式把需要的镜像(如kernel，dtb等)组合到一起生成一个image文件
> 主要使用四个组件：  
a) **its(image source file)文件：**类似于dts，负责描述image的信息，需要自行构造     
b) **itb文件: **最终得到的image文件，类似于dtb文件，u-boot直接对其识别和解析  
c) **mkimage：**负责dtc的角色，通过解析its生成itb
d) **image data file：**实际使用到的镜像文件  
**mkimage将its及对应的image data file打包成一个itb，即uboot可识别的image file(FIT-uImage); 把文件下载到memory中，使用bootm命令就可以执行了**


#### **3.1 配置宏，使能**
```
// 需要配置使能
CONFIG_FIT=y
```

#### **3.2 制作**
##### **3.2.1 its文件制作**
```
// 制作its文件, 其中需要描述被打包的镜像, 主要是kernel镜像, dtb文件和ramdisk镜像
// 示例 [u-boot FIT image介绍](http://www.wowotech.net/u-boot/fit_image_overview.html)
/dts-v1/;
/ {
    description = "U-Boot uImage source file for X project";
    #address-cells = <1>;
    images {
        kernel@tiny210 {
            description = "Unify(TODO) Linux kernel for project-x";
            data = /incbin/("/home/hlos/code/xys/temp/project-x/build/out/linux/arch/arm/boot/zImage");
            type = "kernel";
            arch = "arm";
            os = "linux";
            compression = "none";
            load = <0x20008000>;
            entry = <0x20008000>;
        };
        fdt@tiny210 {
            description = "Flattened Device Tree blob for project-x";
            data = /incbin/("/home/hlos/code/xys/temp/project-x/build/out/linux/arch/arm/boot/dts/s5pv210-tiny210.dtb");
            type = "flat_dt";
            arch = "arm";
            compression = "none";
        };
        ramdisk@tiny210 {
            description = "Ramdisk for project-x";
            data = /incbin/("/home/hlos/code/xys/temp/project-x/build/out/rootfs/initramfs.gz");
            type = "ramdisk";
            arch = "arm";
            os = "linux";
            compression = "gzip";
        };
    };
    configurations {
        default = "conf@tiny210";
        conf@tiny210 {
            description = "Boot Linux kernel with FDT blob";
            kernel = "kernel@tiny210";
            fdt = "fdt@tiny210";
            ramdisk = "ramdisk@tiny210";
        };
    };
};
```
> **可以有多个kernel节点或者fdt节点等, 提高兼容性; 并且可以对kernel、fdt和ramdisk进行多种组合生成多种configurations, 使FIT-uImage可以兼容于多种板子，而无需重新进行编译烧写**

##### **3.2.2 itb文件制作**
```
// 生成itb文件
xingyanl@yocto:~$ mkimage -f ${UIMAGE_ITS_FILE} ${UIMAGE_ITB_FILE}
```

#### **3.3 使用**
```
// 下载FIT-uImage到指定地址
xingyanl@yocto:~$ bootm "指定地址"
```
### **总结：FIT-uImage的格式类似于DTB，uboot通过解析FIT-uImage中的configurations节点，根据节点选择出需要的kernel、dtb、rootfs**
