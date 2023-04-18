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

大致上分为：

* Code区域（低地址：0-1FFF_FFFF）：512MB的空间，包含默认的向量表和部分程序。这部分允许数据访问（写：写入程序；读：CPU读指令）；
* SRAM（2000_0000-3FFF_FFFF）：512MB的空间，通常是on-chip RAM，可以从该位置执行程序；
* 外设（4000_0000-5FFF_FFFF）：512MB的空间，通常是**on-chip 外设**区域；
* RAM（6000_0000-9FFF_FFFF）：1GB的空间，off-chip内存，可以存储数据，也可以执行程序；
* Devices（A000_0000-DFFF_FFFF）：512MB空间，**off-chip外设**区域；
* System（E000_0000-FFFF_FFFF）：系统区域
	* IPPB（内部私有外设总线）用于访问系统组件例如NVIC和SysTick和MPU等。
	* EPPB（外部私有外设总线）用于下游厂商加入自己的IP

参考STM32F37xxx的手册中的memory map章节，可以看到意法半导体也基本follow ARM的原生设计：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202304140856419.png)

我们通常都会把执行的程序存储到RAM或者SRAM上，CPU在执行的时候从RAM里面读取指令然后执行。不过从memory map上看，显然这个是笨拙的，因为要花费额外的时钟周期从RAM中读取指令，远不如从地址总线的内存区域直接读取。但是要知道，地址总线映射区域并不是存储空间，是无法执行程序的。因此，这里就引出了地址属性的概念。对于NVIC与MPU和SCB，这些系统外设占用的空间叫做System Control Space (CSC)。

一些系统组件如图所示：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202304140849293.png)

# 3. 内存+外设与处理器如何连接

ARM使用AMBA来对内存和外设进行连接，AMBA中还包含多种总线协议，应对不同场景，不同的处理器访问对象。

* AMB Lite Protocal：干路总线的接口
* APB Protocal： 用于PPB，访问调试组件

![](https://raw.githubusercontent.com/carloscn/images/main/typora202304140859417.png)

为了获取更好的性能，CODE区域和其他的总线访问分开，这样的意图也是数据访问和取址能够并行执行。

还有一个好处就是，分开的总线能够提高中断的响应效率，试想，在应用层经常出现，在中断处理一个queue的时候，处理器需要进入栈空间的访问vector table读取中断的地址信息，如果总线分开之后可以并行进行处理。

处理器的总线接口有一个等待机制，这个等待机制就位不同速率的存储提供了一个可能，例如处理器处于高速运行状态，当访问速率较低的内存或者速度较慢的外设的时候，该等待机制可以让处理器从总线接口获取等待数据的状态，因此可以引入更多的逻辑来处理高低速不平衡的use case。

处理器总线还有一个error response机制，在总线层级就可以检测内存的合法性，如果访问一个非法区域的空间，总线就会返回一个错误，无需处理器来做判断处理，只需从总线拿到错误汇报即可，减少了CPU的负担。

以上的好处都是得益于总线协议。一个简单的处理器，访问指令空间使用I-CODE和D-CODE总线，而SRAM和外设都接入到系统总线上，如图所示：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202304140913231.png)

我们可以从这个图看到，低速的外设通过总线桥（APB）接入到系统总线（AHB Lite），高速器件都是直接接入到系统总线（AHB Lite）上。这样做的目的是对于不同速率的外设访问一致性的优化，可以最大程度的区分开高速和低速器件，使访问效率以及带宽达到最高。例如DMA属于高速器件，DMA直接使用系统总线，对于USB和网卡的高速数据，都是使用系统总线进行传输。

