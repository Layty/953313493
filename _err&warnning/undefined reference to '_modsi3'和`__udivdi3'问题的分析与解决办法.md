# undefined reference to '_modsi3'和`__udivdi3'问题的分析与解决办法



#### 方法

- cp   lib1funcs.S加入到工程
- makefile 添加 arm-linux-gcc -c -o lib1funcs.o lib1funcs.S
- 链接的时候再加入lib1funcs.o





## 【编译器版本】

​	arm-linux-gcc 3.4.1

## 【问题描述】

​	在做嵌入式底层开发时（基于ARM编译无OS的程序），编写整数转字符串函数，用到了求余操作%和除数操作，部分代码如下：

```
...
while(num)
{
	denom = num % radix;
	num /= radix;
	*ptr++ = denom + '0';
		
	if((num < radix) && num)
	{
		*ptr++ = num + '0';
		*ptr = '\0';
		break;
	}
}
...
```

部分Makefile如下：

```
...
.c.o:
	$(CC) -static -nostartfiles -nostdlib -fno-builtin $(CPPFLAGS) $(CFLAGS) -c $<
...
```

结果报错： 

undefined reference to `__umodsi3'

undefined reference to `__udivdi3'

## 【问题分析】

​	ARM是精简指令集，对求余和除法操作基本上不支持，所以应该尽量避免上述操作。libgcc库包含这些操作，这种错误是没有包含GCC的支持库libgcc.a，缺少库文件。

## 【解决办法】

1.在连接时指定连接GCC库，-lgcc

​	-lgcc 代表链接器将连接GCC的支持库libgcc.a。这里需要注意的是：-lm 代表链接器将连接GCC的标准数学库库libm.a，-lc 代表链接器将连接GCC的标准C库libc.a，如果这三者都需要，则连接排列顺序为-lm -lc -lgcc。

2.自己实现响应的库函数

​	如果不想基于库函数，可以自己参考linux内核源码linux/arch/arm/lib/lib1funcs.S实现这两个库函数，lib1funcs.S实现除法、求模操作，具体的源码如下：

