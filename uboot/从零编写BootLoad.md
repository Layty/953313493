## 概述

1. 初始化硬件：关看门狗、设置时钟、设置SDRAM、初始化NAND FLASH
2. 如果bootloader比较大，要把它重定位到SDRAM
3. 把内核从NAND FLASH读到SDRAM
4. 设置"要传给内核的参数"
5. 跳转执行内核


优化速度

- 提高主频

- 打开ICACH，在uboot中搜索copy

  ```
  	/* 启动ICACHE */
  	mrc p15, 0, r0, c1, c0, 0	@ read control reg
  	orr r0, r0, #(1<<12)
  	mcr	p15, 0, r0, c1, c0, 0   @ write it back
  ```

### 代码实现

- 链接脚本

    SECTIONS {
        . = 0x33f80000;
        .text : { *(.text) }
        
        . = ALIGN(4);
        .rodata : {*(.rodata*)} 
        
        . = ALIGN(4);
        .data : { *(.data) }
        
        . = ALIGN(4);
        __bss_start = .;
        .bss : { *(.bss)  *(COMMON) }
        __bss_end = .;
    }


- nand之前的bug,取址有问题

```
void nand_addr(unsigned int addr)
{
	unsigned int col  = addr % 2048;
	unsigned int page = addr / 2048;
	volatile int i;

	NFADDR = col & 0xff;
	for (i = 0; i < 10; i++);
	NFADDR = (col >> 8) & 0xff;
	for (i = 0; i < 10; i++);
	
	NFADDR  = page & 0xff;
	for (i = 0; i < 10; i++);
	NFADDR  = (page >> 8) & 0xff;
	for (i = 0; i < 10; i++);
	NFADDR  = (page >> 16) & 0xff;
	for (i = 0; i < 10; i++);	
}
```

- makefile

```

CC      = arm-linux-gcc
LD      = arm-linux-ld
AR      = arm-linux-ar
OBJCOPY = arm-linux-objcopy
OBJDUMP = arm-linux-objdump

CFLAGS 		:= -Wall -O2
CPPFLAGS   	:= -nostdinc -nostdlib -fno-builtin

objs := start.o init.o boot.o

boot.bin: $(objs)
	${LD} -Tboot.lds -o boot.elf $^
	${OBJCOPY} -O binary -S boot.elf $@
	${OBJDUMP} -D -m arm boot.elf > boot.dis
	
%.o:%.c
	${CC} $(CPPFLAGS) $(CFLAGS) -c -o $@ $<

%.o:%.S
	${CC} $(CPPFLAGS) $(CFLAGS) -c -o $@ $<

clean:
	rm -f *.o *.bin *.elf *.dis
	
```

### 编译选项

-fno-builtin  自己定义的函数与编译器默认的类型不一致，比如put等函数，这样可以避免警告

-nostdinc  不在标准系统目录中搜索头文件，只在-I指定的目录中搜索

-nostdlib  不连接标准启动文件和标准库文件