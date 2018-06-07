[TOC]
## 打补丁编译

1. 解压 `tar xjf linux-2.6.22.6.tar.bz2 `
2. 打补丁,cat下补丁文件知道需要忽略第一个/  `patch -p1 < linux-2.6.22.6_jz2440.patch`
3. 打包下生成的文件` tar cjvf linux2.6.22_ok.tar.bz2 linux-2.6.22.6`
4. 配置（3种方法）

   1. make menuconfig 每一项都配置
   2. 使用默认配置后修改`find -name "*defconfig*"`,会生出.config，然后 `make menuconfig`,
   3. **厂家提供配置**，直接复制为名为.config，再执行 `make menuconfig`
5. 编译 `make uImage`

### 配置文件相关

默认配置有很多，一般在`./arch/arm/configs/`

```
book@book-desktop:/work/Test/Linux2.6.22/linux-2.6.22.6$ cd ./arch/arm/configs/
book@book-desktop:/work/Test/Linux2.6.22/linux-2.6.22.6/arch/arm/configs$ ls
```

里面有一个s3c2410_defconfig,我们就可以在根目录下执行 `make s3c2410_defconfig`,他所有的结果保存在.config文件中，再 执行`make menuconfig`,就会出来一个图形界面的配置。

如果报错，则

```
book@book-vm:~/work/linux-2.6.22.6$ make s3c2410_defconfig
Makefile:416: *** mixed implicit and normal rules: deprecated syntax
Makefile:1449: *** mixed implicit and normal rules: deprecated syntax
make: *** No rule to make target 's3c2410_defconfig'。 停止。

原因：是由于我的系统的make工具太新，make的旧版规则已经无法兼容新版。

1在makefile中将416行代码
config %config: scripts_basic outputmakefile FORCE
改为
%config: scripts_basic outputmakefile FORCE
2在makefile中将1449行代码
/ %/: prepare scripts FORCE
改为
%/: prepare scripts FORCE
```

我们直接使用韦东山提供配置好的config

```
cp config_ok .config
```

然后`make menuconfig`使用`/`进行搜索，`help`查看帮助然后使用 `make uImage`可以生成内核文件，最后使用uboot启动，menu中的`k`,利用dnw 烧录到nandflash.我们查看下uboot中k对应的具体命令，查看uboot源代码， 在cmd_menu.c，然后输入b启动

```
strcpy(cmd_buf, "usbslave 1 0x30000000; nand erase kernel; nand write.jffs2 0x30000000 kernel $(filesize)");
```
## 配置解析

我们看下`.config`,取其中一行 `CONFIG_DM9000=y`来分析。搜索下 `grep "CONFIG_DM9000" * -nwR`

可以看到源码中的一些宏定义,makefile也有，include 下也有

```

arch/arm/mach-at91/board-sam9261ek.c:79:#if defined(CONFIG_DM9000)
arch/arm/mach-at91/board-sam9261ek.c:134:#endif /* CONFIG_DM9000 */
arch/arm/plat-s3c24xx/common-smdk.c:46:#if defined(CONFIG_DM9000) || defined(CONFIG_DM9000_MODULE)
arch/arm/plat-s3c24xx/common-smdk.c:162:#if defined(CONFIG_DM9000) || defined(CONFIG_DM9000_MODULE)
arch/arm/plat-s3c24xx/common-smdk.c:200:#endif /* CONFIG_DM9000 */
arch/arm/plat-s3c24xx/common-smdk.c:250:#if defined(CONFIG_DM9000) || defined(CONFIG_DM9000_MODULE)

config_ok:599:CONFIG_DM9000=y
drivers/net/Makefile:197:obj-$(CONFIG_DM9000) += dm9dev9000c.o
drivers/net/Makefile:198:#obj-$(CONFIG_DM9000) += dm9000.o
drivers/net/Makefile:199:#obj-$(CONFIG_DM9000) += dm9ks.o

include/config/auto.conf:144:CONFIG_DM9000=y
include/linux/autoconf.h:145:#define CONFIG_DM9000 1
```

C语言的宏来源于源文件(.c.h)所以一般是来源于`include/linux/autoconf.h:145:#define CONFIG_DM9000 1`