```
/*
 * linux/arch/arm/lib/lib1funcs.S: Optimized ARM division routines
 *
 * Author: Nicolas Pitre <nico@cam.org>
 *   - contributed to gcc-3.4 on Sep 30, 2003
 *   - adapted for the Linux kernel on Oct 2, 2003
 */

/* Copyright 1995, 1996, 1998, 1999, 2000, 2003 Free Software Foundation, Inc.

This file is free software; you can redistribute it and/or modify it
under the terms of the GNU General Public License as published by the
Free Software Foundation; either version 2, or (at your option) any
later version.

In addition to the permissions in the GNU General Public License, the
Free Software Foundation gives you unlimited permission to link the
compiled version of this file into combinations with other programs,
and to distribute those combinations without any restriction coming
from the use of this file.  (The General Public License restrictions
do apply in other respects; for example, they cover modification of
the file, and distribution when not linked into a combine
executable.)

This file is distributed in the hope that it will be useful, but
WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the GNU
General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program; see the file COPYING.  If not, write to
the Free Software Foundation, 59 Temple Place - Suite 330,
Boston, MA 02111-1307, USA.  */

/*
#include <linux/linkage.h>
#include <asm/assembler.h>
*/

#define ALIGN		.align 4,0x90
#define __LINUX_ARM_ARCH__  1

#define ENTRY(name) \
  .globl name; \
  ALIGN; \
  name:

.macro ARM_DIV_BODY dividend, divisor, result, curbit

#if __LINUX_ARM_ARCH__ >= 5

	clz	\curbit, \divisor
	clz	\result, \dividend
	sub	\result, \curbit, \result
	mov	\curbit, #1
	mov	\divisor, \divisor, lsl \result
	mov	\curbit, \curbit, lsl \result
	mov	\result, #0
	
#else

	@ Initially shift the divisor left 3 bits if possible,
	@ set curbit accordingly.  This allows for curbit to be located
	@ at the left end of each 4 bit nibbles in the division loop
	@ to save one loop in most cases.
	tst	\divisor, #0xe0000000
	moveq	\divisor, \divisor, lsl #3
	moveq	\curbit, #8
	movne	\curbit, #1

	@ Unless the divisor is very big, shift it up in multiples of
	@ four bits, since this is the amount of unwinding in the main
	@ division loop.  Continue shifting until the divisor is 
	@ larger than the dividend.
1:	cmp	\divisor, #0x10000000
	cmplo	\divisor, \dividend
	movlo	\divisor, \divisor, lsl #4
	movlo	\curbit, \curbit, lsl #4
	blo	1b

	@ For very big divisors, we must shift it a bit at a time, or
	@ we will be in danger of overflowing.
1:	cmp	\divisor, #0x80000000
	cmplo	\divisor, \dividend
	movlo	\divisor, \divisor, lsl #1
	movlo	\curbit, \curbit, lsl #1
	blo	1b

	mov	\result, #0

#endif

	@ Division loop
1:	cmp	\dividend, \divisor
	subhs	\dividend, \dividend, \divisor
	orrhs	\result,   \result,   \curbit
	cmp	\dividend, \divisor,  lsr #1
	subhs	\dividend, \dividend, \divisor, lsr #1
	orrhs	\result,   \result,   \curbit,  lsr #1
	cmp	\dividend, \divisor,  lsr #2
	subhs	\dividend, \dividend, \divisor, lsr #2
	orrhs	\result,   \result,   \curbit,  lsr #2
	cmp	\dividend, \divisor,  lsr #3
	subhs	\dividend, \dividend, \divisor, lsr #3
	orrhs	\result,   \result,   \curbit,  lsr #3
	cmp	\dividend, #0			@ Early termination?
	movnes	\curbit,   \curbit,  lsr #4	@ No, any more bits to do?
	movne	\divisor,  \divisor, lsr #4
	bne	1b

.endm


.macro ARM_DIV2_ORDER divisor, order

#if __LINUX_ARM_ARCH__ >= 5

	clz	\order, \divisor
	rsb	\order, \order, #31

#else

	cmp	\divisor, #(1 << 16)
	movhs	\divisor, \divisor, lsr #16
	movhs	\order, #16
	movlo	\order, #0

	cmp	\divisor, #(1 << 8)
	movhs	\divisor, \divisor, lsr #8
	addhs	\order, \order, #8

	cmp	\divisor, #(1 << 4)
	movhs	\divisor, \divisor, lsr #4
	addhs	\order, \order, #4

	cmp	\divisor, #(1 << 2)
	addhi	\order, \order, #3
	addls	\order, \order, \divisor, lsr #1

#endif

.endm


.macro ARM_MOD_BODY dividend, divisor, order, spare

#if __LINUX_ARM_ARCH__ >= 5

	clz	\order, \divisor
	clz	\spare, \dividend
	sub	\order, \order, \spare
	mov	\divisor, \divisor, lsl \order

#else

	mov	\order, #0

	@ Unless the divisor is very big, shift it up in multiples of
	@ four bits, since this is the amount of unwinding in the main
	@ division loop.  Continue shifting until the divisor is 
	@ larger than the dividend.
1:	cmp	\divisor, #0x10000000
	cmplo	\divisor, \dividend
	movlo	\divisor, \divisor, lsl #4
	addlo	\order, \order, #4
	blo	1b

	@ For very big divisors, we must shift it a bit at a time, or
	@ we will be in danger of overflowing.
1:	cmp	\divisor, #0x80000000
	cmplo	\divisor, \dividend
	movlo	\divisor, \divisor, lsl #1
	addlo	\order, \order, #1
	blo	1b

#endif

	@ Perform all needed substractions to keep only the reminder.
	@ Do comparisons in batch of 4 first.
	subs	\order, \order, #3		@ yes, 3 is intended here
	blt	2f

1:	cmp	\dividend, \divisor
	subhs	\dividend, \dividend, \divisor
	cmp	\dividend, \divisor,  lsr #1
	subhs	\dividend, \dividend, \divisor, lsr #1
	cmp	\dividend, \divisor,  lsr #2
	subhs	\dividend, \dividend, \divisor, lsr #2
	cmp	\dividend, \divisor,  lsr #3
	subhs	\dividend, \dividend, \divisor, lsr #3
	cmp	\dividend, #1
	mov	\divisor, \divisor, lsr #4
	subges	\order, \order, #4
	bge	1b

	tst	\order, #3
	teqne	\dividend, #0
	beq	5f

	@ Either 1, 2 or 3 comparison/substractions are left.
2:	cmn	\order, #2
	blt	4f
	beq	3f
	cmp	\dividend, \divisor
	subhs	\dividend, \dividend, \divisor
	mov	\divisor,  \divisor,  lsr #1
3:	cmp	\dividend, \divisor
	subhs	\dividend, \dividend, \divisor
	mov	\divisor,  \divisor,  lsr #1
4:	cmp	\dividend, \divisor
	subhs	\dividend, \dividend, \divisor
5:
.endm


ENTRY(__udivsi3)

	subs	r2, r1, #1
	moveq	pc, lr
	bcc	Ldiv0
	cmp	r0, r1
	bls	11f
	tst	r1, r2
	beq	12f

	ARM_DIV_BODY r0, r1, r2, r3

	mov	r0, r2
	mov	pc, lr

11:	moveq	r0, #1
	movne	r0, #0
	mov	pc, lr

12:	ARM_DIV2_ORDER r1, r2

	mov	r0, r0, lsr r2
	mov	pc, lr


ENTRY(__umodsi3)

	subs	r2, r1, #1			@ compare divisor with 1
	bcc	Ldiv0
	cmpne	r0, r1				@ compare dividend with divisor
	moveq   r0, #0
	tsthi	r1, r2				@ see if divisor is power of 2
	andeq	r0, r0, r2
	movls	pc, lr

	ARM_MOD_BODY r0, r1, r2, r3

	mov	pc, lr


ENTRY(__divsi3)

	cmp	r1, #0
	eor	ip, r0, r1			@ save the sign of the result.
	beq	Ldiv0
	rsbmi	r1, r1, #0			@ loops below use unsigned.
	subs	r2, r1, #1			@ division by 1 or -1 ?
	beq	10f
	movs	r3, r0
	rsbmi	r3, r0, #0			@ positive dividend value
	cmp	r3, r1
	bls	11f
	tst	r1, r2				@ divisor is power of 2 ?
	beq	12f

	ARM_DIV_BODY r3, r1, r0, r2

	cmp	ip, #0
	rsbmi	r0, r0, #0
	mov	pc, lr

10:	teq	ip, r0				@ same sign ?
	rsbmi	r0, r0, #0
	mov	pc, lr

11:	movlo	r0, #0
	moveq	r0, ip, asr #31
	orreq	r0, r0, #1
	mov	pc, lr

12:	ARM_DIV2_ORDER r1, r2

	cmp	ip, #0
	mov	r0, r3, lsr r2
	rsbmi	r0, r0, #0
	mov	pc, lr


ENTRY(__modsi3)

	cmp	r1, #0
	beq	Ldiv0
	rsbmi	r1, r1, #0			@ loops below use unsigned.
	movs	ip, r0				@ preserve sign of dividend
	rsbmi	r0, r0, #0			@ if negative make positive
	subs	r2, r1, #1			@ compare divisor with 1
	cmpne	r0, r1				@ compare dividend with divisor
	moveq	r0, #0
	tsthi	r1, r2				@ see if divisor is power of 2
	andeq	r0, r0, r2
	bls	10f

	ARM_MOD_BODY r0, r1, r2, r3

10:	cmp	ip, #0
	rsbmi	r0, r0, #0
	mov	pc, lr


Ldiv0:

	str	lr, [sp, #-4]!
/*	bl	__div0	*/
	mov	r0, #0			@ About as wrong as it could be.
	ldr	pc, [sp], #4
```