还需要注意PPB不能用于不同的外设，PPB更像是ARM自己使用的简单高效的工具，但是没有太强的保护措施，所以相比于外设的总线：
* PPB 是CPU私有化的访问；
* 只能够32位访问，无法使用bit级的访问粒度；
* PPB写入需要更多的时钟周期，而且没有写入的缓冲区（针对于[Strongly-Ordered](https://github.com/carloscn/blog/issues/62)设备）；
* 只能是小端访问，外设总线可以配置为小端或者大端模式；

I-CODE和D-CODE总线提供程序的访问。在比较简单的处理器demo中（ARM的设计），两个总线能够被一个BUS MUX混合到一起。二级厂商也能够使用这两条总线来开发定制化flash的访问的加速器，这样允许处理器更快的访问flash memory。

![](https://raw.githubusercontent.com/carloscn/images/main/typora202304140932854.png)

对于意法半导体的Embedded Flash memory的设计，就是用了该方法：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202304140934387.png)

![](https://raw.githubusercontent.com/carloscn/images/main/typora202304140934877.png)

在很多M3和M4的产品中，有很多设计能够找到muliple bus masters的身影，例如DMA，以太网和USB这些高速器件。二级厂商喜欢用“bus matrix” or “multi-layer AHB”来描述。如图所示：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202304140937618.png)

# 4. 内存要求

不同种类的内存能够使用相应的内存接口逻辑和AHB Lite总线连接。尽管总线是32位访问宽度，但是访问的时候还是可以以不同的位宽进行访问，例如（8/16/64/128等）。

尽管市面上有很多种类的RAM，比如SRAM和RAM或者是PSRAM，SDRAM，DDR RAM。对于处理器而言，对于RAM的类型没有限制。针对于ROM空间也是，无所谓FLASH、EEPROM还是OTPROM。只有一点，访问的内存必须是可以**寻址的**，而寻址的宽度必须要支持byte，半word，一个word这些宽度的访问。


# 5. 大小端

Cortex-M3/M4支持大端模式也支持小端模式。在默认情况下，使用的是小端模式。但在一个power-cycle只能有一种模式的存在。换句话说，处理器在复位的时候会决定自己是在小端模式还是在大端模式，一旦被设定，大小端模式就不能被更改，除非处理器复位。

在极少数的use case中，一些外设寄存器，混合使用大小端模式。在这种case中就需要软件工程师在编写程序时，通过软件的手段妥善处理数据，对数据进行大小端的转换，让其从寄存器中读取到正确的数据。

通过union来确定是不是大端：

```c
#define IS_BIG_ENDIAN (!(union { uint16_t u16; unsigned char c; }){ .u16 = 1 }.c)
```

转换：

```c
uint32_t ntohl(uint32_t n)
{
    unsigned char *np = (unsigned char *)&n;

    return ((uint32_t)np[0] << 24) |
        ((uint32_t)np[1] << 16) |
        ((uint32_t)np[2] << 8) |
        (uint32_t)np[3];
}
```

以下是存储32位的大端小端在内存中存储的字节序：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202304170940596.png)

