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
    - 位 `0`: **开始信号** — 当内核可以开始处理数据时，主机应用程序断言。当**完成信号**被置位时必须清除。
    - Bit `1`: **done signal** — Asserted by the kernel when it has completed operation. Cleared on read.
    - Bit `2`: **idle signal** — The kernel asserts this signal when it is not processing any data. The transition from Low to High should occur synchronously with the assertion of the **done** signal    
  - Offset `0x04`- Global Interrupt Enable Register - Used to enable interrupt to the host   
  - Offset `0x08`- IP Interrupt Enable Register - Used to control which IP generated signal are used to generate an interrupt
  - Offset `0x0C`- IP Interrupt Status Register - Provides interrupt status
  - Offset `0x10` and above - Kernel Argument Register(s) - Register for scalar parameters and base addresses for pointers

- One or more of the following interfaces:
  - AXI4 master memory mapped interface to communicate with global memory.
    - All AXI4 master interfaces must have 64-bit addresses.
    - The kernel developer is responsible for partitioning global memory spaces. Each partition in the global memory becomes a kernel argument. The base address (memory offset) for each partition must be set by a control register programmable via the AXI4-Lite slave interface.
    - AXI4 masters must not use Wrap or Fixed burst types, and they must not use narrow (sub-size) bursts. This means that AxSIZE should match the width of the AXI data bus.
    - Any user logic or RTL code that does not conform to the requirements above must be wrapped or bridged.
  - AXI4-Stream interface to communicate with other kernels.

If the original RTL design has a different execution model or hardware interface, you must add logic to ensure that the design behaves in the expected manner and complies with interface requirements.

### Additional Information