这个文件应该是自动生成的，来源于`.config`,查看下`autoconf.h` ，里面有`#define CONFIG_DM9000 1`,宏定义为1，不论是否定义为y or n,那么他是在子目录的makefile中定义的，DM9000在`linux-2.6.22.6/drivers/net`的Makefile中定义`obj-$(CONFIG_DM9000) += dm9dev9000c.o` ，也就是说`CONFIG_DM9000=y`的情况下 ，编译到内核，如果配置成=m 编译成模块加载。

综上查看下 `auto.conf` ---`include/config$ vim auto.conf`

```
CONFIG_DM9000=y
```

`autoconf.h `

```
#define CONFIG_DM9000 1
```

子目录的makefie----`linux-2.6.22.6/drivers/net`

```
obj-$(CONFIG_DM9000) += dm9dev9000c.o
```

auto.conf是.config生成配置下去的，应该被顶层的makefile包含。

**--->.config** 回去创建`autoconf.h`和`auto.conf`,然后在子目录的makefile使用。

autoconf.h ----c语言使用

auto.conf ----子目录下的makefile使用

## 分析Makefile

-  ls Documentation/kbuild/makefiles.txt 有具体的说明
- 第一个入口文件
- 链接脚本

### 子目录下的Makefile

`obj-y += xxxx.o`  编译到内核

 `obj-m += xxxx.o`编译成可加载模块.ko

如果配置内核的时候，不是 y,或者m 则 obj - +=xxx.o 不被处理

### 架构下面的makefile

```
arch/arm/Makefile
```

### 顶层下面的Makefile

包含了架构下的Makefile

```
413 include $(srctree)/arch/$(ARCH)/Makefile
.config 生成的
444 -include include/config/auto.conf
```

生成命令 `make uImage`,查看下`arch\arm\Makefile`的依赖 vmlinux(真正的内核)

```
zImage Image xipImage bootpImage uImage: vmlinux
	$(Q)$(MAKE) $(build)=$(boot) MACHINE=$(MACHINE) $(boot)/$@
```

vmlinux的依赖在顶层的Makefile

```
vmlinux: $(vmlinux-lds) $(vmlinux-init) $(vmlinux-main) $(kallsyms.o) FORCE
-----------------------顶层Makefile--------------↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓
vmlinux-init := $(head-y) $(init-y)
vmlinux-main := $(core-y) $(libs-y) $(drivers-y) $(net-y)
vmlinux-all  := $(vmlinux-init) $(vmlinux-main)
vmlinux-lds  := arch/$(ARCH)/kernel/vmlinux.lds

-----------------------arch\arm\Makefile------------------↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓
head-y		:= arch/arm/kernel/head$(MMUEXT).o arch/arm/kernel/init_task.o
----MMUEXT没有定义，所以就是  arch/arm/kernel/head.o    arch/arm/kernel/init_task.o

-----------------------顶层Makefile--------------↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓
init-y		:= init/
init-y		:= $(patsubst %/, %/built-in.o, $(init-y))
----  %/代表 init/ == init/built-in.o  表示 init/下所有文件，编译为built-in.o

-----------------------顶层Makefile--------------↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓
vmlinux-main := $(core-y) $(libs-y) $(drivers-y) $(net-y)
----- 
core-y		:= usr/
core-y		:= $(patsubst %/, %/built-in.o, $(core-y))
core-y		+= kernel/ mm/ fs/ ipc/ security/ crypto/ block/
===usr/ 下所有的文件编译为 built-in.o
libs-y		:= lib/
libs-y		:= $(libs-y1) $(libs-y2)
libs-y1		:= $(patsubst %/, %/lib.a, $(libs-y))
libs-y2		:= $(patsubst %/, %/built-in.o, $(libs-y))
==== lib/ lib.a   built-in.o
drivers-y	:= drivers/ sound/
drivers-y	:= $(patsubst %/, %/built-in.o, $(drivers-y))
==== drivers下涉及文件编译为/built-in.o sound下涉及文件编译为/built-in.o
net-y		:= net/
net-y		:= $(patsubst %/, %/built-in.o, $(net-y))
==== net下涉及文件编译为/built-in.o 
```

