# Secure boot overview

ZYNQ的secure boot方案提供了三大特性的验证:
* 保密性
* 信息完整性（防篡改，防数据破坏）
* 认证（防否认、中间人攻击）

ZYNQ在secure boot的支持上提供了：
* Hardware Root of Trust (HWRoT) 作为可选的 对于所有分区进行加密
* HWRoT 基于RSA-4096算法SHA384
* HWRoT 有引擎加速
* 保密性使用256 AES密钥 的AES-GCM 消息认证码

本节overview主要是用于如何实现和使用：
* secure boot high-level设计
* Hardware Root of Trust with key revocation（讨论ROT作为启动根密钥）
* 分区加密：Partition encryption with differential power analysis (DPA) countermeasures（换密钥）
* 黑秘钥供应(Black Key Provisioning) 使用PUF作为KEK 

# 1. Secure Boot 所涉及的安全机制

我们需要关注点在于[^2]：
* 启动模式选择哪个？
* AES key本地存储的问题（GCM的密钥）
* AES 的状态（指示加密还是解密）
* 加密和认证的需求
* key的供应（provisioning)

支持安全启动的模式有 QSPI、SD、emmc、USB boot还有NAND。AES的加密后的key和未加密的key存储在eFUSEs上面；BBRAM未加密，NVM已加密。在Zynq UltraScale+MPSoC设备中，分区可以以分区进行加密和/或身份验证。zynq建议是所有的分区都是用rsa进行认证，分区如linux或者uboot如果没有一些非常保密的信息，则不需要被加密。在存在敏感数据和/或专有IP的多个来源/供应商的系统中，使用唯一密钥加密分区可能很重要。

