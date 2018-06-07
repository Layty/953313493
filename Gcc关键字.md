## 语法

### 对齐

- aligned (4)   4字节对齐

- __attribute 对齐

  - `__attribute__ ((packed))`，让所作用的结构体取消在编译过程中的优化对齐，
    按照实际占用字节数进行对齐。
  - `__attribute((aligned (n)))`,让所作用的结构体成员对齐在n字节边界上。如果结构体中有成员变量的字节长度大于n， 则按照最大成员变量的字节长度来对齐。

  ```
  struct  person{
  	char *name;
  	int  age;
  	char score;
  	int  id;
  };

  struct  person1{
  	char *name;
  	int  age;
  	char score;
  	int  id;
  }__attribute__ ((packed));

  struct  person2{
  	char *name;
  	int  age;
  	char score;
  	int  id;
  }__attribute((aligned (4)));
  ```

## 编译选项

https://blog.csdn.net/gtuu0123/article/details/4557858

-fno-builtin  自己定义的函数与编译器默认的类型不一致，比如put等函数，这样可以避免警告

​	`**-fno-builtin-function**` 在这里可以只屏蔽指定的`function`函数

-nostdinc  不在标准系统目录中搜索头文件，只在-I指定的目录中搜索

-nostdlib  不连接标准启动文件和标准库文件

-Wall 打印所有警告和信息

-O0： 表示编译时没有优化。

-O1： 表示编译时使用默认优化。

-O2： 表示编译时使用二级优化。

-O3： 表示编译时使用最高级优化。