查看下如何组织材料的

```
rm vmlinux #先删除
make uImage V=1  # V=1 表示更加详细显示命令
```

```
arm-linux-ld -EL  -p --no-undefined -X -o vmlinux 
-T arch/arm/kernel/vmlinux.lds 
arch/arm/kernel/head.o arch/arm/kernel/init_task.o  init/built-in.o --start-group  usr/built-in.o  arch/arm/kernel/built-in.o  arch/arm/mm/built-in.o  arch/arm/common/built-in.o  arch/arm/mach-s3c2410/built-in.o  arch/arm/mach-s3c2400/built-in.o  arch/arm/mach-s3c2412/built-in.o  arch/arm/mach-s3c2440/built-in.o  arch/arm/mach-s3c2442/built-in.o  arch/arm/mach-s3c2443/built-in.o  arch/arm/nwfpe/built-in.o  arch/arm/plat-s3c24xx/built-in.o  kernel/built-in.o  mm/built-in.o  fs/built-in.o  ipc/built-in.o  security/built-in.o  crypto/built-in.o  block/built-in.o  arch/arm/lib/lib.a  lib/lib.a  arch/arm/lib/built-in.o  lib/built-in.o  drivers/built-in.o  sound/built-in.o  net/built-in.o --end-group .tmp_kallsyms2.o

```

可以看到链接脚本为`arch/arm/kernel/vmlinux.lds `

第一个文件为`arch/arm/kernel/head.o`

lds简单入口分析，文件的顺序在外部由.o的先后顺序决定，具体的段由链接脚本决定

```
. = (0xc0000000) + 0x00008000; //虚拟地址

 .text.head : {
  _stext = .;
  _sinittext = .;
  *(.text.head)
 }
 
 .init : { /* Init code and data		*/
   *(.init.text)
  _einittext = .;
  __proc_info_begin = .;
   *(.proc.info.init)
  __proc_info_end = .;
  。。。。。。。。。。
 
 
```



## 建立si工程

1. 添加所有

2. 移除所有Arch,添加Arch/arm 下除了 Mach_xxx 开头的，Mach_xxx 表示机器型号,添加2410，2440

   剔除 Plat_xxx,加入plat-s3c24xx

   ```
   boot
   common
   configs
   kernel
   lib
   
   mach-s3c2410
   mach-s3c2440
   plat-s3c24xx
   
   mm
   nwfpe
   oprofile
   tools
   vfp
   ```

3. 移除include目录，移除Asm-xxx架构相关的，只加入asm-arm,加入asm-arm顶层文件，**去除两个勾**

   加入架构相关的,加入include下非asm-xxx的

   ```
   arch-s3c2410
   hardware
   mach
   plat-s3c24xx
   ```

## 启动简析

   1. 处理uboot传入的参数
   2. 挂接根文件系统
   3. 最终目的：运行应用程序（在根文件系统上）

## head.s

   我们发现在`arch\arm\boot\compressed`也存在一个head.S的文件，有些内核编译出来比较大，他会以压缩的形式存在，这个文件就是讲压缩的文件解压，在这里不做分析。我们的入口为arch\arm\kernel\head.S

