## **第3章-U-boot命令行**
### **1. 基本概念**
所谓的**命令行模式**是指u-boot初始化完成以后，等待终端输入命令和处理命令的模式
> 通过宏定义CONFIG_CMDLINE使能; 对应命令要打开对应的宏定义，如：要支持bootm命令，则需要打开CONFIG_CMD_BOOTM宏，还有几个相关的TBD


-------
### **2. 数据结构**
#### **2.1 基本概念**
U-boot把所有命令的数据结构放在一个表(命令表)中，其中的每一项代表一个命令，类型是**cmd_tbl_t**   
**数据结构：**
```
struct cmd_tbl_s {
    char    *name;       // 命令名字，对应于执行命令的字符串      
    int     maxargs;     // 命令支持最大参数
    int     repeatable;  // 是否需要重复，也就是空回车键是否会不停执行
    int     (*cmd)(struct cmd_tbl_s *, int, int, char * const []);  // 命令处理函数
    char    *usage;    // 帮助(short)
    char    *help;     // 帮助(long)，通过CONFIG_SYS_LONGHELP使能
    // TBD，根据宏定义使能
	int		(*complete)(int argc, char * const argv[], char last_char, int maxv, char *cmdv[]);
};

typedef struct cmd_tbl_s    cmd_tbl_t;

```
**以bootm命令为例：**
```
// bootm对应数据结构符号: _u_boot_list_2_cmd_2_bootm，起始地址: 0x17881c3c
17881c3c <_u_boot_list_2_cmd_2_bootm>:
17881c3c:	17863203 	strne	r3, [r6, r3, lsl #4]
17881c40:	00000020 	andeq	r0, r0, r0, lsr #32
17881c44:	00000001 	andeq	r0, r0, r1
17881c48:	178044bc 			; <UNDEFINED> instruction: 0x178044bc
17881c4c:	1786b1d8 			; <UNDEFINED> instruction: 0x1786b1d8
17881c50:	17872324 	strne	r2, [r7, r4, lsr #6]
17881c54:	00000000 	andeq	r0, r0, r0
```

#### **2.2 u-boot.map符号表 **
u-boot.map符号表的定义**TBD**  
**cmd相关部分**
```
// 命令表定义在_cmd_1和_cmd_3的位置中, 每个占28个字节， 和cmd_tbl_t结构的大小是一致
.u_boot_list    0x00000000178739d0     0x1450
 *(SORT(.u_boot_list*))
 .u_boot_list_2_cmd_1   // 链接器生成
                0x0000000017873a00        0x0 cmd/built-in.o
 .u_boot_list_2_cmd_1
                0x0000000017873a00        0x0 common/built-in.o
 .u_boot_list_2_cmd_2_base
                0x0000000017873a00       0x1c cmd/built-in.o
                0x0000000017873a00                _u_boot_list_2_cmd_2_base
 .u_boot_list_2_cmd_2_bdinfo
                0x0000000017873a1c       0x1c cmd/built-in.o
                0x0000000017873a1c                _u_boot_list_2_cmd_2_bdinfo
 .u_boot_list_2_cmd_2_bmode
                0x0000000017873a38       0x1c arch/arm/imx-common/built-in.o
                0x0000000017873a38                _u_boot_list_2_cmd_2_bmode
 .u_boot_list_2_cmd_2_boot
                0x0000000017873a54       0x1c cmd/built-in.o
                0x0000000017873a54                _u_boot_list_2_cmd_2_boot
 .u_boot_list_2_cmd_2_bootd
                0x0000000017873a70       0x1c cmd/built-in.o
                0x0000000017873a70                _u_boot_list_2_cmd_2_bootd
......
 .u_boot_list_2_cmd_3
                0x00000000178743a0        0x0 cmd/built-in.o
 .u_boot_list_2_cmd_3  // 链接器生成
                0x00000000178743a0        0x0 common/built-in.o

```

