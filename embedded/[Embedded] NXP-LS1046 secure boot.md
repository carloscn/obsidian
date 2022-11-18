# [Embedded] NXP-LS1046 secure boot

# 1. secure boot overview

## 1.1 introduction

NXP的secure boot流程被分为多个阶段，严格按照TF-A的规则使用，分为BL1, BL2 (at EL3), BL31, BL32, BL33。secure boot的image验证是使用各自的image header完成。

image header有两类：
-   **CSF headers (NXP Chain of Trust)** 参考：[Code signing tool](https://docs.nxp.com/bundle/GUID-487B2E69-BB19-42CB-AC38-7EF18C0FE3AE/page/GUID-932D50F3-D90D-4ED0-BEFC-B1BF825EB422.html).
-   X.509 certificate (Arm Chain of Trust). 参考：[ARM X.509 certificate](https://developer.arm.com/docs/den0006/latest/trusted-board-boot-requirements-client-tbbr-client-armv8-a)

**Note，本文以CSF headers (NXP Chain of Trust)为主**。

对于基于TF-A的secure boot流程简述如下：

1. SoC从reset状态释放之后，控制权hands off BL1。BL1的主要负责去验证BL2的image和BL1自己image header中的信息是否匹配。接着，BL1读取`BOOTLOC`指针的值（这个值在BL2 image header中）。
2. BL2如果被验证成功。BL2 image进一步验证FIP image的每一个header。FIP image主要包含：
	- X.509 certificate/CSF header BL31 + BL31 image
	- X.509 certificate/CSF header BL32 + BL32 image (optional)
	- X.509 certificate/CSF header BL33 + BL33 image
3. BL33（uboot）负责验证和执行uboot所引导层级的固件。

TF-A引导secure boot流程：

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202211161258430.png" width="100%" /></div> 

## 1.2 secure images format

### 1.2.1 bl2_<boot_mode>.pbl

在pbl文件中，主要包含RCW和bl2.bin文件。NXP为bl2.bin文件创造了CSF头文件。

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202211161253136.png" width="100%" /></div> 

### 1.2.2 fip.bin

在fip.bin文件中，包含BL31，BL32(optee.bin)，BL33（uboot/uefi），每一个固件都有一个文件头。

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202211161257646.png" width="100%" /></div> 

# 2. Secure Boot High-Level

## 2.1 签名和验签

### 2.1.1 code sign tool签名

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202211161307776.png" width="100%" /></div> 

这部分由Code Signing Tool (CST)完成。需要输入：

* 私钥和公钥 
* image
* CSF header （Command Sequence File header）
* S/G Table  （Scatter Gather table）

会输出：

* signature
* public key hash （放入eFUSE）

### 2.1.2 验签过程

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202211161312073.png" width="100%" /></div> 

验签过程由板子完成。

### 2.1.3 签名验签元素说明

#### CSF header

Command Sequence File header，主要提供一些image验证信息，例如flag，image指针，offset和长度等。有两个版本分别应用于不同的TA版本（ISBC and ESBC），后面会讲。**LS1046A支持是TA2.X版本**。

#### SG table

Scatter Gather table，可选的。使用该table允许使用不连续的images。

#### Public key list

Super Root Key (SRK) table（后面会展开讲）。一个或多个公钥被放在image后面。然后有CSF头来指定签名验证。

#### signature

最后的签名是，CSF header + SG Table + image + Public key list的集合的SHA256值，被RSA私钥进行签名。

## 2.2 信任链

![](https://raw.githubusercontent.com/carloscn/images/main/typora202211161326629.png)

Chain of Trust (CoT)确保只有被认证的image能够被执行在特定的平台。image在CoT认证可以被分为两个阶段：

* Pre-boot and ISBC
* ESBC

为了保证image的加密特性，image可以被加密以blobs的形式存储在flash中。ESBC uboot image程序可以使用Cryptographic blob mechanism来创建保密特性的信任链。[QorIQ Trust Architecture 3.0 User Guide](https://www.nxp.com.cn/design/training/security-201-introduction-to-qoriq-trust-architecture:TIP-SECURITY-201-INTRODUCTION-TRUST?SAMLart=ST-AAF0E3YJDJej%2BJVBprc7Vu5rkUdez%2FJlD%2F4q%2Fanft6IQwsdABXfRNGo%2F)

### 2.2.1 pre-boot & ISBC (BootROM)

ISBC(Internal Secure Boot Code)是嵌入存储在SoC的BootROM中的验证程序。根凭证的验证程序已经存储在BootROM中。ISBC需要验证下一个阶段的引导程序（在NXP的设计，这个引导程序应该是pbl文件中的BL2）。

### 2.2.2 ESBC (Bootloaders)

External Secure Boot Code (ESBC)，BL2有数字签名验证程序（ESBC）嵌入其中，BL2在执行它之前会进行验证。

ESBC是NXP提供的secure boot的参考实现，用于验证在TFA中的image和uboot image。uboot用于验证Linux，设备树之类的。

NXP提供了相同的ESBC，但是对于ISBC，不同平台有不同的程序。这些平台根据不同的Trust Architecture（TA）种类不同，提出两种：
* TA 2.x or hardware pre-boot loader (PBL) based platforms
* TA 3.x or Service Processor (SP) based platforms

NXP-LS1046A属于TA2.x平台。

# 3. Secure Boot Low-Level

Secure boot low-level主要涉及：
* BootROM阶段
* Bootloaders阶段。

## 3.1 Pre-Boot phase (BootROM)

BootROM阶段的程序是有没有办法修改。这里只能明白其工作和原理。NXP的LS1046A芯片使用的TA 2.x 平台的 bootrom程序。

在开发调试阶段，设定`RCW[SB_EN] = 1`来设定系统启动以secure boot的形式启动。在量产阶段，设定`ITS` bit in `SFP`确保系统执行在secure模式。一旦SFP ITS被写入到FUSE当中，就不能改变了。

#### Hardware pre-boot loader

PBI程序（ pre-boot initialization commands），一旦启动secure boot，其必然被PBL执行。在PBI命令中必须包含加载ESBC的指向CSF header的指针数据（SCRATCHRW1寄存器中）。

接着BootROM程序（ISBC）读取寄存器来判断CSH Header指定的下一个image在哪里，准备进入到验证阶段。如果启动备选image，第二条加载备选image的CSF header的PBI命令会被加载到SCRATCHRW3寄存器。

在LSDK参考实现中，boot源不受到影响，PBI命令被置入到RCW中来加载BL2的image到OCRAM上。在启动secboot的情况下，验证BL2的CSF header也会随着使用PBI命令的BL2一起被加载到OCRAM上。

（这部分知道做什么就可以，无法修改，并不是重点）

#### ISBC phase

当SoC上电之后Power-on Reset (POR)，主**CPU0**被释放，开始执行BootROM上的程序。

过程如下所示：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202211161432065.png)

* **SecMon check** ：CPU0需要确认SecMon的状态，如果SecMon状态没有就绪则进入spinloop状态。
* **ESBC pointer read**：CPU0从SCRTCHRW1寄存器读取ESBC的CSF header的地址，然后从地址拿到文件头信息。此过程会判断header中的barker-code以此判断是否符合平台。
* **CSF header parsing and public key check**：如果CPU0找到了一个正确的CSF头文件，从CSF头中解析public key。在header中可能存在一个或者一组public key。SFP寄存器不会真正存储public key，它只是存储它的sha256的hash值。如果算出的pubhash和存储在SFP中的不相同的话，那么失败。
* **Signature validation**：如果是一个正确的pub hash值，那么就开始对header + image + public keys 一起做RSA验签操作。
* **ESBC Pointer check**： CSF header包含 入口地址。会检测地址范围是否正确。然后，call sec mon进入安全模式，并跳转到入口地址。

如果失败的话，会进入到备选模式，ISBC会从SCRATCHRW3寄存器读取备选image的文件头信息，然后再进入验证。如果备选模式失败，那么就进入到spin loop状态了。

#### ISBC header

ISBC的header和ESBC的header如图所示：

![](https://raw.githubusercontent.com/carloscn/images/main/typoratypora202211161441460.png)

详细参考： https://docs.nxp.com/bundle/GUID-487B2E69-BB19-42CB-AC38-7EF18C0FE3AE/page/GUID-4D5D0916-29CC-4E11-BF82-477C40F31585.html

#### Super Root Keys (SRKs) and signing keys

RSA的密钥对中其中私钥用于签images，公钥用于验证image。公钥被嵌入到了embedded image中，并且被计算hash值存储到了eFUSE上。secboot支持key的长度是1k/2k/4k。

需要注意的的是，rsa私钥需要保护好，如果私钥暴露，攻击者完全可以使用rsa产生一个备选的image存入SCRATCHRW3来pass secure boot。


## 3.2 ESBC phase (Bootloaders)

不像是BootROM阶段的，ESBC阶段的code可以被我们修改。Nxp定义的ESBC包含：
* BL2 image（验证BL31/BL32/BL33 image）
* uboot image （验证linux / dtb）

ESBC的程序在LSDK中已经提供。NXP提供两个类型的secure boot：
* ESBC image validation using NXP CSF headers
	* also known as NXP CoT for ESBC images
* ESBC image validation using X509 certificates
	* Enabled on NXP platform through TF-A
	* meets Arm recommended Trusted Board Boot Requirements (TBBR)
	* also known as Arm CoT for ESBC images

需要注意的是，**Arm CoT is supported only for LX2160ARDB and LX2162AQDS platforms**. 因此，NXP-LX1046A不支持ARM的。

BL2 binary将会加载以下binaries：
-   BL31 binary
-   BL32 binary (OPTEE code)
-   BL33 binary (U-Boot/UEFI)

这部分验证过程和ISBC phase是一致的。

# 4. Code Signing Tool

code sign tool主要就是生成符合NXP验证的secure boot的文件头。内部的实现是通过调用OpenSSL APIs。

主要功能如下：

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202211171256613.png" width="80%" /></div> 

## 4.1 key gen

### genkey
RSA的key pair从以下3部分来：N, E, D:

-   N – Modulus
-   E – Encryption exponent
-   D – Decryption exponent

**Public key**: 来源于E&N
**Private key**: 来源于D&N

生成key需要注意：
* 支持三种长度的key 1024 / 2048 /4096；
* 生成的key格式是pem格式；
* 可以自定义名字。

```
./genkeys <Key length in bits >
```

### gen_otpmk_drbg

工具可以插入hamming code在用户定义的256b的十六进制字符串中。也可以产生256b的随机数并插入hamming code。这个需求可以通过gen_otpmk_drbg完成。

`./gen_otpmk_drbg --b <bit_order> [--s <string>] [--u]`

生成drbg随机数：

`./gen_drv_drbg <hamming_algo> [string]`

## 4.2 header creation

### uni_pbi

```
$ ./uni_pbi [options] <input_file> 
```

参考： https://docs.nxp.com/bundle/GUID-3FFCCD77-5220-414D-8664-09E6FB1B02C6/page/GUID-E0B02E68-62BC-4043-ABF3-95804A92D85A.html

### uni_sign

uni_sign tool能够被使用如下：

-   CSF header generation along with signature for both ISBC and ESBC phases
-   CSF header generation without signature if private key is not provided
-   uni_sign tool (with ESBC = 0 in input file) is used for creating signature and header over Boot1 image to be verified by ISBC
-   uni_sign tool (with ESBC = 1 in input file) is used for creating signature and header over images to be verified by ESBC

运行命令：

```
./uni_sign [options] <input_file>	  	
```

参考： https://docs.nxp.com/bundle/GUID-3FFCCD77-5220-414D-8664-09E6FB1B02C6/page/GUID-054242AF-D433-427B-B020-2602CAA777E7.html

## 4.3 signature gen

### gen_sign

这个工具还能提供计算signature的功能。只需要hash file和私钥作为输入。

`./gen_sign [option] <HASH_FILE> <PRIV_KEY_FILE>`

参考： https://docs.nxp.com/bundle/GUID-3FFCCD77-5220-414D-8664-09E6FB1B02C6/page/GUID-A07DA501-2E7C-41FC-B635-3AD1F917A5D0.html

### sign_embed

把sign的结果嵌入到header里面则需要使用这个工具。

`./sign_embed <hdr_file> <sign_file>`

参考： https://docs.nxp.com/bundle/GUID-3FFCCD77-5220-414D-8664-09E6FB1B02C6/page/GUID-8BEAD88D-4D01-4632-9C1F-7E72805D2B3E.html

# 5. Secure Boot Enabling

Secboot流程能允许被执行：
* Chain of Trust,
* Chain of Trust with confidentiality

## 5.1 secure boot exec flow

总体流程如图所示：

<div align='center'><img src="https://raw.githubusercontent.com/carloscn/images/main/typora202211171326659.png" width="70%" /></div> 

##### 开发模式确认

我们需要确定是开发模式还是产品模式。如果是开发模式，不要进行ITS fuse的熔断，设定RCW[SB_EN] = 1就可以开启secure boot；如果是开发模式，则需要熔断ITS在SFP。

##### Provisioning

无论是开发模式还是在产品模式，都有熔断eFUSE的需求，OTPMK and SRKH两个位需要注意。OTPMK 需要使用`gen_otpmk_drbg`产生符合质量的随机数。详见Provisioning。**熔断OTPMK是必须的，无论是产品模式还是开发模式**。

##### Program secure boot images

在产品模式，只需要把secure boot images烧写到默认bank地址即可。在开发模式，需要编写到alternate bank 地址，由default bank切换到alternate bank再去启动secure boot。

##### power on board

###### 无image加密

在没有image加密的情况下：

如果secure boot 烧写到默认bank(both dev and product mode)，ISBC code获取控制权，并且验证ESBC的image。ESBC进一步验证linux/rootfs/dtb的signature。

如果secure boot烧写到 alternate bank中（dev mode），板子启动将会从默认的bank启动。当切换到alternate bank的时候，ISBC code获取控制权，并且验证ESBC的image。ESBC进一步验证linux/rootfs/dtb的signature。最后启动linux。

###### image加密

如果image使能了加密：

如果secure boot 烧写到默认bank(both dev and product mode)，ISBC code获取控制权，并且验证ESBC的image。ESBC进一步验证linux/rootfs/dtb的signature。

**First boot**：封装步骤（**应在OEM工厂内进行**）
* 默认情况下，encap和decap引导脚本将安装在引导分区中。
* 在生成所有映像后，当板首次启动时，encap bootscript将执行。此引导脚本：
	* 验证和封装Linux和dtb映像，并用新封装的Linux和dtb替换未加密的Linux和dtb映像。
	* ![](https://raw.githubusercontent.com/carloscn/images/main/typora202211171404370.png)
	* 将encap引导脚本和头替换为已存在于引导分区中的decap引导脚本及其头。
	* emit reset信号。

**Subsequent boot**:
* 如果安全引导映像在默认库中闪存（用于生产/开发阶段）:
	* U-Boot将使用decap命令执行脚本
	* 在DDR中Un-blobify linux和dtb映像。
	* 将控制权传递给这些images。
	![](https://raw.githubusercontent.com/carloscn/images/main/typora202211171407967.png)

如果secure boot烧写到 alternate bank中（dev mode），板子启动将会从默认的bank启动。当切换到alternate bank的时候，ISBC code获取控制权，并且验证ESBC的image。ESBC进一步验证linux/rootfs/dtb的signature。最后启动linux。

## 5.2 prepare board for secure boot

### enable POVDD

检查POVDD，对于LS1046RDB，需要Put J21 to enable PWR_PROG_SFP。

### Byte swap for reading and writing SRKH/OTPMK

应谨慎编写SRKH和OTPMK。如果使用Core写入SRKH和OTPMK，则需要交换SRKH与OTPMK。但是，如果使用DAP或SFP写入SRKH和OTPMK，则不需要交换。有关详细信息，请参阅下表。

![](https://raw.githubusercontent.com/carloscn/images/main/typora202211171411035.png)

参考： https://docs.nxp.com/bundle/GUID-3FFCCD77-5220-414D-8664-09E6FB1B02C6/page/GUID-036CF49E-4211-48B8-803A-870C57777C8A.html

###  Program OTPMK

After [enabling POVDD](https://docs.nxp.com/bundle/GUID-3FFCCD77-5220-414D-8664-09E6FB1B02C6/page/GUID-0EFF85FB-9070-4D76-A926-B973CA6C8FB5.html#GUID-0EFF85FB-9070-4D76-A926-B973CA6C8FB5), follow these steps to program OTPMK at U-Boot:

参考： https://docs.nxp.com/bundle/GUID-3FFCCD77-5220-414D-8664-09E6FB1B02C6/page/GUID-F780D1D5-F1B2-478B-86AE-267D74F9C790.html

### Program SRKH mirror registers

参考： https://docs.nxp.com/bundle/GUID-3FFCCD77-5220-414D-8664-09E6FB1B02C6/page/GUID-2CF1D60F-C79F-4A35-800B-2BFE504EBAC5.html

###  Write SFP_INGR register

参考： https://docs.nxp.com/bundle/GUID-3FFCCD77-5220-414D-8664-09E6FB1B02C6/page/GUID-EFF8FF41-C8C0-4A3B-AF95-E801D585B7C6.html

## 5.3 build secure boot image

### 5.3.1 using flexbuild

After setting up the flexbuild environment, run the following commands to generate images for NXP CoT:

```
$ flex-builder -i clean-firmware
$ flex-builder -i mkfw -m <machine> -b <boottype> -T nxp-cot  (optional)
$ flex-builder -i mkfw -m <machine> -b <boottype> -s ('-s' is equivalent to '-T nxp-cot')
```

For example:

```
$ flex-builder -i clean-firmware
$ flex-builder -i mkfw -m lx2162aqds -b xspi -T nxp-cot  (optional)
$ flex-builder -i mkfw -m ls1046ardb -b xspi -s ('-s' is equivalent to '-T nxp-cot')
```

The images will be available at: `/build/images/`.

### 5.3.2 using manually TF-A

需要在TF-A中设定：
* -   Set `TRUSTED_BOARD_BOOT=1` to enable trusted board boot.
    NXP CoT is enabled automatically when `TRUSTED_BOARD_BOOT=1` and `MBEDTLS_DIR path` is not specified.
* Specify path of the CST repository as `CST_DIR` to generate CSF headers. In NXP CoT, CSF header is embedded to the BL31, BL32, and BL33 images.
* Default input files for CSF header generation are available in `CST_DIR`.
* As per the default input file, you need to generate following RSA key pairs and add them to the ATF repository:
	-   srk.pri
	-   srk.pub
* The RSA key pairs can be generated using the gen_keys CST tool. To change the input file, you can use the options `BL33_INPUT_FILE`, `BL32_INPUT_FILE`, `BL31_INPUT_FILE`.

注意：TRUSTED_BOARD_BOOT 也可以被使能在非安全启动。 然而，ROTPK在非安全启动流中被忽略，并且故障不会导致SNVS转换。

```
make PLAT=<platform> TRUSTED_BOARD_BOOT=1 CST_DIR=$CST_DIR_PATH \
       
       RCW=$RCW_BIN \
       BL32=$TEE_BIN SPD=opteed\
       BL33=$UBOOT_SECURE_BIN \
       pbl \
       fip
```
To prepend CSF headers to BL31, BL32, and BL33 images:
```
make PLAT=<platform> all fip pbl SPD=opteed BL32=tee.bin BL33=u-boot.bin \
       RCW = <secure bot RCW>	\
       TRUSTED_BOARD_BOOT=1 CST_DIR=<cst dir path> BL33_INPUT_FILE=<ip file> BL32_INPUT_FILE=<ip_file> \
       BL31_INPUT_FILE = <ip file>
```
The secure boot binaries for NXP CoT are available in the `atf` directory:
-   `build/<platform>/release/fip.bin`
-   `build/<platform>/release/bl2_flexspi_nor_sec.pbl`

