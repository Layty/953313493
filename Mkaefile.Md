#### 常规选项

| 选项      | 功能                                             |
| --------- | ------------------------------------------------ |
| -v        | 查看gcc编译器的版本，显示gcc执行时的详细过程     |
| -o <file> | 指定输出文件名为file，这个名称不能跟源文件名同名 |
| -E        | 只预处理，不会编译、汇编、链接t                  |
| -S        | 只编译，不会汇编、链接                           |
| -c        | 编译和汇编，不会链接                             |

举例如下： 实际应用中可以直接使用  `gcc -o hello hello.c`,gcc 会自动识别c文件进行编译汇编

```
gcc -E -o hello.i hello.c
gcc -S -o hello.s hello.i
gcc -c -o hello.o hello.s
gcc -o hello hello.o
```

#### 常用自动化变量

```
%.o：表示所用的.o文件
%.c：表示所有的.c文件
$@：表示目标
$<：表示第1个依赖文件
$^：表示所有依赖文件
```

#### 简单调试

可以使用echo显示

```
SRCS=$(wildcard *.c)
OBJS=$(SRCS:.c=.o)
DEPS=$(SRCS:.c=.d)

all:
    @echo SRCS = $(SRCS)
    @echo OBJS = $(OBJS)
    @echo DEPS = $(DEPS)
```





# Linux Makefile生成*.d依赖文件及 gcc -M -MF -MP等相关选项说明



记录：

OBJS=$(SRCS:.c=.o)   就是将SRCS中以.c 结尾的替换为.o为结尾的名称，还有一种替换是用静态模式

```
foo := a.o b.o c.o
bar := $(foo:%.o=%.c)
```





## 1. 为什么要使用后缀名为.d的依赖文件？

在 Makefile 中， 我们的依赖关系可能需要包含一系列的头文件。 
比如源文件main.c内容如下：

```
    #include "stdio.h"
    #include "defs.h"

    int main(int argc, char *argv[])
    {
        return 0;
    }       1234567
```

```
那么我们的依赖关系应该如下：  
    main.o : main.c stdio.h defs.h ...   
123
```

但如果是一个比较大型的工程，你必需清楚每一个 C 文件包含了哪些头文件，并且，你在加入或删除头文件时，也需要小心地修改 Makefile，这是一个很没有维护性的工作。为了避免这种繁重而又容易出错的事情，我们可以使用 C/C++编译的一个功能。大多数的C/C++编译器都支持一个“-M”的选项，即自动找寻源文件中包含的头文件，并生成一个依赖关系。例如，执行下面的命令：

```
gcc -M main.c   
其输出如下：  
    main.o : main.c defs.h
1234
```

于是由编译器自动生成了依赖关系，这样做的好处有以下几点：

- 不必手动书写若干文件的依赖关系，由编译器自动生成
- 不管是.c文件还是.h文件有更新，目标文件都会重新编译

## 2. 使用说明：

参数介绍：

- -M 
  生成文件的依赖关系,同时也把一些标准库的头文件也包含了进来。本质是告诉预处理器输出一个适合make的规则，用于描述各目标文件的依赖关系。对于每个源文件，预处理器输出 一个make规则，该规则的目标项(target)是源文件对应的目标文件名，依赖项(dependency)是源文件中 ‘#include’引用的所有文件，生成的规则可以是单行，但如果太长,就用’\’换行符续成多行。规则 显示在标准输出，不产生预处理过的C程序。 
  **注意：该选项默认打开了-E选项，-E参数的用处是使得编译器在预处理结束时就停止编译**

> 例如： gcc -M main.c 
> 则在终端上输出如下： 
> main.o: main.c defs.h \ 
> /usr/include/stdio.h \ 
> /usr/include/features.h \ 
> /usr/include/sys/cdefs.h /usr/include/gnu/stubs.h \ 
> /usr/lib/gcc-lib/i486-suse-linux/2.95.3/include/stddef.h \ 
> /usr/include/bits/types.h \ 
> /usr/include/bits/pthreadtypes.h \ 
> /usr/include/_G_config.h /usr/include/wchar.h \ 
> /usr/include/bits/wchar.h /usr/include/gconv.h \ 
> /usr/lib/gcc-lib/i486-suse-linux/2.95.3/include/stdarg.h \ 
> /usr/include/bits/stdio_lim.h

- -MM 
  生成文件的依赖关系，和-M类似，但不包含标准库的头文件

> 例如：gcc -MM main.c 
> 则在终端上输出如下： 
> main.o: main.c defs.h

- -MG 
  要求把缺失的头文件按存在对待,并且假定他们和源程序文件在同一目录下.必须和 ‘-M’选项一起用.
- -MF File 
  当使用了’-M’或者’-MM’选项时，则把依赖关系写入名为’File’的文件中。若同时也使用了’-MD’或’-MMD’，’-MF’将覆写输出的依赖文件的名称

```
例如：gcc -M -MF main.d main.c
    则 '—M' 输出的内容就存在于 main.d 文件中了12
```

- -MD 
  等同于 ‘-M -MF File’，但是默认关闭了’-E’选项. 其输出的文件名是基于’-o’选项，若给定了’-o’选项，则输出的文件名是’-o’指定的文件名，并添加.d后缀，若没有给定，则输入的文件名作为输出的文件名，并添加.d后缀，同时继续指定的编译工作 
  **注意：’-MD’不会像’-M’那样阻止正常的编译任务. 因为它默认关闭了’-E’选项, 比如命令中使用了-c选项，其结果要生成.o文件，若使用了’-M’选项，则不会生成.o文件，若使用的是’-MD’选项，则会生成.o文件**

