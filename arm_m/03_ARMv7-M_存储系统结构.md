# 1. Overview

Cortex-M是32位的处理器，因此处理器能够寻址到4GB的地址空间。M-CPU采用的是哈佛结构，因此数据空间和指令空间在同一个空间维度。这些空间被分为多个区域，后面会介绍memory map。关于M核的存储系统，有以下比较重点features：
* 多总线接口允许指令和数据并发访问地址空间。
* AMBA总线协议
* 大小端存储都支持
* 支持非对齐的数据传输
* 支持排他访问（操作系统的信号量）
* 比特级的内存访问宽度
* 内存属性和访问权限的配置（指定区域）
* MPU

# 2. Memory Map

4GB的寻址空间，一部分被内部的外设预留，作为外设访问的地址空间，例如NVIC和一些调试的组件。这些外设的预留的空间都是固定的。还有一部分的空间被分配如图：

![](https://raw.githubusercontent.com/carloscn/images/main/typora20230413202154.png)

![](https://raw.githubusercontent.com/carloscn/images/main/typora20230413203424.png)


这样的分配:
* 处理器这样设计是为了支持多种内存，并且对于二级厂家开箱即用。
* 内存的分区编排也是为了提更高的性能；

在ARM公司层面，给出memory map只是一个参考建议，对于数字芯片设计者，他们可以根据自己的case来对内存进行设计。对于NXP，ST的这些ARM芯片，他们都是有自己的memory map的。ARM提供的仅仅是一个很简单的内存映射的demo。

我们通常都会把执行的程序存储到RAM或者SRAM上，CPU在执行的时候从RAM里面读取指令然后执行。不过从memory map上看，显然这个是笨拙的，因为要花费额外的时钟周期从RAM中读取指令，远不如从地址总线的内存区域直接读取。但是要知道，地址总线映射区域并不是存储空间，是无法执行程序的。因此，这里就引出了地址属性的概念。对于NVIC与MPU和SCB，这些系统外设占用的空间叫做System Control Space (CSC)。