###  处理流程

   1. 判断是否支持CPU

   2. 判断是否支持单板（机器ID） `#define MACH_TYPE_S3C2440              362`,由于uboot调用格式为`theKernel (0, bd->bi_arch_number, bd->bi_boot_params);`所以机器ID在R1寄存器

   3. 创建页表，加载地址在lds中为`. = (0xc0000000) + 0x00008000;`,并不是实际地址，需要启动MMU

   4. 在__switch_data中，跳转C函数`b	start_kernel` 文件`head-common.S`中，是内核第一个C函数

   5. uboot的参数在`start_kernel`这个c函数中处理的

      `setup_arch(&command_line);setup_command_line(command_line);`

      **单板判断**

      ```
      	.type	__lookup_machine_type, %function
      __lookup_machine_type:
      
      	adr	r3, 3b
      	@r3=3b的地址,3b就是前面3的位置，物理地址，uboot还没启动mmu
      	@3:	.long	.
      	@.long	__arch_info_begin
      	@.long	__arch_info_end
      	
      	ldmia	r3, {r4, r5, r6}
      	@传值
      	@r4='.' 表示一个虚拟地址,r5=__arch_info_begin,r6=__arch_info_end
      	
      	sub	r3, r3, r4			@ get offset between virt&phys
      	@得到虚拟地址，物理地址的偏差
      	
      	add	r5, r5, r3			@ convert virt addresses to
      	add	r6, r6, r3			@ physical address space
      	@得到r5,r6 真正的物理地址 
      	@__arch_info_begin  __arch_info_end 在链接脚本中定义
      	@arch\arm\kernel\lds  虚拟地址
      	@  __arch_info_begin = .;
        @ *(.arch.info.init)
        @__arch_info_end = .;
        /*
        在 arch.h 中有定义  定义某个结构体的段属性
        #define MACHINE_START(_type,_name)			\
        static const struct machine_desc __mach_desc_##_type	\
        __used							\
        __attribute__((__section__(".arch.info.init"))) = {	\
        .nr		= MACH_TYPE_##_type,		\
        .name		= _name,
      
        #define MACHINE_END				\
       };
       
       有如下应用
      MACHINE_START(S3C2440, "SMDK2440")
      	/* Maintainer: Ben Dooks <ben@fluff.org> */
      	.phys_io	= S3C2410_PA_UART,
      	.io_pg_offst	= (((u32)S3C24XX_VA_UART) >> 18) & 0xfffc,
      	.boot_params	= S3C2410_SDRAM_PA + 0x100,
      
      	.init_irq	= s3c24xx_init_irq,
      	.map_io		= smdk2440_map_io,
      	.init_machine	= smdk2440_machine_init,
      	.timer		= &s3c24xx_timer,
      MACHINE_END
      
      
      
      展开下这个宏，定义 machine_desc的一个结构体设置段属性
        static const struct machine_desc __mach_desc_S3C2440	
        __used							
        __attribute__((__section__(".arch.info.init"))) = {	
        .nr		= MACH_TYPE_S3C2440,		
        .name		= SMDK2440,
        .phys_io	= S3C2410_PA_UART,
      	.io_pg_offst	= (((u32)S3C24XX_VA_UART) >> 18) & 0xfffc,
      	.boot_params	= S3C2410_SDRAM_PA + 0x100,
      
      	.init_irq	= s3c24xx_init_irq,
      	.map_io		= smdk2440_map_io,
      	.init_machine	= smdk2440_machine_init,
      	.timer		= &s3c24xx_timer,
       };
       
       
       查看下machine_desc 这个结构体内容,可以发现支持多少单板，就有多少这个宏的使用
       struct machine_desc {
      	/*
      	 * Note! The first four elements are used
      	 * by assembler code in head-armv.S
      	 */
      	unsigned int		nr;		/* architecture number	*/
      	unsigned int		phys_io;	/* start of physical io	*/
      	unsigned int		io_pg_offst;	/* byte offset for io 
      						 * page tabe entry	*/
      
      	const char		*name;		/* architecture name	*/
      	unsigned long		boot_params;	/* tagged list		*/
      
      	unsigned int		video_start;	/* start of video RAM	*/
      	unsigned int		video_end;	/* end of video RAM	*/
      
      	unsigned int		reserve_lp0 :1;	/* never has lp0	*/
      	unsigned int		reserve_lp1 :1;	/* never has lp1	*/
      	unsigned int		reserve_lp2 :1;	/* never has lp2	*/
      	unsigned int		soft_reboot :1;	/* soft reboot		*/
      	void			(*fixup)(struct machine_desc *,
      					 struct tag *, char **,
      					 struct meminfo *);
      	void			(*map_io)(void);/* IO mapping function	*/
      	void			(*init_irq)(void);
      	struct sys_timer	*timer;		/* system tick timer	*/
      	void			(*init_machine)(void);
      };
      
        */
      @逐个比较机器ID	
      1:	ldr	r3, [r5, #MACHINFO_TYPE]	@ get machine type
      	teq	r3, r1				@ matches loader number?
      	beq	2f				@ found
      	add	r5, r5, #SIZEOF_MACHINE_DESC	@ next machine_desc
      	cmp	r5, r6
      	blo	1b
      	mov	r5, #0				@ unknown machine
      2:	mov	pc, lr
      
      ```

