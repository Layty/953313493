## 内核使用init_post 启动第一个应用程序

```
if (sys_open((const char __user *) "/dev/console", O_RDWR, 0) < 0)
	printk(KERN_WARNING "Warning: unable to open an initial console.\n");
(void) sys_dup(0);
(void) sys_dup(0);
```

`sys_dup(0);`是复制的意思，具体的还不清楚。`/dev/console`在这里是串口

这里三个代表了标准输入，标准输出，标准错误。

通过`run_init_process`来启动命令行`init=/linuxrc`

下面的`execute_command`执行的时候，会先看下uboot是否传递了`init=`开头的命令行参数，如果有就执行

或者没有的话执行下一个，如过有一个`run_init_process`成功，则不会执行下一个，一般是死循环在里面

```
	if (execute_command) {
		run_init_process(execute_command);
		printk(KERN_WARNING "Failed to execute %s.  Attempting "
					"defaults...\n", execute_command);
	}

	run_init_process("/sbin/init");
	run_init_process("/etc/init");
	run_init_process("/bin/init");
	run_init_process("/bin/sh");
```

命令行参数相关代码如下：init_setup是一个函数，明显的`_setup("init=", init_setup);`构造一个有具体属性的一个结构，包含了这个函数

```
static int __init init_setup(char *str)
{
	unsigned int i;

	execute_command = str;
	/*
	 * In case LILO is going to boot us with default command line,
	 * it prepends "auto" before the whole cmdline which makes
	 * the shell think it should execute a script with such name.
	 * So we ignore all arguments entered _before_ init=... [MJ]
	 */
	for (i = 1; i < MAX_INIT_ARGS; i++)
		argv_init[i] = NULL;
	return 1;
}
__setup("init=", init_setup);
```

在这里，我们定义了`init=/linuxrc`，会执行`run_init_process(/linuxrc)`,一去不复返，不会返回

### 测试

1. 使用oflash 烧录uboot到nandflash

2. 烧录uimage，擦出root `nand erase root`

3. 输入`menu` 返回菜单，输入b启动内核

4. 发现提示错误，这个和代码中的提示是一致的init_post (){...}

   ```
   VFS: Mounted root (yaffs filesystem).       
   挂接了根文件系统，但是flash是空的，默认识别为yaffs
   Freeing init memory: 140K
   Warning: unable to open an initial console.
   flash是空的，无法启动应用程序
   Failed to execute /linuxrc.  Attempting defaults...
   错误指示--命令行
   Kernel panic - not syncing: No init found.  Try passing init= option to kernel.


   ```

5. 输入y，下载文件系统 fs_mini.yaffs2

6. 输入b启动

7. 输入ps看下启动的应用程序

   ```
   # ps
     PID  Uid        VSZ Stat Command
       1 0          3092 S   init
       2 0               SW< [kthreadd]
       3 0               SWN [ksoftirqd/0]
       4 0               SW< [watchdog/0]
       5 0               SW< [events/0]
       6 0               SW< [khelper]
      55 0               SW< [kblockd/0]
      56 0               SW< [ksuspend_usbd]
      59 0               SW< [khubd]
      61 0               SW< [kseriod]
      73 0               SW  [pdflush]
      74 0               SW  [pdflush]
      75 0               SW< [kswapd0]
      76 0               SW< [aio/0]
     710 0               SW< [mtdblockd]
     745 0               SW< [kmmcd]
     767 0          3096 S   -sh
     769 0          3096 R   ps
   ```


## init进程之busybox

linux 以类似`run_init_process("/sbin/init");`的形式调用命令,这个init一般由busybox提供

源码：busybox-1.7.0.tar.bz2

嵌入式中实际上shell命令是指向busybox的一个链接,使用ls -l命令查看下，也可以直接使用类似命令 busybox ls

