<table style="width:100%">
  <tr>
    <td align="center" width="100%" colspan="6"><img src="https://www.xilinx.com/content/dam/xilinx/imgs/press/media-kits/corporate/xilinx-logo.png" width="30%"/><h1>2018.3 SDAccel™开发环境教程</h1>
    <a href="https://github.com/Xilinx/SDAccel-Tutorials/branches/all">查看其他版本</a>
</td>
  </tr>
  <tr>
    <td align="center"><a href="README.md">Introduction</td>
    <td align="center"><a href="lab-1-introduction-to-the-sdaccel-gui.md">Lab 1: Introduction to the SDAccel Development Environment</td>
    <td align="center">Lab 2: Introduction to the SDAccel Makefile</a></td>
  </tr>
</table>

# Lab 2: Introduction to the SDAccel Makefile

以下实验使用了可以找到的Xilinx®SDAccel™示例GitHub存储库中的示例 [here](https://github.com/Xilinx/SDAccel_Examples).

## Step 1: 准备和设置SDAccel环境

在此步骤中，您将设置SDx™环境以在命令行中运行并克隆SDAccel™的GitHub存储库。

1. 启动终端窗口并使用以下命令之一获取SDx环境中的设置脚本：

   ```c
   source <SDx_install_location>/<version>/settings64.csh
   ```

   ```c
   source <SDx_install_location>/<version>/settings64.sh
   ```

   这使您无需使用GUI即可运行SDx命令行。

   如果您通过SDx IDE下载了SDAccel示例（如实验1中所述），则可以从该位置访问这些文件。在Linux上，使用以下命令将文件下载到 `/home/<user>/.Xilinx/SDx/<version>/sdaccel_examples/` 到您选择的工作区：

   ```
   git clone https://github.com/Xilinx/SDAccel_Examples <workspace>/examples
   ```

   >**注意**: 这个GitHub存储库大小约为400 MB。确保本地或远程磁盘上有足够的空间。

2. 下载完成后，使用以下命令找到SDAccel示例中的 `vadd` 目录：

   ```C
   cd <workspace>/examples/getting_started/host/helloworld_c
   ```

   在此目录中，运行 `ls` 命令并查看文件，其中显示以下内容：

   ```C
   [sdaccel@localhost helloworld_c]$ ls
   Makefile    README.md    description.json src utils.mk
   ```

   如果在 `src` 目录上运行 `ls` 命令，它将显示以下内容：

   ```C
   [sdaccel@localhost helloworld_c]$ ls src
   host.cpp    vadd.cpp
   ```

## Step 2: 初始设计和Makefile探索

`helloworld_c` 目录包含Makefile文件，您将使用它：

* 在硬件和软件仿真中编译设计
* 生成系统运行

1. 在文本编辑器中，打开Makefile。

2. 查看内容并熟悉其编写方式，例如它使用的bash样式语法。

   >**注意**: 该文件本身引用了所有GitHub示例设计使用的通用makefile。

   前几行包含所有示例使用的其他通用makefile的 `include` 语句。

   ```c
   COMMON_REPO = ../../../
   ABS_COMMON_REPO = $(shell readlink -f $(COMMON_REPO))

   include ./utils.mk
   ```

3. 向下滚动到第22行并查看以下内容：

   ```c
   TARGETS:=hw
   ```

   这里，`TARGETS` 定义了默认构建（如果没有在makefile命令行中指定）。默认情况下，它设置为 `hw`（系统构建）。当您处理自己的设计时，您将设置所需的值。

4. 打开 `./utils.mk` makefile，它包含构建主机和源代码所需的标志和命令行编译器信息。

   ```c
   # By Default report is set to none, no report will be generated  
   # 'estimate' for estimate report generation  
   # 'system' for system report generation  
   REPORT:=none
   PROFILE ?= no
   DEBUG ?=no

   ifneq ($(REPORT),none)  
   CLFLAGS += --report $(REPORT)  
   endif

   ifeq ($(PROFILE),yes)
   CLFLAGS += --profile_kernel data:all:all:all
   endif

   ifeq ($(DEBUG),yes)
   CLFLAGS += --dk protocol:all:all:all
   endif
   ```

   >**注意**: `REPORT`, `PROFILE` 和 `DEBUG` 是终端中 `make` 命令的输入标志（参数）。请注意， `CLFLAGS` 正在构建一个要使用的 `xocc` 命令行标志的长列表。
   
5. 关闭 `utils.mk` 并重新回到 Makefile.

6. 回顾第44行及以后的内容。请注意，此文件处理源代码所在的大部分位置，并命名内核和应用程序可执行文件。

## Step 3: 运行软件仿真

现在你已经了解了makefile构造的一部分，现在是时候编译代码来运行Software Emulation了。

1. 要编译软件仿真的应用程序，请运行以下命令：

   ```C
   make all REPORT=estimate TARGETS=sw_emu DEVICES=xilinx_u200_xdma_201830_1`
   ```

   生成的四个文件是：

   * host (host executable)
   * `xclbin/vadd.sw_emu.xilinx_u200_xdma_201830_1.xclbin` (binary container)
   * A system estimate report
   * `emconfig.json`

2. 要验证是否生成了这些文件，请在目录中运行 `ls` 命令。

   显示以下内容：

   ```C
   [sdaccel@localhost helloworld_c]$ ls
   description.json
   Makefile
   README.md
   src
   host
   _x  this directory contains the logs and reports from the build process.
   xclbin
   [sdaccel@localhost helloworld_c]$ ls xclbin/
   vadd.sw_emu.xilinx_u200_xdma_201830_1.xclbin
   vadd.sw_emu.xilinx_u200_xdma_201830_1.xo
   xilinx_u200_xdma_201830_1 this folder contains the emconfig.json file
   ```

3. 要在仿真中运行应用程序，请运行以下命令：

   ```C
   make check PROFILE=yes TARGETS=sw_emu DEVICES=xilinx_u200_xdma_201830_1
   ```

   >**注意**: 确保上面指定的 `DEVICES` 的值与 **Initial Design and Makefile Exploration** 部分中用于编译的值相同。

   在此流程中，它运行上一个命令和应用程序。

   如果应用程序成功运行，则终端中将显示以下消息：

   ```C
   [sdaccel@localhost helloworld_c]$ make check TARGETS=sw_emu DEVICES=xilinx_u200_xdma_201830_1
   cp -rf ./xclbin/xilinx_u200_xdma_201830_1/emconfig.json .
   XCL_EMULATION_MODE=sw_emu ./host
   Found Platform
   Platform Name: Xilinx
   XCLBIN File Name: vadd
   INFO: Importing xclbin/vadd.sw_emu.xilinx_u200_xdma_201830_1.xclbin
   Loading: 'xclbin/vadd.sw_emu.xilinx_u200_xdma_201830_1.xclbin'
   TEST PASSED
   sdx_analyze profile -i sdaccel_profile_summary.csv -f html
   INFO: Tool Version : 2018.3
   INFO: Done writing sdaccel_profile_summary.html
   ```

   要生成其他报告，您可以执行以下操作之一：

   * 设置环境变量
   * 使用适当的信息和权限创建一个名为 `sdaccel.ini` 的文件。

## Step 4: 生成其他报告

在本教程中，您将在 `helloworld_c` 目录中创建 `sdaccel.ini` 文件，并添加以下内容：

```C
[Debug]
timeline_trace = true
profile = true
```

1. 运行以下命令的命令：

   ```C
   make check PROFILE=yes TARGETS=sw_emu DEVICES=xilinx_u200_xdma_201830_1
   ```

   应用程序完成后，还有一个名为 `sdaccel_timeline_trace.csv`的附加时间线跟踪文件。

2. 要在GUI中查看跟踪报告，请使用以下命令将CSV文件转换为WDB文件：

   ```C
   sdx_analyze trace sdaccel_timeline_trace.csv
   ```

   该应用程序以CSV格式生成名为 `sdaccel_profile_summary` 的分析摘要报告。

3. 要在SDx IDE中浏览报告，请将 `sdaccel_timeline_trace.csv` 转换为实验1：配置文件摘要中显示的报告类型。运行以下命令：

   ```C
   sdx_analyze profile sdaccel_profile_summary.csv
   ```

   这会生成一个 `sdaccel_profile_summary.xprf` 文件。

4. 要查看此报告，请在SDx IDE中选择 **File** > **Open File**，然后从菜单中选择文件。

   报告如下所示。

   ![error: missing image](./images/183_lab2-sw_emu_profile.png)

   >**注意**: 要查看这些报告，您不需要使用先前在实验1中使用的工作空间。要在本地创建工作空间以查看这些报告，请使用以下命令：
   >```C
   >sdx -workspace ./lab2
   >```
   >
   > 您可能还需要关闭“Welcome Window”才能查看报告。
   >
   >软件仿真不提供所有分析信息（例如内核和全局内存之间的数据传输）。此信息可在硬件仿真和系统中获得。

   系统估计报告 (`system_estimate.xtxt`)也会生成。这是使用`xocc`命令编译时使用的 `--report` 转换。

   ![error: missing image](./images/183_lab2_sw_emu_sysestimate.png)

5. 如前所述，在SDx IDE中，选择 **File** > **Open File** 以找到 `sdaccel_timeline_trace.wdb` 文件，该文件将打开以下报告：

   ![error: missing image](./images/183_lab2-sw_emu_timeline.png)

## 运行硬件仿真

1. 软件仿真完成后，您可以运行硬件仿真。要在不更改makefile的情况下执行此操作，请运行以下命令：

   ```C
   make all REPORT=estimate TARGETS=hw_emu DEVICES=xilinx_u200_xdma_201830_1
   ```

   当您以这种方式定义 `TARGETS` 时，它会传递该值并覆盖在makefile中设置的默认值。
   
   >**注意**: 硬件仿真比软件仿真需要更长的编译时间。

2. 重新运行已编译的主应用程序。

   ```C
   make check TARGETS=hw_emu DEVICES=xilinx_u200_xdma_201830_1
   ```

   >**注意**: makefile将环境变量设置为 `hw_emu`。
   >
   >您不需要重新生成 `emconfig.json`，因为设备信息没有更改。但是，需要为硬件仿真设置仿真。

   输出类似于Software Emulation输出，如下图所示。

   ```C
   [sdaccel@localhost helloworld_c]$ make check TARGETS=hw_emu DEVICES=xilinx_u200_xdma_201830_1
   cp -rf ./xclbin/xilinx_u200_xdma_201830_1/emconfig.json .
   XCL_EMULATION_MODE=hw_emu ./host
   Found Platform
   Platform Name: Xilinx
   XCLBIN File Name: vadd
   INFO: Importing xclbin/vadd.hw_emu.xilinx_u200_xdma_201830_1.xclbin
   Loading: 'xclbin/vadd.hw_emu.xilinx_u200_xdma_201830_1.xclbin'
   INFO: [SDx-EM 01] Hardware emulation runs simulation underneath. Using a large data set will result in long simulation times. It is recommended that a small dataset is used for faster execution. This flow does not use cycle accurate models and hence the performance data generated is approximate.
   TEST PASSED
   INFO: [SDx-EM 22] [Wall clock time: 00:10, Emulation time: 0.109454 ms] Data transfer between kernel(s) and global memory(s)
   vadd_1:m_axi_gmem-DDR[1]          RD = 32.000 KB              WR = 16.000 KB       

   sdx_analyze profile -i sdaccel_profile_summary.csv -f html
   INFO: Tool Version : 2018.3
   Running SDx Rule Check Server on port:40213
   INFO: Done writing sdaccel_profile_summary.html
   ```

3. 要查看配置文件摘要和时间线跟踪，必须将它们转换为SDx IDE以读取和查看更新的信息。使用以下命令：

   ```C
   sdx_analyze profile sdaccel_profile_summary.csv
   sdx_analyze trace sdaccel_timeline_trace.csv
   ```

   将显示“Profile Summary”，如下图所示。
   
   ![error: missing image](./images/183_lab2-hw_emu_profile.png)

## 系统运行

1. 要编译系统运行，请使用以下命令：

   ```C
   make all TARGETS=hw DEVICES=xilinx_u200_xdma_201830_1
   ```

   >**注意**: 构建系统可能需要很长时间，具体取决于计算机资源。

2. 要在U200加速卡上运行设计，请安装板和部署软件，如 _Getting Started with Alveo Data Center Accelerator Cards_ 所述([UG1301](https://www.xilinx.com/support/documentation/boards_and_kits/accelerator-cards/ug1301-getting-started-guide-alveo-accelerator-cards.pdf)).

3. 成功安装并验证卡后，使用以下命令运行该应用程序：

   ```C
   make check TARGETS=hw DEVICES=xilinx_u200_xdma_201830_1
   ```

4. 系统运行完成后，要将配置文件摘要和时间线跟踪转换为SDx环境可以读取的文件，请使用以下命令：

   ```C
   sdx_analyze profile sdaccel_profile_summary.csv
   sdx_analyze trace sdaccel_timeline_trace.csv
   ```

## 总结

完成本教程后，您将了解如何执行以下操作：

* 设置SDx环境以运行终端中的所有命令。
* 复制GitHub存储库。
* 运行xcpp，xocc，emconfigutil，sdx_analyze profile，sdx_analyze trace命令以生成应用程序，二进制容器和仿真模型。
* 编写一个makefile来编译OpenCL™内核和主机代码。
* 在文本编辑器或SDx IDE中查看仿真生成的文件。
* 设置环境并部署要与平台一起使用的设计。

<hr/>
<p align="center"><sup>Copyright&copy; 2019 Xilinx</sup></p>