## start_kernel

### 概览

```
start_kernel
	setup_arch			//解析uboot传入的参数
	setup_command_line
	parse_args
		do_early_param
			从__setup_start 中调用early函数
	unknown_bootoption
		obsolete_checksetup
			从__setup_start 中调用非early函数 段属性
	rest_init
		kernel_init
			prepare_namespace
				mount_root	//根文件系统，
               init_post	// 执行应用程序
```

### setup_arch

```
if (mdesc->boot_params)
	tags = phys_to_virt(mdesc->boot_params);
	
//在上面定义机器id结构体的时候，.boot_params	= S3C2410_SDRAM_PA + 0x100,=0x30000100
#define S3C2410_CS6 (0x30000000)
#define S3C2410_SDRAM_PA    (S3C2410_CS6)
//我们在uboot的时候存储参数的地址也是这个
board_init -----gd->bd->bi_boot_params = 0x30000100;
然后开始处理tags


  static const struct machine_desc __mach_desc_S3C2440	
  __used							
  __attribute__((__section__(".arch.info.init"))) = {	
  .nr		= MACH_TYPE_S3C2440,		
  .name		= SMDK2440,
  .phys_io	= S3C2410_PA_UART,
	.io_pg_offst	= (((u32)S3C24XX_VA_UART) >> 18) & 0xfffc,
	.boot_params	= S3C2410_SDRAM_PA + 0x100,

	.init_irq	= s3c24xx_init_irq,
	.map_io		= smdk2440_map_io,
	.init_machine	= smdk2440_machine_init,
	.timer		= &s3c24xx_timer,
 };

```

在parse_tags(tags); 处理具体的tag

### setup_command_line(command_line);

如果uboot没有传递命令行参数，会使用默认的命令行,否则进行处理

### rest_init

创建一个内核进程`kernel_thread(kernel_init, NULL, CLONE_FS | CLONE_SIGHAND);`在kernel_init中调用`prepare_namespace`->`mount_root();`会挂载根文件系统,然后init_post(); 执行应用程序

## 命令行参数

`bootargs=noinitrd root=/dev/mtdblock3 init=/linuxrc console=ttySAC0`

根文件系统`root=/dev/mtdblock3`,从函数`prepare_namespace`为入口点，

```
static int __init root_dev_setup(char *line)
{
	strlcpy(saved_root_name, line, sizeof(saved_root_name));
	return 1;
}
__setup("root=", root_dev_setup);

#define __setup(str, fn)					\
	__setup_param(str, fn, fn, 0)

//定义了一些结构体，里面有函数，有一些段属性，
#define __setup_param(str, unique_id, fn, early)			\
	static char __setup_str_##unique_id[] __initdata = str;	\
	static struct obs_kernel_param __setup_##unique_id	\
		__attribute_used__				\
		__attribute__((__section__(".init.setup")))	\
		__attribute__((aligned((sizeof(long)))))	\
		= { __setup_str_##unique_id, fn, early }

```

通过`__setup("root=", root_dev_setup);` 去分析命令`root=/dev/mtdblock3`, 找到`root_dev_setup`函数并且调用他去保存后面的定义`/dev/mtdblock3`到变量`saved_root_name`,根据宏，查看下lds,搜索下__setup_start，这里面是一些函数

```
  __setup_start = .;
   *(.init.setup)
  __setup_end = .;

```

具体的分区信我们可以先启动分区，发现有个bootloader分区，代码搜索在common-smdk.c中

```
static struct mtd_partition smdk_default_nand_part[] = {
	[0] = {
        .name   = "bootloader",
        .size   = 0x00040000,
		.offset	= 0,
	},
	[1] = {
        .name   = "params",
        .offset = MTDPART_OFS_APPEND,
        .size   = 0x00020000,
	},
	[2] = {
        .name   = "kernel",
        .offset = MTDPART_OFS_APPEND,
        .size   = 0x00200000,
	},
	[3] = {
        .name   = "root",
        .offset = MTDPART_OFS_APPEND,
        .size   = MTDPART_SIZ_FULL,
	}
};

```