#### **2.3 定义一个命令 **
**以bootm为例：**
```
U_BOOT_CMD(
	bootm,	CONFIG_SYS_MAXARGS,	1,	do_bootm,
	"boot application image from memory", bootm_help_text
);
// 命令处理函数
int do_bootm(cmd_tbl_t *cmdtp, int flag, int argc, char * const argv[])
```

**U_BOOT_CMD实现**
```
// 接口
#define U_BOOT_CMD(_name, _maxargs, _rep, _cmd, _usage, _help)

// 实现在include/common.h文件中
#define U_BOOT_CMD(_name, _maxargs, _rep, _cmd, _usage, _help)      \
    U_BOOT_CMD_COMPLETE(_name, _maxargs, _rep, _cmd, _usage, _help, NULL)

#define U_BOOT_CMD_COMPLETE(_name, _maxargs, _rep, _cmd, _usage, _help, _comp) \
    ll_entry_declare(cmd_tbl_t, _name, cmd) =            \
        U_BOOT_CMD_MKENT_COMPLETE(_name, _maxargs, _rep, _cmd,    \
                        _usage, _help, _comp);

#define U_BOOT_CMD_MKENT_COMPLETE(_name, _maxargs, _rep, _cmd,      \
                _usage, _help, _comp)           \
        { #_name, _maxargs, _rep, _cmd, _usage,         \
            _CMD_HELP(_help) _CMD_COMPLETE(_comp) }

#define ll_entry_declare(_type, _name, _list)                \
    _type _u_boot_list_2_##_list##_2_##_name __aligned(4)        \
            __attribute__((unused,                \
            section(".u_boot_list_2_"#_list"_2_"#_name)))

    ll_entry_declare(cmd_tbl_t, _name, cmd)

// 最终得到数据结构如下，并保持在.u_boot_list_2_cmd_2_bootm段中：
cmd_tbl_t _u_boot_list_2_cmd_2_bootm=
{
    _name=bootm，
    _maxargs=CONFIG_SYS_MAXARGS，
    _rep=1，
    _cmd=do_bootm，
    _usage="boot application image from memory"，
    _help=bootm_help_text，
    _comp=NULL，
}
```

-------
### **3. 命令行模式**
命令行模式有两种：  
1. 通用模式：获取数据，解析和处理命令
2. hush模式：命令的解析和处理使用busybox中的hush工具，对应代码为hush.c  

u-boot执行完所有初始化程序后，调用**run_main_loop**进入主循环，接着进入命令行模式
```
// common/board_r.c文件
static int run_main_loop(void)
{
    for (;;)
        // 进入了主循环，autoboot也在主循环内实现
        main_loop();
    return 0;
}

// common/main.c文件
void main_loop(void)
{
    // cli的初始化，主要是hush模式下的初始化
    cli_init();

    // 命令行模式循环模式，不返回
    cli_loop();
    panic("No CLI available");
}


// common/cli.c文件
void cli_loop(void)
{
    // hush命令模式
    parse_file_outer();

    // 通用命令行模式
    cli_simple_loop();
}

```

#### **3.1 通用模式**
**通用模式处理流程**
```
void cli_simple_loop(void)
{
    for (;;) {
        // 打印CONFIG_SYS_PROMPT，然后从串口读取一行作为命令，存储在console_buffer中
        len = cli_readline(CONFIG_SYS_PROMPT);

        if (len > 0)
            // 从console_buffer中copy到lastcommand中
            strlcpy(lastcommand, console_buffer,
                CONFIG_SYS_CBSIZE + 1);
        else if (len == 0)
            // 得到换行符，设置重复标识，后续重复执行上条命令
            flag |= CMD_FLAG_REPEAT;

        if (len == -1)
            // 直接打印中断
            puts("<INTERRUPT>\n");
        else
            // 执行lastcommand中的命令
            rc = run_command_repeatable(lastcommand, flag);

        if (rc <= 0) {
            // 命令执行出错时，清空lastcommand，防止下次再执行命令
            /* invalid command or not repeatable, forget it */
            lastcommand[0] = 0;
        }
    }
}

int run_command_repeatable(const char *cmd, int flag)
{
    return cli_simple_run_command(cmd, flag);
}

int cli_simple_run_command(const char *cmd, int flag)
{
        // 把lastcommand中保存的内容转化成argv和argc格式
        argc = cli_simple_parse_line(finaltoken, argv);

        // 调用cmd_process处理命令
        if (cmd_process(flag, argc, argv, &repeatable, NULL))
}

```

