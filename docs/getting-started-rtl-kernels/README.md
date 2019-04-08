<table>
 <tr>
   <td align="center"><img src="https://www.xilinx.com/content/dam/xilinx/imgs/press/media-kits/corporate/xilinx-logo.png" width="30%"/><h1>2018.3 SDAccel™开发环境教程</h1>
   <a href="https://github.com/Xilinx/SDAccel-Tutorials/branches/all">查看其他版本</a>
   </td>
 </tr>
 <tr>
 <td align="center"><h1>Getting Started with RTL Kernels</h1>
 </td>
 </tr>
</table>

# 介绍

本教程演示了如何使用SDAccel环境将RTL内核编程到FPGA中并使用通用开发流程构建硬件仿真:

  1.将RTL块打包为Vivado™Design Suite IP。
  2.创建内核描述XML文件。
  3.将RTL内核打包到Xilinx对象（`.xo`）文件中。
  4.使用SDAccel将RTL内核编程到FPGA上，并在硬件或硬件仿真中运行。

# 教程概述

符合某些软件和硬件接口要求的RTL设计可以打包到Xilinx对象（`.xo`）文件中。该文件可以链接到二进制容器中，以创建一个`xclbin`文件，主机代码应用程序使用该文件将内核编程到FPGA中。

本教程提供以下参考文件：
 - 一个简单的向量累加示例，执行`B [i] = A [i] + B [i]`运算。
 - 主机应用程序，在双倍数据速率（DDR）内存上创建读/写缓冲区，以在主机和FPGA之间传输数据。
 - 主机将入队RTL内核（在FPGA上执行），读取DDR的缓冲区，执行`B [i] = A [i] + B [i]`，然后将结果写回DDR。
 - 主机将回读数据以比较结果。

使用这些参考文件，本教程将指导您从创建SDx™项目的第一步到构建和运行项目的最后一步。

# 将RTL设计用作RTL内核的要求

要在SDAccel环境框架内使用RTL内核，它必须满足SDAccel内核执行模型和硬件接口要求。

## 内核执行模型

RTL内核使用与C / C ++内核相同的软件接口和执行模型。它们被宿主应用程序视为具有void返回值，标量参数和指针参数的函数。例如：

```C
void vadd_A_B(int *a, int *b, int scalar)
```

这意味着RTL内核具有类似软件功能的执行模型：
 - 它必须在被叫时开始。
 - 它负责处理必要的结果。
 - 处理完成后必须发送通知。

更具体地说，SDAccel执行模型依赖于以下机制和假设：
 - 标量参数通过AXI4-Lite从接口传递给内核。
 - 指针参数通过全局内存（DDR或PLRAM）传输。
 - 指针参数的基址通过其AXI4-Lite从接口传递给内核。
 - 内核通过一个或多个AXI4内存映射接口访问全局内存中的指针参数。
 - 内核由主机应用程序通过其AXI4-Lite接口启动
 - 内核必须在通过AXI4-Lite接口或特殊中断信号完成操作时通知主机应用程序

## 硬件接口要求

为了符合此执行模型，SDAccel要求内核满足特定的硬件接口要求：

- 用于访问可编程寄存器的唯一一个AXI4-Lite从接口（控制寄存器，标量参数和指针基址）.
  - 偏移量 `0x00` - 控制寄存器 - 控制并提供内核状态
    - 位 `0`: **开始信号** — 当内核可以开始处理数据时，主机应用程序有效。当**完成信号**被置位时必须清除。
    - 位 `1`: **完成信号** — 内核在完成操作时断言。阅读时清除。
    - 位 `2`: **空闲信号** — 内核在不处理任何数据时有效此信号。从低到高的转换应与**完成信号**的断言同步发生    
  - 偏移量 `0x04`- 全局中断使能寄存器 - 用于对主机的中断   
  - 偏移量 `0x08`- IP中断使能寄存器 - 用于控制使用哪个IP生成的信号生成中断
  - 偏移量 `0x0C`- IP中断状态寄存器 - 提供中断状态
  - 偏移量 `0x10` 以上 - 内核参数寄存器 - 寄存器指针的标量参数和基址

