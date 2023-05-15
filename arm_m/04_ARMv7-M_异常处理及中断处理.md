
# 1. Overview

中断是一个典型的组件在一般的微控制器中。中断时间通常由硬件产生，例如外设和外部的引脚。中断会改变程序的执行路径。当一个中断产生的时候，通常会发生：

1. 外设assert中断请求到处理器；
2. 处理器会暂停当前的任务；
3. 处理器跳转到当前中断的服务程序中ISR；
4. 处理器恢复中断之前的任务。

在Cortex-M处理器中提供了一个Nested Vectored Interrupt Controller（NVIC）。除了中断，还有其他事件需要服务，我们称之为“异常”。在ARM的术语中，中断属于异常的一种，而在工程使用中有个比较习惯的表述，中断更侧重于由外设，外围器件这种角色产生的；而异常是由处理器本身产生的。无论是外设的中断还是处理器异常，在ARM中都通过异常处理机制来对这种信号进行管理。在典型的Cortex-M控制器中，NVIC接收各种各样的中断源：

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202304231142210.png" width="80%" /></div> 

有外设、有IO口、有处理器产生的异常、有定时器。在ARMv8里面和NVIC同等位置的IP是GIC，[12_ARMv8_异常处理（三）- GICv1/v2中断处理](https://github.com/carloscn/blog/issues/51)。而NVIC更像是ARMv8架构中的Legacy的中断。[11_ARMv8_异常处理（二）- Legacy 中断处理](https://github.com/carloscn/blog/issues/49)。

Cortex-M3 and Cortex-M4的NVIC支持240IRQs，和Non-Makable Interrupt(NMI)，和System Tick定时器中断，还有数个系统异常。注意，NMI是无法屏蔽的中断，像是watchdog或者Brown-Out Detector (BOD)。

为了能够从中断程序中恢复中断现场。ARM处理器需要保存中断之前的状态，在中断结束之后，从保存的区域中恢复。这个功能的实现可以通过硬件，也可以软硬结合。在Cortex-M中，一些寄存器的值会被保存到栈上，等中断结束之后会从栈上恢复状态。

# 2. 异常与中断介绍

## 2.1 异常类型

M核处理器将数个系统级异常和外部中断在异常体系中做了功能归并。1-15号异常分配给了系统，16号以上中断（不包含IO pins）分配给处理器的中断引脚。大部分的中断都是支持定制优先级，少部分的系统级的异常则无法更改优先级。

不同的处理器架构（M3/M4）中断编号不同，也有着不同的优先级配置。这个要看二级厂商怎么定义了。

值得注意的是，1-15号是系统级的中断（注意没有0号异常）。

|中断号|类型|优先级|描述|
|-|-|-|-|
|1|Reset|-3（最高）|复位中断|
|2|NMI|-2|不可屏蔽中断，是由处理器级外设或者外部的中断源产生|
|3|Hard Fault|-1|所有的Fault在没有定义handler的时候就会产生该中断|
|4|MemManage Fault|可编程|内存管理fault，比如违反了MPU的规则，或者向可执行的总线写入数据|
|5|Bus Fault|可编程|总线错误，通常发生在当AHB总线从slave设备接收到一个错误的返回值；或者与pre-fetch abort错误|
|6|Usage Fault|可编程|由于编程错误或者试图访问协处理器（M3/M4不支持协处理器）|
|7-10|Reserved|NA|-|
|11|SVC|可编程|SuperVisor Call,在OS环境，允许引用访问系统服务|
|12|Debug Monitor|可编程|当一个debug事件，类似于执行到断点，观测点的时候发生|
|13|Reserved|NA|-|
|14|PendSV|可编程|在系统进程中，进程之间的切换|
|14|SYSTICK|可编程|Timer产生，包括外设的timer或者是处理器。|
|16|Interrupt #0|可编程|外设使用|
|17|Interrupt #1|可编程|外设使用|
|...|Interrupt # n|可编程|外设使用|
|255|Interrupt # 239|可编程|外设使用|

这些中断编号，用于识别是哪种异常发生了，被广泛用于ARMv7-M的架构中。在CMSIS-Core的软件栈中，中断标识使用枚举类型来表述，注意，0号指的是16号的interrupt0，并非系统0号中断。而系统中断使用负值表示。在CMSIS-Core的软件栈中定义中断如下所示：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202304240932490.png)