#### **3.2 hush模式(**TBD**)**

-------

### **4. 命令执行流程**
#### **4.1 对以上接收到命令进行处理**
```
// common/command.c文件
enum command_ret_t cmd_process(int flag, int argc, char * const argv[],
                   int *repeatable, ulong *ticks)
{
    // 根据命令argv[0]，获取它对应的表项cmd_tbl_t
    cmdtp = find_cmd(argv[0]);

    // 执行命令表项中的命令，返回0表示成功
    rc = cmd_call(cmdtp, flag, argc, argv);
}

// 获取命令表
cmd_tbl_t *find_cmd(const char *cmd)
{
    // 获取命令表的地址，start表示指向命令表的指针
    cmd_tbl_t *start = ll_entry_start(cmd_tbl_t, cmd);

    // 获取命令表的长度
    const int len = ll_entry_count(cmd_tbl_t, cmd);

    // 查找和cmd匹配的表项，返回给调用者。
    return find_cmd_tbl(cmd, start, len);
}

```

#### **4.2 获取命令表地址和长度**
```
// include/linker_lists.h文件

// _list == cmd，则获取.u_boot_list_2_cmd_1的起始地址
#define ll_entry_start(_type, _list)                    \
({                                  \
    static char start[0] __aligned(4) __attribute__((unused,    \
        section(".u_boot_list_2_"#_list"_1")));         \
    (_type *)&start;                        \
})

//  _list == cmd，则获取.u_boot_list_2_cmd_3的结束地址
#define ll_entry_end(_type, _list)                  \
({                                  \
    static char end[0] __aligned(4) __attribute__((unused,      \
        section(".u_boot_list_2_"#_list"_3")));         \
    (_type *)&end;                          \
})

// _list == cmd，计算命令表的长度
#define ll_entry_count(_type, _list)                    \
    ({                              \
        _type *start = ll_entry_start(_type, _list);        \
        _type *end = ll_entry_end(_type, _list);        \
        unsigned int _ll_result = end - start;          \
        _ll_result;                     \
    })

// 查找匹配表项common/command.c文件, 参数table和table_len可通过上面计算得到
cmd_tbl_t *find_cmd_tbl(const char *cmd, cmd_tbl_t *table, int table_len)
{
    // 查找table中的每一个cmd_tbl_t
    for (cmdtp = table; cmdtp != table + table_len; cmdtp++) {
        if (strncmp(cmd, cmdtp->name, len) == 0) {
            if (len == strlen(cmdtp->name))
                // 命令字符串和表项中的name完全匹配，且长度相等，直接返回
                return cmdtp;

            // 若命令字符串和表项中的name的前面部分匹配的话（我们称为部分匹配），则暂时保存下来，并记录有几个这样的表项
            // 目的是支持自动补全的功能
            cmdtp_temp = cmdtp;
            n_found++;
        }
    }
    if (n_found == 1) {  
        // 如果部分匹配的表项是唯一，则将这个表项返回
        // 目的是支持自动补全的功能  
        return cmdtp_temp;
    }
}
```

#### **4.3 执行命令**
```
// 执行对应命令表项中的命令
static int cmd_call(cmd_tbl_t *cmdtp, int flag, int argc, char * const argv[])
{
    // 执行命令表项cmd_tbl_t中的cmd命令处理函数，返回0代表成功
    result = (cmdtp->cmd)(cmdtp, flag, argc, argv);
}

```
