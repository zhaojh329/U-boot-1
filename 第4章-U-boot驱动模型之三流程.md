## **第4章-U-boot驱动模型之三流程**
### **3. DM的初始化**
#### **3.1 主要工作**
##### **3.1.1 初始化DM**
**a) 创建根设备root的udevice，放在gd->dm_root中**
> 根设备其实是一个虚拟设备，主要为其他设备提供一个挂载点

**b) 初始化uclass链表gd->uclass_root**

##### **3.1.2 udevice和uclass的解析**
**a) 创建udevice和uclass**   
**b) 绑定udevice和uclass**   
**c) 绑定udevice和driver**   
**d) 绑定uclass和uclass_driver**    
**e) 调用部分的driver**   

#### **3.2 入口说明**
> dts节点中的“u-boot,dm-pre-reloc”属性，当设置了这个属性时，则表示这个设备在relocate之前就需要使用

```
// 只对带有“u-boot,dm-pre-reloc”属性的节点进行解析，情况少
dm_init_and_scan(true);

// 对所有节点进行解析，重点说明
dm_init_and_scan(false);
```

##### **3.2.1 dm_init_and_scan说明**
![initf_dm](./images/initf_dm.png)

```
// driver/core/root.c
int dm_init_and_scan(bool pre_reloc_only)
{
    ret = dm_init();    // DM的初始化

    ret = dm_scan_platdata(pre_reloc_only);  // 从平台设备中解析udevice和uclass

    if (CONFIG_IS_ENABLED(OF_CONTROL)) {
        ret = dm_scan_fdt(gd->fdt_blob, pre_reloc_only); // 从dtb中解析udevice和uclass
    }

    ret = dm_scan_other(pre_reloc_only);
}
```

##### **3.2.2 dm_init**
> a) 创建根设备root的udevice，放在gd->dm_root中, 即创建设备链表头   
  b) 初始化uclass链表gd->uclass_root, 即创建uclass链表头   

 ![dm_init](./images/dm_init.png)
```
// driver/core/root.c
#define DM_ROOT_NON_CONST       (((gd_t *)gd)->dm_root) // 宏定义根设备指针gd->dm_root
#define DM_UCLASS_ROOT_NON_CONST    (((gd_t *)gd)->uclass_root) // 宏定义gd->uclass_root，uclass的链表

int dm_init(void)
{
    // 初始化uclass链表
    INIT_LIST_HEAD(&DM_UCLASS_ROOT_NON_CONST);

    // DM_ROOT_NON_CONST是指根设备udevice，root_info是表示根设备的设备信息
    // 查找和设备信息匹配的driver，创建对应的udevice，和对应uclass绑定，并把udevice放在DM_ROOT_NON_CONST链表中
    ret = device_bind_by_name(NULL, false, &root_info, &DM_ROOT_NON_CONST);

    // 对根设备执行probe操作
    ret = device_probe(DM_ROOT_NON_CONST);
}

// 找driver
int device_bind_by_name(struct udevice *parent, bool pre_reloc_only,
			const struct driver_info *info, struct udevice **devp)
{
    // 根据name找driver
	drv = lists_driver_lookup_name(info->name);  
	return device_bind_common(parent, drv, info->name,
			(void *)info->platdata, 0, -1, platdata_size, devp);
}

//  绑定
static int device_bind_common(struct udevice *parent, const struct driver *drv,
			      const char *name, void *platdata,
			      ulong driver_data, int of_offset,
			      uint of_platdata_size, struct udevice **devp)
{
    // 根据uclass_id找uclass
    ret = uclass_get(drv->id, &uc);

    // 创建一个udevice，和uclass关联
    dev->uclass = uc;
    ......

    // 给udevice一个req_seq
    fdtdec_get_alias_seq(gd->fdt_blob,
        uc->uc_drv->name, of_offset,
        &dev->req_seq)

    // 把udevice链接到uclass中，检查是否需要执行uclass_driver的操作 	
    ret = uclass_bind_device(dev);

    // driver和udevice进行绑定， 函数指针，由具体设备指定
    ret = drv->bind(dev);
}

// 绑定后激活
int device_probe(struct udevice *dev)
{
    // 根据uclass_id和req_seq，得到dev->seq
    seq = uclass_resolve_seq(dev);

    // probe前操作
    ret = uclass_pre_probe_device(dev);

    // **************************
    // 函数指针，对应实际的udevice的driver
    ret = dev->parent->driver->child_pre_probe(dev);
    ret = drv->ofdata_to_platdata(dev);
    ret = drv->probe(dev);
    // **************************

    // probe后操作
    ret = uclass_post_probe_device(dev);
}
```

##### **3.2.3 从平台设备中解析udevice和uclass——dm_scan_platdata**
![dm_scan_platdata](./images/dm_scan_platdata.png)
TBD

##### **3.2.4 dm_scan_fdt: 从dtb中解析udevice和uclass**

![dm_scan_fdt](./images/dm_scan_fdt.png)