## 2.2 中断管理

上面提到有很多可配置优先级的中断，这就是需要一个中断管理器来对中断进行管理。中断管理中使用寄存器来对中断的信息进行保留。

>注意，寄存器的概念：
>
><div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202304261118830.png" width="60%" /></div> 
>
>寄存器不同于内存。寄存器在数电中是通过逻辑电路的来实现的，里面有锁存器来存储信息。ARM里面有很多锁存器堆叠起来形成寄存器，这是不同于内存的。ARM总线通过地址映射，让我们可以用C语言访问地址来访问到寄存器。

大多数的寄存器在NVIC和System Control Block(SCB)中，（在ARM内部，SCB是NVIC的一部分，但是在软件CMSIS-Core中定义的数据结构是分开的）。除此之外，还有一些特殊的寄存器（PRIMASK, FAULTMASK和BASEPRI）来管理中断的屏蔽。

NVIC和SCB都在SCS（System Control Space: 0xE000_E000）这个位置4KB的长度。SCS也包含SysTick定时器，MPU，Debug Registers这些的配置选项。**要访问SCS寄存器，处理器必须处于特权模式才可以**。仅仅有一个叫做Software Trigger Interrupt Register (STIR)的寄存器，才可以以非特权模式访问。

对于应用层级编程，最好的是使用CMSIS-Core软件库。例如这个软件库中提供了一些中断控制的函数接口：

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202304261132266.png" width="80%" /></div> 

如果有需要也可以直接访问在NVIC或者SCB中的寄存器。但是这样的话，从软件设计的角度，就有一些局限性，限制了程序的可移植性。

在复位之后，所有的中断会被管理，并且默认给定的优先级都是0。在使用中断之前，必须：
* 根据业务情况配置所需要的优先级；
* 配置外设中断的触发方式；
* 在NVIC中使能中断；

在大多数的APP中，这些都需要编写APP的人来去处理。当中断被触发之后，相应的ISR会被执行。在代码启动的时候会设定一个中断向量表，这个ISR的名字在中断向量表中定义。

## 2.3 中断优先级

在M3/M4，处理器是否执行某个中断的handler，取决于异常的优先级。高优先级的中断可以抢占低优先级的中断。注意，高优先级的priority-level数值小，反之亦然。M核还支持嵌套中断。还需要注意的是，前面有说一些中断的优先级不能被配置，例如reset、NMI和HardFault。

在原生ARM里面，M3/M4是支持3个固定最高优先级和256等级的可编程的中断优先级
。这个实际的数量取决于二级厂商。如图所示：

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202304270818962.png" width="80%" /></div> 

对于优先级配置的寄存器是8bits宽度的，但是这里仅仅有128 pre-emption。这是因为8bits宽度的寄存器被分为两部分，`group priority`和`sub-priority`。因为这样分组之后，可以通过组来设定优先级，设定一个组优先级之后，该组内所有的中断优先级都会被生效。

对于相同优先级的中断，处理器按序号进行二次优先级认定，即中断编号低的优先级更高。


## 2.4 中断向量

当M处理器接受一个异常请求的时候，处理器需要知道这个异常handler的开始地址在哪里。handler像是一个表格一样存储在内存里面。默认状态，vector table在0地址，并且内handler的按照异常需要在表格中排序。**异常向量表通常被放在code的起始位置**。如下图所示，为M3/M4的异常向量表：

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202304280901787.png" width="80%" /></div> 

