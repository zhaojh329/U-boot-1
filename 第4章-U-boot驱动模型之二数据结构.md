# 数据结构

------
## 2. uclass_id
**每种uclass有对应的ID号，定义在其uclass_driver中，这个和ID号和其对应的udevice中的driver中的uclass_id一样**
```
// include/dm/uclass-id.h
enum uclass_id {
    UCLASS_ROOT = 0,
    UCLASS_DEMO,
    UCLASS_CLK,     
    UCLASS_PINCTRL,    
    UCLASS_SERIAL,   
}
```

------
## 3. uclass
### 3.1 数据结构
```
struct uclass {
    // uclass的私有数据指针
    void *priv;  
    // 对应的uclass driver
    struct uclass_driver *uc_drv;
    // 链表头，挂接它所有相关联的udevice
    struct list_head dev_head;
    // 链表节点，用于把uclass连接到uclass_root链表上
    struct list_head sibling_node;
};
```
### 3.2 生成   
有对应uclass driver并且下面挂有udevice的uclass才会被uboot自动生成  

### 3.3 存放位置  
所有生成的都会挂接到gd->uclass_root链表上  

### 3.4 获取对应的API  
```
// 根据uclass_id遍历gd->ulcass_root链表
int uclass_get(enum uclass_id key, struct uclass **ucp);
```

------
## 4. uclass_driver
### 4.1 数据结构
```
// include/dm/uclass.h，每个uclass都有对应的uclass_driver
struct uclass_driver {
    // 该uclass_driver的名字
    const char *name;
    // 对应的uclass id
    enum uclass_id id;

    /* 以下函数指针主要是调用时机的区别 */
    int (*post_bind)(struct udevice *dev); // 在udevice被绑定到该uclass之后调用
    int (*pre_unbind)(struct udevice *dev); // 在udevice被解绑出该uclass之前调用
    int (*pre_probe)(struct udevice *dev); // 在该uclass的一个udevice进行probe之前调用
    int (*post_probe)(struct udevice *dev); // 在该uclass的一个udevice进行probe之后调用
    int (*pre_remove)(struct udevice *dev);// 在该uclass的一个udevice进行remove之前调用
    int (*child_post_bind)(struct udevice *dev); // 在该uclass对应的某个udevice的某个设备被绑定到该udevice之后调用
    int (*child_pre_probe)(struct udevice *dev); // 在该uclass对应的某个udevice的某个设备进行probe之前调用
    int (*init)(struct uclass *class); // 安装该uclass的时候调用
    int (*destroy)(struct uclass *class); // 销毁该uclass的时候调用
    int priv_auto_alloc_size; // 需要为对应的uclass分配多少私有数据
    const void *ops; //操作集合
};

```
### 4.2 生成   
- 通过UCLASS\_DRIVER定义uclass\_driver
```
// 以serial-uclass为例
UCLASS_DRIVER(serial) = {
    .id        = UCLASS_SERIAL,
    .name        = "serial",
    .flags        = DM_UC_FLAG_SEQ_ALIAS,   
    .post_probe    = serial_post_probe,
    .pre_remove    = serial_pre_remove,
    .per_device_auto_alloc_size = sizeof(struct serial_dev_priv),
};
```
- UCLASS_DRIVER宏定义

```
#define UCLASS_DRIVER(__name)                       \
    ll_entry_declare(struct uclass_driver, __name, uclass)

#define ll_entry_declare(_type, _name, _list)               \
    _type _u_boot_list_2_##_list##_2_##_name __aligned(4)       \
            __attribute__((unused,              \
            section(".u_boot_list_2_"#_list"_2_"#_name)))
```
- 最终会得到如下结构体，并且存放在.u\_boot\_list\_2\_uclass\_2\_serial段中