```
例如1：gcc -E -MD main.c
本目录下生成了以下文件：
    main.d 
同时在终端上输出了main.c文件的预处理结果

经实测发现,不使用'-o'指定输出文件名,以下情况有细微的差别:
gcc -E main.c //不使用'-o',则把结果输出在终端上
gcc -S main.c //不使用'-o',则把结果默认输出到以输入文件名为名称的.s文件中,即main.s
gcc -c main.c //同上
gcc main.o   //不使用'-o',则把结果默认输出到a.out可执行文件中12345678910
```

```
例如2：gcc -E -o tmp.i -MD main.c
本目录下生成了以下文件：
    tmp.d tmp.i

例如3：gcc -c -MD main.c
本目录下生成了以下文件：
    main.d main.o

例如4：gcc -c -o tmp.o -MD main.c
本目录下生成了以下文件：
    tmp.d tmp.o

例如5: gcc -MD main.c
本目录下生成了以下文件:
    a.out main.d 

例如6: gcc -M -MD main.c
本目录下生成了以下文件:
    main.d  //并不会生成a.out可执行文件,因为'-M'默认打开了'-E'选项,使得编译器在预处理结束后就停止编译12345678910111213141516171819
```

- -MMD 
  类似于’-MD’，但是输出的依赖文件中，不包含标准头文件
- -MP 
  生成的依赖文件里面，依赖规则中的所有.h依赖项都会在该文件中生成一个伪目标，其不依赖任何其他依赖项。该伪规则将避免删除了对应的头文件而没有更新”Makefile”去匹配新的依赖关系而导致make出错的情况出现。 
  (英文描述:This option instructs CPP to add a phony target for each dependency 
  other than the main file, causing each to depend on nothing. These 
  dummy rules work around errors ‘make’ gives if you remove header 
  files without updating the ‘Makefile’ to match.)

```
例如1: gcc -c -MM -MD main.c
生成的main.d文件内容如下:
main.o: main.c defs.h

例如1: gcc -c -MM -MD main.c -MP
生成的main.d文件内容如下:
main.o: main.c defs.h
defs.h:  //该选项会生成该伪目标，其没有任何依赖项,若不使用'-MP'选项,则不会生成该伪目标规则12345678
```

- -MT Target 
  在生成的依赖文件中,指定依赖规则中的目标

```
例如: gcc -MF main.d -MG -MM -MP -MT main.d -MT main.o main.c

$ cat main.d    #查看生成的依赖文件的内容
main.d main.o: main.c
注:依赖规则中main.d 和 main.o 目标都是通过'-MT'选项指定的12345
```

## 3. 使用参考：

以上简单介绍了gcc -M相关的选项,旨在让make自动推导并生成文件的依赖关系. 
以下提供一个比较好的gcc -M选项的参考示例， 它将自动生成依赖文件，并保存在指定目录下的’.d’文件中。

makefile如下所示:

```
SRCS=$(wildcard *.c)
OBJS=$(SRCS:.c=.o)
DEPS=$(SRCS:.c=.d)

.PHONY: all clean

all: main

-include $(APPDEPS)    #注释:'-'号的作用在于加载错误时，会继续执行make,其主要是指首次make时，目录中还不存在'*.d'文件，所以此时加载会产生错误

%.o:%.c
    gcc -c -g -Wall $< -o $@ -MD -MF $*.d -MP 

main: $(OBJS)
    gcc $^ -o $@   #注释:$^:表示所有的依赖文件 $@:表示目标文件

clean: 
        rm -f  *.d *.o main
```

仍旧以本篇文章开头的源文件进行make，将生成如下文件： 
main : 可执行文件 
main.o : 编译的二进制目标文件 
main.d：保存了main.o的依赖关系的文件

```
注释: $* 表示目标模式中'%'及其之前的部分.如果目标是'dir/a.foo.b',
并且目标的模式为'a.%.b',那么'$*'的值就是'dir/a.foo'. 
如果目标中没有模式的定义,那么'$*'就不能被推导出.
但是,如果目标文件是make所识别的,那么'$*'就是除了后缀的那一部分,

例如:目标是'foo.c',因为'.c'是make所能识别的后缀名,
所以'$*'的值就是'foo'.这个特性是GNU make的.1234567
```

## 4. 延伸说明：

```
Makefile文件中使用比较多的自动变量:

 - $@ : 表示一个规则中的目标.当规则中有多个目标时,$@所指的是其中任何造成规则的命令运行的目标
 - $^ : 表示规则中的所有依赖项
 - $< : 表示规则中的第一个依赖项

例如Makefile文件内容如下:
target1 target2:dep1 dep2 dep3
    @echo "Tar:$@, First Dep:$<, All Dep:$^"
dep1 dep2 dep3:    #注释:不写该行目标规则,执行make会报错,因为makefile目录下不仅没有dep1 dep2 dep3文件,makefile文件中也没有以这些依赖文件做为目标的规则.

运行make: 
1. make target1 或者 make(把文件中第一个目标当作首要目标)
终端输出如下:
    Tar:target1, First Dep:dep1, All Dep:dep1 dep2 dep3

2. make target2
终端输出如下:
    Tar:target2, First Dep:dep1, All Dep:dep1 dep2 dep3

3. make target1 target2
终端输出如下:
    Tar:target2, First Dep:dep1, All Dep:dep1 dep2 dep3
    Tar:target2, First Dep:dep1, All Dep:dep1 dep2 dep3123456789101112131415161718192021222324
```

相信大家在看了以上例子,能够更加理解这三个自动化变量的含义,尤其是’**$@**‘, 前面已经说明了其含义,这里就不再赘述了.