在启动时候被使用的中断向量表也包含一个**主**栈指针（MSP）。这是必要的，因为当处理器刚刚从重置中出来，并且在执行任何其他初始化步骤之前，可能会发生一些异常，如NMI。在传统的ARM处理器中，向量表包含诸如分支到适当处理程序的分支指令之类的指令，而在Cortex-M中，向量表格包含异常处理程序的起始地址。通常，程序的boot开始地址在0x0000_0000，可能是flash也可能是ROM，通常这个地址是不能更改的。那么这个需求如何修改呢？在M3/M4处理器支持一个特性，Vector Table Relocation。这个VTR提供一个可配置的寄存器，叫做VTOR。这个寄存器定义了vector table的起始地址。

VTOR寄存器复位之后是全0，并且在CMSIS-compliant的驱动中有接口可以写入。

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202304271024475.png" width="80%" /></div> 

当使用VTOR的时候，新vector的基地址必须和向量表的大小的2次幂对齐。

```C
// Macros for word access
#define HW32_REG(ADDRESS) (*((volatile unsigned long *)(ADDRESS)))
#define VTOR_NEW_ADDR 0x20000000

int i; // loop counter
// Copy original vector table to SRAM first before programming VTOR
for (i=0;i<48;i++){ // Assume maximum number of exception is 48
	// Copy each vector table entry from flash to SRAM
	HW32_REG((VTOR_NEW_ADDR + (i<<2))) = HW32_REG((i<<2));
}
__DMB(); // Data Memory Barrier
// to ensure write to memory is completed
SCB->VTOR = VTOR_NEW_ADDR; // Set VTOR to the new vector table
//location
__DSB(); // Data Synchronization Barrier to ensure all
// subsequence instructions use the new configuation
```

### 2.4.1 示例1：32个中断源

中断表大小 (32 + 16) x 4 = 192 (0xc0)。 其中32是中断，16是系统异常。接着向上round_up到256bytes。因此向量表的基地址可以是 0, 0x100, 0x200。

### 2.4.2 示例2：75个中断源

中断表大小 (75 + 16) x 4 = 364 (0x16c)。向上round_up到512bytes。因此向量表的基地址可以是 0, 0x200, 0x400。

### 2.4.3 bootloader和vector关系

在一些控制器中会有很多的程序存储空间，例如bootROM和flash。很多程序都会从bootROM启动。当控制器启动之后，会执行bootROM中的程序，执行完毕之后会branch到用户的应用程序。如图所示Vector table relocation in devices with boot ROM and user flash memory

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202304271024077.png" width="90%" /></div> 

* bootROM启动，使用bootROM中的向量表；
* 执行bootloader中的程序
* 编写VTOR寄存器，替换向量表的地址为用户的定义的向量表的地址；
* branch到用户向量表的reset中断的handler程序；

此时不经有人会疑问，ARM处理器进入reset状态时候会是handler特权模式，如果应用程序的main入口在reset handler中，那岂不是用户的应用程序也要执行在特权模式？根据ARM的手册：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202304280917908.png)

https://developer.arm.com/documentation/dui0056/d/handling-processor-exceptions/reset-handlers

processor mode需要在reset handler里面做更改的。

### 2.4.4 Load Ram和Vector关系

一些处理器，程序会被从外部内存（SD卡，甚至是网络）拷贝到RAM执行。在这种情况下，bootloader程序还要初始化这些外部内存。

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202304280919760.png" width="90%" /></div> 

* 初始化硬件并且拷贝程序到RAM中
* 编写VTOR放入RAM的地址（vector存储）
* 切换到RAM的reset handler。


## 2.5 NVIC中断嵌套

ARM是支持中断嵌套的， 那就是在一个中断发生期间，又来了一个中断，从这个功能上我们很容易想到，要实现嵌套功能至少：

