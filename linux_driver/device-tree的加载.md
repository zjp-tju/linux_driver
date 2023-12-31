# 设备树的加载过程
在Linux中，所有设备都以文件的形式存放在/dev目录下，都是通过文件的方式进行访问，设备节点是Linux内核对设备的抽象，一个设备节点就是一个文件。

简单概述一下：
它基本上就是画一棵电路板上CPU、总线、设备组成的树，Bootloader会将这棵树传递给内核，然后内核可以识别这棵树，并根据它展开出Linux内核中platform_device、i2c_client、spi_device等设备，而这些设备用到的内存、IRQ等资源，也被传递给了内核，内核会将这些资源绑定给展开的相应的设备。

  设备树文件的格式为dts，包含的头文件格式为dtsi，dts文件是一种人可以看懂的编码格式。但是uboot和linux不能直接识别，他们只能识别二进制文件，所以需要把dts文件编译成dtb文件。dtb文件是一种可以被kernel和uboot识别的二进制文件。把dts编译成dtb文件的工具是dtc。

编译好的设备树文件由bootloader在启动时加载到内存中，并在启动内核时将设备树的地址一并传递给内核。
当kernel启动后，kernel会首先根据设备树中的平台信息确认内核是否支持
接下来会解析出设备树中的运行时配置，主要包括bootarg信息，根节点的address-cells，size-cells的值，memory节点的reg信息

通过上述可知道，最后我们内核加载的是dtb文件，现在问题来了：dtb的加载过程是怎样的呢？

分割设备树
首先将设备树分割成两部分：

主DT。由SOC供应商提供的SOC公用部分和默认配置。
叠加DT。由原始设计制造商(ODM)/原始设备制造商(OEM)提供的设备专属配置.
编译主DT和叠加DT
要编译主DT，请执行以下操作：

（1）将主.dts编译为.dtb文件。
（2）将.dtb文件刷写到引导加载程序在运行时可访问的分区。
要编译叠加DT，请执行以下操作：

（1）将叠加DT .dts编译为.dtbo文件。虽然文件格式与已格式化为扁平化设备树的.dtb文件相同，但是用不同的文件扩展名可以将其与主DT分开来。 //高通用这种方式
（2）将.dtbo文件刷写到引导加载程序在运行时可访问的分区

对DT进行分区
在内存中确定加载程序在运行时可访问和可信的位置以放入.dtb和.dtbo。

主DT的实例位置：

引导分区的一部分，已附加到内核(image.gz)
单独的DT blob(.dtb)，位于专用的(dtb)中。
在引导加载程序中运行
将 .dtb 从存储加载到内存中
将 .dtbo 从存储加载到内存中
用 .dtbo 叠加 .dtb 以形成合并的 DT
启动内核（已给定合并 DT 的内存地址）


## 最新 
1、dts的提出是为方便统一硬件资源，减少内核的代码冗余。dts转为dtb后在u-boot阶段加载进内核。
2、内核解析dtb-->device_node-->platform_device 
3、在msm-4.9平台上，dtbo横空出世（准确来说是出厂搭载安卓9的要求）。device tree被拆分到了两个地方，
一个是boot分区中的老位置，另一个则是dtbo分区。谷歌做这件事的初衷在于：希望分离芯片厂商和手机厂商的修改，
芯片厂商只修改内核中的dtb，而手机厂商只修改dtbo分区

# #dtbo编译
       多 DTBO 结构，编译时同一个 BSP_BUILD_FAMILY 下的多个 DTBO 都会打包到 dtbo.img 中。以 SC9863A 为例，图 图 1-1 为其多 DTBO 结构，其中 sp9863a-3h10-overlay.dtbo-base、sp9863a-1c10-overlay.dtbo-base 和 sp9863a-1h10-overlay.dtbo-base 指定的 sp9863a.dtb 都会编译并打包到 dtb.img 中。

![图 0](../images/0bfc10852f892ad5eb9dc223b1d51d5d276ee02443ebe22df0169176d0a2ecbb.png)  

     U-Boot 在启动时，它会匹配 DTBO 并将其内容合入或者覆盖到匹配的 DTB 中，随后加载到 DDR，再将地址传给 Kernel.
     dtb.img 在 boot header V2 时会打包到 boot.img，而在 boot header V3/V4 时会打包到 vendorboot.img。
     系统引入 DTBO 后，设备树分割为两部分：
      Native DT：Vendor 供应商提供的 SoC 公用部分和 Vendor 默认配置。
      Overlay DT：ODM/OEM 提供的专属配置或客制化差异配置。 