- 一个或多个以下接口:
  - AXI4主存储器映射接口与全局存储器通信。
    - 所有AXI4主接口必须具有64位地址。
    - 内核开发人员负责分区全局内存空间。全局内存中的每个分区都成为内核参数。每个分区的基址（存储器偏移量）必须由可通过AXI4-Lite从接口编程的控制寄存器设置。
    - AXI4主机不能使用Wrap或Fixed突发类型，并且它们不能使用窄（子大小）突发。这意味着AxSIZE应匹配AXI数据总线的宽度。
    - 必须包装或桥接任何不符合上述要求的用户逻辑或RTL代码。
  - AXI4-Stream接口与其他内核通信。
如果原始RTL设计具有不同的执行模型或硬件接口，则必须添加逻辑以确保设计以预期方式运行并符合接口要求。

### 附加信息

有关SDAccel环境内核的接口要求的更多信息，请参阅 [Requirements for Using an RTL Design as an RTL Kernel](https://www.xilinx.com/html_docs/xilinx2018_3/sdaccel_doc/creating-rtl-kernels-qnk1504034323350.html#qbh1504034323531) section of the _SDAccel Environment User Guide_ ([UG1023](https://www.xilinx.com/cgi-bin/docs/rdoc?v=2018.3;d=ug1023-sdaccel-user-guide.pdf)).

## Vector-Accumulate RTL IP

对于本教程，执行 `B[i]=A[i]+B[i]` 的Vector-Accumulate RTL IP满足上述所有要求，并具有以下特征：


- 两个AXI4内存映射接口：
   - 一个接口用于读取A.
   - 一个接口用于读写B
   - 本设计中使用的AXI4主机不使用环绕，固定或窄脉冲类型。
-  AXI4-Lite从控制接口：
   - 偏移量为0x00的控制寄存器
   - 内核参数寄存器偏移量为0x10，允许主机将标量值传递给内核。
   - 内核参数寄存器偏移量为0x18，允许主机将全局内存中A的基址传递给内核
   - 内核参数寄存器偏移量为0x1C，允许主机将全局内存中B的基址传递给内核


这些规范用作RTL内核向导的输入。有了这些信息，RTL内核向导将生成以下内容：
 - 将RTL设计打包为SDAccel内核 `.xo` 文件所需的XML文件
 - 执行A [i] = A [i] + 1的示例内核（RTL代码，测试平台和主机代码）
 - 内核的Vivado项目


在继续学习本教程时，您将使用预先存在的Vector-Accumulate设计替换示例内核，并将其打包为 `.xo` 文件。

# 创建一个SDx项目

1. 使用 `sdx` 命令在Linux的终端窗口中启动SDx开发环境。
将显示“工作区启动器”对话框。 
![error: missing image](images/183_launcher.png)  

2. 选择工作区的位置，这是项目所在的位置。
3. 单击 **Launch**.  
欢迎窗口打开。  
![error: missing image](images/welcome.png)  
>**注意**: 首次使用该工具时，或者选择 **Help > Welcome**,欢迎窗口打开。

4. 在“欢迎”窗口中，单击 **Create Application Project**.  
将打开“New SDx Application Project”窗口。  
![error: missing image](images/newapplicationproject.png)

5. 创建一个新的SDx项目:  
   1. 输入项目名称。  
   2. 选择 **Use default location**.  
   3. 单击 **Next**.  
  将显示“平台”页面。  
![error: missing image](./images/183_hardware_platform_dialog.png)

6. 选择 `xilinx_u200_xdma_201830_1`, 然后单击 **Next**.  
将打开“模板”窗口，其中显示可用于开始构建SDAccel环境项目的模板。除非您已下载其他SDx示例，否则您应该只将空应用程序和向量添加视为可用模板。  
![error: missing image](./images/183_example_empty.png)

   硬件平台的选择将项目定义为SDAccel或SDSoC™环境项目。在这种情况下，您选择了一个SDAccel环境加速平台，因此该项目将是一个SDAccel项目。

7. 选择 `Empty Application`, 然后单击 **Finish**. 这将关闭新项目向导并返回主SDx GUI窗口。

8.  从SDx GUI的顶部菜单栏, 单击 **Xilinx**->**RTL Kernel Wizard**, 如下图所示。  
![Missing Image: RTL Wizard Menu](images/RTLKernelWizard.png)  

一个 *"Welcome to SDx RTL Kernel Wizard"* 页面，其中包含内核向导功能的摘要。查看使用信息。

# 使用RTL内核向导进行配置

RTL内核向导将指导您完成指定RTL内核的接口特性的过程。使用RTL内核向导可确保将RTL IP打包到可由SDAccel集成到系统中的有效内核中。使用该向导还有一个额外的好处，即可以自动执行将RTL IP打包到内核中的一些必要任务。

## General Settings

- **Kernel Identification**: 指定供应商，内核名称和库，称为IP的“供应商：库：名称：版本”（VLNV）。内核名称应与您用于RTL内核的IP的顶层模块名称匹配。
- **Kernel Options**: 指定设计类型。
  - **RTL** (default): 以Verilog格式生成关联文件。
  - **Block design**: 为Vivado™工具IP Integrator生成块设计。块设计包括MicroBlaze™子系统，该子系统使用Block RAM交换存储器来模拟控制寄存器。
- **Clock and Reset Options**: 指定内核使用的时钟数以及内核是否需要顶级重置端口。

1. 对于内核标识，请将内核名称指定为 `Vadd_A_B`。

2. 对于其余选项，请保留默认值，然后单击 **Next**.  
![Missing Image: P1.PNG](./images/P1.PNG)

## Scalars

标量参数用于将输入参数从主机应用程序传递到内核。对于指定的输入参数的数量，创建相应的寄存器以便于将参数从软件传递到硬件。为每个参数分配一个ID值，用于从主机应用程序访问该参数。可以在向导的摘要页面上找到此ID值。
- **Argument name**: 参数的名称。  
- **Argument type**: 标量参数的类型，表示为本机C / C ++数据类型。例如：（u）char，（u）short或（u）int。

1. 保留默认值，然后单击 **Next**.  
![scalar.PNG](./images/scalar.PNG)

## Global Memory

全局内存用于在主机和内核之间以及内核与其他内核之间传递大型数据集。内核可以通过AXI4内存映射主接口访问该内存。对于每个AXI4主接口，您可以自定义接口名称，数据宽度和相关参数的数量。
- **Number of AXI master interfaces**: 指定内核中AXI接口的数量
- **AXI master definition**: 指定接口名称，数据宽度（以字节为单位）以及与每个AXI4接口关联的参数数量。
- **Argument definition**: 指定分配给每个AXI4接口的指针参数。为每个参数分配一个ID值，该值可用于从主机应用程序访问参数。可以在向导的摘要页面上找到此ID值分配。  

![Missing Image: maxi.PNG](./images/maxi.PNG)  

1. 在“Number of AXI master interfaces”行, 选择 **2** ，是因为Vector-Accumulate内核有两个AXI4接口。

2. 在“AXI master definition”区域:
   1. 不用修改“interface names”。
   2. 不用修改“width”。
   3. 在“Number of arguments”列, 选择 **1** ，是因为每个AXI4接口都专用于单个指针参数。

3. 在“Argument definition section”区域, 在“Argument name”列:
   1. 在“m00_axi”行, 写 `A`. 通过此AXI4接口访问数据集A.
   2. 在“m01_axi”行, 写 `B`. 通过此AXI4接口访问数据集B.
    
设置应与上面的屏幕截图相同。  

4. 单击 **Next**.  
将显示摘要页面。  

## 示例摘要页面

![Missing Image: summary.PNG](./images/summary.PNG)

- **Target platform**: 指定将为其编译RTL内核的平台。必须重新编译RTL内核以支持不同的平台。
- **Function prototype**: 如果它是C函数，则传达内核调用的内容。
- **Register map**: 显示主机软件ID，参数名称，硬件寄存器偏移量，数据类型和关联的AXI接口之间的关系。

1. 在生成内核之前，请查看摘要页面以获取正确性。

RTL内核向导使用通过各个步骤获取的数据，并在摘要页面中进行汇总以生成：
 - 内核描述XML文件。
   - `kernel.xml` 指定您在向导中定义的运行时和SDAccel环境流所需的属性，例如寄存器映射。
 - 一个名为VADD的示例内核实现A [i] = A [i] +1，包括：
   -  RTL代码
   - 验证测试台
   - 主机代码
 - 一个VADD示例内核的Vivado项目。

2. 要启动Vivado Design Suite以打包RTL IP并创建内核，请单击**OK**.

# Vivado设计套件 -  RTL设计

当Vivado Design Suite GUI打开时，Sources窗口将显示由RTL内核向导自动生成的源文件，如上所述。将这些预定义文件替换为 `reference-files` 目录中您自己的源文件：  
- `IP` 目录：包含10个RTL IP源文件。设计层次结构如下所示：  
  - `Vadd_A_B.v`: 内核顶级模块。  
      - `Vadd_A_B_control_s_axi.v`: RTL控制寄存器模块。  
      - `Vadd_A_B_example.sv`: 用于AXI4存储器映射和加法器模块的包装器。 
          - `Vadd_A.sv`: 用于Vector A的AXI4读取模块。 
          - `Vadd_B.sv`: 用于向量B和加法器的AXI4读/写模块。
- `testbench` 目录：包含示例VADD内核的测试平台。
  - `Vadd_A_B_tb.sv`
- `host` 目录：包含示例VADD内核的示例主机代码。
  - `vadd.cpp`: 创建读/写缓冲区以在主机和FPGA之间传输数据。

## 删除现有的 `A+1` 源文件

从项目中删除示例VADD内核 (`A+1` 源文件和测试平台)，然后将它们替换为您自己的RTL IP文件，用于Vector-Accumulate (`B[i] = A[i]+B[i]`).

1. 在Vivado Design Suite GUI的Sources窗口中，选择 **Compile Order** > **Synthesis**, 然后展开 **Design Sources** 树。

2. 选择所有八个源文件，右键单击，然后选择 `Remove File from Project...`.

3. 在显示的“Remove Sources”对话框中，要从项目中删除文件，请单击 **OK**.

将打开“Invalid Top Module ”对话框。

4. 单击 **OK** 继续.

   >**注意**: 您现在已从RTL内核项目中删除了自动生成的RTL文件，这意味着您已准备好将自己的RTL IP文件添加回项目中。在本教程中已经为您提供了RTL IP文件，但是您可以在此处插入自己的RTL IP和支持的文件层次结构。
5. 在同一窗口中，将源更改为模拟并仅删除 `Vadd_A_B_wizard_0_tb.sv `文件。

6. 重复步骤3和4.

## 添加资源

1. 右击 **`Design Sources`**, 然后单击 **Add Sources**.  
将显示“Add Sources”窗口。

2. 单击 **Add or create design sources**, 然后单击 **Next**.

3. 单击 **Add Directories**, 浏览到 `reference-files`, 然后选择 `IP` 目录 (包含RTL源)。
    >**注意**: 要添加自己的RTL IP，请指定所需的文件夹或文件。

4. 单击 **Add Files**, 浏览到 `testbench`, 然后选择 `Vadd_A_B_tb.sv`:  
![Missing Image: add_design_sources.PNG](./images/add_design_sources.PNG)  

5. 将文件添加到当前项目, 单击 **Finish**。

6. 要查看项目的层次结构，请在“Sources”窗口中选择 **Hierarchy** 选项。

   > **重要**: 测试平台已被选为顶级设计文件。虽然这在技术上是正确的（测试平台包括IP），但就您的目的而言，测试平台不应该是RTL内核的顶级。

7. 右击 `Vadd_A_B_tb.sv`, 然后选择 **Move to Simulation Sources**.  

这定义了用于仿真的测试平台，并使Vivado Design Suite能够将 `Vadd_A_B.v` 文件识别为设计的新顶级。此RTL IP具有与SDAccel开发环境中的RTL内核要求兼容的接口。这可以在模块 `Vadd_A_B` 定义中看到，如下图所示：  
![Missing Image: Top_level_interface.PNG](./images/Top_level_interface.PNG)  

现在可以使用XML文件和RTL源将内核打包成 `.xo` 文件。

## RTL验证

在将RTL设计打包为内核 `.xo` 文件之前，应确保使用标准RTL验证技术（例如验证IP，随机化和/或协议检查程序）对其进行完全验证。

在本教程中，Vector-Accumulate内核的RTL代码已经过独立验证。  

如果要跳过此步骤并开始打包RTL内核IP，请转到下一部分。

> **注意**: Vivado IP目录中提供AXI验证IP（AXI VIP），以帮助验证AXI接口。本教程中包含的测试平台包含此AXI VIP。

要使用此测试平台来验证Vector添加内核：

1. 在“Flow Navigator”中，右击 **Simulation**, 然后选择 **Simulation Settings**。

2. 在“Settings”对话框中，选择 **Simulation**。

3. 改变 **xsim.simulate.runtime** 的值为 `all`, 如下图所示:

   ![Missing Image: simulation_settings.PNG](./images/simulation_settings.PNG)

4. 单击 **OK**.

5. 在“Flow Navigator”中，单击 **Run Simulation**, 然后选择 **Run Behavioral Simulation**。

   Vivado模拟器运行测试平台完成，然后在Tcl控制台窗口中显示消息。忽略任何错误消息。

   `Test Completed Successfully` 确认内核验证成功。

   > **注意**: 您可能需要在Tcl控制台中向上滚动才能看到确认消息。

## 打包RTL内核IP

现在可以将Vivado项目打包为内核 `.xo` 文件，以便与SDx环境一起使用。

1. 从“Flow Navigator”中，单击 **Generate RTL Kernel**。将打开“生成RTL内核”对话框，如下图所示。 
![Missing Image: generate_rtl_kernel.png](./images/generate_rtl_kernel.PNG)
    >**Packaging Option Details**
    >- **Sources-only kernel**: 直接使用RTL设计源打包内核。
    >- **Pre-synthesized kernel**: 使用RTL设计源打包内核，以及稍后可在SDx链接流中使用的综合高速缓存输出，以避免重新合成内核并改善运行时。
    >- **Netlist (DCP) based kernel**: 使用由内核的合成输出生成的网表将内核打包为块框。此输出可以选择加密，以便在保护您的IP的同时提供给第三方。
    >- **Software Emulation Sources** (optional): 使用可用于软件仿真的软件模型打包RTL内核。在本教程中，不使用软件模型;将此选项留空。

2. 在本教程中，选择 **Sources-only**.

3. 单击 **OK**.  
Vivado工具使用`package_xo` 命令生成 `.xo` 文件，如下所示。

   ```
   package_xo -xo_path Vadd_A_B.xo -kernel_name Vadd_A_B -kernel_xml kernel.xml -ip_directory ./ip/
   ```

该文件在Vivado内核项目的 `./sdx_imports` 目录中生成。您可以检查Tcl控制台以获取该过程的记录。

4. 出现提示时，退出Vivado Design Suite。

   Vivado工具关闭并将控制权返回给SDx集成开发环境（IDE）。将RTL内核导入到打开的SDAccel环境项目中，该项目将打开“内核向导”对话框。

5. 单击 **OK**。

# 在SDAccel项目中使用RTL内核

## 删除并导入主机代码

退出Vivado工具后，以下文件将添加到SDAccel环境中的Project Explorer：
- `Vadd_A_B.xo`: 已编译的内核对象文件。
- `host_example.cpp`: 示例主机应用程序文件。  

1. 在“Project Explorer”视图中，展开 `src`, 如下所示：  
![Missing Image: project_explorer.png](./images/project_explorer.PNG)
   > **注意**: `Vadd_A_B.xo` 显示闪电图标。在SDx GUI中，这表示硬件功能。该工具识别RTL内核并将其标记为加速功能。

2. 选择并删除 `host_example.cpp`。  
此时，导入为本教程准备的宿主应用程序。

3. 要导入为此示例准备的宿主应用程序项目，请在“Project Explorer”视图中，右击教程项目，然后单击 **Import Sources**。

4. 单击 **Browse...**, 找到 `reference-files/host`, 然后单击 **OK**。

5. 要将主应用程序代码添加到项目中，请选择 `vadd.cpp`, 然后单击 **Finish**。 
`vadd.cpp` 被添加到 `src`下面.

6. 双击 `vadd.cpp`, 在代码编辑器窗口中打开。

## 主代码结构

主代码的结构分为三个部分：

1.设置OpenCL运行时环境
2.执行内核
3. FPGA器件的后处理和发布

以下是一些重要的OpenCL API调用，允许主机应用程序与FPGA交互：

- 创建内核（第239行）:
```C
clCreateKernel(program, "Vadd_A_B", &err);
```

- 创建缓冲区是为了在主机和FPGA之间来回传输数据（第256行）：
```C
clCreateBuffer(context, CL_MEM_READ_WRITE,sizeof(int) * number_of_words, NULL, NULL);
```

- 值（A和B）写入缓冲区，缓冲区传输到FPGA（第266,274行）：
```C
clEnqueueWriteBuffer(commands, dev_mem_ptr, CL_TRUE, 0,sizeof(int) * number_of_words, host_mem_ptr, 0, NULL, NULL);
```

- 只要A和B转移到设备，就可以执行内核（第299行）：
```C
clEnqueueTask(command_queue, kernel, 0, NULL, NULL);
```

- 内核完成后，主机应用程序使用新值B读回缓冲区（第312行）：
```C
clEnqueueReadBuffer(command_queue, dev_mem_ptr, CL_TRUE, 0, sizeof(int)*number_of_words,host_mem_output_ptr, 0, NULL, &readevent );
```

主机应用程序的结构和要求将在 _SDAccel Environment User Guide_ ([UG1023](https://www.xilinx.com/html_docs/xilinx2018_3/sdaccel_doc/pjq1528392379194.html#awb1528664475358)) 和  _SDAccel Environment Programmers Guide_ ([UG1277](https://www.xilinx.com/html_docs/xilinx2018_3/sdaccel_doc/vpy1519742402284.html#vpy1519742402284))中详细讨论。

## 建立项目

有关构建项目的详细信息，请参阅 [Getting Started with C/C++ Kernels](../getting-started-c-kernels/README.md)。

通过将主机应用程序代码 (`vadd.cpp`) 和RTL内核代码 (`Vadd_A_B.xo`) 添加到项目中，您现在可以构建并运行该项目了。

1. 要创建二进制容器，请选择RTL内核作为硬件功能。
    >**注意**: 对于软件仿真，RTL内核流程需要内核的C / C ++软件模型。在本教程中，您尚未提供此类模型，因此您将无法运行Software Emulation构建。

2. 在SDx应用程序项目设置中，将 **Active build configuration** 更改为 **Emulation-HW**。  
硬件仿真目标对以下内容非常有用：
    - 验证将进入FPGA的逻辑功能。
    - 检索加速器和主机应用程序的初始性能估计。

3. 构建并运行硬件仿真配置，然后验证结果。

### 可选：在硬件平台上构建和运行系统

1. 在SDx应用程序项目设置中，将 **Active build configuration** 更改为 **System**。  
在系统配置中，内核代码在FPGA器件上实现，从而产生将在所选平台卡上运行的二进制代码。 

2. 如果您有可用的硬件平台，请构建并运行系统，然后验证结果。

# 总结

1.您使用SDx GUI中的RTL内核向导来指定新RTL内核的名称和接口（基于现有的RTL IP）。
    -  RTL内核向导根据您的规范创建了一个XML模板，自动生成模板IP (`Vadd A+1`)的RTL文件，然后启动Vivado Design Suite。

2. 在Vivado Design Suite中，您删除了模板RTL文件并添加到您自己的RTL IP文件中。

3. 您使用测试平台模拟IP以合并AXI验证IP（AXI VIP）。  
    >**注意**: 使用自己的RTL IP时，必须创建此测试平台。

4. 您将RTL IP项目打包到SDAccel开发环境所需的已编译的 `.xo` 文件中。

5. 您将RTL内核添加到主机应用程序并构建了硬件仿真配置。  
   - 在SDAccel开发环境中，使用 `.xo` 文件创建了一个二进制容器，并编译了一个 `xclbin` 文件。

<hr/>
<p align="center"><sup>Copyright&copy; 2019 Xilinx</sup></p>