* 中断必须有方法能够被使能和关闭；
* 中断优先级必须能够配置（否则不知道谁嵌套谁）；
* 中断必须能够被pending和保存当前中断的现场信息（状态，返回地址等）；
* 中断必须能够被激活并且恢复中断现场信息；

为了能够实现上面的功能，NVIC通过寄存器编程的方式，来设定：
* 中断enable bit位
* pending 状态位
* read only激活查询位

处理器可以接收中断请求：

* pending状态被设定；
* 中断使能中；
* 优先级高于当前执行中断。

处于pending状态的中断，是等待处理器响应该中断。在一些实例中，有些处理中断一变成pending状态，处理器立刻处理该中断；有些则是处理器会处理更高等级的中断。这些就有区别于传统的ARM处理器，例如中断请求IRQ/FIQ必须持住中断请求直到处理器来处理它。而现在Cortex-M使用NVIC来处理器中断，只需要设定中断的状态机为Pending，处理器将会根据规则处理中断。

如图所示，当一个处理器开始处理中断请求，中断的pending状态将会被自动的清除掉，然后再去进入相应的handler mode去处理中断。

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202305081012314.png" width="90%" /></div> 

当处理器开始处理中断，中断就处于激活状态。请注意，在中断的入口时序中，多个寄存器的值会被push进入到栈中。**这个过程叫做stacking**。于此同时，ISR的入口地址会被从vector table中取出来。

在很多控制器的设计中，外设的中断通常是由一段时间的电平触发，这种状态下，在ISR中就不得不手动地清除中断请求。例如，当中断服务完成之后，处理器要从异常状态返回，寄存器的值要自动的从stack里面pop出来。

当一个中断被激活的时候，你不可能再去释放一个相同的中断请求，需要等到上一个中断完成或者从异常状态返回。

pending状态存储在pending寄存器里面，并且可以通过软件寻址到这个寄存器。因此，你可以通过软件来干预pending的状态。当处理器正在服务于一个高优先级的中断并且想要的中断pending被清除了，那么这个中断就不会被响应了。

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202305081023222.png" width="80%" /></div> 

如果一个外设中断频繁地被assert，并且软件尝试去清除pending状态，那么这个pending会被下一次中断覆盖掉。

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202305081024943.png" width="80%" /></div> 

如果一个中断在它被服务完之后，中断还不断的去assert。这个中断会再次进入到pending状态，并且处理器还会去处理它。

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202305081025321.png" width="80%" /></div> 

对于周期性的脉冲中断请求，在处理器开始处理之前，这些请求被认定为只有一个请求。

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202305081027688.png" width="80%" /></div> 

一个中断的pending状态恰好被再一次设定当处理器正在服务（因为pending状态在服务之前就会被处理器自动清除），处理器还会进行处理。

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202305081029251.png" width="80%" /></div> 

请注意，pending状态是可以被设定的，即便是这个中断处于disable状态。当这个中断被使能的时候，那么就满足了处理器处理中断的条件就会被处理。换句话说，切换中断的开关不影响pending状态。如果这样的话，需要十分小心，当开启中断之前，需要将中断的状态机手动清除掉。

## 2.6 异常处理过程

