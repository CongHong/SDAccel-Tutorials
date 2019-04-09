<table>
 <tr>
   <td align="center"><img src="https://www.xilinx.com/content/dam/xilinx/imgs/press/media-kits/corporate/xilinx-logo.png" width="30%"/><h1>2018.3 SDAccel™开发环境教程</h1>
   <a href="https://github.com/Xilinx/SDAccel-Tutorials/branches/all">查看其他版本</a>
   </td>
 </tr>
 <tr>
 <td align="center"><h3>Using Multiple Compute Units</h3>
 </td>
 </tr>
</table>

## 简介

本教程演示了一个灵活的内核链接过程，以增加FPGA上的内核实例数量。每个指定的内核实例也称为计算单元（CU）。此过程改进了组合主机 - 内核系统中的并行性。

## 背景

默认情况下，SDAccel™工具为每个内核创建一个硬件实例（也称为计算单元）。对于不同的数据集，主机程序可以多次使用相同的内核。在这些情况下，生成内核的多个计算单元以使这些计算单元同时运行并提高整个系统的性能非常有用。  

有关更多信息，请参见 _Multiple Instances of a Kernel_ in the _SDAccel Environment Programmers Guide_ ([UG1277](https://www.xilinx.com/support/documentation/sw_manuals/xilinx2018_3/ug1277-sdaccel-programmers-guide.pdf)).

<!-- We should provide some prerequisites for this tutorial, as well as some directions for accessing the lab materials either through cloning the repository or downloading a zip file from xilinx.com.-->

## 教程示例的说明

本教程使用图像过滤器示例来演示多个计算单元功能。主机应用程序处理图像，提取Y，U和V平面，然后运行内核三次以过滤图像的每个平面。默认情况下，这三个内核使用相同的硬件资源按顺序运行，因为FPGA只包含内核的单个硬件实例。本教程演示了如何增加主机应用程序调用的计算单元数，以同时过滤Y，U和V平面。

## 课程

在本教程中，您将：

1.创建新的应用程序项目并导入源文件。

2.运行硬件仿真并检查仿真报告以识别多个串行内核执行。

3.调整主机代码以启用无序命令执行。

4.更改内核链接过程以创建同一内核的多个实例。

5.重新运行硬件仿真，并确认计算单元的并行执行。 

## 教程演练

### 设置工作区

1.打开示例目录：`cd using-multiple-cu/reference-files/`。

2.调用SDx™环境GUI：`sdx`。

3.指定工作区，创建新的应用程序项目，然后将项目名称指定为`filter2d`。

4.选择`xilinx_u200_xdma_201830_1`作为平台。

5.在**Templates**中保留**Empty Application**，然后单击**Finish**。

SDx工具将在指定的工作区中创建和打开项目。

### 配置设置

1. 在"Project Explorer"窗口中，从`src / host`目录导入主机源文件，然后选择所有文件。
2. 在"Project Explorer"窗口中，从`src / kernel`目录导入`Filter2DKernel.xo`内核对象文件。
>**注意**: 内核代码已经是本教程中使用的编译目标文件（.xo）。实际上，`Filter2DKernel.xo`文件可以从C / C ++或RTL生成。从编译的目标代码开始，它们基本相同。您仍然可以从`.xo`文件开始自定义链接过程。
3. 在主项目窗口中，选择**Filter2DKernel**作为硬件功能。
4. 指定主机代码链接器选项： 
主机代码使用OpenCV™库进行映像文件操作，因此我们需要指定相关的链接器选项。
  1. 在"Project Explorer"窗口中，右击`filter2d`项目的顶级文件夹，然后选择**C/C ++ Build Settings**。
  2. 在“Settings”对话框中，选择 **SDX GCC Host Linker (x86_64)** 工具设置。
  3. 在Settings对话框的顶部，将**Configuration**下拉列表设置为**All Configuration**，以便链接器选项适用于所有流程。
  4. 在“Expert Settings: Command”字段中，将以下文本追加到当前字符串的末尾： 
```
  -L${XILINX_SDX}/lnx64/tools/opencv -lopencv_core -lopencv_highgui -Wl,-rpath,${XILINX_SDX}/lnx64/tools/opencv
```  
  5. 选择 **Apply and Close**按键。  

6. 设置运行时参数： 
从顶部的“Run”菜单中，使用`-x ../binary_container_1.xclbin -i ../../../../img/test.bmp -n 1`设置 **Program arguments**。

### 运行硬件仿真

为**Active build configuration**选择**Emulation-HW**，然后单击**Run**键运行硬件仿真： (![](./images/RunButton.PNG))

### 检查主机代码

在进行仿真运行时，我们将查看主机代码。通过展开**Project Explorer**窗口中的`src`文件夹打开文件，然后双击`host.cpp`文件。

滚动到第266-268行，您可以在其中看到过滤器功能分别为Y，U和V通道调用三次：  
![](./images/host_file1.png)

这个函数在第80行描述。在这里，您可以看到设置了内核参数，并且内核由`clEnqueueTask`命令执行：
![](./images/host_file2.png)

所有三个`clEnqueueTask`命令都使用单个有序命令队列（75行）排队。因此，使用此命令队列的所有命令将按照它们添加到队列的顺序依次执行。
![](./images/Command_queue.JPG)

### 仿真结果

硬件仿真运行完成后，在“Assistant”窗口（左下角）内单击。选择**Emulation-HW** -> **filter2d-Default**。从这里，您可以看到几个重要的报告，例如`Profile Summary`和`Application Timeline`。  
![](./images/assistant_2.JPG)

1. 双击**Reports**窗口中的项目，打开**Profile Summary**报告。 
  * 此报告提供与应用程序运行方式相关的数据。
  * 请注意，在**Top Kernel Execution**下，内核执行三次。

2. 从**Emulation-HW**运行打开应用程序时间线报告。
   * “Application Timeline”报告在公共时间轴上收集和显示主机和设备事件，以帮助您了解和可视化系统的整体运行状况和性能。
   * 在时间轴的底部，您可以看到3个蓝色条：每个内核从主机中排出一个。主机按顺序（按顺序）将内核执行排队，因为它使用单个有序命令队列。
   * 在蓝色条下方，您可以看到3个绿色条：每个内核执行一个。他们依次在FPGA上工作。 
   ![](./images/serial_kernel_enqueue.JPG)


### 改进并发内核入队的主机代码

更改75行中的主机代码，将命令队列声明为_out-of-order_命令队列。

改变之前:
```
mQueue   = clCreateCommandQueue(Context, Device, CL_QUEUE_PROFILING_ENABLE, &mErr);
```

改变之后:
```
mQueue   = clCreateCommandQueue(Context, Device, CL_QUEUE_PROFILING_ENABLE | CL_QUEUE_OUT_OF_ORDER_EXEC_MODE_ENABLE, &mErr);
```
按**CTRL** + **S**保存文件。  
>**可选步骤：**您可以使用更改的主机代码运行硬件仿真。如果选择运行硬件仿真功能，请使用时间轴跟踪来观察使用无序队列使内核几乎在彼此同时执行。

但是，尽管主机同时调度了所有这些执行，但由于FPGA上的内核实例有限（FPGA仍然按顺序执行内核），一些执行请求会延迟。
![](./images/sequential_kernels_2.JPG)

在下一步中，我们将增加FPGA上的内核实例数，以允许同时执行三个主机内核。

我们可以使用多个有序队列来实现从主机代码执行相同的并发命令，而不是使用单个无序队列。有关更多信息，请参阅 [this SDAccel Github Host code example](https://github.com/Xilinx/SDAccel_Examples/blob/master/getting_started/host/concurrent_kernel_execution_ocl/src/host.cpp) ，其中显示了单个输出of-order命令队列和多个有序命令队列方法。

### 增加内核实例数

执行以下步骤将内核实例数增加到三个：
1. 要返回SDx项目设置，请单击**Filter2D**选项卡。
2. 找到窗口下半部分的“Hardware Functions”部分。
3. 将计算机单元的数量从`1`增加到`3`。 
![](./images/SetNumComputeUnits_2.PNG)

### 运行硬件仿真并检查更改

1. 以与先前运行硬件仿真相同的方式运行硬件仿真。
2. 运行完成后，您可以查看“配置文件摘要”报告。
3. 在“Application Timeline”窗口中，您现在可以看到内核执行重叠：  
![](./images/overlapping_kernels_2.JPG)

您已经学习了如何改变内核链接过程以在FPGA上同时执行相同的内核函数。

## 可选步骤

本教程通过硬件仿真演示了该机制。您可以将"Active Build Configuration"更改为**System**，然后单击**Run**按钮以在实际FPGA板上编译并执行此操作。运行完成后，您还可以确认系统运行时的时间线跟踪报告。

<hr/>
<p align="center"><sup>Copyright&copy; 2019 Xilinx</sup></p>
