## Zynq KR260裸机固件上传

本文提供生成BOOT.bin并上传到KR260开发板的方式，供读者参考。
写本文是因为在这一块踩了坑，现在摸索出来了，写出来方便大家。

### 笔者踩了的坑

KR260板上的QSPI带官方的默认固件，且KR260从该固件启动。然而该固件是U-Boot，因此读取SD卡时并不会加载裸机BOOT.bin，而是加载Linux启动用的`boot.scr.uimg`。因此除非QSPI内有带能通过USB集线器读取SD卡内BOOT.bin的FSBL，否则直接拷贝BOOT.bin到SD卡内并不能正确启动裸机程序。

另外笔者注意到KR260自带的SD读卡器是连接到板上的USB集线器的，因此不能直接通过PS侧的SD卡外设读取SD卡内部的BOOT.bin。希望有帮到想探索SD卡启动的读者。

另另外，如果你在Vivado里修改了设计，在Vitis 2025.2里重新导入xsa文件后，一些关键的文件是不会更新的。比如所有app的`_ide`目录下的比特流和初始化tcl脚本、硬件平台自带的FSBL下的`workspace根目录/platform/zynqmp_fsbl/psu_init.c`。这会导致BSP生成的FSBL初始化过程过期以及调试时用的TCL初始化脚本过期。FSBL的`psu_init.c`的问题可以通过导入FSBL模板解决（应该是每导入一次xsa就导入一次FSBL模板）。TCL初始化脚本的问题，可以在调试时切换为FSBL初始化，或者调试启动设置里选择`platform/export/hw/sdt/`下的比特流和初始化TCL脚本。

最后，Vitis生成的BOOT.bin可能会有问题，如果遇到问题可以考虑重新生成或重新上传固件。这是真的，在文章末尾用自定义FSBL生成BOOT.bin并上传后，两个串口中的一个不能输出。重新生成一份就ok了。也有可能是上传时出了问题，但考虑到有CRC32校验，这个可能性不太高。

环境：Vivado & Vitis 2025.2。其它版本步骤应该相似。

> 笔者的Vitis 2025.2可能在几个方面和旧版不一样：
> 开始工程前需要设置workspace，旧版可能是直接从Vivado里Launch SDK，不需要这一步。
> Platform Component应该对应旧版的hardware，即从XSA导入的硬件。
> App Component对应app project。
> 操作基本上和旧版是一样的，UI会有区别。
>
> 笔者用2025.2是因为2022在我的OS上有严重影响使用的bug，不一定是最好用的版本。

### 创建Vivado工程并导出硬件
这里创建一个Vivado示范工程。

Create Block Design，添加一个Zynq UltraScale+ MPSoC IP核。
点击界面上方的Run Block Automation对Zynq IP核应用开发板的初始设置。