```
struct uclass_driver  _u_boot_list_2_uclass_2_serial = {
    .id        = UCLASS_SERIAL,   // 设置对应的uclass id
    .name        = "serial",
    .flags        = DM_UC_FLAG_SEQ_ALIAS,   
    .post_probe    = serial_post_probe,
    .pre_remove    = serial_pre_remove,
    .per_device_auto_alloc_size = sizeof(struct serial_dev_priv),
}
```

### 4.3 存放位置  
通过查看u-boot.map可以得到：
```
.u_boot_list_2_uclass_1
                0x23e368e0        0x0 drivers/built-in.o
.u_boot_list_2_uclass_2_gpio
                0x23e368e0       0x48 drivers/gpio/built-in.o
                0x23e368e0                _u_boot_list_2_uclass_2_gpio  // gpio uclass driver的符号
.u_boot_list_2_uclass_2_root
                0x23e36928       0x48 drivers/built-in.o
                0x23e36928                _u_boot_list_2_uclass_2_root // root uclass drvier的符号
.u_boot_list_2_uclass_2_serial
                0x23e36970       0x48 drivers/serial/built-in.o
                0x23e36970                _u_boot_list_2_uclass_2_serial // serial uclass driver的符号
.u_boot_list_2_uclass_2_simple_bus
                0x23e369b8       0x48 drivers/built-in.o
                0x23e369b8                _u_boot_list_2_uclass_2_simple_bus
.u_boot_list_2_uclass_3
                0x23e36a00        0x0 drivers/built-in.o
                0x23e36a00                . = ALIGN (0x4)
```
- 即所有的uclass driver都会放在.u_boot_list_2_uclass_1和.u_boot_list_2_uclass_3的区间中，这个列表也称为uclass_driver table**

### 4.4 获取对应的API 
- 先获取uclass\_driver table，然后遍历获得对应的uclass\_driver
```
// 根据.u_boot_list_2_uclass_1的段地址获得uclass_driver table的地址
struct uclass_driver *uclass =
     ll_entry_start(struct uclass_driver, uclass);

// 获得uclass_driver table的长度
const int n_ents = ll_entry_count(struct uclass_driver, uclass);

// 根据uclass_id从uclass_driver table中得到相应的uclass_driver
struct uclass_driver *lists_uclass_lookup(enum uclass_id id)
```

------
## 5. udevice
### 5.1 数据结构
```
// include/dm/device.h, 描述的是设备树内容
struct udevice {
    const struct driver *driver; // 该udevice对应的driver
    const char *name; // 设备名
    void *platdata; // 该udevice的平台数据
    void *parent_platdata; // 提供给父设备使用的平台数据
    void *uclass_platdata; // 提供给所属uclass使用的平台数据
    int of_offset; // 该udevice的dtb节点偏移，代表了dtb里面的这个节点node
    ulong driver_data; // 驱动数据
    struct udevice *parent; // 父设备
    void *priv; // 私有数据的指针
    struct uclass *uclass; // 所属uclass
    void *uclass_priv; // 提供给所属uclass使用的私有数据指针
    void *parent_priv; // 提供给其父设备使用的私有数据指针
    struct list_head uclass_node; // 挂接到它所属uclass的链表上
    struct list_head child_head; // 链表头，挂接它下面的子设备
    struct list_head sibling_node; // 挂接到父设备的链表上
    uint32_t flags; // 标识
    int req_seq;  // 请求的seq
    int seq;     // 分配的seq
};
```

### 5.2 生成  
uboot解析dtb以后动态生成

### 5.3 存放位置 
1. 挂接到对应的uclass上，即挂接到uclass->dev\_head  
2. 挂接到父设备的子设备链表中， 即挂接到udevice->child\_head, 最上层的父设备是gd->dm\_root

### 5.4 获取对应的API  
- 通过uclass获取对应的udevice
> 不应该是dtb解析获得的吗？的确是通过dtb解析获得