![Pasted image 20221104152951](https://user-images.githubusercontent.com/16836611/199932676-1ed9c004-bf7f-45b9-8089-f2c5d8df2986.png)

### Hardware Root of Trust[¶](https://xilinx.github.io/Embedded-Design-Tutorials/docs/2021.2/build/html/docs/Introduction/ZynqMPSoC-EDT/9-secure-boot.html#hardware-root-of-trust "Permalink to this heading")

信任的根是存储（RTS）、完整性（RTI）、验证（RTV）、测量（RTM）和报告（RTR）的安全原语。RoT由硬件、固件和软件组成。HWRoT比软件RoT具有优势，因为HWRoT是不可变的，攻击面更小，行为更可靠。

HWRoT基于CSU、eFUSE、BBRAM（电池支持的RAM）和隔离元件。HWRoT负责验证操作环境和配置是否未被修改。RoT充当引导的锚，因此对手无法在检测机制启动之前插入恶意代码。

固件和软件在引导期间在HWRoT上运行。Zynq UltraScale+提供了不可变的引导ROM代码、第一阶段引导加载程序、设备驱动程序以及在HWRoT上运行的XILSKEY和XILSECURE库。这些提供了一个经过良好测试、经过验证的使用中API，这样开发人员就不会通过有限的测试从头开始创建安全组件。

#### Data Integrity[¶](https://xilinx.github.io/Embedded-Design-Tutorials/docs/2021.2/build/html/docs/Introduction/ZynqMPSoC-EDT/9-secure-boot.html#data-integrity "Permalink to this heading")

数据完整性是指硬件、固件和软件没有损坏。数据完整性功能验证对手未篡改配置和操作环境。

Zynq UltraScale+使用对称密钥（AES-GCM）和非对称密钥（RSA）身份验证验证分区的完整性。RSA使用私钥/公钥对。现场的嵌入式系统只有公钥。窃取公钥的价值有限，因为在当前技术下，无法从公钥中导出私钥。

加密分区也使用AES的伽罗瓦计数器模式（GCM）模式进行身份验证。在安全引导中，分区首先经过身份验证，然后根据需要进行解密。

#### Authentication[¶](https://xilinx.github.io/Embedded-Design-Tutorials/docs/2021.2/build/html/docs/Introduction/ZynqMPSoC-EDT/9-secure-boot.html#authentication "Permalink to this heading")

下图显示了分区的RSA签名和验证。Bootgen工具使用私钥从安全设施对分区进行签名。在设备中，ROM验证FSBL，FSBL或U-Boot使用公钥验证后续分区。

使用primary和secondary私钥/公钥对。主私钥/公钥对的功能是对次私钥/公共密钥对进行身份验证。辅助密钥的功能是对分区进行签名/验证。

![Pasted image 20221104153548](https://user-images.githubusercontent.com/16836611/199932743-adb3232d-a93b-4651-9ddd-8ee4769d2a81.png)

要签署分区，Bootgen首先计算分区数据的SHA3。然后使用私钥对384位散列进行RSA签名。生成的RSA签名被放置在身份验证证书中。在图像中，每个签名分区都有分区数据，后面是一个包含RSA签名的身份验证证书。

FSBL的验证由CSU ROM代码处理。为了验证后续分区，FSBL或U-Boot使用XILSECURE库。

有一种用于身份验证的调试模式，称为引导头身份验证(boot header authentication)。在这种认证模式中，CSU ROM代码不检查存储在设备**eFUSE中的主公钥摘要、会话密钥ID或密钥撤销位**。**因此，此模式不安全**。然而，它对于测试和调试很有用，因为它不需要对eFUSE进行编程。

**本教程使用此模式**。然而，现场系统不应使用引导头身份验证。本节末尾包含了完全安全系统的示例BIF文件。

### Boot Image Confidentiality and DPA[¶](https://xilinx.github.io/Embedded-Design-Tutorials/docs/2021.2/build/html/docs/Introduction/ZynqMPSoC-EDT/9-secure-boot.html#boot-image-confidentiality-and-dpa "Permalink to this heading")

AES用于确保敏感数据和IP的机密性。Zynq UltraScale+使用AES伽罗瓦计数器模式（GCM）和256位AES密钥。Zynq UltraScale+提供的AES增强功能主要是增强了对差分功率分析（DPA）攻击的抵抗力，以及启动后AES加密/解密的可用性。

Bootgen和FSBL软件支持AES加密。私钥用于AES加密，AES加密由Bootgen使用密钥文件完成。密钥文件可以由Bootgen或OpenSSL生成。操作键的使用限制了device key的暴露。下一节将讨论操作键在key rolling中的使用。为了保持引导映像的机密性，可以使用Bootgen创建加密的引导映像。Vitis中还提供了对BBRAM和eFUSE键编程。

### DPA Protections[¶](https://xilinx.github.io/Embedded-Design-Tutorials/docs/2021.2/build/html/docs/Introduction/ZynqMPSoC-EDT/9-secure-boot.html#dpa-protections "Permalink to this heading")

key rolling用于DPA抵抗。key rolling和黑密钥存储可以在同一设计中使用。在密钥滚动中，软件和比特流被分成多个数据块，每个数据块都用唯一的AES密钥加密。**初始密钥存储在BBRAM或eFUSE NVM中**。连续数据块的密钥在前一数据块中被加密。在初始密钥之后，密钥更新寄存器用作密钥源。

96位初始化向量包含在NKY密钥文件中。IV使用96位初始化AES计数器。当使用密钥滚动时，在引导头中提供128位IV。32个最低有效位定义了使用当前密钥解密的数据块大小。IV中定义的初始块之后的块大小定义为Bootgen图像格式（BIF）文件中的属性。

一种有效的key rolling使用操作键。使用操作密钥，Bootgen使用用户指定的操作密钥和第一个块IV创建加密的安全头。eFUSE或BBRAM中的AES密钥仅用于使用256位操作密钥解密384位安全头。这限制了设备密钥暴露于DPA攻击。

> 什么是key rolling？
> 
> 功耗分析这样的旁路攻击，是通过统计学原理从密文推算出密钥，再还原成原文。为了不给攻击者足够的信息推算密钥，Key Rolling 的主要想法是一个密钥只用于有线长度的原文。原文切成很多小份，每一小份原文加一个新的密钥打包后用密钥加密，新的密钥用于加密后面的一份原文和密钥，这样滚动前进。
> 
> 解密时就反向操作，eFUSE 里的密钥只用于解密 section 1，解密出原文1 和密钥1，再用密钥1 解密 section 2，得到原文2 和密钥2，依此类推。
> 

### Black Key Storage[¶](https://xilinx.github.io/Embedded-Design-Tutorials/docs/2021.2/build/html/docs/Introduction/ZynqMPSoC-EDT/9-secure-boot.html#black-key-storage "Permalink to this heading")

> 什么是 AES Black Key
> 
>AES 是一种对称加密技术。所谓对称加密，就是加密和解密用的是同一个密钥。这样的话，保护密钥就变得尤为重要。把 AES Key 直接烧录到 eFUSE 中被称为 Red Key，如果有人能够打开芯片探测到 eFUSE 的烧录情况，就可以获得密钥。为了防止这种情况发生，Xilinx 提供了 Black Key 方案。
>
>每片 MPSoC 芯片中都有一个硬件模块叫做 PUF: Physical Uncloneable Function。它可以根据每片芯片物理上本身工艺尺度上的微小差异，对 Red Key 进行加密生成 Black Key。当 Black Key 烧录在 eFUSE 后，这片芯片的 PUF 模块通过一些输入信息还可以将 Red Key 还原回来，再用得到的 Red Key 解密安全启动的镜像。因为烧录在 eFUSE 中的是 Black Key 而不是 Red Key，它不能通过简单的方法还原出黑客希望获得的密钥来解密镜像，整个解密过程只能在芯片内部进行，使用 Black Key 技术相当于为产品的安全再增加了一堵墙。

PUF能够以加密（黑色）格式存储AES密钥。黑密钥可以存储在eFUSE或引导头中。当需要解密时，使用PUF生成的密钥加密密钥（KEK）解密eFUSE中的加密密钥或引导头。

使用PUF进行黑密钥存储有两个步骤：
* 使用PUF注册软件来生成PUF助手数据和PUF KEK。PUF注册数据允许PUF每次生成KEK时重新生成相同的密钥。有关使用PUF注册软件的更多详细信息，请参阅引 https://xilinx.github.io/Embedded-Design-Tutorials/docs/2021.2/build/html/docs/Introduction/ZynqMPSoC-EDT/9-secure-boot.html#using-puf-in-boot-header-mode 。有关PUF注册-eFUSE模式的更多信息，请参阅编程BBRAM和eFUSE https://xilinx.github.io/Embedded-Design-Tutorials/docs/2021.2/build/html/docs/Introduction/ZynqMPSoC-EDT/9-secure-boot.html#using-puf-in-boot-header-mode
* 如果使用PUF eFUSE模式，则助手数据和加密的用户密钥必须存储在eFUSEs中，如果使用PUFboot头模式，则必须存储在引导头中。PUF引导头模式的过程在引导头模式中使用PUF中进行了讨论。有关在eFUSE模式下使用PUF的过程。

本教程使用PUF引导头模式，因为它不需要对eFUSE进行编程，因此对于测试和调试非常有用。然而，最常见的模式是PUF eFUSE模式，因为PUB引导头模式需要为每个设备运行唯一的Bootgen。

# 2. Secure boot High-Level Design

ZYNQ所有的关于secure boot的顶层设计全部在bootgen tool的设计中表述[^1]。对于secure boot在high-level层面我们只需要关注两个问题，一个是image的签名和验签（认证），另一个是image的加解密。对于这两个问题，需要共同关注对于key的使用和管理，这部分就需要结合第1节的安全机制。综上所述，本节我们需要弄清楚：
* image如何签名和验签？
	* 如何管理公钥和私钥？
	* 怎么去配置公钥和私钥？
	* 启动的时候如何公钥验签？公钥如何存储在板子上？
* image如何加密和解密？
	* 如何管理key？
	* 怎么去配置key？
	* 什么场景应该使用什么key？
[^5]
## 2.1 image加解密

ZYNQ设备使用GCM（32-bit, word-based）256-bit key 作为数据加解密，消息认证，完整性检查，这种安全机制可以cover数据保密性、完整性和消息认证，但是没有办法防否认。关于GCM的介绍可以参考：[3.3_Security_对称密钥算法之AEAD](https://github.com/carloscn/blog/issues/145)

mbedtls的接口定义可以参考：

https://siliconlabs.github.io/Gecko_SDK_Doc/mbedtls/html/gcm_8h.html

```C
int mbedtls_gcm_crypt_and_tag (mbedtls_gcm_context *ctx, 
							   int mode, 
							   size_t length, 
							   const unsigned char *iv, 
							   size_t iv_len, 
							   const unsigned char *add, 
							   size_t add_len, 
							   const unsigned char *input, 
							   unsigned char *output, 
							   size_t tag_len, 
							   unsigned char *tag);
int mbedtls_gcm_auth_decrypt (mbedtls_gcm_context *ctx, 
							  size_t length, 
							  const unsigned char *iv, 
							  size_t iv_len, 
							  const unsigned char *add, 
							  size_t add_len, 
							  const unsigned char *tag, 
							  size_t tag_len, 
							  const unsigned char *input, 
							  unsigned char *output)
// note, /p add is additional data.
```

关于密钥产生，bootgen能够产生满足要求的GCM的Key，使用CMAC作为伪随机数为核心的生成器。比如在Key Rolling的时候则需要随机数作为支撑。而且可以指定种子（没有指定使用key0）作为种子。

### 2.1.1 image加解密原理

ZYNQ提供了bootgen工具的用以加密引导image的分区，用户可以制定BIF格式的文件用来描述image分区的加密命令和加密属性。ZYNQ只提供了AES对称加密算法，加密和解密的密钥是一致的，因此，key应该在两个地方：
* key需要在运行bootgen tool的本地host，用来加密；
* key需要在设备端，为了保密性则放置在eFUSE上或者BBRAM中。

加密过程如下：key需要写入到eFUSE或者BBRAM中，同时key需要给同给bootgen tool用来加密image分区。

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221106131204.png" width="70%" /></div>

Note，根据zynq的设计，每个分区都可以使用不同的对称加密的密钥。

解密过程如下：一个SoC设备是由BootROM和FSBL来进行解密的，换句话说，BootROM作为根启动是不能进行加密的。在BootROM引导程序中，BootROM从flash里面读取FSBL，然后对FSBL进行解密，然后把FSBL加载到OCM中，接着把hands off控制权到FSBL，FSBL会读取剩余的image分区，然后加载、解密、hands off。

在这个过程中BootROM和FSBL是从eFUSE或者BBRAM中读取到AES的key，并且使用加密硬件引擎操作，保障了密钥不被泄露。BootROM和FSBL可以从image分区的boot header中有一个`key source`区域，知道AES加密key的来源是哪里。

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221106131244.png" width="70%" /></div>

### 2.1.2 AES key管理

#### Operational Key

关于Operational key的密码学意义可以参考[^3]，简言之就是一个份很私密的密钥因为业务原因不得不保存在host端，这就存在一个显著的安全风险，尤其是跨团队的操作，导致这个密钥被copy了很多份。Operational Key提供了保护私密的密钥途径，ZYNQ对于两个团队之间传输私密的OPT Key产生过程，以防止密钥泄露，可以参考[^4]。

bootgen tool可以创建一个加密的安全header，其中包含用户指定的opt_key，以及启动此功能时第一个块IV。结果就是，存储在eFUSE或BBRAM的密钥长度是384位（AES-GCM需要256，剩余128密钥位就是opt_key加成的结果），**这种方式可以低于侧[信道攻击](https://zh.wikipedia.org/wiki/%E6%97%81%E8%B7%AF%E6%94%BB%E5%87%BB)**。

对于一个image，使用opt_key的一个BIF示例：
```
image_v:
{
	[fsbl_config] opt_key
	[keysrc_encryption] bbram_red_key 
	 
	[bootloader, 
	 destination_cpu = a53-0,
	 encryption      = aes, 
	 aeskeyfile      = aes_p1.nky]fsbl.bin
	 
	[destination_cpu = a53-3,
	 encryption      = aes, 
	 aeskeyfile      = aes_p2.nky]hello.bin
}
```

我们使用bootgen来产生这个文件：

`bootgen -arch zynqmp -image ./image.bif -w -o u-boot-enc.bin -p xc7z020clg484`

注意，如果指定elf文件，会check elf文件头。

此时就会生成：
<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora20221106134644.png" width="90%" /></div>

生成的aes_p1.nky内容如下：
```bash
Device       xc7z020clg484;

Key 0        1FA4A2573EEF8C10DB860F7C227637CD7ECBEE08BE8046273D350C8FA9D9CB2B;
IV 0         828BDAB962189EB6CA107718;

Key 1        868BF0CC442D11B428CFBFEF158FA3E73BBC0DEF071B328DC17B9622BF21DED9;
IV 1         B9B3BD4CB8453B2A78DB4198;

Key Opt      69B1CC18E961A941B034E1DD6478817832BD242BA79D6AA5C6D0500C647A62EE;
```

这里生成key是 64字节（512bit）， IV是24字节（192bit）。能够注意到Key Opt包含在了这个文件里。

#### Rolling Key

GCM也支持Rolling Key，防止差分功率分析的侧信道攻击。Roling Key会把加密信息分为几个不同大小的块，每一个块使用一个不同的密钥进行加密。bootgen tool支持这个模式。

```
image_v:
{
	[keysrc_encryption] bbram_red_key
		
	[
		bootloader, 
		destination_cpu = a53-0,
		encryption      = aes, 
		aeskeyfile      = aes_p1.nky,
		blocks          = 1024(2);2048;4096(2);8192(2);4096;2048;1024 
	]    fsbl.bin
	 	
	[
		destination_cpu = a53-3,
		encryption      = aes, 
		aeskeyfile      = aes_p2.nky,
		blocks          = 4096(1);1024 
	]    hello.bin
}
```

根据blocks的信息产生了这么多的key：

aes_p1.nky:
```bash
Device       xc7z020clg484;

Key 0        B0C44304775BFDD04BCF4B4241BFB0C1C44190ED21FA7EA82DB32826149EDFB9;
IV 0         C1437DEF64184BEA5AFF1C5C;

Key 1        0B4B513FCBFFDE1349CD3FACF61E99F4FA7896237D166B7C382D7A13F153FE7B;
IV 1         E9657B925A5103828B18C162;

Key 2        BC0AB79DD18D30229CC68D9845462430E234224752C2A1A1732ACD68821E2BDD;
IV 2         FC948884A7D0A81030B192A9;

Key 3        8884118253D979AF785C23C5FC62EA0F202385A56AA9F10B6D57D47A6EE70704;
IV 3         F0A91531B1B5E2BBF36EEF98;

Key 4        1E7D07483FE4AFF738981E49566C8BD73D70B1B06E8720F0E81D46F94798BA5A;
IV 4         B0789AB7AAD3F977B89CCAC7;

Key 5        33907B62ED7C782BB90D2241B5BF045F64032F59CE4863EF55BBE401C744E0A0;
IV 5         4B73575E804F708DE01420A6;

Key 6        B2C16A5B2F336CC568F8410FA2D9657268A24D857086469610606CBD0805B3F1;
IV 6         4CDC8B1E7990AFB7C8AB011E;

Key 7        7D6275C2106F9375037146487C6F4AE58FB3C243DAD7855D909E208FDBA35E6A;
IV 7         D54A560B3832D7309092C7BC;

Key 8        1F8A35BCD094ED3CEA3216913BD02F34002507ACB13B63F806039479C67A2FB1;
IV 8         D8AF645F32BD360958938D27;

Key 9        D30B3FB5683E00CDE542043AEB415F6AAA17869358BA84D5A24B7E57CD09076D;
IV 9         DCF06E464F2BD730BCAE022A;

Key 10        F92D9F15CDA82571EEEAA8891EFC01416802E2B92DA05831D469A6229314270E;
IV 10         23CA961490154C7E98846420;
```

aes_p2.nky:
```bash
Device       xc7z020clg484;

Key 0        B0C44304775BFDD04BCF4B4241BFB0C1C44190ED21FA7EA82DB32826149EDFB9;
IV 0         C1437DEF64184BEA5AFF1C5C;

Key 1        AF82DBE8927A7C064B20FD38D3190E37597890C79952D135812D4AAA25613270;
IV 1         8DA02A66E4DC4D8FDF674DB7;

Key 2        334B72DD04730889EE2BA628F268EE67859340CDDE0D85828FCCA4CCCCA14E1A;
IV 2         EDE114061B4EC91BD65F6564;
```
**Note:**
-   Number of keys in the key file should always be equal to the number of blocks to be encrypted.
    -   If the number of keys are less than the number of blocks to be encrypted, Bootgen returns an error.
    -   If the number of keys are more than the number of blocks to be encrypted, Bootgen ignores (does not read) the extra keys.
-   If you want to specify multiple Key/IV Pairs, you should specify `no. of blocks + 1` pairs
    -   The extra Key/IV pair is to encrypt the secure header.
    -   No Key/IV pair should be repeated in a any of the aes key files given in a single bif except the Key0 and IV0.

#### Gray/Obfuscated Key

Gray Key（模糊Key）这个Key的业务逻辑是，允许一批设备使用相同的Family Key加密red key。比如我们经常说的Model Key，这个密钥可以**存储在eFUSE或在image的boot header中(没有BBRAM)**。对于希望以模糊形式存储加密密钥的用户，可以使用Gray密钥作为加密密钥对红色密钥进行加密。ZynqMP将解密模糊密钥以获得实际的红色密钥。请注意，系列密钥在ZynqMP SoC系列中的所有设备上都是相同的。该解决方案允许用户将混淆的密钥分发给合同制造商，而不泄露实际的加密密钥。

```
image_v:
{
	[keysrc_encryption] efuse_gry_key 
	[bh_key_iv] bhiv.txt
	[
		bootloader, 
		destination_cpu = a53-0,
		encryption      = aes, 
		aeskeyfile      = aes_p1.nky
	]    fsbl.bin 
	[
		destination_cpu = r5-0,
		encryption      = aes,
		aeskeyfile      = aes_p2.nky 
	]    hello.bin
}
```

注意，创建Gray密钥还需要创建bhiv.txt文件，在这个文件中放置IV，这个字段会出现在Boot Header的BH IV中。

```
8DA02A66E4DC4D8FDF674DB7
```

`bootgen -arch zynqmp -image ./gray_key.bif -w -o u-boot-enc.bin -p xc7z020clg484`

产生Family Key是有安全要求的，我们需要使用bootgen来产生Family Key。产生Key的bif如下：
```
obf_key:
{
	[aeskeyfile] aes.nky  
	[familykey] familyKey.cfg 
	[bh_key_iv] bhiv.txt
}
```

The command to generate the Obfuscated key is（需要自己创造IV）:
```
bootgen -arch zynqmp -image all.bif -generate_keys obfuscatedkey
```

> The family key is not distributed with Xilinx development tools, and user need to request family key by sending request to email address [secure.solutions@xilinx.com](mailto:secure.solutions@xilinx.com) . Install the family key received from Xilinx in the development tool installation directory, and then use development tools (bootgen) to create the obfuscated key.

我们拿不到familyKey.cfg文件，这部分需要找xilinx团队要。

#### Black/PUF Keys

black key解决方案是对安全性能的增强，属于KEK（Key encryption Key），而且这种Key并不是使用算法或者软件进行加密，而是使用PUF这种硬件期间。PUF会加扰写在eFUSE上的密钥。黑密钥可以**写入eFUSE或者a part of the authenticated boot header**。

```
image_v:
{ 
	[puf_file] pufdata.txt
	[bh_key_iv] black_iv.txt
	[bh_keyfile] black_key.txt
	[fsbl_config] puf4kmode, shutter=0x0100005E, pufhd_bh
	[keysrc_encryption] bh_blk_key 
	
	[
	  bootloader,
	  destination_cpu = a53-0,
	  encryption      = aes, 
	  aeskeyfile      = aes_p1.nky
	] fsbl.bin
		 
	[
	  destination_cpu = r5-0,
	  encryption      = aes,
	  aeskeyfile      = aes_p2.nky
	] hello.bin
}
```

![](https://raw.githubusercontent.com/carloscn/images/main/typora20221106144845.png)

关于PUF的一些参数如上图所示。

**note**: puf_file是一个输入文件，关于这个输入文件，需要参考[External Secure Storage Using the PUF (XAPP1333)](https://docs.xilinx.com/r/en-US/xapp1333-external-storage-puf/External-Secure-Storage-Using-the-PUF-Application-Note)

#### 多个加密密钥文件

早期版本的Bootgen支持通过使用单个加密密钥加密多个分区来创建引导映像。对于每个分区，重复使用相同的密钥。这是一个安全漏洞，不建议使用。每个key在流中只能使用一次。

Bootgen支持每个分区单独的加密密钥。如果有多个密钥文件，请确保每个加密密钥文件使用相同的Key0（设备密钥）、IV0和操作密钥。如果每个加密密钥文件中的启动映像不同，Bootgen不允许创建启动映像。您必须指定多个加密密钥文件，一个用于映像中的每个分区。指定分区使用指定的密钥加密。

比如：
```
all:
{
	[keysrc_encryption] bbram_red_key
	// FSBL (Partition-0)
	[
		bootloader, 
		destination_cpu = a53-0, 
		encryption = aes,
		aeskeyfile = key_p0.nky
		
	]fsbla53.elf
				 
	// application (Partition-1)
	[
		destination_cpu = a53-0,
		encryption = aes,
		aeskeyfile = key_p1.nky
			
	]hello.elf  
}
```

-   The partition fsbla53.elf is encrypted using the keys from key_p0.nky file.
-   Assuming hello.elf has three partitions because it has three loadable sections, then partition hello.elf.0 is encrypted using keys from the test2.nky file.
-   Partition hello.elf.1 is then encrypted using keys from `test2.1.nky`.
-   Partition hello.elf.2 is encrypted using keys from `test2.2.nky`.



# Ref
[^1]:[bootgen-user-guide](https://docs.xilinx.com/r/en-US/ug1283-bootgen-user-guide/Installing-Bootgen)
[^2]:[Embedded-Design-Tutorials - secure-boot](https://xilinx.github.io/Embedded-Design-Tutorials/docs/2021.2/build/html/docs/Introduction/ZynqMPSoC-EDT/9-secure-boot.html#)
[^3]:[Operational keys](https://www.ibm.com/docs/en/zos/2.1.0?topic=tab-operational-keys)
[^4]:[Using-Op-Key-to-Protect-the-Device-Key-in-a-Development-Environment](https://docs.xilinx.com/r/en-US/ug1283-bootgen-user-guide/Using-Op-Key-to-Protect-the-Device-Key-in-a-Development-Environment)
[^5]:[Zynq+Ultrascale+MPSoC+Security+Features](https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/18841708/Zynq+Ultrascale+MPSoC+Security+Features#ZynqUltrascale%2BMPSoCSecurityFeatures-ObfuscatedKey)