## 【类似问题--ARM中的64位除法问题】

​	32位嵌入式系统中（目前多数系统都是，比如ARM的片子），对于普通的a除以b（b为32位）：
（1）当a为32位，Linux 内核中，常用uint32_t 类型，可以直接写为 a/b
（2）但是，对于a是64位，uint64_t的时候，就要用到专门的除操作相关的函数，linux内核里面一般为do_div(n, base)，注意，此处do_div得到的结果是余数，而真正的a/b的结果，是用a来保存的。do_div(n,base)的具体定义，和当前体系结构有关，对于arm平台，在arch/arm/include/asm/div64.h和linux/arch/arm/lib/div64.S中，其实现很复杂，感兴趣的自己去代码里看。因此，如果你当前写代码，a/b，如果a是uint64_t类型，那么一定要利用do_div(a,b)，而得到结果a，而不能简单的用a/b，否则编译可以正常编译，但是最后链接最后出错，会提示上面的那个错误：undefined reference to "__udivdi3"。

​	知道原因，就好办了。办法就是，在你代码里面找到对应的用到除法的地方，即类似于a/b的地方，其中被除数a为64位，Linux中一般用uint64_t，将a/b用do_div(a,b)得到的a去代替（注意，不是直接用do_div()，因为do_div(a,b)得到的是余数。）即可。

​	Linux内核早已经帮我们实现了对应的64位的unsingned和signed两个函数：

```
static inline u64 div_u64(u64 dividend, u32 divisor);
static inline s64 div_s64(s64 dividend, s32 divisor);
```

我们可以直接拿过来用了，注意用此函数时，要包含对应头文件：

\#include <linux/math64.h>

如果需要在进行64位除数的时候，同时得到余数remainder，可以直接用#include <linux/math64.h>中的：

```
static inline u64 div_u64_rem(u64 dividend, u32 divisor, u32 *remainder);
static inline s64 div_s64_rem(s64 dividend, s32 divisor, s32 *remainder);
```