```
# ls -l /bin/ls
lrwxrwxrwx    1 1000     1000            7 Jan  6  2010 /bin/ls -> busybox

# busybox ls
bin         lib         mnt         sbin        usr
dev         linuxrc     proc        sys
etc         lost+found  root        tmp
```

其实，/sbin/init也是到busybox的链接

```
run_init_process("/sbin/init");
run_init_process("/etc/init");
run_init_process("/bin/init");
run_init_process("/bin/sh");

# ls -l /sbin/init
lrwxrwxrwx    1 1000     1000           14 Jan  6  2010 /sbin/init -> ../bin/busybox
```

### 源码分析

linux 以类似`run_init_process("/sbin/init");`的形式调用命令，没有传递参数

一个命令就有xx.c,比如ls.c,cp.c,我们这里执行init命令，则是有init.c,有函数`init_main`，他需要做到以下：

1. 配置文件读取
2. 解析配置文件
3. 执行用户程序

```
busybox
  init_main
    parse_inittab  linux调用没有传入参数
      file = fopen(INITTAB, "r");//打开配置文件"/etc/inittab"
      打开说明文件inittab 发现格式如下
      Format for each entry: <id>:<runlevels>:<action>:<process>
      最终使用 new_init_action
    然后指令链表程序  run_actions↓↓ ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓
    ↓↓
    ↓↓
    ↓↓
   	run_actions(SYSINIT);
   				waitfor(a, 0);//执行a，等待执行结束
   					run(a);//执行创建process子进程
   					 waitpid(runpid, &status, 0);
				delete_init_action(a);//删除链表
	/* Next run anything that wants to block */
	run_actions(WAIT);
	   				waitfor(a, 0);//执行a，等待执行结束
   					run(a);//执行创建process子进程
   					 waitpid(runpid, &status, 0);
				delete_init_action(a);//删除链表
	/* Next run anything to be run only once */
	run_actions(ONCE);
					run(a);
					delete_init_action(a);
		/* Now run the looping stuff for the rest of forever */
	while (1) {//重新运行pid已经退出的子进程
		run_actions(RESPAWN);
				if (a->pid == 0) {  //默认pid为0
					a->pid = run(a);}

		run_actions(ASKFIRST);
				if (a->pid == 0) {
					a->pid = run(a);}
					//打印"\nPlease press Enter to activate this console. ";，
					//等待输入回车
					//创建子进程
		wpid = wait(NULL);//等待子进程退出
		while (wpid > 0) {
				a->pid--；//推出后pid=0
			}
		}
	}



  配置文件：一般放在etc目录下
    1. 制定应用程序
    2. 何时执行
  Format for each entry: <id>:<runlevels>:<action>:<process>
  id=> /dev/id, 用作终端， 标准输入输出以及err
  runlevels 忽略
  action  何时执行
  process 应用程序或者脚本
```

### new_init_action

```
例子
new_init_action(ASKFIRST, bb_default_login_shell, VC_2);
#define ASKFIRST    0x004
bb_default_login_shell= "-bin/sh"
# define VC_2 "/dev/tty2"
===》new_init_action(ASKFIRST, "-bin/sh",  "/dev/tty2");

```

**目的：**

1. 创建一个init_action结构，放入init_action_list链表

举例分析，假设没有配置文件，函数会创建一个默认配置，反推一下

```
		new_init_action(CTRLALTDEL, "reboot", "");
		/* Umount all filesystems on halt/reboot */
		new_init_action(SHUTDOWN, "umount -a -r", "");
		/* Swapoff on halt/reboot */
		if (ENABLE_SWAPONOFF) new_init_action(SHUTDOWN, "swapoff -a", "");
		/* Prepare to restart init when a HUP is received */
		new_init_action(RESTART, "init", "");
		/* Askfirst shell on tty1-4 */
		new_init_action(ASKFIRST, bb_default_login_shell, "");
		new_init_action(ASKFIRST, bb_default_login_shell, VC_2);
		new_init_action(ASKFIRST, bb_default_login_shell, VC_3);
		new_init_action(ASKFIRST, bb_default_login_shell, VC_4);
		/* sysinit */
		new_init_action(SYSINIT, INIT_SCRIPT, "");
```

