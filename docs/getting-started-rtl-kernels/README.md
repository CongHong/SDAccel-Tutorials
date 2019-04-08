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

2. To launch Vivado Design Suite to package the RTL IP and create the kernel, click **OK**.

# Vivado Design Suite — RTL Design

When the Vivado Design Suite GUI opens, the Sources window displays the source files that were automatically generated by the RTL Kernel Wizard, as described. Replace these pre-defined files with your own source files in the `reference-files` directory:  
- `IP` directory: Contains 10 source files for your RTL IP. The design hierarchy is shown below:  
  - `Vadd_A_B.v`: Kernel top level module.  
      - `Vadd_A_B_control_s_axi.v`: RTL control register module.  
      - `Vadd_A_B_example.sv`: Wrapper for AXI4 memory mapped and adder module.  
          - `Vadd_A.sv`: AXI4 read module for Vector A.  
          - `Vadd_B.sv`: AXI4 read/write module for Vector B and an adder.
- `testbench` directory: Contains a testbench for the sample VADD kernel.
  - `Vadd_A_B_tb.sv`
- `host` directory: Contains sample host code for the sample VADD kernel.
  - `vadd.cpp`: Creates read/write buffers to transfer the data between the host and the FPGA.

## Remove Existing `A+1` Source Files

Remove the sample VADD kernel (`A+1` source files and test bench) from the project, and then replace them with your own RTL IP files for Vector-Accumulate (`B[i] = A[i]+B[i]`).

1. In the Vivado Design Suite GUI, in the Sources window, select **Compile Order** > **Synthesis**, and then expand the **Design Sources** tree.

2. Select all eight source files, right-click, and then select `Remove File from Project...`.

3. In the Remove Sources dialog box that is displayed, to remove the files from the project, click **OK**.

The Invalid Top Module dialog box opens.

4. Click **OK** to continue.

   >**NOTE**: You have now removed the automatically-generated RTL files from the RTL kernel project, which means you are ready to add your own RTL IP files back into the project. The RTL IP files have been provided for you in this tutorial, but this is the point at which you would insert your own RTL IP and the supporting hierarchy of files.

5. In the same window, change the sources to simulation and delete _only_ the `Vadd_A_B_wizard_0_tb.sv `file.

6. Repeat steps 3 and 4.

## Add Sources

1. Right-click **`Design Sources`**, and then click **Add Sources**.  
The Add Sources window is displayed.

2. Click **Add or create design sources**, and then click **Next**.

3. Click **Add Directories**, browse to `reference-files`, and then select the `IP` directory (which contains the RTL sources).
    >**NOTE**: To add your own RTL IP, specify the required folder or files.

4. Click **Add Files**, browse to `testbench`, and then select `Vadd_A_B_tb.sv`:  
![Missing Image: add_design_sources.PNG](./images/add_design_sources.PNG)  

5. To add the files to the current project, click **Finish**.

6. To view the hierarchy of the project, in the Sources window, select the **Hierarchy** tab.

   > **IMPORTANT**: The test bench has been selected as the top-level design file. While this is technically correct (the test bench includes the IP), for your purposes, the test bench should not be the top level of the RTL kernel.

7. Right-click `Vadd_A_B_tb.sv`, and then select **Move to Simulation Sources**.  

This defines the test bench for use in simulation and enables the Vivado Design Suite to identify the `Vadd_A_B.v` file as the new top level of the design. This RTL IP has an interface which is compatible with the requirements for RTL kernels in the SDAccel development environment. This can be seen in the module `Vadd_A_B` definition, as shown in the following figure:  
![Missing Image: Top_level_interface.PNG](./images/Top_level_interface.PNG)  

The XML file and the RTL sources can now be used to package the kernel into an `.xo` file.

## RTL Verification

Before packaging an RTL design as a kernel `.xo` file, you should ensure that it is fully verified using standard RTL verification techniques such as verification IP, randomization and/or protocol checkers.