```
// 以dtb基地址为参数
int dm_scan_fdt(const void *blob, bool pre_reloc_only)
{
    return dm_scan_fdt_node(gd->dm_root, blob, 0, pre_reloc_only);
}

// parent=gd->dm_root，表示以root设备作为父设备开始解析
// blob=gd->fdt_blob，对应dtb入口地址
// offset=0，从偏移0的节点开始扫描
int dm_scan_fdt_node(struct udevice *parent, const void *blob, int offset,
             bool pre_reloc_only)
{
    // 遍历每一个dts节点并且调用lists_bind_fdt对其进行解析
    // 获得blob设备树的offset偏移下的节点的第一个子节点
    for (offset = fdt_first_subnode(blob, offset);
        offset > 0;
        // 循环查找下一个子节点
        offset = fdt_next_subnode(blob, offset)) {
        // 节点状态disable的直接忽略
        if (!fdtdec_get_is_enabled(blob, offset))
        // 解析绑定这个节点，dm_scan_fdt的核心
        err = lists_bind_fdt(parent, blob, offset, NULL);
}

// driver/core/lists.c
// 通过blob和offset可以获得对应的设备的dts节点
int lists_bind_fdt(struct udevice *parent, const void *blob, int offset,
           struct udevice **devp)
{
    // 获取driver table地址
    struct driver *driver = ll_entry_start(struct driver, driver);
    // 获取driver table长度
    const int n_ents = ll_entry_count(struct driver, driver);

    // 遍历driver table中的所有driver
    for (entry = driver; entry != driver + n_ents; entry++) {
        // 判断driver中的compatibile字段和dts节点是否匹配
        ret = driver_check_compatible(blob, offset, entry->of_match,
                          &id);

        // 获取节点名称
        name = fdt_get_name(blob, offset, NULL);

        // 找到对应driver，创建对应udevice和uclass进行绑定
        ret = device_bind_with_driver_data(parent, entry, name,
						   id->data, offset, &dev);
    }
}

int device_bind_with_driver_data(struct udevice *parent,
				 const struct driver *drv, const char *name,
				 ulong driver_data, int of_offset,
				 struct udevice **devp)
{
    // 同上
    return device_bind_common(parent, drv, name, NULL, driver_data,
				  of_offset, 0, devp);
}

int device_bind(struct udevice *parent, const struct driver *drv,
        const char *name, void *platdata, int of_offset,
        struct udevice **devp)
{
    // 同上
    return device_bind_common(parent, drv, name, platdata, 0, of_offset, 0,
				  devp);
}
```

> 以上完成dtb的解析，udevice和uclass的创建，以及各个组成部分的绑定
**注意：这里只是绑定，即调用了driver的bind函数，但是设备还没有真正激活，也就是还没有执行设备的probe函数**


**TBD，差一个流程图**
-------
### **4. DM的probe**
#### **4.1 device_probe函数**
**a) 分配设备的私有数据**    
**b) 对父设备probe，执行probe device之前uclass需要调用的一些函数**  
**c) 调用driver的ofdata_to_platdata，将dts信息转化为设备的平台数据**   
**d) 调用driver的probe函数，执行probe device之后uclass需要调用的一些函数**  

![device_probe](./images/device_probe.png)
```
// driver/core/device.c
int device_probe(struct udevice *dev)
{
    // 同上
}
```

#### **4.2 通过uclass获取一个udevice并且probe**
```
// driver/core/uclass.c
//通过索引从uclass的设备链表中获取udevice，进行probe
int uclass_get_device(enum uclass_id id, int index, struct udevice **devp)

//通过设备名从uclass的设备链表中获取udevice，进行probe
int uclass_get_device_by_name(enum uclass_id id, const char *name,
                  struct udevice **devp)
//通过序号从uclass的设备链表中获取udevice，进行probe                  
int uclass_get_device_by_seq(enum uclass_id id, int seq, struct udevice **devp)

//通过dts节点的偏移从uclass的设备链表中获取udevice，进行probe
int uclass_get_device_by_of_offset(enum uclass_id id, int node,
                   struct udevice **devp)

//通过设备的“phandle”属性从uclass的设备链表中获取udevice，进行probe                   
int uclass_get_device_by_phandle(enum uclass_id id, struct udevice *parent,
                 const char *name, struct udevice **devp)

//从uclass的设备链表中获取第一个udevice，进行probe                 
int uclass_first_device(enum uclass_id id, struct udevice **devp)

//从uclass的设备链表中获取下一个udevice，进行probe
int uclass_next_device(struct udevice **devp)
```
**以上接口主要是获取设备的方法上有所区别，但是probe设备的方法都是一样的**  

**以uclass_get_device为例**
```
int uclass_get_device(enum uclass_id id, int index, struct udevice **devp)
{
    //通过索引从uclass的设备链表中获取对应的udevice
    ret = uclass_find_device(id, index, &dev);

    // 获得device，进行probe
    return uclass_get_device_tail(dev, ret, devp);
}

int uclass_get_device_tail(struct udevice *dev, int ret,
                  struct udevice **devp)
{
    // probe设备
    ret = device_probe(dev);
}
```