Cortex-M的异常处理，相比于ARMv8比较简单。 [10_ARMv8_异常处理（一） - 入口与返回、栈选择、异常向量表](https://github.com/carloscn/blog/issues/47#top) 没有EL等级的切换。

这一个小结从四个步骤来讨论M核的异常处理的过程，分别是：
* 进入异常的条件
* 异常入口
* 异常handler
* 异常返回

### 2.6.1 异常条件

当处理器满足以下所有的条件的时候，会激发进入异常：
* 处理器正在running状态（不是halted或者reset状态）
* 异常被使能（NMI和HardFaul异常比较特殊，它们always使能）
* 当前assert的异常优先级高于现在执行的优先级
* PRIMASK寄存器中没有mask当前的异常

**注意`SVC`异常，如果SVC指令用于一个异常handler服务函数中，而这个异常handler的优先级大于等于SVC异常的优先级，那么此时会有HardFault异常产生**。

### 2.6.7 异常现场保护

过程如图所示：

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202305090849970.png" width="40%" /></div> 

#### stacking registers

进入异常的时候需要记录中断现场，包括返回的地址，选择的栈是什么。如果当前处理器处于Thread mode并且使用Process Stack Pointer（PSP）栈指针，此时记录中断现场则使用PSP栈指针。否则，则使用Main Stack Pointer （MSP）栈指针。Cortex-M是双栈指针结构，具体可参考：[02_ARMv7-M_编程模型与模式](https://github.com/carloscn/blog/issues/123)

#### 取中断向量和执行handler

需要拿到ISR函数的起始地址，这个过程可以和上一步并行完成，关于怎么取这个地址，可以参考 **2.4 中断向量**。

拿到入口地址之后，就从该地址load中断handler的指令。

#### 更新寄存器

寄存器已经被备份了，备份的原因也是异常需要使用这些寄存器，在执行完上述操作之后，需要更新NVIC寄存器和一些核心寄存器。需要包含：

* pending状态
* 激活状态
* PSR（Program Status Register）
* LR （Link Register）
* PC（Program Count）
* SP（Stack Pointer）

对于栈指针，MSP和PSP的使用，还是取决于进入异常时候的状态。对于PC，是跟随exception handler的；对于LR，会被注入一个非常特殊和重要的值`EXC_RETURN`，用于记录异常的返回地址。

### 2.6.8 handler执行

在handler中可以实现对于外设的一些业务逻辑。**当执行ISR的时候，此时此刻处理器已经处于Handler mode**。在Handler Mode：

* 无论保存现场的栈指针是什么，都要切换MSP，使用主栈指针来操作；
* 处理器进入特权模式

当更高优先级的异常在执行当前的handler的时候出现了。新的异常将会被打断，这个叫做中断的嵌套。如果另一个优先级等于或者小于当前的优先级，此时新的中断状态机为pending状态，等待该中断完成之后才能切换到新的中断。

在异常handler结束之后，异常应该返回，此时从LR寄存器中读出返回地址，并且加载到PC中，返回主函数。


### 2.6.9 异常返回

在一些处理器架构中，会使用特殊的指令用于异常的返回。然后这就意味着异常handler不能使用C函数来编写。在ARM M系列的处理器中，异常返回机制被一个特殊的返回地址`EXC_RETURN`触发。这个值是在进入中断入口的时候，被保存在LR寄存器中产生的。当这个值被**一个允许异常返回的指令**写入到PC寄存器的时候，此时就会触发异常返回操作。

一个允许异常返回的指令，被列在下面：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202305090908985.png)

当异常返回机制被触发的时候，处理器会将之前在入口备份到stack的registers的值恢复，这个有个术语叫做Unstacking。除此之外，NVIC寄存器，和上面提到的寄存器也会被更新。Unstacking操作和恢复中断现场的过程可以并行执行的。

用于异常返回的`EXC_RETURN`值可以用C语言写入。在程序中，C编译器会处理`EXC_RETURN`值。由于`EXC_RETURN`值的机制的设定，因此就不可能有一个正常的功能能够返回到0xF0000000 到 0xFFFFFFFF地址。然后，因为架构指定这些地址范围不能用于code属性，（XN Execute Never属性），所以这也没什么关系。

## 2.7 中断寄存器配置

### 2.7.1 NVIC寄存器

### 2.7.2 SCB寄存器

### 2.7.3 特殊寄存器


## 2.8 经典案例

## 2.9 软中断

## 2.10 注意事项


# 3. 异常处理


## 3.1 异常处理综述


## 3.2 异常序列


## 3.3 中断延迟和异常处理调优