## 开发板介绍

我们计算机组成原理与硬件综合设计课程中使用的[Nexys 4 DDR](https://digilent.com/reference/programmable-logic/nexys-4-ddr/start) FPGA开发板能够运行龙芯开发的[Chiplab](https://gitee.com/loongson-edu/chiplab)软核。真实的Chiplab软核与QEMU存在一些差异，可以帮助同学们发现操作系统中未考虑到真实硬件的一些问题（例如Cache一致性、MMIO保序等问题）。

## 步骤

### 1. 使用`loongarch32r-linux-gnusf-strip`对elf文件不必要的部分进行裁剪，否则PMON会报错。

在ucore-loongarch32的obj文件夹执行以下命令：
```bash
loongarch32r-linux-gnusf-strip ucore-kernel-initrd
```

### 2. 将PMON（Bootloader）烧写到Nexys 4 DDR开发板的SPI Flash

1. 将开发板连上电脑，打开Vivado的Hardware Manager
2. 下载[PMON文件](../static/gzrom-uart_div_18.bin)

    !!! warning

        注意，Nexys 4 DDR开发板烧写SPI Flash的方式与龙芯杯开发板以及百芯计划开发板不同。请勿按照[官方流程](https://chiplab.readthedocs.io/zh/latest/FPGA_run_linux/flash.html)进行。

        此外，目前龙芯提供的PMON文件（2022年6月6日版本），串口波特率存在较大偏差。
        
        根据计算，串口分频比=33MHz/16/115200bauds=17.90，如果分频比设置为17而非18会导致串口波特率偏移达到5%，实际上这一数值应该四舍五入。
        
        但Nexys 4 DDR开发板的USB串口芯片对波特率非常敏感，如果使用该版本PMON会导致串口时常出现乱码的情况。


3. 在Hardware Manager中连接开发板，并选中FPGA芯片，点击右键，选择`Add Configuration Memory Device`。

4. 在弹出的窗口中搜索芯片`s25fl128sxxxxxx0-spi-x1_x2_x4`。

5. 使用刚刚下载的PMON文件`gzrom-uart_div_18.bin`进行烧写。


### 3. 使用Vivado生成Chiplab的FPGA bit流。

下载助教做好的Vivado工程。

```bash
git clone https://gitee.com/cyyself/chiplab.git -b n4ddr_with_cpu
cd chiplab
open fpga/nexys4ddr/system_run/system_run.xpr # 如果没有open命令可以自己用Vivado打开这个xpr文件
```

直接生成bit流，然后烧到开发板上即可。（这个流程大家做过数字逻辑和组成原理的实验应该很熟悉。）

### 4. 将bit流写到开发板上。

这个流程大家做过数字逻辑和组成原理的实验应该很熟悉。

需要注意的是SPI Flash已被我们的PMON占用，所以请勿固化。

### 5. 剩余流程

以下流程可直接参考[龙芯Chiplab文档](https://chiplab.readthedocs.io/zh/latest/FPGA_run_linux/linux_run.html#id1)，从4.3节开始，4.5节除外（因为Nexys 4 DDR没有NAND Flash）。

- 使用串口软件连接开发板的USB串口，使用115200波特率，8n1，无流控。
- 使用网线连接自己的电脑和开发板
- 在PMON中配置IP

    我们可以为连接开发板的网卡随意配置一个静态IP，例如PC配置为169.254.0.1。

    之后在串口中的PMON也配置同网段IP：

    ```bash
    ifconfig dmfe0 169.254.0.2
    ping 169.254.0.1
    ```

- 在电脑上运行tftp服务器

    - 对于Windows用户，推荐使用[Tftpd64](https://pjo2.github.io/tftpd64/)

    - 对于Linux用户，推荐使用tftpd-hpa，对于Ubuntu可以参考[Wiki](https://help.ubuntu.com/community/TFTP)。

    使用WSL2的同学注意：WSL2中网卡为NAT模式，TFTP协议使用UDP传输，请确保你使用的端口转发方式能够正确处理UDP，否则建议将文件复制出来，在Host端Windows运行。

- 在PMON中载入uCore内核

    在PMON中执行：
    ```bash
    load tftp://169.254.0.1/ucore-kernel-initrd
    g
    ```
    
    注：uCore不会读取Bootloader传递的Kernel command line，因此不需要像启动Linux一样在g后面添加参数。

## 更多帮助

如果同学们想要在这个开发板上运行LoongArch32的Linux，可以阅读更多说明：

1. [Chiplab帮助文档](https://chiplab.readthedocs.io/)
2. [Chiplab移植N4DDR说明](https://gitee.com/cyyself/chiplab/tree/n4ddr_with_cpu/fpga/nexys4ddr)