In this tutorial, the RTL code for the Vector-Accumulate kernel has already been independently verified.  

If you want to skip this step and begin packaging the RTL kernel IP, go to the next section.

> **NOTE**: The AXI Verification IP (AXI VIP) is available in the Vivado IP catalog to help with verification of AXI interfaces. The test bench included with this tutorial incorporates this AXI VIP.

To use this test bench for verifying the Vector addition kernel:

1. In the Flow Navigator, right-click **Simulation**, and then select **Simulation Settings**.

2. In the Settings dialog box, select **Simulation**.

3. Change the **xsim.simulate.runtime** value to `all`, as shown in the following figure:

   ![Missing Image: simulation_settings.PNG](./images/simulation_settings.PNG)

4. Click **OK**.

5. In the Flow Navigator, click **Run Simulation**, and then select **Run Behavioral Simulation**.

   The Vivado simulator runs the test bench to completion, and then displays messages in the Tcl Console window. Ignore any error messages.

   `Test Completed Successfully` confirms that the verification of the kernel is successful.

   > **NOTE**: You might need to scroll up in the Tcl Console to see the confirmation message.

## Package the RTL Kernel IP

The Vivado project is now ready to be packaged as a kernel `.xo` file for use with the SDx environment.

1. From the Flow Navigator, click **Generate RTL Kernel**. The Generate RTL Kernel dialog box is opened, as shown in the following figure.  
![Missing Image: generate_rtl_kernel.png](./images/generate_rtl_kernel.PNG)
    >**Packaging Option Details**
    >- **Sources-only kernel**: Packages the kernel using the RTL design sources directly.
    >- **Pre-synthesized kernel**: Packages the kernel with the RTL design sources, and a synthesized cached output that can be used later in the SDx linking flow to avoid re-synthesizing the kernel and improve runtime.
    >- **Netlist (DCP) based kernel**: Packages the kernel as a block box, using the netlist generated by the synthesized output of the kernel. This output can be optionally encrypted to provide to third-parties while protecting your IP.
    >- **Software Emulation Sources** (optional): Packages the RTL kernel with a software model that can be used in software emulation. In this  tutorial, a software model is not used; leave this option empty.

2. For this tutorial, select **Sources-only**.

3. Click **OK**.  
The Vivado tool uses the `package_xo` command to generate an `.xo` file, as shown below.

   ```
   package_xo -xo_path Vadd_A_B.xo -kernel_name Vadd_A_B -kernel_xml kernel.xml -ip_directory ./ip/
   ```

The file is generated in the `./sdx_imports` directory of the Vivado kernel project. You can examine the Tcl Console for a transcript of the process.

4. When prompted to, exit Vivado Design Suite.

   The Vivado tool closes and returns control to the SDx integrated development environment (IDE). The RTL kernel is imported into the open SDAccel environment project, which opens the Kernel Wizard dialog box.

5. Click **OK**.

# Using the RTL Kernel in an SDAccel project

## Delete and Import Host Code

After exiting the Vivado tool, the following files are added to Project Explorer in the SDAccel environment:
- `Vadd_A_B.xo`: The compiled kernel object file.
- `host_example.cpp`: An example host application file.  

1. In the Project Explorer view, expand `src`, as shown below:  
![Missing Image: project_explorer.png](./images/project_explorer.PNG)
   > **NOTE**: `Vadd_A_B.xo` is displayed with a lightning bolt icon. In the SDx GUI, this indicates a hardware function. The tool recognizes the RTL kernel and marks it as an accelerated function.

2. Select and delete `host_example.cpp`.  
At this point, import the host application that has been prepared for this tutorial.

3. To import the host application project prepared for this example, in the Project Explorer view, right-click the tutorial project, and then click **Import Sources**.

4. Click **Browse...**, navigate to `reference-files/host`, and then click **OK**.

5. To add the host application code to your project, select `vadd.cpp`, and then click **Finish**.  
`vadd.cpp` is added under `src`.

6. Double-click `vadd.cpp`, which opens it in the Code Editor window.

