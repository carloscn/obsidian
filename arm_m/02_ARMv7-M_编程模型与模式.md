# 02_ARMv7-M_编程模型与模式
ARM有个专业术语叫做Programmer's model，可能对这个概念很陌生，也无法解释，那么我们把它视为一个词根。一个model中包含：

* 寄存器有哪些（通用的+特殊的）
* ARM的执行状态有哪些

我们对于ARM的控制，抛开C语言，最终都转换成汇编语言，汇编是ARM的指令，指令操作需要依赖寄存器，ARM从寄存器中得知需要有什么行为，例如我们的运算，拷贝。因此编程者模型更多的是站在汇编的层面来讲故事。而我们需要关注的是，我们直接操作的寄存器，会对ARM产生什么行为。ARM设计了什么寄存器？这些寄存器的使用条件是什么？

以下编程模型来自于arm的白皮书：

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202305100933677.png" width="80%" /></div> 

M系列的CPU的P model设计上都保持了一致性。例如，在寄存器方面`R0-R15`，`PSR`，`CONTROL`和`PRIMASK`都保持了一样的设计。但是也有一些不同的地方，例如，两个特殊寄存器，`FAULTMASK`和`BASEPRI`只在M3/M4/M7/M33存在，floating point bank寄存器和`FPSCR`(Floating Point Status and Control Register)只在Cortex-M4/M7/M33上存在。

在ARMv7-M架构中，中断非常多，通常都有相应的优先级配置，`BASEPRI`寄存器主要用于阻塞某些特定优先级或者低优先级的中断和异常。而在ARMv6-M或者ARMv8-M中，中断的优先级被限制到了4个可配置的等级中。`FAULTMASK`用于处理复杂的handler。

非特权模式这个概念在ARMv6-M架构上是可以存在或者不存在的，而在v7-m和v8-m上是一定会有这个概念。这种差异意味着CONTROL寄存器在不同的Cortex-M处理器之间可能有微小的差异。FPU的配置选项也影响CONTROL寄存器。

CONTROL寄存器如下：

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202305111026711.png" width="80%" /></div> 

可以看出，关于M核心的执行状态问题，都是使用CONTROL寄存器来进行配置的。

PSR(Program Status Register) 在M核之间也有不同。在所有的Cortex-M中，PSR能被分为：
* Application PSR
* Eexecution PSR
* Interrupt PSR