格式按照上文所提`Format for each entry: <id>:<runlevels>:<action>:<process>`

查看文件`busybox-1.7.0\examples\inittab` 能够看到

```
# <action>: Valid actions include: sysinit, respawn, askfirst, wait, once,
#                                  restart, ctrlaltdel, and shutdown.
```

得到上述程序实现了

```
new_init_action 第三个参数为终端，""为空
:: ctrlaltdel:reboot
:: shutdown:umount -a -r
ENABLE_SWAPONOFF 没有用到
:: restart:init
:: askfirst:-bin/sh
/dev/tty2:: askfirst:-bin/sh
/dev/tty3:: askfirst:-bin/sh
/dev/tty4:: askfirst:-bin/sh
::sysinit::/etc/init.d/rcS
```

然后把这个结构放到放入init_action_list链表，使用run_actions 执行

### run_actions

```
run_actions(SYSINIT);
   				waitfor(a, 0);//执行a，等待执行结束
   					run(a);//执行创建process子进程
   					 waitpid(runpid, &status, 0);
				delete_init_action(a);//删除链表
	/* Next run anything that wants to block */
	run_actions(WAIT);
	   				waitfor(a, 0);//执行a，等待执行结束
   					run(a);//执行创建process子进程
   					 waitpid(runpid, &status, 0);
				delete_init_action(a);//删除链表
	/* Next run anything to be run only once */
	run_actions(ONCE);
					run(a);
					delete_init_action(a);
		/* Now run the looping stuff for the rest of forever */
	while (1) {//重新运行pid已经退出的子进程
		run_actions(RESPAWN);
				if (a->pid == 0) {  //默认pid为0
					a->pid = run(a);}

		run_actions(ASKFIRST);
				if (a->pid == 0) {
					a->pid = run(a);}
					//打印"\nPlease press Enter to activate this console. ";，
					//等待输入回车
					//创建子进程
		wpid = wait(NULL);//等待子进程退出
		while (wpid > 0) {
				a->pid--；//推出后pid=0
			}
		}
	}
```

### init进程小结

1. 打开终端 dev/console
2. dev/null 不设置id的时候定位到此
3. 需要配置文件  /etc/initab
4. 配置文件指定的文件也需要存在  比如-bin/sh
5. 很多东西不是自己实现的 比如printf fopen，所以他需要一些库文件

### 最小的根文件系统

1. dev/console
2. dev/null
3. sbin/init---busybox提供
4. etc/inittab
5. 配置文件制定的应用程序
6. C库



## 配置编译busybox

说明文档在busybox根目录的INSTALL

```
  make menuconfig     # This creates a file called ".config"
  make                # This creates the "busybox" executable  
  make install        # or make CONFIG_PREFIX=/path/from/root install  不能使用这个，这个是安装到pc的
```

如果新版本ubunto编译不过去，则

```

背景：新装ubantu-16.04.2，2.已修改busybox-1.7.0 顶层Makefile 405行:config %config: scripts_basic outputmakefile FORCE
改为:
%config: scripts_basic outputmakefile FORCE

修改busybox-1.7.0 顶层Makefile 1242行:
/ %/: prepare scripts FORCE
改为:
%/: prepare scripts FORCE

如果还不行
要安装 sudo apt-get install libncurses5-dev libncursesw5-dev 这两个库才可以
```

设置makefie CROSS_COMPILE	?=arm-linux-

设置tab补全 在图形界面 busybox settings ---busybox libry tuning ----tab completion,输入y选择

关于C库，和其他编译选项具体看书本

- 然后make

- 千万不能直接make install ，会破坏pc