## Host Code Structure

The structure of the host code is divided into three sections:

1. Setting up the OpenCL Runtime environment
2. Execution of kernels
3. Post-processing and release of FPGA device

Here are some of the important OpenCL API calls allowing the host application to interact with the FPGA:

- A handle to the kernel is created (line 239):
```C
clCreateKernel(program, "Vadd_A_B", &err);
```

- Buffers are created to transfer data back and forth between the host and the FPGA (line 256):
```C
clCreateBuffer(context, CL_MEM_READ_WRITE,sizeof(int) * number_of_words, NULL, NULL);
```

- Values (A and B) are written into the buffers and the buffers transferred to the FPGA (lines 266,274)
```C
clEnqueueWriteBuffer(commands, dev_mem_ptr, CL_TRUE, 0,sizeof(int) * number_of_words, host_mem_ptr, 0, NULL, NULL);
```

- Once A and B have been transferred to the device, the kernel can be executed (line 299):
```C
clEnqueueTask(command_queue, kernel, 0, NULL, NULL);
```

- After the kernel completes, the host application reads back the buffer with the new value of B (line 312):
```C
clEnqueueReadBuffer(command_queue, dev_mem_ptr, CL_TRUE, 0, sizeof(int)*number_of_words,host_mem_output_ptr, 0, NULL, &readevent );
```

The structure and requirements of the host application are discussed in greater detail in the _SDAccel Environment User Guide_ ([UG1023](https://www.xilinx.com/html_docs/xilinx2018_3/sdaccel_doc/pjq1528392379194.html#awb1528664475358)) and in the _SDAccel Environment Programmers Guide_ ([UG1277](https://www.xilinx.com/html_docs/xilinx2018_3/sdaccel_doc/vpy1519742402284.html#vpy1519742402284)).

## Build the Project

For detailed information on building the project, see [Getting Started with C/C++ Kernels](../getting-started-c-kernels/README.md).

With the host application code (`vadd.cpp`) and the RTL kernel code (`Vadd_A_B.xo`) added to the project, you are now ready to build and run the project.

1. To create a binary container, select the RTL kernel as the hardware function.  
    >**NOTE**: For Software Emulation, the RTL kernel flow requires a C/C++ software model of the kernel. In this tutorial, you have not been provided such a model, so you will not be able to run a Software Emulation build.

2. In SDx Application Project Settings, change **Active build configuration** to **Emulation-HW**.  
The Hardware Emulation target is useful for:
   - Verifying the functionality of the logic that will go into the FPGA.
   - Retrieving the initial performance estimates of the accelerator and host application.

3. Build and run the Hardware Emulation configuration, and then verify the results.

### Optional: Build and Run the System on a Hardware Platform

1. In the SDx Application Project Settings, change **Active build configuration** to **System**.  
In the system configuration, the kernel code is implemented onto the FPGA device, resulting in a binary that will run on the selected platform card.  

2. If you have an available hardware platform, build and run the system, and then verify the results.

# Summary

1. You used the RTL Kernel Wizard from the SDx GUI to specify the name and interfaces of a new RTL kernel (based on an existing RTL IP).
   - The RTL Kernel Wizard created an XML template according to your specifications, automatically generated the RTL files for a template IP (`Vadd A+1`), and then launched the Vivado Design Suite.

2. In the Vivado Design Suite, you removed the template RTL files and added in your own RTL IP files.

3. You simulated the IP using a test bench to incorporate the AXI Verification IP (AXI VIP).  
    >**NOTE**: You must create this test bench when using your own RTL IP.

4. You packaged the RTL IP project into the compiled `.xo` file needed by the SDAccel development environment.

5. You added the RTL kernel to a host application and built the Hardware Emulation configuration.  
   - In the SDAccel development environment, a binary container was created using the `.xo` file, and an `xclbin` file was compiled.

<hr/>
<p align="center"><sup>Copyright&copy; 2019 Xilinx</sup></p>
