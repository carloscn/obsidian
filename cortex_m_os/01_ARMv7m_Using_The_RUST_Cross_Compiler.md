# 0. 导语

今年的study-2023计划中，包含了rust和Cortex-M处理器架构的学习。截止到目前为止RUST已经学的七七八八，偶然间找到了RUST编写简易操作系统的博客[https://os.phil-opp.com/](https://os.phil-opp.com/) 大为震撼。该博主基于x86架构的芯片使用rust编写了操作系统，里面包含了，中断异常处理、内存分配、还有任务管理的一些机制。受到该博主的启发，或许我们可以基于Cortex-M的同步进行开发。这样做的好处，我们既可以熟悉rust，又可以熟悉Cortex-M，又结合了操作系统的知识，可以说一举三得。我相信理论基础的学习也仅仅是第一步，自己动手编写和调试才能有很深刻的掌握。

我不打算把rbp-os作为一个要交付的“产品”，它不会是一个从顶层到底层的的设计方式。它更像是一种“乐高积木”，一个系统要素一个要素的去实现。在自定义的操作系统中，我们可以随意的增加自己的系统模块。我们按照博主的思路，先去做一个最小的内核，然后慢慢开发中断异常的处理，后续增加内存MMU管理，heap分配器，而这些机制可能一开始我们做一个能够“work”的最简单的实现方式，后续我们慢慢的增强，借鉴Linux内核的处理机制不断的去完善更复杂的场景。

本文会大量的引用：

* Linux操作系统或者FreeRTOS中的操作系统机制来解决baremental的问题
	* https://github.com/carloscn/blog#rtos
	* https://github.com/carloscn/blog#linux-kernel
* 基于Cortex-M的架构级的机制
	* https://github.com/carloscn/blog#armv7-m-cortex-m

本节中会涉及大量关于嵌入式应用方面的知识，硬件架构知识和系统模型分别在上述连接中进行引用。


# 1. 嵌入式平台

## 1.1 硬件基本介绍

参考 https://stevenbai.top/rustbook/book/intro/hardware.html 的博客，这里提供了很基础的硬件思路，我们将借助该文档对Cortex-M的rust开发环境进行入门级的整理。在博客中使用了[STM32F303VCT6](https://www.st.com/en/microcontrollers/stm32f303vc.html) 微控制器。我自己也买了一个这个平台的微控制器：

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202304121022391.png" width="70%" /></div> 

该控制器基本的feature为：
* 单核**ARM Cortex-M4F**处理器（单精度浮点数）参考：https://developer.arm.com/Processors/Cortex-M4
* 最大时钟频率为72 MHz
* 256KiB的Flash
* 例如定时器，I2C，SPI和USART。
* 通用输入输出(GPIO)和其他类型的引脚
* 一个USB接口 标有"USB USER"的USB端口。

以下是Cortex-M4的一些IP资源：

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202304121041115.png" width="70%" /></div> 

板子的bsp和文档下载地址： https://www.st.com/en/evaluation-tools/stm32f3discovery.html


## 1.2 Compiler

### 1.2.1 C programming 

https://developer.arm.com/Tools%20and%20Software/Arm%20Compiler%20for%20Embedded

The following diagram shows how the different toolchain components interact with each other in a typical embedded application build process:

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202304121042757.png" width="70%" /></div> 

### 1.2.2 LLVM基础设施和RUST

LLVM（**L**ow **L**evel **V**irtual **M**achine）提供了一套适合编译器系统的中间语言（Intermediate Representation，IR），有大量变换和优化都围绕其实现。LLVM是许多编程语言后端引擎。 它被C、C++、Rust、Go和Swift等使用[^1]。

编译过程可以分为三个部分。 前端、优化、后端。 编译过程从前端开始。 首先，预处理器开始组织源代码。 这是外部库扩展到目标源代码的阶段。 在Rust中，该指令由use语句定义。 C和C++中的一个类似指令是#include语句。其次，是解析。 在此阶段，代码会被解析，从而发现语法错误，并构建[抽象语法树 (AST)](https://en.wikipedia.org/wiki/Abstract_syntax_tree?utm_source=ttalk.im&utm_medium=website&utm_campaign=Tech%2520Talk)。 前端编译的第三步是IR Generation。 在这一步，编译器将AST转换为中间代码（IR）并输出结果。**在优化阶段**，编译器对程序执行各种转换和清理。 这提高了程序性能，通过减少各种Bug的产生使程序更安全可靠，同时运行一些辅助程序完成一些工作。 稍后我们将探索IR并查看编译器附带的一些优化过程。程序优化后进入后端阶段。 在这，Compiler Back-Endd将IR转换为特定于目标的汇编代码。 然后汇编程序将特定于目标的汇编代码转换为特定于目标的机器代码。 最后，Linker将多个机器代码文件组合成一个文件，我们称之为可执行文件。

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202304121117279.png" width="80%" /></div> 

目前我们已经有几种特定于语言的前端。我们有C和C++的clang，Go有gollvm，而Rust有一个叫做rustc的编译器。

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202304121121908.png" width="90%" /></div> 

#### RUST工具链

安装rustup: 
```
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

rustup的默认安装仅支持本机编译，因此需要添加对ARM Cortex-M的交叉编译支持。对于STM32F3DISCOVERY这个本书示例使用的开发板，请使用target "thumbv7em-none-eabihf"。

`$ rustup target add thumbv7em-none-eabihf`

还需要安装cargo-binutils

```bash
$ cargo install cargo-binutils 
$ rustup component add llvm-tools-preview
$ cargo install cargo-generate
```

#### no_std[^2]

Cortex-M是一个裸机环境。在裸机环境中，系统在运行你的代码之前,没有未加载任何代码。因为没有操作系统的支持，我们将无法使用标准库。 相反，程序及其使用的crate只能直接使用硬件(裸机)来运行。为了防止Rust加载标准库，必须使用`no_std`[^3]。可通过[核心库](https://doc.rust-lang.org/core/)获得标准库中与平台无关的部分。核心库还排除了嵌入式环境中并不总是需要的东西。其中之一是用于动态内存分配的内存分配器。如果您需要此功能或任何其他功能，通常会有第三方crate实现。

`#![no_std]` 是一个crate级属性，指示该crate将链接到核心库，而不是标准库。[核心库](https://stevenbai.top/rustbook/book/intro/(https://doc.rust-lang.org/core/))是标准库的与平台无关的子集,它不对程序运行的系统做任何假设。它只提供了语言相关(例如浮点数，字符串和切片)的API，以及处理器功能(例如原子操作和SIMD指令)的API。但是，它缺少涉及平台集成的任何东西的API。 由于这些属性，`no_std`和[核心库](https://stevenbai.top/rustbook/book/intro/(https://doc.rust-lang.org/core/))代码可用于任何类型的引导程序(阶段0)代码，例如bootloader，固件或内核。

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202304121131728.png" width="60%" /></div> 

* 仅当您使用`alloc` crate并使用合适的分配器(如[alloc-cortex-m](https://github.com/rust-embedded/alloc-cortex-m))时。
* 仅当您使用`collections`crate并配置全局默认分配器时。

#### 注意rustc的路径

rust工具集可能存在多个安装途径，例如snap和apt安装，他们分别在/snap/bin 或者/usr/bin 目录下面，因此你可能遇到：

```
error[E0463]: can't find crate for `core`
  |
  = note: the `thumbv7m-none-eabi` target may not be installed
  = help: consider downloading the target with `rustup target add thumbv7m-none-eabi`
```

然而，你按照建议安装该工具，又出现了

`info: component 'rust-std' for target 'thumbv7m-none-eabi' is up to date`

![](https://raw.githubusercontent.com/carloscn/images/main/typora202304130852121.png)

解决方案[^4]如下：

>I had installed **rustup** via **snap** in Ubuntu , so it was in **/snap/bin** while **rustc** , **rust-gdb** were installed in **/usr/bin** .  
> According to this installation tutorial , [Install Rust - Rust Programming Language 49](https://www.rust-lang.org/tools/install) all the software was supposed to be in **~/.cargo/bin** which only had **cargo** and **rust-fmt** .  
>So I manually deleted all rust related things from **/snap/bin** . Did **sudo apt remove rustc rust-gdb** . Also , I removed **~/.cargo/** .  
>Then I followed the installation tutorial . Chose the stable installation.  
>Finally I added the target , **rustup add target thumbv7m-none-eabi**

# 2. Hello World

## 2.1 非标准函数

我们将使用[`cortex-m-quickstart`](https://github.com/rust-embedded/cortex-m-quickstart)项目模板生成一个新项目。

使用[`cargo-generate`](https://stevenbai.top/rustbook/book/start/qemu.html#%E4%BD%BF%E7%94%A8cargo-generate)

首先安装cargo-generate

`cargo install cargo-generate`

然后生成一个新项目

`cargo generate --git https://github.com/rust-embedded/cortex-m-quickstart`

```
Project Name: hello_world
Creating project called `hello-world`...  
```

`cd hello-world`

在`.cargo/config.toml`中引入正确的版本

```rust
#![no_std]
#![no_main]

// pick a panicking behavior
use panic_halt as _; // you can put a breakpoint on `rust_begin_unwind` to catch panics
// use panic_abort as _; // requires nightly
// use panic_itm as _; // logs messages over ITM; requires ITM support
// use panic_semihosting as _; // logs messages to the host stderr; requires a debugger

use cortex_m::asm;
use cortex_m_rt::entry;
use cortex_m_semihosting::{debug, hprintln};

#[entry]
fn main() -> ! {
    hprintln!("Hello, world!\n");

    // exit QEMU
    // NOTE do not run this on hardware; it can corrupt OpenOCD state
    debug::exit(debug::EXIT_SUCCESS);

    loop 
```

* `#![no_std]`表示此程序_不会_链接到标准库，而是链接到其子集--核心库。
* `#![no_main]`表示该程序将不使用大多数Rust程序使用的标准`main`接口。使用no_main的主要原因是在no_std上下文中使用main函数需要Rust的nightly版本。
* `extern crate panic_halt;`。这个crate提供了一个 `panic_handler`，它定义了程序的panic行为。
* [`#[entry]`](https://docs.rs/cortex-m-rt-macros/latest/cortex_m_rt_macros/attr.entry.html)是[`cortex-m-rt`](https://crates.io/crates/cortex-m-rt)crate提供的属性，用于标记程序的入口。由于我们没有使用标准的“ main”接口，因此需要另一种方式来指示程序的入口，即 `#[entry]`。
* 注意main函数的签名是`fn main() -> !` ,因为我们的程序是目标硬件上唯一的程序，所以我们不希望它结束​​！我们使用[发散函数](https://doc.rust-lang.org/rust-by-example/fn/diverging.html)(函数签名中的`->！`表示没有返回值)来在编译时确保main不会结束。

`cargo build --target thumbv7m-none-eabi` 编译程序在`target/thumbv7m-none-eabi/debug/app`中有一个非本地的ELF二进制文件。我们可以使用`cargo-binutils`检查它。

### 2.1.1 bin-utils

现在我们在`target/thumbv7m-none-eabi/debug/app`中有一个非本地的ELF二进制文件。我们可以使用`cargo-binutils`检查它。

#### 查看elf头文件

使用readelf 可以查看elf文件段信息，这部分和[02_ELF文件结构_浅析内部文件结构](https://github.com/carloscn/blog/issues/6#top) 完全一致。

![](https://raw.githubusercontent.com/carloscn/images/main/typora202304130945951.png)

#### objdump

使用`rust-objdump -S hello-world` 可以查看汇编信息。

![](https://raw.githubusercontent.com/carloscn/images/main/typora202304130947339.png)


## 2.1 QEMU

参考： https://stevenbai.top/rustbook/book/start/qemu.html

需要安装qemu-arm

```sh
sudo apt install gdb-multiarch openocd qemu-system-arm
```

要在QEMU上运行此二进制文件，请运行以下命令：

```
qemu-system-arm \
  -cpu cortex-m3 \
  -machine lm3s6965evb \
  -nographic \
  -semihosting-config enable=on,target=native \
  -kernel target/thumbv7m-none-eabi/debug/examples/hello
```

![](https://raw.githubusercontent.com/carloscn/images/main/typora202304130949418.png)

* `qemu-system-arm` 这是QEMU仿真器。QEMU支持很多不同的架构。从名字可以看出,这是ARM处理器的完整仿真。
* `-cpu cortex-m3`。这告诉QEMU模拟Cortex-M3 CPU。指定CPU型号可以让我们捕获一些编译参数不当错误：例如，运行针对具有硬件FPU的Cortex-M4F编译的程序，QEMU将在其运行期间产生错误。
* `-machine lm3s6965evb`。这告诉QEMU模拟LM3S6965EVB，这是一个包含LM3S6965微控制器的开发板。
* `-nographic`。这告诉QEMU不要启动其GUI。
* `-semihosting-config(..)`。这告诉QEMU启用半主机。半主机使仿真设备可以使用主机stdout，stderr和stdin并在主机上创建文件。
* `-kernel $file`。这告诉QEMU在模拟机上加载并运行哪个二进制文件。

输入这么长的QEMU命令太麻烦了！我们可以设置一个自定义运行器以简化过程。`.cargo/config`有一行启动 QEMU的运行器被注释掉了,让我们去掉这行注释：

`head -n3 .cargo/config.tmol`

```
[target.thumbv7m-none-eabi]
# uncomment this to make `cargo run` execute programs on QEMU
runner = "qemu-system-arm -cpu cortex-m3 -machine lm3s6965evb -nographic -semihosting-config enable=on,target=native -kernel"

```

`cargo run target/thumbv7m-none-eabi/debug/hello-world`


## 2.3 调试

调试的第一步是在调试模式下启动QEMU：

```
qemu-system-arm \ -cpu cortex-m3 \ -machine lm3s6965evb \ -nographic \ -semihosting-config enable=on,target=native \ -gdb tcp::3333 \ -S \ -kernel target/thumbv7m-none-eabi/debug/examples/hello
```

此命令不会在控制台上显示任何内容，并且会阻塞终端。这次我们额外传递了两个参数：

-   `-gdb tcp::3333`。这告诉QEMU监听TCP端口3333,等待GDB的连接。
-   `-S` 这告诉QEMU在启动时冻结计算机。没有这个，可能我们还没有来得及启动调试器,程序就已经结束了！

接下来，我们在另一个终端中启动GDB，并告诉它加载示例的调试符号：

```console
gdb-multiarch -q target/thumbv7m-none-eabi/debug/examples/hello
```

然后在GDB Shell中，我们连接到QEMU，它正在TCP端口3333上等待连接。

```console
target remote :3333
```

```console
Remote debugging using :3333 Reset () at $REGISTRY/cortex-m-rt-0.6.1/src/lib.rs:473 473 pub unsafe extern "C" fn Reset() -> ! {
```

您会看到该进程已停止，并且程序计数器指向了一个名为“Reset”的函数。那就是重启入口：即Cortex-M启动时执行程序的入口。

该函数最终将调用我们的main函数。让我们使用断点和`continue`命令一路跳过：

`break main`

`Breakpoint 1 at 0x400: file examples/panic.rs, line 29.`

`continue`

`Continuing.  Breakpoint 1, main () at examples/hello.rs:17 17          let mut stdout = hio::hstdout().unwrap();`

我们现在接近打印“ Hello，world！”的代码。让我们继续使用“ next”命令。

`next`

`18          writeln!(stdout, "Hello, world!").unwrap();`

`next`

`20          debug::exit(debug::EXIT_SUCCESS);`

此时，您应该在运行`qemu-system-arm`的终端上看到"Hello, world!"。

`$ qemu-system-arm (..) Hello, world!`

再次调用`next`将终止QEMU过程。

`next`

`[Inferior 1 (Remote target) exited normally]`

现在，您可以退出GDB会话。

`quit`

具体可参考： https://gist.github.com/carloscn/f628bb08453cdda3a33de58caa06ba1f

# Ref

[^1]:[LLVM基础设施和Rust]( https://rustmagazine.github.io/rust_magazine_2021/chapter_12/llvm-infrastructure-and-rust.html#llvm%E5%9F%BA%E7%A1%80%E8%AE%BE%E6%96%BD%E5%92%8Crust )
[^2]:[Writing an OS in Rust - A Freestanding Rust Binary]( https://os.phil-opp.com/freestanding-rust-binary/#the-no-std-attribute )
[^3]:[1184-stabilize-no_std.md]( https://github.com/rust-lang/rfcs/blob/master/text/1184-stabilize-no_std.md )
[^4]:[Can’t find core , the `thumbv7m-none-eabi` target may not be installed](https://users.rust-lang.org/t/cant-find-core-the-thumbv7m-none-eabi-target-may-not-be-installed/39183/3)