![](https://raw.githubusercontent.com/carloscn/images/main/typora202305111030293.png)

# 1. Programmer's model

## 1.1 Model

### 1.1.1 Operation Mode 与 states

M3/M4处理器有两个执行状态，相应的有两种模式。除此之外，处理器有特权和非特权访问等级。特权访问level可以访问处理器上的所有的资源，非特权访问意味着一些内存区域，或者一些指令操作不能被执行。在一些文档中，unprivilege模式还被称为“user”模式，这个术语来源于classic ARM，例如ARM7TDMI。

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202305151010921.png" width="80%" /></div> 

#### Operation States

执行状态分为Debug State和Thumb state。

* Debug State：当处理器被halted，例如被调试器或者遇到断点的时候。此时arm就会进入到debug状态或者停止执行指令的状态。
* Thumb State：如果处理器正在运行程序（Thumb指令）此时处理器就处于Thumb状态。不像是Classic ARM处理器，没有ARM state，因为M处理器不支持ARM指令集。

#### Operation Modes

* Handler Mode：当一个异常handler被执行的时候，例如处理一个中断的ISR程序。处于handler mode的时候，处理器总是privileged访问权限；换句话说，中断处理过程中，执行的指令可以访问处理器上的任何资源。
* Thread Mode：当执行正常的main loop中的程序的时候，处理器可能处于privileged访问权限也可能处于unprivileged访问权限。怎么界定？通过控制一个特殊寄存器CONTROL来控制在Thread mode下执行的权限。

注意：在Thread mode下，软件能够在特权模式切换到非特权模式（特权->非特权）。然而，无法从非特权升级为特权模式。**如果要在Thread模式下申请升权，需要用到异常机制来完成这个操作**。

ARM这样设计的初衷也是为了安全考虑，基于一个基本的安全模型，系统的设计者可以利用保护机制对于特殊的敏感的内存区域加上访问限制，以开发出健壮的嵌入式系统。怎么说呢？硬件的机制其实对标的是操作系统机制，操作系统中有kernel权限和user权限的区分。例如：[0x21_LinuxKernel_内核活动（一）之系统调用](https://github.com/carloscn/blog/issues/69) 。 理所当然的，用户空间是用非特权模式，内核空间使用特权模式。同时，ARM上引入MPU机制来阻止应用程序访问内核空间的数据或者外设。当发生这种访问的时候MPU会抛出异常，我们可以设定异常处理来做一些补救措施，让其ARM恢复到正常的工作状态，而不是当一个应用层级的程序访问了内核空间就让整个处理器陷入不可恢复的异常状态。这就大大的提高了健壮性和安全性。

以上是从内存引发的一些思考。除此之外，对于特权和非特权模式，还有指令层级的划分。一些指令的设定只能是在特权模式执行，这个是为什么呢？上面谈到，指令的本质是依托寄存器进行一些操作。因此，一些寄存器只能是在特权模式进行访问。例如NVIC寄存器只能是特权模式才可以访问的。

借助于特权模式和非特权模式隔离的设计思想，同样的，thread模式和handler模式也有类似的编程模式。Thread模式能够切换使用一个独立的shadowed Stack Pointer（SP）。这样就有助于把kernel的执行现场和应用的执行现场分开。怎么说？比如，low-memory是user的，high-memory是kernel的，我们当前正在执行一个应用，此时SP栈指针指向的是user空间的内存，当异常发生的时候切换到kernel也就是handler模式，我们需要把SP的栈指针地址备份到Kernel空间，然后把SP内的内容放入kernel的高内存的地址。这个过程就增大了模式切换的开销。如果SP独立化，根本不需要做这个备份，直接回归到应用的SP的位置执行就好了。

默认状态，M核处理器上电就处于特权模式，这个和ARMv8设计一致，开始则为最大可访问权限然后逐步降级，这样也是为了初始化资源方便。在一些简单的应用场景（并非需要OS调度的）就不需要去降级到非特权模式，使用普通的SP指针即可。**注意M0+没有特权模式**。如图所示：

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202305151044134.png" width="80%" /></div> 

还需要注意debug state。这个状态进入的点在于被debugger控制和停止了ARM处理器的运行。这个状态开放了寄存器值的外部读取和修改。

### 1.1.2 Registers

在M3/M4中ARM执行数据处理和控制依托于寄存器。这些寄存器组合成一个单元叫做register bank。参考[^1]，register bank里面包含很多register，

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202305160844392.png" width="80%" /></div> 

在ARM的数据处理过程中，涉及数据处理就要频繁的访问寄存器，至少有一个原寄存器和目的寄存器。数据处理的过程通常是把数据从内存中加载到寄存器组的寄存器，数据处理完毕之后，需要把数据从寄存器中回写到内存中，这种架构通常称为“load-store”架构。这种情况也提高了运算的效率，因为数据在寄存器内进行运算不需要频繁的通过总线访问内存。但也需要注意，寄存器属于CPU内部的机制，在多核的SoC中，或者有DMA的存在，多个访问者挂载共用的内存中时候，这个时候如果两个CPU都在自己的寄存器中进行运算，那么就会发生数据错误。因此，我们需要用C语言的时候增加volatile关键字，告诉编译器，这个值访问的时候需要从memory中重新load。参考[Compiler optimization and the volatile keyword.md](https://gist.github.com/carloscn/354c7b91e49fa44110dafa1b8b2776c3)

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202305160853868.png" width="80%" /></div> 

M3和M4的寄存器组有16个寄存器，他们中的13个是通用寄存器（R0-R12），剩下的是特殊寄存器（R13-R15）。

#### R0-R12

R0-R12寄存器是32位的通用寄存器。R0-R7也叫做low registers。由于指令集对于空间的限制，很多16-bit指令是可以访问low register的； 后8个寄存器 high register（R8-R12）能够被32-bit的指令访问和部分16-bit执行访问，例如MOV。要有一个意识，R0-R12上电之后的默认值是没有定义的（随机）。

#### R13-SP

R13是栈指针寄存器。栈指针寄存器主要适用于访问stack内存的，和栈指针寄存器打交道的指令也是PUSH/POP指令。[07_ELF文件_堆和栈调用惯例以ARMv8为例](https://github.com/carloscn/blog/issues/50#top) 在 1.2 不同架构出栈和入栈 中展了A32指令的PUSH和POP在函数调用过程中的作用。

在ARM核的设定中，有两个不同的SP指针，一个是MSP（Main Stack Pointer）主要用于handler模式的栈指针，是上电后默认的指针；还有一个是Process Stack Pointer，用于thread模式。ARM选择哪个栈指针，是用过配置CONTROL寄存器进行的。

两个SP指针都是32位的。SP的最低的2个比特位总是，如果对这两个位强行写入的话总是被置为0，换句话说，SP永远是4 bytes aligned。在M处理器中，PUSH和POP指令操作的永远都是32位的，传输数据也必须是32位对齐的。

在大多数的应用场景中，如果没有嵌入式操作系统的话，没有必要使用PSP。所有的操作全部在内核空间即可。PSP使用的场景经常是需要引入OS kernel和任务分开。PSP的初始值undfined，MSP的初始值通常是reset handler的地址。




# Ref
[^1]:[Chapter 2: Registers, Register Banks, Memory and Arithmetic-Logic Units]( http://www.marmaralectures.com/chapter-2-registers-register-banks-memory-and-arithmetic-logic-units/)