For more information on the interface requirements for an SDAccel environment kernel, see the [Requirements for Using an RTL Design as an RTL Kernel](https://www.xilinx.com/html_docs/xilinx2018_3/sdaccel_doc/creating-rtl-kernels-qnk1504034323350.html#qbh1504034323531) section of the _SDAccel Environment User Guide_ ([UG1023](https://www.xilinx.com/cgi-bin/docs/rdoc?v=2018.3;d=ug1023-sdaccel-user-guide.pdf)).

## Vector-Accumulate RTL IP

For this tutorial, the Vector-Accumulate RTL IP performing `B[i]=A[i]+B[i]` meets all the requirements described above and has the following characteristics:

- Two AXI4 memory mapped interfaces:
  - One interface is used to read A
  - One interface is used to read and write B
  - The AXI4 masters used in this design do not use wrap, fixed, or narrow burst types.
- An AXI4-Lite slave control interface:
  - Control register at offset 0x00
  - Kernel argument register at offset 0x10 allowing the host to pass a scalar value to the kernel.
  - Kernel argument register at offset 0x18 allowing the host to pass the base address of A in global memory to the kernel
  - Kernel argument register at offset 0x1C allowing the host to pass the base address of B in global memory to the kernel

These specifications serve as the inputs to the RTL Kernel Wizard. With this information, the RTL Kernel Wizard generates the following:
- An XML file necessary to package the RTL design as an SDAccel kernel `.xo` file
- A sample kernel (RTL code, testbench, and host code) performing A[i] = A[i] + 1
- A Vivado project for the kernel

As you continue through the tutorial, you will replace the sample kernel with the pre-existing Vector-Accumulate design, and package it as an `.xo` file.

# Create an SDx Project

1. Use the `sdx` command to launch the SDx development environment in a terminal window in Linux.  
The Workspace Launcher dialog box is displayed.  
![error: missing image](images/183_launcher.png)  

2. Select a location for your workspace, which is where the project will be located.  

3. Click **Launch**.  
The Welcome window opens.  
![error: missing image](images/welcome.png)  
>**NOTE**: The Welcome window opens when you use the tool for the first time, or by selecting **Help > Welcome**.

4. In the Welcome window, click **Create Application Project**.  
The New SDx Application Project window opens.  
![error: missing image](images/newapplicationproject.png)

5. Create a new SDx project:  
   1. Enter a project name.  
   2. Select **Use default location**.  
   3. Click **Next**.  
  The Platform page is displayed.  
![error: missing image](./images/183_hardware_platform_dialog.png)

6. Select `xilinx_u200_xdma_201830_1`, and then click **Next**.  
The Templates window opens, showing templates you can use to start building an SDAccel environment project. Unless you have downloaded other SDx examples, you should only see Empty Application and Vector Addition as available templates.  
![error: missing image](./images/183_example_empty.png)

   The selection of the hardware platform defines the project as an SDAccel or SDSoC™ environment project. In this case you have selected an SDAccel environment acceleration platform, so the project will be an SDAccel project.

7. Select `Empty Application`, and click **Finish**. This closes the new project wizard and takes you back to the main SDx GUI window.

8.  From the top menu bar of the SDx GUI, click **Xilinx**->**RTL Kernel Wizard**, as shown in the following figure.  
![Missing Image: RTL Wizard Menu](images/RTLKernelWizard.png)  

A *Welcome to SDx RTL Kernel Wizard* page is displayed with a summary of the kernel wizard features. Review the usage information.

# Configuration with the RTL Kernel Wizard

The RTL Kernel Wizard guides you through the process of specifying the interface characteristics for an RTL kernel. Using the RTL Kernel Wizard ensures that the RTL IP is packaged into a valid kernel that can be integrated into a system by SDAccel. Using the wizard has an added benefit of automating some necessary tasks for packaging the RTL IP into a kernel.

## General Settings

- **Kernel Identification**: Specifies the vendor, kernel name, and library, known as the "Vendor:Library:Name:Version" (VLNV) of the IP. The kernel name should match the top module name of the IP you are using for the RTL kernel.
- **Kernel Options**: Specifies the design type.
  - **RTL** (default): Generates the associated files in a Verilog format.
  - **Block design**: Generates a block design for the Vivado™ tools IP Integrator. The block design consists of a MicroBlaze™ subsystem that uses a block RAM exchange memory to emulate the control registers.
- **Clock and Reset Options**: Specifies the number of clocks used by the kernel and whether the kernel needs a top-level reset port.

1. For Kernel Identification, specify the kernel name as `Vadd_A_B`.

2. For the remaining options, keep the default values, and click **Next**.  
![Missing Image: P1.PNG](./images/P1.PNG)

## Scalars

Scalar arguments are used to pass input parameters from the host application to the kernel. For the number of input arguments specified, a corresponding register is created to facilitate passing the argument from software to hardware. Each argument is assigned an ID value that is used to access that argument from the host application. This ID value can be found on the summary page of the wizard.
- **Argument name**: Name of the argument.  
- **Argument type**: Type of the scalar argument expressed as a native C/C++ datatype. For example: (u)char, (u)short or (u)int.

1. Keep the default values, and then click **Next**.  
![scalar.PNG](./images/scalar.PNG)

## Global Memory

Global Memory is used to pass large data sets between the host and kernels, and between kernels to other kernels. This memory can be accessed by the kernel through an AXI4 memory mapped master interface. For each AXI4 master interface, you can customize the interface name, data width, and the number of associated arguments.
- **Number of AXI master interfaces**: Specifies the number of AXI interfaces in the kernel
- **AXI master definition**: Specifies the interface name, the data width (in bytes) and the number of arguments associated with each AXI4 interface.
- **Argument definition**: Specifies the pointer arguments assigned to each AXI4 interface. Each argument is assigned an ID value, that can be used to access the argument from the host application. This ID value assignment can be found on the summary page of the wizard.  

![Missing Image: maxi.PNG](./images/maxi.PNG)  

1. In Number of AXI master interfaces, select **2** since the Vector-Accumulate kernel has two AXI4 interfaces.

2. In the AXI master definition section:
   1. Do not modify the interface names.
   2. Do not modify the width.
   3. For Number of arguments, select **1** since each AXI4 interface is dedicated to a single pointer argument.

3. In the Argument definition section, under Argument name:
   1. For m00_axi, enter `A`. Dataset A is accessed through this AXI4 interface.
   2. For m01_axi, enter `B`. Dataset B is accessed through this AXI4 interface.
    The settings should be the same as the above screen capture.  

4. Click **Next**.  
The Summary Page is displayed.  

## Example Summary Page

![Missing Image: summary.PNG](./images/summary.PNG)

- **Target platform**: Specifies what platform the RTL kernel will be compiled for. The RTL kernel must be recompiled to support different platforms.
- **Function prototype**: Conveys what a kernel call would be like if it was a C function.
- **Register map**: Displays the relationship between the host software ID, argument name, hardware register offset, datatype, and associated AXI interface.

1. Before generating the kernel, review the summary page for correctness.

The RTL Kernel Wizard uses the specification captured through the various steps and summarized in the Summary Page to generate:
- A kernel description XML file.
  - `kernel.xml` specifies the attributes you defined in the wizard that are needed by the runtime and SDAccel environment flows, such as the register map.
- A sample kernel called VADD implementing A[i]=A[i]+1, including:
  - RTL code
  - Verification test bench
  - Host code
- A Vivado project for the VADD sample kernel.

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