- 创建一个文件夹比如fsfirst ， 使用make CONFIG_PREFIX=/work/fsfirst install

  ```
  book@100ask:/work/fsfirst$ ls
  bin  linuxrc  sbin  usr

  book@100ask:/work/fsfirst$ ls bin/ls -l
  lrwxrwxrwx 1 book book 7 6月   4 23:02 bin/ls -> busybox


  ```

# 构建最小的根文件系统

## 编译busybox

编译busybox之后，会生成一个文件目录，里面有`bin  linuxrc  sbin  usr`,其中`linuxrc`是`bin/busybox`下的链接文件



## 构建dev目录（dev/console，null)

1. 查看下PC里面这两个文件的属性,

   ```
   book@100ask:/work/fsfirst/bin$ ls /dev/console  /dev/null -l
   crw------- 1 root root 5, 1 6月   4 22:36 /dev/console
   crw-rw-rw- 1 root root 1, 3 6月   4 22:36 /dev/null
   ```

   **解释** : c表示字符设备，root 5, 1  这个5表示主设备号，1表示次设备号

2. 打开刚才编译busybox的文件目录，在里面创建`dev`目录下文件

```
mkdir dev
cd dev
sudo mknod console c 5 1
sudo mknod null  c  1 3
```

## 构造一个 etc/inittab

如果没有自己去构造inittab,则linux会生成默认的参数表,具体分析见init进程分析.我们并不需要这些

```
::ctrlaltdel:reboot
::shutdown:umount -a -r
ENABLE_SWAPONOFF 没有用到
::restart:init
::askfirst:-bin/sh
/dev/tty2::askfirst:-bin/sh
/dev/tty3::askfirst:-bin/sh
/dev/tty4::askfirst:-bin/sh
::sysinit:/etc/init.d/rcS
```

我们可以先只要一个简单的,只包含`:: askfirst:-bin/sh`,将标准输入,输出,错误定位到console上

```
cd etc
vi inittab
book@100ask:/work/first_fs/etc$ cat inittab
console::askfirst:-bin/sh
```

## 构建C库

拷贝所有的`.so`文件,cp命令中使用-d的目的是:假设源文件为链接格式,那么cp的对象也是链接格式的

查看下我们当前的gcc目录,`.a`表示静态库不需要,使用`-d`来拷贝

```
which arm-linux-gcc
cd /opt/gcc-3.4.5-glibc-2.3.6/arm-linux/lib
ls -l 查看文件
mkdir /work/dirst_fs/lib
cp *.so* /work/first_fs/lib -d
ls -l 查看文件,可以发现链接文件还是链接文件
```

## 制作映像文件

使用工具`yaffs_source_util_larger_small_page_nand.tar`

```
tar xjf yaffs_source_util_larger_small_page_nand.tar.bz2
得到目录Development_util_ok
cd yaffs2/
cd utils/
make  新版本over了,直接使用资料光盘的

sudo cp mkyaffs2image /usr/local/bin
sudo chmod +x /usr/local/bin/mkyaffs2image
```

查看下这个工具的用法

```
book@100ask:/work/first_fs$ mkyaffs2image
mkyaffs2image: image building tool for YAFFS2 built Jan  6 2010
usage: mkyaffs2image dir image_file [convert]
           dir        the directory tree to be converted
           image_file the output file to hold the image
           'convert'  produce a big-endian image from a little-endian machine
```

生成`mkyaffs2image  first_fs first_fs.yaffs2`,使用DNW烧写文件系统,然后输入b启动

最后能够看到

```
Please press Enter to activate this console.
starting pid 763, tty '/dev/console': 'bin/sh'
```

## 增加其他命令(ps为例)

我们使用ps命令来查看有哪些应用程序,但是会提示没有命令

```
# ps
  PID  Uid        VSZ Stat Command
ps: can't open '/proc': No such file or directory
```

我们需要挂载虚拟的根文件系统,内核通过一个虚拟的文件系统收集内核的信息.手动在单板上挂载,然后就可以了