![](https://raw.githubusercontent.com/carloscn/images/main/typora202304170940060.png)

在M处理器中，大端的存储形式被叫做`byte-invariant big endian, or BE-8`被v6, v6-M, v7, and v7-M所支持。在经典ARM处理器中，大端的存储形式被叫做`word-invariant big endian, or BE-32.` 他们在内存排布的方式是一致的，区别就在于AHB总线传输方面。 具体参考： https://developer.arm.com/documentation/ddi0344/k/programmers-model/memory-formats/byte-invariant-big-endian-format

>[关于ARM大小端模式和CPU有关还是编译器有关](https://segmentfault.com/a/1190000041436540)
> 结论，ARM大小端模式和CPU有关也和编译器有关系。
> 
> ARM默认状态配置为小端模式，编译器不指定编译模式也默认是小端模式。但有些ARM是可以配置为大端模式的。例如：
>-   **ARMv7-A**: In ARMv7-A, the mapping of instruction memory is always little-endian.
>-   **ARMv7-R**: SCTLR.IE, bit[31], that indicates the instruction endianness configuration.
>-   **ARMv7-M**: Armv7-M supports a selectable endian model in which, on a reset, a control input determines whether the endianness is big endian (BE) or little endian (LE). This endian mapping has the following restrictions: The endianness setting only applies to data accesses. Instruction fetches are always little endian. All accesses to the SCS are little endian, see System Control Space (SCS) on page B3-595.
>-   **ARMv8-aarch32**: aarch32: When using AArch32, having the CPSR.E bit have a different value to the equivalent System Control register EE bit when in EL1, EL2, or EL3 is now deprecated
>-   **ARMv8-aarch64**: aarch64: This data endianness is controlled independently for each Execution level. For EL3, EL2 and EL1, the relevant register of SCTLR_ELn.
> 
> 如果在ARM上面配置了大端模式，gcc编译器则需要增加参数`-mbig-endian`
>
> -mlittle-endian
Generate code for a processor running in little-endian mode. This is the default for all standard configurations.
> -mbig-endian
Generate code for a processor running in big-endian mode; the default is to compile code for a little-endian processor.
> 与CPU配置保持一致。
>
>[https://gcc.gnu.org/onlinedoc...](https://link.segmentfault.com/?enc=2ImG%2BQLTfPBXDupV0zBoZQ%3D%3D.MoB34JArRb%2BcVHg5mCGQRfu0BXlYcs0qQQG2jAAXAh5x1ip9lDMlB3knHIKtQzYvF9UeX7nW%2FNWFCOR3YIdRsw%3D%3D)  
>[https://developer.arm.com/doc...](https://link.segmentfault.com/?enc=QZ4DxwZBWHzorAJEHiOGfw%3D%3D.Aob7LJBgxkA%2F%2BDjuVRBOOzvI3W1pkMjPD2NDCgTF6sq3txrkiMKXC%2FWwOqJ93PxWsrAmkUw%2Femm0QI0%2Butb85D6fp5wY35%2Bt6CQErG%2BEV4m%2Bj1XrIQWDcGM597kUvwbL)  
>[https://developer.arm.com/doc...](https://link.segmentfault.com/?enc=6o1LVkEIhEKKNporfS9mRA%3D%3D.r1m8NmOoAGzJ7qOXzuTQNPTF3dzzGT6FKoX1pYSEc4yIY0RPWM0TMBNPdist9OjoD%2Bx6F%2BYg3PzpY1mfjCyTcU9OS4LHOhnCENGfwh9X9YEjTEIz%2Bw83DDsxTet7J3EaWvzg%2F5Mr3s3iesxv%2F%2B%2BOhTn7IptLRUo98KekGdPlQ%2BMCVoZbp9eCeJzPK2CMhlykatz9%2BKnBIklKs1nlm10cHA%3D%3D)

对于M系列的CPU：
* 处理器取指令总是使用小端模式；
* 访问E000_0000 - E00F_FFFF的 `System Control Space (SCS)`，调试模块还有PPB总线总是使用小端模式；
* 指令REV, REVSH, and REV16可以转换大小端数据；

# 6. 数据对齐和非对齐访问

内存系统是32位的，则一个数据的访问可以是32位的（4字节，或一个字），或者16位的（2个字节，半个字）。从编程者的角度，这个数据的访问可以是对齐的，也可以是非对齐的。对齐的数据传输意味着地址是字节大小的倍数。例如，一个32位的数据传输能够被搬移到地址 `0x0000_0000`， `0x0000_0004`，... ， `0x0000_1000`（这些地址都是32位对齐的，可以整除32）；一个16位的数据可以被移动到`0x0000_0000`，`0x0000_0002`，`0x0000_1000`，`0x0000_0002`。

下面的图表示非对齐和对齐数据传输的实例：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202304180832360.png)

在经典的ARM处理器中，**仅仅允许对齐的数据传输**。从数据地址规律来看，32位的对齐数据，地址1比特位和地址0比特位都是0；一个16位的数据地址0比特位是0。（ARM就是通过这个方法来检测数据对齐异常的）。

Cortex-M3和Cortex-M4在架构层级就支持非对齐的数据传输，使用`LDR,LDRH,STR,STRH`指令就可以完成。但是这些指令也有一些限制：

* 非对齐的访问不支持Load/store多条指令；
* Stack的操作（PUSH/POP）必须对齐；
* 独占访问（Exclusive accesses）`LDREX`和`STREX`必须对齐；否则 ，就会发生异常；
* 非对齐的数据不支持bit-band级别的内存操作

还需要注意的是，在处理器层面，对于非对齐的操作本质上是拆分成多个对齐的操作进行处理。在软件层面的确是不需要关心数据是否对齐了，然而，对于处理器而言，非对齐数据需要增加额外的开销，因此，如果要达到最好的性能，需要让数据对齐。

在编译器层面，编译器会对数据进行优化，不去产生非对齐数据访问。因此非对齐访问发生在：

* 直接使用指针操作数据；
* 访问一个数据结构体`__packed`属性，且这个结构体中包含非对齐的数据；
* Inline 内嵌汇编程序。

我们也可以对M处理器进行配置，让其不支持非对齐数据的访问，行为是当发生非对齐的内存访问，就会发生一个非对齐的异常。通过`UNALIGN_TRP` (Unaligned Trap) 比特位在CCR上面（Configuration Control Register，0xE000_ED14）在SCB上。
