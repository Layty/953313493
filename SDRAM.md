## SDRAM 地址分布

## 内存知识

- P-Bank, 物理位宽，要等同于CPU的数据总线宽度，也是北桥内存总线宽度，适用于SDRAM以及以前产品，在RDRAM中以通道代替
- SDRAM synchronous Dynamic Random Access Memory 同步动态随机存储器
- SIMM single In-line Memory moudle 单列内存模组
- DIMM double in-line Memory moudle 双列内存模组
- SDRAM 芯片位宽  SDRAM芯片的数据总线
- 内存颗粒==内存芯片
- L-bank logic bank ,SDRAM芯片内部的bank，一般4个
- 内存芯片容量=行\*列\*L-bank\*位宽
- 引脚
  - Dqn 数据总线
  - An 行列地址线
  - DQM 数据掩码
  - CAS# 列选中
  - RAS# 行选中
  - CK 时钟信号
  - CKE 时钟有效
  - Ban L-bank线
  - WE# 写有效

### SDRAM 使用

#### 初始化协商 MSR

SDRAM 在上电的时候需要BIOS对其初始化设置MSR 模式，也就是协商一些参数

- 操作模式
- CAS 潜伏期  列地址潜伏期
- BT 突发传输模式
- BL 突发长度

#### 寻址

[(允许同时)CS片选，L-bank选择，行有效]列有效

#### 数据读

有个参数 CAS Latency，CAS 潜伏期=又被称为读取潜伏期（RL，Read Latency），这个在初始化时设定

#### 数据写

注意参数 twr 回写时间

#### 突发模式

连续读取，只需要发送起始列地址，BL在协商的时候规定了





![SDRAM地址分析](SDRAM.assets/SDRAM地址分析.png)

- 手册上有写地址线表述 行A0-A12，列 A0-A8 
- 16位宽的，所以一个阵列的点是2字节的
- 1bank内存=2^13\*2^9\*16/8/1024/1024=8M
- 1片的内存=4bank=32M
- 因为程序拟做32位寻址，地址线忽略低[1:0]，0x04=0b0100，所以地址线从l[2~14]

| bank                                     | 行[A12:A0]  | 列[A8:A0]   | 地址线固定为0 |
| ---------------------------------------- | ----------- | ----------- | ------------- |
| LADDR24和 LADDR25                        | LADDR[14:2] | LADDR[14:2] | 没有接        |
| [25:24],这里的接线正是 LADDR24和 LADDR25 | [23：11]    | [10:2]      | [1:0]={0,0}   |

- 从上表就可以看出为啥bank线接24,25了，也可以这么计算：

  ```
  寻址64M，4个片选也就是64/4=16M,16M=2^(20+4),所以0->2^23=16M,其再高1位就是24线了
  ```

- 开发板中使用两片16位的SDRAM芯片并联组成32位的位宽，与CPU的32根数据线(DATA0—DATA31)相连。 BANK6的起始地址为0x30000000，所以SDRAM的访问地址为0x30000000～低0x33FFFFFF，共64MB。