```
int uclass_get_device(enum uclass_id id, int index, struct udevice **devp); // 通过索引从uclass中获取udevice
int uclass_get_device_by_name(enum uclass_id id, const char *name, struct udevice **devp); // 通过设备名从uclass中获取udevice
int uclass_get_device_by_seq(enum uclass_id id, int seq, struct udevice **devp);
int uclass_get_device_by_of_offset(enum uclass_id id, int node,
                   struct udevice **devp);
int uclass_resolve_seq(struct udevice *dev);
```

------
## 6. driver
### 6.1 数据结构
```
// include/dm/device.h
struct driver {
    char *name;    // 驱动名
    enum uclass_id id;  // 对应的uclass id
    const struct udevice_id *of_match;    // compatible字符串的匹配表，用于和device tree里面的设备节点匹配
    int (*bind)(struct udevice *dev);   // 用于绑定目标设备到该driver中
    int (*probe)(struct udevice *dev);   // 用于probe目标设备激活
    int (*remove)(struct udevice *dev); // 用于remove目标设备禁用
    int (*unbind)(struct udevice *dev); // 用于解绑目标设备到该driver中
    int (*ofdata_to_platdata)(struct udevice *dev); // 在probe之前，解析对应udevice的dts节点，转化成udevice的平台数据
    int (*child_post_bind)(struct udevice *dev); // 如果目标设备的一个子设备被绑定之后调用
    int (*child_pre_probe)(struct udevice *dev); // 在目标设备的一个子设备被probe之前调用
    int (*child_post_remove)(struct udevice *dev); // 在目标设备的一个子设备被remove之后调用
    int priv_auto_alloc_size; //需要分配多少空间作为其udevice的私有数据
    int platdata_auto_alloc_size; //需要分配多少空间作为其udevice的平台数据
    int per_child_auto_alloc_size;  // 对于目标设备的每个子设备需要分配多少空间作为父设备的私有数据
    int per_child_platdata_auto_alloc_size; // 对于目标设备的每个子设备需要分配多少空间作为父设备的平台数据
    const void *ops;    /* driver-specific operations */ // 操作集合的指针，提供给uclass使用，没有规定操作集的格式，由具体uclass决定
};
```

### 6.2 生成   
- 通过U\_BOOT\_DRIVER定义一个driver
```
// 以serial为例
U_BOOT_DRIVER(serial_s5p) = {
    .name    = "serial_s5p",
    .id    = UCLASS_SERIAL,
    // 定义为设备树中的compatible属性
    .of_match = s5p_serial_ids,
    .ofdata_to_platdata = s5p_serial_ofdata_to_platdata,
    .platdata_auto_alloc_size = sizeof(struct s5p_serial_platdata),
    .probe = s5p_serial_probe,
    .ops    = &s5p_serial_ops,
    .flags = DM_FLAG_PRE_RELOC,
};
```
- U\_BOOT\_DRIVER宏定义如下，生成的结构体保存在.u\_boot\_list\_2\_driver\_2\_serial\_s5p段中**

```
// U_BOOT_DRIVER宏定义
#define U_BOOT_DRIVER(__name)                        \
    ll_entry_declare(struct driver, __name, driver)

#define ll_entry_declare(_type, _name, _list)               \
    _type _u_boot_list_2_##_list##_2_##_name __aligned(4)       \
            __attribute__((unused,              \
            section(".u_boot_list_2_"#_list"_2_"#_name)))

// 生成结构体
struct driver _u_boot_list_2_driver_2_serial_s5p= {
    .name    = "serial_s5p",
    .id    = UCLASS_SERIAL,
    .of_match = s5p_serial_ids,
    .ofdata_to_platdata = s5p_serial_ofdata_to_platdata,
    .platdata_auto_alloc_size = sizeof(struct s5p_serial_platdata),
    .probe = s5p_serial_probe,
    .ops    = &s5p_serial_ops,
    .flags = DM_FLAG_PRE_RELOC,
};
```