```
# mount -t proc none /proc
# ps
  PID  Uid        VSZ Stat Command
    1 0          3092 S   init
    2 0               SW< [kthreadd]
    3 0               SWN [ksoftirqd/0]
    4 0               SW< [watchdog/0]
    5 0               SW< [events/0]
    6 0               SW< [khelper]
   55 0               SW< [kblockd/0]
   56 0               SW< [ksuspend_usbd]
   59 0               SW< [khubd]
   61 0               SW< [kseriod]
   73 0               SW  [pdflush]
   74 0               SW  [pdflush]
   75 0               SW< [kswapd0]
   76 0               SW< [aio/0]
  710 0               SW< [mtdblockd]
  745 0               SW< [kmmcd]
  763 0          3096 S   -sh
  769 0          3096 R   ps

```

我们可以去proc目录下查看有那些文件

```
# ls proc/
1              745            diskstats      locks          sys
2              75             driver         meminfo        sysrq-trigger
3              76             execdomains    misc           sysvipc
4              763            fb             modules        timer_list
5              770            filesystems    mounts         tty
55             asound         fs             mtd            uptime
56             buddyinfo      interrupts     net            version
59             bus            iomem          partitions     vmstat
6              cmdline        ioports        scsi           yaffs
61             cpu            irq            self           zoneinfo
710            cpuinfo        kallsyms       slabinfo
73             crypto         kmsg           stat
74             devices        loadavg        swaps
```

其中1表示进程,我们可以`cd 1`,查看fd,表示标准输入输出和错误

```
# ls -l fd
lrwx------    1 0        0              64 Jan  1 00:07 0 -> /dev/console
lrwx------    1 0        0              64 Jan  1 00:07 1 -> /dev/console
lrwx------    1 0        0              64 Jan  1 00:07 2 -> /dev/console

```

### 使用配置文件去挂载(mount -t)

在pc上更改文件系统,增加配置文件,增加一个脚本`::sysinit:/etc/init.d/rcS`

```
mkdir proc
vi etc/inittab
增加脚本文件到配置文件
book@100ask:/work/first_fs$ cat etc/inittab
console::askfirst:-bin/sh
::sysinit:/etc/init.d/rcS
```

创建脚本文件,并加上手动增加的那句话,并增加上可执行的属性

```
mkdir etc/init.d
book@100ask:/work/first_fs$ cat etc/init.d/rcS
mount -t proc none /proc
chmod +x rcS
```

然后重新制作映像文件烧录`mkyaffs2image  first_fs first_fs.yaffs2`,进入系统后可以直接使用ps命令了

### 使用mount -a 挂载指定配置虚拟文件系统

`mount -a`能去读取`etc/fstab` 配置去挂载,我们可以看下busybox下的例子,`examples\bootfloppy\etc\fstab`

格式如下: 我们是虚拟的文件系统,device随便写,挂载到mount-point这个位置,文件系统是type

```
# device  mount-point  type   options dump  fsck order
```

所以需要更改两个文件   etc/init.d/rcS 和etc/fstab

```
book@100ask:/work/first_fs$ cat etc/fstab
# device  mount-point  type   options dump  fsck order
proc /proc proc defaults 0 0

book@100ask:/work/first_fs$ cat etc/init.d/rcS
#mount -t proc none /proc
mount -a
```

然后再`mkyaffs2image  first_fs first_fs.yaffs2`后烧录,也能使用ps命令,使用`cat  /proc/mounts` 查看已经挂载的文件系统

```
# cat  /proc/mounts
rootfs / rootfs rw 0 0
/dev/root / yaffs rw 0 0
proc /proc proc rw 0 0
```

## 使用m/udev 自动创建设备节点

`udev`用来自动创建`/dev/设备节点`, `mdev`是其简化版本,busybox中`docs/mdev.txt`有详细说明

```
Here's a typical code snippet from the init script:
[1] mount -t sysfs sysfs /sys
[2] echo /bin/mdev > /proc/sys/kernel/hotplug
[3] mdev -s

Of course, a more "full" setup would entail executing this before the previous
code snippet:
[4] mount -t tmpfs mdev /dev
[5] mkdir /dev/pts
[6] mount -t devpts devpts /dev/pts
```

