## 结构体

```
//申明一个结构体 
struct book 
{
    char title[MAXTITL];//一个字符串表示的titile 题目 ； 
    char author[MAXAUTL];//一个字符串表示的author作者 ； 
    float value;//一个浮点型表示的value价格； 
};//注意分号不能少，这也相当于一条语句； 
struct book library； //需要这么定义变量


typedef struct book 
{
    char title[MAXTITL];//一个字符串表示的titile 题目 ； 
    char author[MAXAUTL];//一个字符串表示的author作者 ； 
    float value;//一个浮点型表示的value价格； 
}bookstru;//注意分号不能少，这也相当于一条语句； 
struct book library； //需要这么定义变量
```

也可以这样`bookstru  library;//也可以这样`,但是`book library`是有问题的。

**注意**

可以这么初始化，需要C99支持，keil里这么写特别注意。

```
struct s
{
  int a;
  int b;
};
struct s s1 = 
{
  .b = 1,
  .a = 2 
};
```



