- [Respberry Pi Specification](#respberry-pi-specification)
  - [Key Files](#key-files)
  - [Key Structs](#key-structs)
    - [dtb types](#dtb-types)
    - [Memory Arragement](#memory-arragement)
  - [Key Functions](#key-functions)
    - [入口函数](#入口函数)
    - [驱动处理函数](#驱动处理函数)
    - [架构/SoC 层函数（arch/arm/mach-bcm283x/）](#架构soc-层函数archarmmach-bcm283x)
      - [1. __内存映射配置__](#1-内存映射配置)
      - [2. __mach\_cpu\_init()__](#2-mach_cpu_init)
      - [3. __兼容性匹配表__](#3-兼容性匹配表)
    - [__Raspberrypi Specific Function__](#raspberrypi-specific-function)
      - [__dram\_init()__](#dram_init)
      - [__board\_init()__](#board_init)
      - [__misc\_init\_r()__](#misc_init_r)
      - [__board\_get\_usable\_ram\_top()__](#board_get_usable_ram_top)
      - [__board\_fdt\_blob\_setup()__](#board_fdt_blob_setup)
      - [__ft\_board\_setup()__](#ft_board_setup)
    - [do\_bootefi (Intel GRUB Only)](#do_bootefi-intel-grub-only)
  - [dtb Hanlding](#dtb-hanlding)
    - [1. __start.elf（固件）传递 fw\_dtb\_pointer__](#1-startelf固件传递-fw_dtb_pointer)
    - [2. __board\_fdt\_blob\_setup() 使用固件DTB__](#2-board_fdt_blob_setup-使用固件dtb)
    - [3. __dram\_init() 从 mailbox 获取内存信息__](#3-dram_init-从-mailbox-获取内存信息)
    - [====================== board\_f above, board\_r below ======================](#-board_f-above-board_r-below-)
    - [4. __board\_init() 识别板级型号__](#4-board_init-识别板级型号)
    - [5. __misc\_init\_r() 设置环境变量__](#5-misc_init_r-设置环境变量)
    - [6. __ft\_board\_setup() 同步固件DT属性__](#6-ft_board_setup-同步固件dt属性)
- [Compilation](#compilation)
  - [Cmds](#cmds)
- [Boot Linux](#boot-linux)
  - [Connect UART](#connect-uart)
  - [Steps](#steps)
  - [辅助命令](#辅助命令)
    - [printenv](#printenv)
    - [mmc dev 0](#mmc-dev-0)
    - [raspberry pi 典型布局](#raspberry-pi-典型布局)
    - [根文件系统挂载过程](#根文件系统挂载过程)
    - [查看文件，以及文件内容](#查看文件以及文件内容)
- [Trouble shooting](#trouble-shooting)
  - [addr2line](#addr2line)

# Respberry Pi Specification
## Key Files
- common/board_f.c
- common/board_r.c
- arch/arm/mach-bcm283x/init.c
- board/raspberrypi/rpi/rpi.c


## Key Structs
### dtb types
```
static const struct rpi_model rpi_models_new_scheme[] = {
    	[0x17] = {
		"5 Model B",
		DTB_DIR "bcm2712-rpi-5-b.dtb",
		true,
	},
	[0x18] = {
		"Compute Module 5",
		DTB_DIR "bcm2712-rpi-cm5-cm5io.dtb",
		true,
	},
	[0x19] = {
		"500",
		DTB_DIR "bcm2712-rpi-500.dtb",
		true,
	},
	[0x1A] = {
		"Compute Module 5 Lite",
		DTB_DIR "bcm2712-rpi-cm5l-cm5io.dtb",
		true,
	},
};
```

### Memory Arragement
```
高地址 +-----------------+ ← gd->ram_top (初始)
       |   未分配内存     |
       +-----------------+
       |   pram区域      | ← reserve_pram()
       +-----------------+
       |   视频缓冲区    | ← reserve_video()
       +-----------------+
       |   跟踪缓冲区    | ← reserve_trace()
       +-----------------+
       |   U-Boot镜像    | ← reserve_uboot()
       +-----------------+ ← gd->start_addr_sp (初始)
       |   malloc堆      | ← reserve_malloc()
       +-----------------+
       |   板级信息      | ← reserve_board()
       +-----------------+
       |   全局数据      | ← reserve_global_data()
       +-----------------+
       |   设备树        | ← reserve_fdt()
       +-----------------+
       |   栈空间        | ← reserve_stacks()
低地址 +-----------------+ ← 最终 gd->start_addr_sp
```

## Key Functions

### 入口函数
```
  arch/arm/cpu/armv8/start.S
  └── bl  _main
          └── arch/arm/lib/crt0_64.S
              ├── ENTRY(_main)
              └── bl board_init_f
              │      └── common/board_f.c
              │          └── setup_reloc() -> move/copy old gd (and others) to new
              └── b	board_init_r
                     └── common/board_r.c
                         └── initcall_run_r
                             └── run_main_loop
                                  └── [U_BOOT_CMD] do_bootm
                                      └── boot_os[IH_OS_LINUX]
                                          └─ do_bootm_linux()
                                              └─ boot_jump_linux()
                                                  ├─ do_nonsec_virt_switch()  // 刷新缓存，准备切换
                                                  └─ armv8_switch_to_el2()    // 实际执行 EL 切换
                                                      └─ 跳转到内核入口点
```

### 驱动处理函数
```
initf_dm
├── bootstage_start(BOOTSTAGE_ID_ACCUM_DM_F, "dm_f")
├── dm_init_and_scan(true) -> create device instance
├── dm_autoprobe() -> probe and activate devices
└── bootstage_accum(BOOTSTAGE_ID_ACCUM_DM_F)
```

### 架构/SoC 层函数（arch/arm/mach-bcm283x/）

#### 1. __内存映射配置__

```c
// arch/arm/mach-bcm283x/init.c
static struct mm_region bcm2712_mem_map[MEM_MAP_MAX_ENTRIES] = {
    {
        /* 第一个 1GB DRAM */
        .virt = 0x00000000UL,
        .phys = 0x00000000UL,
        .size = 0x40000000UL,
        .attrs = PTE_BLOCK_MEMTYPE(MT_NORMAL) | PTE_BLOCK_INNER_SHARE
    }, {
        /* AXI 总线起始（uSD 控制器所在）*/
        .virt = 0x1000000000UL,  // 64位地址空间
        .phys = 0x1000000000UL,
        .size = 0x0002000000UL,
        .attrs = PTE_BLOCK_MEMTYPE(MT_DEVICE_NGNRNE) | ...
    }, {
        /* SoC 总线 */
        .virt = 0x107c000000UL,
        .phys = 0x107c000000UL,
        .size = 0x0004000000UL,
        .attrs = PTE_BLOCK_MEMTYPE(MT_DEVICE_NGNRNE) | ...
    }
};
```

#### 2. __mach_cpu_init()__

- __功能__: SoC 特定的 CPU 初始化

- __任务__:

  1. 更新内存映射：`rpi_update_mem_map()`

  2. 从设备树获取 I/O 基地址

  3. 设置外设基地址：

     - `rpi_mbox_base` = 邮箱
     - `rpi_sdhci_base` = SDHCI 控制器
     - `rpi_wdog_base` = 看门狗（bcm2712-pm）
     - `rpi_timer_base` = 系统定时器

#### 3. __兼容性匹配表__

```c
static const struct udevice_id board_ids[] = {
    { .compatible = "brcm,bcm2712", .data = (ulong)&bcm2712_mem_map},
    // ... 其他兼容性
};
```



### __Raspberrypi Specific Function__

```
| 功能     | 文件                        | 函数           | RPi 5 特定处理           |
|---------|-----------------------------|---------------|-------------------------|
| 内存初始化 | board/raspberrypi/rpi/rpi.c | dram_init() | mailbox 查询，64位地址支持 |
| 板级识别 | board/raspberrypi/rpi/rpi.c | get_board_revision() | 识别 rev_type 0x17-0x1A |
| 设备树设置 | board/raspberrypi/rpi/rpi.c | board_fdt_blob_setup() | 使用固件传递的 DTB |
| 内存映射 | arch/arm/mach-bcm283x/init.c | bcm2712_mem_map[] | 64位外设地址映射 |
| SoC 初始化 | arch/arm/mach-bcm283x/init.c | mach_cpu_init() | 设置 bcm2712 外设基地址 |
| 环境设置 | board/raspberrypi/rpi/rpi.c | misc_init_r() | 设置 fdtfile=bcm2712-rpi-5-b.dtb |
```

#### __dram_init()__

- __功能__: 通过 VideoCore mailbox 获取 ARM 内存大小

- __实现__:

  ```c
  int dram_init(void)
  {
      ALLOC_CACHE_ALIGN_BUFFER(struct msg_get_arm_mem, msg, 1);
      // 通过 BCM2835_MBOX_PROP_CHAN 查询内存大小
      ret = bcm2835_mbox_call_prop(...);
      gd->ram_size = msg->get_arm_mem.body.resp.mem_size;
      gd->ram_size &= ~MMU_SECTION_SIZE;  // 对齐到段大小
      return 0;
  }
  ```

#### __board_init()__

- __功能__: 板级初始化，获取板级修订号，启用 USB 电源
- __调用__: `get_board_revision()` → 识别 RPi 5 (rev_type 0x17-0x1A)
- __返回__: `bcm2835_power_on_module(BCM2835_MBOX_POWER_DEVID_USB_HCD)`

#### __misc_init_r()__

- __功能__: 运行时杂项初始化

- __包含__:

  - `set_fdt_addr()`: 设置设备树地址
  - `set_fdtfile()`: 设置设备树文件名（bcm2712-rpi-5-b.dtb）
  - `set_usbethaddr()`: 从固件获取 MAC 地址
  - `set_board_info()`: 设置环境变量 board_revision, board_name 等
  - `set_serial_number()`: 获取序列号

#### __board_get_usable_ram_top()__

- __功能__: 获取可用的 RAM 顶部地址，避免与固件设备树重叠

```c
phys_addr_t board_get_usable_ram_top(phys_size_t total_size)
{
    if ((gd->ram_top - fw_dtb_pointer) > SZ_64M)
        return gd->ram_top;
    return fw_dtb_pointer & ~0xffff;
}
```

#### __board_fdt_blob_setup()__

- __功能__: 使用固件传递的设备树
- __检查__: `fdt_magic(fw_dtb_pointer) == FDT_MAGIC`

#### __ft_board_setup()__

- __功能__: 设备树设置，从固件 DT 复制属性到 U-Boot DT

- __调用__: `update_fdt_from_fw()` 复制：

  - `/model` - 精确的模型名称
  - `/memreserve` - 内存保留区域
  - `/reserved-memory/linux,cma` - CMA 内存设置
  - `/chosen/kaslr-seed` - KASLR 种子
  - 各种外设特定属性

### do_bootefi (Intel GRUB Only)
```
U-Boot 命令行
  └─ do_bootefi("0x1000000", ...) [cmd/bootefi.c:162]
      └─ efi_binary_run(image_buf, size, fdt, NULL, 0) [cmd/bootefi.c:245]
          └─ efi_binary_run_dp(image, size, fdt, NULL, 0, ...) [lib/efi_loader/efi_bootbin.c:250]
              └─ efi_run_image(source_buffer, source_size, dp_dev, dp_img) [lib/efi_loader/efi_bootbin.c:156]
                  └─ do_bootefi_exec(handle, load_options) [lib/efi_loader/efi_bootbin.c:169]
                      ├─ switch_to_non_secure_mode() [arch/arm/cpu/armv8/exception_level.c:43]
                      ├─ efi_set_watchdog(300)
                      └─ EFI_CALL(efi_start_image(...))
```


## dtb Hanlding
```
固件DTB (start.elf创建)
    ↓ 传递指针
U-Boot 接收并验证
    ↓ 作为gd->fdt_blob
U-Boot 查询补充信息（内存、型号等）
    ↓ 更新到DTB
U-Boot 同步固件属性
    ↓ 最终DTB
传递给内核
```

### 1. __start.elf（固件）传递 fw_dtb_pointer__

- __DTB 来源__: Raspberry Pi 的 GPU 固件（start.elf）在加载 U-Boot 之前已经初始化了硬件，并创建了一个包含当前硬件配置的 DTB
- __传递机制__: 固件将 DTB 的物理地址保存在 `fw_dtb_pointer` 变量中（在 `lowlevel_init.S` 中设置）
- __代码位置__:

```c
// board/raspberrypi/rpi/rpi.c
unsigned long __section(".data") fw_dtb_pointer;  // 由汇编代码设置
```

### 2. __board_fdt_blob_setup() 使用固件DTB__

- __DTB 验证__: 检查 `fw_dtb_pointer` 指向的 DTB 魔数是否为 `FDT_MAGIC`
- __DTB 选择__: 如果验证通过，将其设置为 U-Boot 的主 DTB (`gd->fdt_blob`)
- __代码实现__:

```c
int board_fdt_blob_setup(void **fdtp)
{
    if (fdt_magic(fw_dtb_pointer) != FDT_MAGIC)
        return -ENXIO;
    
    *fdtp = (void *)fw_dtb_pointer;  // 使用固件的 DTB
    return 0;
}
```

### 3. __dram_init() 从 mailbox 获取内存信息__

- __DTB 补充__: 虽然固件 DTB 可能包含内存信息，但 RPi 通过 mailbox 向 VideoCore 查询更准确的内存大小
- __数据同步__: 获取的内存信息最终会更新到 DTB 的 `/memory` 节点
- __为什么需要__: 固件 DTB 中的内存信息可能不完整或需要调整（如对齐到 MMU 段大小）

### ====================== board_f above, board_r below ======================

### 4. __board_init() 识别板级型号__

- __DTB 查询__: 通过 mailbox 获取板子修订号，确定具体型号（RPi 5、RPi 4 等）
- __DTB 影响__: 根据型号选择合适的设备树文件（如 `bcm2712-rpi-5-b.dtb`）
- __环境变量设置__: 将型号信息保存到环境变量，影响后续 DTB 加载

### 5. __misc_init_r() 设置环境变量__

- __DTB 相关环境变量__:

  - `fdtfile`: 设置 DTB 文件名（`bcm2712-rpi-5-b.dtb`）
  - `fdt_addr`: 设置 DTB 在内存中的地址
  - `serial#`: 从 DTB 或 mailbox 获取序列号

- __目的__: 为后续内核加载准备正确的 DTB 文件

### 6. __ft_board_setup() 同步固件DT属性__

- __DTB 增强__: 将固件 DTB 中的重要属性复制到 U-Boot 的 DTB 中

- __同步的属性包括__:

  - `/model`: 精确的硬件型号
  - `/memreserve`: 内存保留区域
  - `/reserved-memory/linux,cma/size`: CMA 内存大小
  - `/chosen/kaslr-seed`: 内核地址随机化种子
  - 各种外设的 `dma-ranges`、`reg` 等属性

- __代码关键部分__:

```c
void update_fdt_from_fw(void *fdt, void *fw_fdt)
{
    // 复制固件DTB中的属性到U-Boot DTB
    copy_property(fdt, fw_fdt, "/", "model");
    copy_property(fdt, fw_fdt, "/", "memreserve");
    copy_property(fdt, fw_fdt, "/reserved-memory/linux,cma", "size");
    // ... 更多属性
}
```

# Compilation
## Cmds
```
 make menuconfig
 make rpi_arm64_defconfig
 make CROSS_COMPILE=aarch64-linux-gnu- -j8
 bear -- make CROSS_COMPILE=aarch64-linux-gnu- -j8
```

# Boot Linux
## Connect UART
```
option-1: screen
启动：sudo screen /dev/ttyACM0 115200
退出：Ctrl+A, k -> yes -> quit

option-2: minicom
启动：minicom raspberrypi5 (raspberrypi5 见配置部分说明)
退出：Ctrl+A, q -> yes -> quit


配置：
minicom -s
进入后看到主菜单：

text
            [Configuration]
            +-----[configuration]------+
            | Filenames and paths      |
            | File transfer protocols  |
            | Serial port setup        |
            | Modem and dialing        |
            | Screen and keyboard      |
            | Save setup as dfl        |
            | Save setup as..          |
            | Exit                     |
            | Exit from Minicom        |
            +--------------------------+
1. Serial port setup —— 串口参数设置
这是最常用的配置项，按 S 进入，会显示类似下面的界面：

text
   +-----------------------------------------------------------------------+
   | A -    Serial Device      : /dev/ttyUSB0                              |
   | B - Lockfile Location     : /var/lock                                 |
   | C -   Callin Program      :                                           |
   | D -  Callout Program      :                                           |
   | E -    Bps/Par/Bits       : 115200 8N1                                |
   | F - Hardware Flow Control : No                                        |
   | G - Software Flow Control : No                                        |
   |                                                                       |
   |    Change which setting?                                              |
   +-----------------------------------------------------------------------+
各子项含义：

A - Serial Device：串口设备文件路径。Linux 下通常为 /dev/ttyUSB0、/dev/ttyS0 等，根据实际连接的串口设备填写。

B - Lockfile Location：锁文件存放目录，默认为 /var/lock，用于防止多个程序同时占用同一串口。

C - Callin Program / D - Callout Program：通常为空，用于指定拨入/拨出时要执行的程序，串口调试无需设置。

E - Bps/Par/Bits：波特率、校验位和数据位。格式如 115200 8N1 表示：

波特率 115200

8 数据位

N 无奇偶校验（N=None, O=Odd, E=Even）

1 停止位
按 E 可以修改这些参数，minicom 会提示逐项选择。

F - Hardware Flow Control：硬件流控（RTS/CTS），一般设为 No，除非你的设备需要硬件握手。

G - Software Flow Control：软件流控（XON/XOFF），通常也设为 No，否则可能会干扰二进制数据传输。

修改方法：按对应字母（如 A）即可编辑该行。

2. Modem and dialing —— 调制解调器与拨号设置
此菜单用于配置 modem 相关的拨号参数，在直接连接串口设备（如 Raspberry Pi）时通常无需修改，保持默认即可。常用子项包括：

A - Init string：初始化 modem 的 AT 命令，默认如 AT。

B - Reset string：重置 modem 的命令。

拨号前缀/后缀：定义拨号时发送的指令。

拨号脚本：自动拨号的脚本等。

若只是作为串口终端使用，可忽略此项。

3. Screen and keyboard —— 屏幕与键盘行为
影响终端显示和键盘输入的处理方式，常用选项：

A - Command key is：定义 minicom 的转义键，默认是 Meta-A（通常指 Ctrl+A）。

B - Backspace key sends：Backspace 键发送的字符，一般选 DEL 或 BS，需与连接的设备匹配。

C - Add linefeed：是否在收到回车时自动添加换行，通常设为 No。

D - Local echo：本地回显，若连接的设备不支持回显，可设为 Yes 以便看到自己键入的内容。默认 No。

E - Emulation：终端模拟类型，常用 ANSI 或 VT100。

其他：如屏幕行列数、自动换行等。

4. Filenames and paths —— 文件名与路径
设置日志保存、上传下载的默认目录等：

A - Download directory：从串口接收文件时保存的路径。

B - Upload directory：发送文件时查找的路径。

C - Script directory：存放运行脚本的目录。

D - Capture file：默认的捕获文件名（用于记录串口输出）。

5. File transfer protocols —— 文件传输协议
配置上传/下载所使用的协议及其参数，如 xmodem、ymodem、zmodem 等。一般保持默认即可。

6. Save setup as dfl —— 保存为默认配置
将当前所有设置保存到 ~/.minirc.dfl 文件中，以后直接运行 minicom 就会加载此配置。

7. Save setup as.. —— 另存配置
将当前设置保存到自定义名称的文件（raspberrypi5），之后可以通过 minicom raspi 启动对应配置。必须是 sudo minicom -s 才可以保存，因为其文件路径必须为： /etc/minirc.raspberrypi5.

cat /etc/minirc.raspberrypi5 
# Machine-generated file - use "minicom -s" to change parameters.
pu port             /dev/ttyACM0


8. Exit —— 退出配置，进入终端
保存配置后选择此项，minicom 会应用当前配置并进入串口通信界面。

9. Exit from Minicom —— 完全退出 minicom
不进入终端，直接退出 minicom 程序。
```

## Steps
```
fatload mmc 0:1 0x00200000 kernel_2712.img
fatload mmc 0:1 0x05600000 bcm2712-rpi-5-b.dtb
setenv bootargs "console=tty1,115200 root=/dev/mmcblk0p2 rootfstype=ext4 rootwait rw"
booti 0x00200000 - 0x05600000
```

## 辅助命令
### printenv
```
U-Boot> printenv
arch=arm
baudrate=115200
board=rpi
board_name=5 Model B
board_rev=0x17
board_rev_scheme=1
board_revision=0xD04170
boot_targets=mmc usb pxe dhcp
bootcmd=bootflow scan
bootdelay=2
bootm_size=0x20000000
cpu=armv8
dhcpuboot=usb start; dhcp u-boot.uimg; bootm
ethaddr=2c:cf:67:32:00:90
fdt_addr=2efec500
fdt_addr_r=0x05600000
fdtcontroladdr=3f731370
fdtfile=broadcom/bcm2712-rpi-5-b.dtb
kernel_addr_r=0x00080000
kernel_comp_addr_r=0x02000000
kernel_comp_size=0x03400000
loadaddr=0x1000000
preboot=pci enum; usb start;
pxefile_addr_r=0x05500000
ramdisk_addr_r=0x05700000
scriptaddr=0x05400000
serial#=fa2868239d327b29
soc=bcm283x
stderr=serial,vidconsole
stdin=serial,usbkbd
stdout=serial,vidconsole
usb_ignorelist=0x1050:*,
usbethaddr=2c:cf:67:32:00:90
vendor=raspberrypi

Environment size: 768/16380 bytes
```

### mmc dev 0
```
U-Boot> mmc dev 0
switch to partitions #0, OK
mmc0 is current device


U-Boot> mmc part -> 查看当前 dev 0 下的所有分区 partition
```

### raspberry pi 典型布局
```
MMC/SD卡：
├── 分区1 (FAT32): /dev/mmcblk0p1
│   ├── bootcode.bin
│   ├── start*.elf
│   ├── kernel*.img
│   ├── *.dtb
│   └── config.txt
│
└── 分区2 (ext4): /dev/mmcblk0p2  ← 根文件系统
    ├── /
    ├── /bin, /sbin, /usr, /etc, /home, /var, /lib
    └── 完整的 Linux 发行版

```

### 根文件系统挂载过程
```
// 简化的内核代码流程
mount_root() {
    // 1. 打开块设备
    dev = name_to_dev_t("/dev/mmcblk0p2");
    
    // 2. 读取超级块识别文件系统类型
    sb = read_super(dev, "ext4");
    
    // 3. 挂载为根文件系统
    mount_block_root("/dev/mmcblk0p2", "ext4");
    
    // 4. 切换到根文件系统
    chroot("/");
    chdir("/");
}
```

### 查看文件，以及文件内容
```
fatls mmc 0:1 -> 查看 mmc 0:1 fat 文件系统下所有文件名
ext4ls mmc 0:1 -> 查看 mmc 0:2 ext4 文件系统下的所有目录，文件
cat mmc 0:1 config.txt -> 查看文件 config.txt 的内容
```

# Trouble shooting
## addr2line
- aarch64-linux-gnu-addr2line -e u-boot 0x0000000080203b54