### 6.3 存放位置  
通过查看u-boot.map可以得到：
```
 .u_boot_list_2_driver_1
                0x00000000178743a0        0x0 drivers/built-in.o
 .u_boot_list_2_driver_2_gpio_mxc
                0x00000000178744b0       0x44 drivers/gpio/built-in.o
                0x00000000178744b0                _u_boot_list_2_driver_2_gpio_mxc
 .u_boot_list_2_driver_2_gpio_regulator
                0x00000000178744f4       0x44 drivers/power/regulator/built-in.o
                0x00000000178744f4                _u_boot_list_2_driver_2_gpio_regulator
 .u_boot_list_2_driver_2_i2c_generic_chip_drv
                0x0000000017874538       0x44 drivers/i2c/built-in.o
                0x0000000017874538                _u_boot_list_2_driver_2_i2c_generic_chip_drv
 .u_boot_list_2_driver_2_i2c_mxc
                0x000000001787457c       0x44 drivers/i2c/built-in.o
                0x000000001787457c                _u_boot_list_2_driver_2_i2c_mxc
 .u_boot_list_2_driver_2_imx6_pinctrl
                0x00000000178745c0       0x44 drivers/built-in.o
                0x00000000178745c0                _u_boot_list_2_driver_2_imx6_pinctrl
......
 .u_boot_list_2_driver_3
                0x00000000178748f0        0x0 drivers/built-in.o
```
- 即所有driver都会放在.u\_boot\_list\_2\_driver\_1和.u\_boot\_list\_2\_driver\_3的区间中，这个列表也称为driver table

### 6.4 获取对应的API  
- 先获取driver table，然后遍历获得对应的driver
```
// 根据.u_boot_list_2_driver_1的段地址获得driver table的地址
struct driver *drv =
    ll_entry_start(struct driver, driver);

// 获得driver table的长度
const int n_ents = ll_entry_count(struct driver, driver);

// 根据driver name从driver table中得到相应的driver
struct driver *lists_driver_lookup_name(const char *name)
```

------
## 7. 相关API
### 7.1 uclass
```
// 从gd->uclass_root链表获取对应的uclass
// uclass是从gd->uclass_root链表中获得, 参数有uclass_id
int uclass_get(enum uclass_id key, struct uclass **ucp);
```

### 7.2 uclass_driver
```
// 根据uclass id从uclass_driver table中获取
// uclass_driver是从uclass_driver 中获得，参数是uclass_id
struct uclass_driver *lists_uclass_lookup(enum uclass_id id)
```

### 7.3 udevice
```
#define uclass_foreach_dev(pos, uc) \
    list_for_each_entry(pos, &uc->dev_head, uclass_node)

#define uclass_foreach_dev_safe(pos, next, uc)  \
    list_for_each_entry_safe(pos, next, &uc->dev_head, uclass_node)

// 通过name获取driver，调用device_bind对udevice初始化，与对应uclass、driver绑定
int device_bind_by_name(struct udevice *parent, bool pre_reloc_only,
                const struct driver_info *info, struct udevice **devp)

// 初始化udevice，与对应uclass、driver绑定
int device_bind(struct udevice *parent, const struct driver *drv,
        const char *name, void *platdata, int of_offset,
        struct udevice **devp)

// 在uclass的设备链表中绑定新的udevice
int uclass_bind_device(struct udevice *dev)
{
    uc = dev->uclass;
    list_add_tail(&dev->uclass_node, &uc->dev_head);
}

// 通过索引从uclass中获取udevice，注意，在获取的过程中就会对设备进行probe
int uclass_get_device(enum uclass_id id, int index, struct udevice **devp);
// 通过设备名从uclass中获取udevice
int uclass_get_device_by_name(enum uclass_id id, const char *name,
                  struct udevice **devp);
int uclass_get_device_by_seq(enum uclass_id id, int seq, struct udevice **devp);
int uclass_get_device_by_of_offset(enum uclass_id id, int node,
                   struct udevice **devp);
......
int uclass_resolve_seq(struct udevice *dev);
```

### 7.4 driver
```
// 根据name从driver table中获取
struct driver *lists_driver_lookup_name(const char *name)
```