先创建sys目录

```
mkdir sys
```

修改挂载配置文件`etc/fstab`实现1和4

```
book@100ask:/work$ cat  first_fs/etc/fstab
# device  mount-point  type   options dump  fsck order
proc /proc proc defaults 0 0
sysfs /sys sysfs defaults 0 0
tmpfs /dev tmpfs defaults 0 0
```

修改脚本文件

```
#mount -t proc none /proc
mount -a
mkdir /dev/pts
mount -t devpts devpts /dev/pts
echo  /sbin/mdev > /proc/sys/kernel/hotplug
mdev -s
```

烧录` mkyaffs2image  first_fs first_fs.yaffs2`下载后使用`ls dev`可以看到自动生成很多节点设备

```
# ls dev
console          ptyrf            tty17            ttyq7
dsp              ptys0            tty18            ttyq8
event0           ptys1            tty19            ttyq9
----......
```

再使用`cat  /proc/mounts` 查看已经挂载的文件系统

```
# cat proc/mounts
rootfs / rootfs rw 0 0
/dev/root / yaffs rw 0 0
proc /proc proc rw 0 0
sysfs /sys sysfs rw 0 0
tmpfs /dev tmpfs rw 0 0
devpts /dev/pts devpts rw 0 0
```

# 使用jffs2 文件系统格式

zlib 是个压缩库,jffs2是一个压缩的文件系统.

**工具文件:zlib-1.2.3.tar.gz**解压,注意这里是使用`xzf`,不是`xjf`,命令是`tar xzf zlib-1.2.3.tar.gz`

先配置`./configure  --shared --prefix=/usr/` ,`shared `表示动态库,`prefix`表示安装路径,然后make

最后安装到系统,`sudo make install`

**映像生成工具**mtd-utils-05.07.23.tar.bz2,先解压`tar xjf mtd-utils-05.07.23.tar.bz2`,然后`cd util/`去make

然后安装 `sudo make install`

生成映像文件`mkfs.jffs2 -n -s 2048 -e 128KiB -d  first_fs -o  first_fs.jffs2`

```
-s  一页大小是2048
-e  一个块大小 128KiB
-d  源目录
-o 输出
```

启动单板,输入`j`使用dnw烧写,启动后发现内核还是以yaff去识别`VFS: Mounted root (yaffs filesystem).`

显然不对.需要在启动命令行指定根文件系统的类型

```
Enter your selection: q
OpenJTAG> print
bootargs=noinitrd root=/dev/mtdblock3 init=/linuxrc console=ttySAC0
bootcmd=nand read.jffs2 0x30007FC0 kernel; bootm 0x30007FC0
baudrate=115200
ethaddr=08:00:3e:26:0a:5b
ipaddr=192.168.7.17
serverip=192.168.7.11
netmask=255.255.255.0
mtdids=nand0=nandflash0
mtdparts=mtdparts=nandflash0:256k@0(bootloader),128k(params),2m(kernel),-(root)
bootdelay=10
stdin=serial
stdout=serial
stderr=serial
partition=nand0,0
mtddevnum=0
mtddevname=bootloader

Environment size: 444/131068 bytes

```

设定根文件系统格式并保存

```
set bootargs noinitrd root=/dev/mtdblock3 rootfstype=jffs2 init=/linuxrc console=ttySAC0
save
```

输入boot启动`VFS: Mounted root (jffs2 filesystem).`

# NFS网络文件系统

设置虚拟机为桥接连接,进入单板linux,配置ip

```
ifconfig eth0 up
ifconfig eth0 192.168.5.200
ping下win
ping下ubuntu
```

## FLASH 根文件系统引导

37min29













# linux 命令

mknod  用于创建Linux中的字符设备文件和块设备文件。 
