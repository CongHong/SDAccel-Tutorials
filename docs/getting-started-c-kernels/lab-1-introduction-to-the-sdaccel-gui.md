<table style="width:100%">
  <tr>
    <td  align="center" width="100%" colspan="6"><img src="https://www.xilinx.com/content/dam/xilinx/imgs/press/media-kits/corporate/xilinx-logo.png" width="30%"/><h1>2018.3 SDAccel™开发环境教程</h1>
    <a href="https://github.com/Xilinx/SDAccel-Tutorials/branches/all">查看其他版本</a>
</td>
  </tr>
  <tr>
    <td colspan="3" align="center"><h1>Getting Started with C/C++ Kernels</h1></td>
  </tr>  <tr>
    <td align="center"><a href="README.md">Introduction</td>
    <td align="center">Lab 1: Introduction to the SDAccel Development Environment</td>
    <td align="center"><a href="lab-2-introduction-to-the-sdaccel-makefile.md">Lab 2: Introduction to the SDAccel Makefile</a></td>
  </tr>
</table>

# Lab 1: Introduction to the SDAccel Development Environment

本实验演示了两种不同的流程：步骤1-3解释GUI流程，步骤4解释Makefile流程。

本实验使用Xilinx®SDAccel™示例GitHub存储库中的示例，该存储库可在此处找到。 [here](https://github.com/Xilinx/SDAccel_Examples).

## Step 1: 从GitHub示例创建SDAccel项目

1. 使用 `sdx` 命令在Linux的终端窗口中启动SDx开发环境。

   将显示“Workspace Launcher”对话框。

2. 选择工作区的位置，这是项目所在的位置。

   ![error: missing image](./images/183_launcher.png)

3. 单击 **Launch**。

   欢迎窗口打开。

   >**注意**: 首次使用该工具时，或者通过选择 **Help > Welcome**，将打开“Welcome”窗口。

4. 在“Welcome”窗口, 单击 **Create Application Project**。

   ![SDxIDEWelcome.png](images/SDxIDEWelcome.png)

   将打开“New SDx Application Project”对话框。

5. 创建项目。
   1. 在项目名称中，输入 `helloworld`。
   2. 选择 **Use default location**。
   3. 单击 **Next**。  

   ![error: missing image](./images/183_application_project.png)

   “Platform”页面打开。

6. 选择 `xilinx_u200_xdma_201830_1`, 然后单击 **Next**。

   ![error: missing image](./images/183_hardware_platform_dialog.png)

   将打开“Templates”页面，其中显示可用于开始构建SDAccel环境项目的模板。除非您已下载其他SDx示例，否则只应将_Empty Application_和_Vector Addition_视为可用模板。

   >**注意**: 您选择的硬件平台类型将项目确定为SDAccel或SDSoC™环境项目。

7. 在本实验中，您将使用GitHub存储库中的Helloworld示例。要下载示例，请单击 **SDx Examples**。

   ![error: missing image](./images/183_example_empty.png)

8. 选择 **SDAccel Examples**, 然后单击 **Download**。

   ![error: missing image](./images/183_example_download.png)

   系统将GitHub存储库克隆到指定位置。下载完成后，将填充并扩展SDAccel示例树表。

   >**注意**: 下载可能需要很长时间，具体取决于您的连接速度。将显示“Progress Information”对话框，直到完成存储库克隆。

9. 单击 **OK**。 这将关闭窗口并返回“Templates”窗口。

   ![error: missing image](./images/183_example_full.png)

   在“Templates”模板窗口中填充了SDAccel GitHub示例。 

10. 在 **Find**中, 输入 `hello`, 然后从Host Examples中找到Hello World（HLS C / C ++ Kernel）。

11. 单击 **Finish**。

    ![error: missing image](./images/183_github_example_new.png)

   _helloworld_项目在SDAccel环境中创建并显示，如下图所示。

   ![error: missing image](./images/183_helloworld_project.png)

有关SDx IDE功能的更多信息，请参阅 _SDAccel Environment User Guide_ ([UG1023](https://www.xilinx.com/cgi-bin/docs/rdoc?v=2018.3;d=ug1023-sdaccel-user-guide.pdf))。

## Step 2: 运行软件仿真

此步骤通过执行以下操作向您显示如何为设计运行软件仿真：

* 设置“运行配置”设置
* 打开报告
* 启动调试

有关报告和调试的详细信息，请参阅 _SDAccel Environment User Guide_ ([UG1023](https://www.xilinx.com/cgi-bin/docs/rdoc?v=2018.3;d=ug1023-sdaccel-user-guide.pdf))。

1. 转到"SDx Application Project Settings"，在右上角，将 **Active build configuration** 设置为 **Emulation-SW**。

   ![error: missing image](./images/183_project_settings_sw.png)

   GitHub示例包含设计加速器，因此您无需添加硬件功能。

   >**注意**: 要将硬件功能添加到设计中，请单击 ![error: missing image](./images/qpg1517374817485.png). 这将分析C / C ++代码并确定可用于加速的函数。
2. 单击 ![[the Run Button]](./images/lvl1517357172451.png) (**Run**)。

   这会在运行仿真之前构建项目。

   >**注意**: 构建和仿真过程可能需要几分钟或更长时间才能完成。在此期间，打开“ Run Configurations”对话框，以查看如何添加特定命令行选项以自定义构建。

3. 转到“Run”菜单，然后选择 **Run Configurations**。

   在Arguments选项卡下的Program Arguments字段中，您可以添加Xilinx OpenCL™ Compiler (XOCC)命令行标志和开关。有关命令选项的说明，请参阅 _SDx Command and Utility Reference Guide_ ([UG1279](https://www.xilinx.com/cgi-bin/docs/rdoc?v=2018.3;d=ug1279-sdx-command-utility-reference-guide.pdf))。 在本教程中，设计无需任何命令行参数即可运行。
   在“Profile”选项卡中，有一个用于生成时间线跟踪报告的下拉菜单。您可以单击选项以查看生成的报告类型。此选项卡中还有一个用于 **Enable Profiling** 的复选框。

4. 关闭窗口而不更改任何内容。

   >**注意**: 如果在“Run Configurations”对话框中进行更改，请单击 **Run** 以重新运行当前模拟步骤并查看所做的更改。
   控制台窗口现在显示 `TEST PASSED`.

5. 仿真运行完成后，您可以查看“Profile Summary”和“Application Timeline”报告，以获取有关进一步优化的详细信息。在“Assistant”窗口中，双击 **Profile Summary**，如下所示。

   ![error: missing image](./images/183_assistant_reports_sw.png)

   在这里，您可以查看可用于优化设计的操作，执行时间，带宽和其他有用数据。
   
   > **注意**: 您的摘要编号可能与下图不同。

   ![error: missing image](./images/183_profile_summary_sw.png)

6. 要查看“Application Timeline”报告，请在“Assistant”窗口中，双击 **Application Timeline**。

   这显示了主机代码和内核代码的细分，以及每个代码的执行时间。要放大特定区域，请单击并向右拖动鼠标。

   ![error: missing image](./images/183_application_timeline.png)  

   “Profile Summary”和“Application Timeline”提供有关主机代码和内核如何通信和处理内核信息的数据。您可以使用“Debug”功能来完成主机内核处理并识别问题。
7. 在Project Explorer窗口中，要在编辑器中打开文件，请双击 **host.cpp** (位于 **Explorer > src` 目录**).

8. 在可以在Debug中运行之前，必须设置断点。在执行的关键点设置断点有助于识别问题。要在内核调试之前暂停主机代码，请在(`OCL_CHECK(err, err = q.enqueueMigrateMemObjects({buffer_in1, buffer_in2},0/* 0 means from host*/));`)上的蓝色区域（见下图）中右键单击 <`line 99`> 并选择 **Toggle Breakpoint**。

   ![error: missing image](./images/debug_breakpoint_hw.PNG)

9. 运行Debug，请单击 ![missing image: the Debug icon](./images/cwo1517357172495.png)。

   将打开一个对话框，显示切换透视图的选项。

10. 单击 **Yes**.

    默认情况下，调试器在 `main`的第一行插入一个自动断点。在“Runs Configuration”对话框的“Debugger”选项卡上，有一个选项可以在 `main` 函数上停止，该函数默认启用（如下所示）。这对于需要更彻底调试的有问题的功能是有帮助的。

11. _(Optional)_ Using Eclipse 调试，您可以更详细地检查主机和内核代码。逐步调试的所有控件都在“Run”菜单或主工具栏菜单中。

12. 恢复到下一个断点：
    1. 按 **F8**.
    2. 选择 **Run**>**Resume**.

       ![error: missing image](./images/debug_configuration_hw.PNG)

13. 在恢复调试之后，SDx工具为内核代码启动另一个gdb实例，该实例在函数开头也有一个断点。

    您可以使用此断点来执行内核的详细分析，以及如何将数据读入函数并写入内存。在gdb中完成内核执行后，该实例终止，然后返回主调试线程。

14. 继续，请按 **F8**。

    >**注意**  控制台视图仍显示内核调试输出。单击 ![error: missing image](./images/gqm1517357172417.png) 以返回到vadd.exe控制台并查看主机代码的输出。

15. 要关闭Debug透视图，请执行以下操作之一：

    * 在窗口的右上角显示 ![error: missing image](./images/cwo1517357172495.png) (Debug), 右击, 然后选择 **Close**。
    * 单击 ![error: missing image](./images/sdx_perspective_icon.PNG) (SDx) 切换到标准SDx透视图。

16. 进入"SDx Perspective"后，关闭“Project Editor”窗口中的所有选项卡，“Application Project Settings window”窗口除外。

## Step 4: 运行硬件仿真

此步骤包括运行硬件仿真功能，以及查看分析和报告的基础知识。

Emulation-SW和Emulation-HW之间的主要区别在于，当模拟硬件时，Emulation-HW会构建一个更接近平台上所见的设计，为内核代码合成RTL。这意味着与带宽，吞吐量和执行时间相关的数据更准确。但是，这会导致设计编译时间更长。

1. 要运行硬件仿真，请转到SDx应用程序设置，并确保 **Active build configuration** 设置为Emulation-HW，然后单击 **Run**。这需要一些时间来完成。

2. 在“Assistant”选项卡的Emulation-HW配置下，打开“System Estimate report”报告。此文本报告提供有关内核信息，设计时序，时钟周期和设备中使用的区域的信息。

   ![error: missing image](./images/183_system_estimate_hw.png)

3. 在“Reports”选项卡中，双击以打开“Profile Summary”报告。此报告提供与内核操作，数据传输和OpenCL API调用相关的详细信息，以及与资源使用情况相关的分析信息，以及与内核/主机之间的数据传输。

   >**注意**: 硬件仿真中使用的仿真模型是近似的。显示的配置文件编号只是一个估计值，可能与实际硬件中获得的结果不同。

   ![error: missing image](./images/183_profile_summary_report_hw.png)

   在“Console”选项卡旁边，将显示“Guidance”选项卡。这是未完成检查提供有关如何优化内核的一些信息的地方。

   ![error: missing image](./images/183_guidance_view_hw.png)

   >**注意**: 要查看其他性能优化技术和方法，请参阅 _SDAccel Profiling and Optimization Guide_ ([UG1207](https://www.xilinx.com/cgi-bin/docs/rdoc?v=2018.3;d=ug1207-sdaccel-optimization-guide.pdf)).

4. 在“Reports”选项卡中，双击以打开“Application Timeline ”报告。此报告显示主机和内核完成任务所需的估计时间，并提供有关bottlenecks可能的更详细的信息。添加标记，缩放和扩展信号有助于识别bottlenecks。

   ![error: missing image](./images/183_timeline_hw.png)  

5.要打开HLS报告，请展开Emulation-HW选项卡，然后展开相关的内核选项卡。

   该报告提供了Vivado®HLS在内核转换和综合方面提供的详细信息。报告底部的选项卡提供了有关内核和其他性能相关数据花费大部分时间的更多信息。本报告还显示了包括延迟和时钟周期在内的性能数据。

   ![error: missing image](./images/183_hls_hw.png)  

## Step 5: 使用Makefile流程

此步骤说明了makefile流程的基础知识以及SDx™IDE如何使用它。使用此流程的优点包括：

* 易于移植到任何系统
* 小型设计变更的周转时间更短

1. 在Project Explorer中，导航到Emulation-SW目录，然后查找makefile文件。
2. 双击该文件以在编辑器中将其打开。 SDx IDE创建此makefile并将其用于构建和运行仿真。或者，您可以导航到 `Emulation-HW` 目录并查找makefile文件。

   请注意，每个构建都有一个唯一的makefile。在编辑器窗口中打开的makefile中，查看第21行。注意，如果是 `hw_emu` 或 `sw_emu`，它指定一个目标。

   >**提示**: 您还可以使用SDx IDE生成的makefile在GUI之外构建项目。

3. 打开新的终端会话并找到工作区。

4. 找到Emulation-SW目录并输入 `make incremental`。

   该过程产生典型的SDx日志输出。
   
   >**注意**: 如果未对主机或内核代码进行任何更改，则无效，因为编译已完成。它可能会显示如下消息：
   >
   >_make: Nothing to be done for `incremental`._

[Lab 2: Introduction to the SDAccel Makefile](./lab-2-introduction-to-the-sdaccel-makefile.md) 详细介绍了如何使用makefile和命令行流程。
## 总结

完成本教程后，您将了解如何执行以下操作：

* 从GitHub示例设计创建SDAccel环境项目。
* 为设计创建二进制容器和加速器。
* 运行Software Emulation并在主机和内核代码上使用Debug环境。
* 运行硬件仿真并使用报告来了解可能的优化。
* 了解软件和硬件仿真报告之间的差异。
* 读取项目makefile并运行makefile命令行。

## Lab 2: Introduction to the SDAccel Makefile

[Lab 2: Introduction to the SDAccel Makefile](./lab-2-introduction-to-the-sdaccel-makefile.md) 详细介绍了如何使用makefile和命令行流程。

<hr/>
<p align="center"><sup>Copyright&copy; 2019 Xilinx</sup></p>