![image-20260107123402749](https://github.com/fishBone000/KR260-BOOT.bin-HOWTO/blob/main/image-20260107123402749.png)将IP核右侧的两个pl_clk连接到左侧的maxihpmX_fpd_aclk。
双击进入IP核设置，把两个UART打开。UART1的MIO选择`MIO 36 .. 37`，UART0的输出选择EMIO走板上pin口。

> 这里选择MIO 36和37是因为KR260的这两个MIO输出到板上Micro-USB接口，可以通过官方的schematic电路图的JTAG/UART页看到。直接在schematic pdf里搜索MIO36和MIO37即可。

![image-20260107123605917](https://github.com/fishBone000/KR260-BOOT.bin-HOWTO/blob/main/image-20260107123605917.png)

点OK保存IP核设置。把IP核右侧的UART_0导出为外部接口（Make external）。

注意这里应用的默认设置打开了QSPI，这个是必要的，后面FSBL通过QSPI外设和QSPI储存沟通以加载比特流和应用ELF。

保存Block Design。生成HDL Wrapper。

接下来点Open Elaborated Design，为UART0设置针脚。`UART_0_0_rxd`选`W13`（对应KR260树莓派接口PIN10），`UART_0_0_rxd`选`W14`（对应树莓派接口PIN8）。电平选择`LVTTL`（`LVCMOS33`也可以，都一样）从KR260树莓派接口往风扇看去，远一点的那一排，从左往右数，第三个针脚是PIN3（接地），第四个针脚是PIN8，第五个针脚是PIN10。将电脑上的USB转串口连接到这三个针脚。

![image-20260107124940710](https://github.com/fishBone000/KR260-BOOT.bin-HOWTO/blob/main/image-20260107124940710.png)

CTRL-S保存为`io.xdc`。文件命名无所谓。

然后生成比特流。导出xsa文件，记得包含比特流。

### 创建Vitis工程并生成BOOT.bin

打开Vitis，导入XSA文件。

这里对BSP做一些选项修改：

1. 修改FSBL的BSP的stdin和stdout，设置为`psu_uart_1`
2. 修改应用的BSP的stdin和stdout，检查是否为`psu_uart_1`

![image-20260107125941165](https://github.com/fishBone000/KR260-BOOT.bin-HOWTO/blob/main/image-20260107125941165.png)

![image-20260107130048192](https://github.com/fishBone000/KR260-BOOT.bin-HOWTO/blob/main/image-20260107130048192.png)

第一步其实是没有必要的，但是加上之后可以看到FSBL在串口上的输出。

> 实测FSBL的stdio在我这边不能调成None（如果有开启任何UART），宏会报错。如果不想在串口上输出FSBL信息，可以调成coresight，或者去改FSBL的代码（在`platform/zynqmp_fsbl`下）

然后重新生成两个BSP（这里是因为修改了`stdio`后某些编译配置文件不会更新，要重新生成BSP。这个局限应该是仅对于FSBL的BSP而言，但为了省时间应用的BSP也重新生成一下）。

![image-20260107130413857](https://github.com/fishBone000/KR260-BOOT.bin-HOWTO/blob/main/image-20260107130413857.png)

![image-20260107130456461](https://github.com/fishBone000/KR260-BOOT.bin-HOWTO/blob/main/image-20260107130456461.png)

编译BSP。

创建裸机应用工程。

创建main.c，代码如下：

```c
#include <xil_printf.h>
#include <sleep.h>
#include <xparameters.h>
#include <xuartps.h>

int main()
{
    xil_printf("main entry!\r\n");
    XUartPs uart;
    XUartPs_Config *pCfg = XUartPs_LookupConfig(XPAR_XUARTPS_0_BASEADDR);
    XUartPs_CfgInitialize(&uart, pCfg, pCfg->BaseAddress);
    u8 s[] = "Hello world!\r\n";
    while (1)
    {
        sleep(1);
        XUartPs_Send(&uart, s, sizeof(s));
    }
}
```

然后编译。

生成BOOT.bin。FSBL选platform component里刚编译好的FSBL，在Vitis 2025.2里是默认选好的。路径对应：`workspace根目录/platform/export/platform/sw/boot/fsbl.elf`。这里platform是platform component的名字。

这里笔者的比特流默认选择的是`workspace根目录/app_component/_ide/bitstream/design_1_wrapper.bit`。因为重新导入xsa文件后Vitis 2025.2不会更新这个文件，所以建议选择`workspace根目录/platform/export/platform/hw/sdt/design_1_wrapper.bit`。

Vitis 2025.2不会默认选择好bif输出路径，随便找个路径输出就好了。

### 上传固件

重点来了！

Zynq UltraScale+ MPSoC启动后PMU/CSU固件会读取BOOT MODE针脚以确定启动方式。从KR260电路图上可以看到针脚设置为QSPI启动模式。这意味着固件会以QSPI方式启动。

官方的QSPI里有默认的出厂固件，官方为了防止用户把固件搞坏锁死了QSPI，因此用户不能直接往QSPI里覆写固件。同时这个QSPI固件内置启动镜像恢复工具，允许用户上传BOOT.bin到QSPI内的两个镜像槽中，分别为镜像A和B。也就是说，用户不能直接往QSPI里写入固件，但是可以通过恢复工具写入QSPI中官方指定的两个位置。下面是官方的wiki链接：
https://xilinx-wiki.atlassian.net/wiki/spaces/A/pages/1641152513/Kria+SOMs+Starter+Kits#Boot-Image-Recovery-Tool

这里我们也同样用恢复工具上传刚刚生成的BOOT.bin。

先将KR260和电脑用网线连接起来。KR260上插四个网口的右下角那一个（必须是这一个）。电脑上设置网络为静态IP `192.168.0.xxx`，子网掩码`255.255.255.0`。注意不能是`192.168.0.111`，因为这个是KR260镜像恢复工具用的IP。

再连接USB线到电脑。注意KR260端是连接Micro USB口，因为UART1走MIO36、37后会连接到板上的串口转USB芯片，因此UART1串口数据最后是通过Micro USB数据线传输的。连接后电脑上应该会新出现四个串口设备，我们打开第二个，波特率调115200，格式是8N1。如果你不知道8N1是什么可以略去这一步，大部分串口工具默认都是8N1。同样也打开UART0的串口，波特率和格式相同。
建议查看固件在串口上的输出信息时用PuTTY等带终端功能的软件，有的固件（比如笔者板上带的镜像A）会在串口上输出终端信息，如果用不带终端的软件查看，可能会看到一堆乱码。此处附上Linux的PuTTY指令做参考：`putty /dev/ttyUSB1 -serial -sercfg 115200,8,n,1,N`。不过这里我们的FSBL不输出终端信息，直接`cat`或者用串口调试工具也是没问题的。

找到KR260板上的`FWUEN`和`RESET`按钮。前者是固件更新使能按钮，后者是重置按钮。两个按钮都在风扇旁边。按住`FWUEN`按钮的同时短按`RESET`按钮，这样板上的官方QSPI固件就能启动镜像恢复工具了。短按`RESET`按钮后就可以松开`FWUEN`了。

这里可以在串口上看到镜像恢复工具的启动信息。这里是否能看到和我们之前是否选择UART1和MIO36~37应该无关。

![image-20260107132817703](https://github.com/fishBone000/KR260-BOOT.bin-HOWTO/blob/main/image-20260107132817703.png)

然后打开浏览器，访问网址`http://192.168.0.111`。注意前面要有http前缀，否则浏览器可能会访问https对应的443端口。

然后就能看到界面了。

![image-20260107133008568](https://github.com/fishBone000/KR260-BOOT.bin-HOWTO/blob/main/image-20260107133008568.png)

上传刚刚生成的BOOT.bin到Image B（这里推荐Image B，防止默认的Image A被写入)，然后配置Requested Boot Image为Image B。

接下来短按`RESET`重置开发板。

可以看到串口上有简短的FSBL信息，我们的应用也开始`Hello World!`刷屏了！

![image-20260107133155469](https://github.com/fishBone000/KR260-BOOT.bin-HOWTO/blob/main/image-20260107133155469.png)

因为我们之前把FSBL和应用的BSP的stdio设置调成了uart1，因此我们可以从USB数据线上收到来自FSBL和应用的`xil_printf`信息。
应用代码里用UART0输出`Hello world!`，所以后者是在树莓派扩展针脚上收到的。又因为UART0走EMIO，经过PL到针脚，这表示我们的PL也配置成功了！

### 自定义FSBL

Vitis可以导入FSBL模板，这意味着我们对FSBL有更好的控制，比如增加调试信息甚至修改代码。同时因为导入XSA并不会更新`workspace根目录/platform/zynqmp_fsbl/psu_init.c`，我们可以导入FSBL模板来对这个问题work around。

这一章节不是必要的，但是是推荐的。

![image-20260108003110427](https://github.com/fishBone000/KR260-BOOT.bin-HOWTO/blob/main/image-20260108003110427.png)

Vitis里导入`ZynqMO FSBL`模板。笔者提示应用用的domain（可以理解为BSP）不符合FSBL模板的需求，这里新建一个domain，起名`fsbl_domain`。注意选择`Processor`为`psu_cortexa53_0`（默认是`2`）。

![image-20260107133845379](https://github.com/fishBone000/KR260-BOOT.bin-HOWTO/blob/main/image-20260107133845379.png)

修改`fsbl_domain`下的BSP的stdio为UART1。

![image-20260107134034367](https://github.com/fishBone000/KR260-BOOT.bin-HOWTO/blob/main/image-20260107134034367.png)

编译platform。

在刚刚创建的`zynqmp_fsbl`的编译选项里添加`FSBL_DEBUG_DETAILED`符号。这个宏定义可以在`xfsbl_debug.h`里找到，开启后FSBL会以最高verbosity输出。

然后编译FSBL。

回到一开头创建的应用，重新生成启动镜像。注意FSBL选择`workspace根目录/zynqmp_fsbl/build/zynqmp_fsbl.elf`。注意路径！导入XSA时也会生成`zynqmp_fsbl`，对应`workspace根目录/platform/zynqmp_fsbl/`，但我们选择的不是这一个FSBL。

然后上传固件，步骤和上面用镜像恢复工具一样。这里注意`Select Image to be recovered`自动变成了`Image A`（之前我们上传到B的缘故），我选择了`B`以防覆盖默认的`Image A`。

然后短按`RESET`按钮重置开发板。

可以看到详细的FSBL信息和应用的`Hello world!`！

![image-20260107141007173](https://github.com/fishBone000/KR260-BOOT.bin-HOWTO/blob/main/image-20260107141007173.png)
