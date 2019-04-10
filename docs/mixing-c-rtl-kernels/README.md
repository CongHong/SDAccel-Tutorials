<table>
 <tr>
   <td align="center"><img src="https://www.xilinx.com/content/dam/xilinx/imgs/press/media-kits/corporate/xilinx-logo.png" width="30%"/><h1>2018.3 SDAccel™开发环境教程</h1>
   <a href="https://github.com/Xilinx/SDAccel-Tutorials/branches/all">查看其他版本</a>
   </td>
 </tr>
 <tr>
 <td align="center"><h3>Mixing C and RTL Kernels</h3>
 </td>
 </tr>
</table>

## 介绍
SDAccel™开发环境提供了一个加速硬件功能的平台。虽然主机软件或应用程序是使用OpenCL™API调用在C / C ++中开发的，但硬件组件或内核可以用C / C ++，OpenCL C或RTL开发。实际上，SDAccel环境应用程序可以使用以不同语言开发的任何内核组合。本教程演示了使用两个内核的应用程序，其中主机代码以相同的方式访问内核：
* 用C++开发的内核
* 用RTL开发的内核

### 工作流程
在本教程中，您将在GUI模式下使用SDAccel工具来初始创建新项目，并添加基于C ++的内核。然后，您将生成一个二进制容器（xclbin），其中包含基于C ++的内核，以便在FPGA上实现。然后，在软件仿真中运行设计。您将查看生成的应用程序时间轴报告，以查看主机应用程序正在调用和运行的内核。

在第二部分中，您将使用RTL内核向导创建一个简单的RTL内核并将其添加到项目中。二进制容器将更新为包含基于RTL的内核以及C ++内核。您将更新主机代码以使用基于RTL的内核并运行硬件仿真。再一次，您将查看应用程序时间轴，并查看主机应用程序调用和运行的两个内核。

## 使用C / C ++内核
首先，启动SDAccel环境，创建新项目并导入源。作为参考，_SDAccel Environment User Guide_ ([UG1023](https://www.xilinx.com/cgi-bin/docs/rdoc?v=2018.3;d=ug1023-sdaccel-user-guide.pdf)) 提供了具体的参考细节。[Getting Started with C/C++ Kernels](./docs/getting-started-c-kernels/README.md) 实验室还详细介绍了具体步骤。

### 启动SDAccel环境
1. 打开Linux终端，运行以下命令以GUI模式启动SDAccel：
```
$ sdx
```  
2. 在Workspace Launcher对话框中，创建一个名为 `Tutorial`的新工作区，然后单击 **OK** 继续。您可以选择工作区的任何位置。

### 使用“Project Creation”向导创建新项目

1. 从File / New菜单中选择 **SDx Application Project** 。
2. 指定以下项目名称： `mixed_c_rtl`。
3. 单击 **Next**.
4. 在“Platform”对话框中，选择 `xilinx_u200_xdma_201830_1` 平台，然后单击 **Next**。
5. 最后，在“Templates”对话框中，选择 **Empty Application**, 然后单击 **Finish**。

### 将源文件添加到prj_c_rtl项目

1. 在“SDAccel Project Explorer”窗格中，单击 **Import Sources**, 在下图中以红色圈出：  
![Missing Image:ImportSources](images/import_sources_icon.PNG)  
2. 浏览到 `mixing-c-rtl-kernels/reference-files` 然后单击 **OK**。
3. 选择目录中的所有文件，然后单击 **Finish**。这会将主机和C ++内核源文件从 `reference-files` 目录导入到 `mixing_c_rtl` 项目的 `src` 目录。

内核 (`krnl_vadd.cpp`) 添加两个输入向量并生成输出结果。
主机代码 (`host.cpp`) 设置平台并定义全局内存缓冲区和内核连接。下面描述了主机代码中的四组重要的OpenCL API调用。您可以通过打开 `host.cpp` 文件来查看这些调用。

第一组代码在 `host.cpp` 文件的第135-138行创建了要执行的程序。它使用仅包含基于C ++的内核的二进制容器。

```
cl::Program::Binaries bins;
bins.push_back({buf,nb});
devices.resize(1);
cl::Program program(context, devices, bins);
```

第二行，在第142行，从程序中获取C ++ `krnl_vadd` 内核对象，并指定名称 `krnl_vector_add`。它允许主机使用内核。

```
cl::Kernel krnl_vector_add(program,"krnl_vadd");
```

第162到165行的第三组代码将 `krnl_vector_add` 内核参数赋给缓冲区：

```
krnl_vector_add.setArg(0,buffer_a);
krnl_vector_add.setArg(1,buffer_b);
krnl_vector_add.setArg(2,buffer_result);
krnl_vector_add.setArg(3,DATA_SIZE);
```


参数数字0,1,2和3匹配`krnl_vadd.cpp`中的`krnl_vadd`定义中的参数顺序，如下所示：

```
void krnl_vadd(
                int* a,
                int* b,
                int* c,
                const int n_elements)
```
最后，在第171行，以下OpenCL API启动`krnl_vector_add`内核：

```
q.enqueueTask(krnl_vector_add);
```

当我们添加基于RTL的代码时，我们将为该内核添加相同的调用。
如您所见，高级OpenCL调用独立于内核编程的语言。

>**注意**: 有关主机代码编程的完整详细信息，请参见 _SDAccel Environment Programmers Guide_ ([UG1277](https://www.xilinx.com/support/documentation/sw_manuals/xilinx2018_3/ug1277-sdaccel-programmers-guide.pdf)).

### 创建一个包含C ++内核的二进制容器

1. 在“Project Explorer”下，双击**project.sdx**以打开“SDx Application Project Settings”窗口。  
![Missing Image:OpenSDxAppPrj](images/mixing-c-rtl-kernels_open_sdx_app_prj_settings.PNG)  

2. 单击“Hardware Function”部分中的闪电图标，如下所示。  
![Missing Image:HW_Functions](images/mixing-c-rtl-kernels_hw_functions.PNG)  
单击闪电图标后，SDx™开发环境将扫描所有源文件，并自动识别项目中存在的内核（在这种情况下，只有一个内核）。
3. 选择内核，然后单击 **OK**。 
创建一个包含默认“binary_container_1”名称的二进制容器，其中包含krnl_vadd，如下所示。 
![Missing Image:BinContainer](images/mixing-c-rtl-kernels_bin_container_with_cpp.PNG)  

现在，该项目已准备好进行编译（即构建）。

### 编译项目

1. 在“SDx Application Project Settings”窗口中，确保配置为**Emulation-SW**：  
![Missing Image:SW_EMU](images/mixing-c-rtl-kernels_sw_emulation.PNG)  
2. 单击锤子图标以开始编译过程：  
![Missing Image:HammerCompile](images/mixing-c-rtl-kernels_hammer.PNG)  
>**注意**: 您将使用软件模拟来验证主机代码和内核的功能。软件仿真运行时非常快，因为主机代码和内核代码都是在开发机器的x86处理器上编译和运行的。
3. 运行模拟，单击运行图标： 
![Missing Image:RunIcon](images/run_icon.PNG)  
当应用程序成功完成时，您将在控制台窗口中看到`TEST WITH ONE KERNEL PASSED`。

### 申请时间表审核

查看软件仿真期间生成的应用程序时间轴。应用程序时间轴在公共时间轴上收集和显示主机和设备事件。您可以使用它来可视化主机事件和运行的内核。
1. 在“Assistant”视图中，展开**Emulation-SW**，然后展开**mixed_c_rtl-Default**，如下所示。  
![Missing Image:Application Timeline 1](images/mixed_c_rtl_default_from_assistant.png)
2. 双击**Application Timeline** 以查看主机事件，包括创建程序和缓冲区。
3.	在**Device** - > **Binary Container**下，您将看到一行名为`Compute Unit krnl_vadd_1`的行。沿时间轴移动并放大以查看计算单元`krnl_vadd_1`的运行位置。  
![Missing Image:Application Timeline 1](images/mixing-c-rtl-kernels_timeline_one_kernel.PNG)  
查看后，关闭“Application Timeline”窗口。  
>**注意:** 计算单元是FPGA上内核的实例。

## 创建RTL内核

既然您已经成功添加并运行了C ++内核，那么您将在`prj_c_rtl`项目中添加一个RTL内核。然后，您将构建并运行C ++和RTL内核。对于SDAccel环境，内核的源代码可以是任何一种受支持的语言：
* OpenCL
* C/C++
* RTL

正如您将看到的，无论内核是如何设计的，主机代码都通过类似的函数调用来访问内核。  

### 使用RTL内核向导

首先，您将使用RTL内核向导生成一个内核，该内核将向输入向量添加常量。 RTL内核向导可以自动执行将RTL设计打包到可由SDAccel环境访问的已编译内核对象文件（`.xo`）中所需的一些步骤。

默认情况下，向导会创建一个简单的向量添加内核，就像本教程第一部分中使用的C ++内核一样。您将创建此RTL内核并将其添加到项目中。  
>**重要**：为了在本教程中创建RTL内核，您将很快完成这些步骤。但是，RTL内核向导将在 [Getting Started with RTL Kernels](./docs/getting-started-rtl-kernels/README.md) 教程中详细讨论。此外，可以在 _SDAccel Environment User Guide_ ([UG1023](https://www.xilinx.com/cgi-bin/docs/rdoc?v=2018.3;d=ug1023-sdaccel-user-guide.pdf))中找到RTL内核向导的完整详细信息。

##### 打开RTL内核向导

1.	在“Xilinx”菜单下，选择**RTL Kernel Wizard**。这将打开RTL内核向导，该向导以欢迎页面开始。
2. 单击 **Next**.
3. 在“General Settings”对话框中，保留所有默认设置，然后单击 **Next**。
4. 在“Scalars”对话框中，将标量参数的数量设置为`0`，然后单击**Next**。
5. 在“Global Memory”对话框中，保留所有默认设置，然后单击 **Next**。  
“Summary”对话框提供了RTL内核设置的摘要，并包含一个函数原型，它将内核调用的内容表示为C函数。
6. 单击 **OK**。


### Vivado项目

此时，Vivado™设计套件将打开一个项目，用于生成的RTL代码。由RTL内核向导生成和配置的RTL代码对应于`A = A + 1 function`。您可以查看源文件，甚至可以运行RTL模拟。但是，对于本教程，您将只生成RTL内核。

1. 在"Flow Navigator"中，单击**Generate RTL Kernel**，如下所示。  
![Missing Image:Flow Navigator](images/mixing-c-rtl-kernels_flow_navigator.png)  
2. 在"Generate RTL Kernel"对话框中，选择**Sources-only**内核打包选项。
3. 对于软件仿真源（可选），您可以添加RTL内核的C ++模型，该模型可在软件仿真期间用于SDAccel工具。 C ++模型必须由设计工程师编写。通常，没有可用的C ++模型，并且使用硬件仿真来测试设计。
由于RTL向导创建了VADD设计的C ++模型，现在可以添加它。
4. 单击 `…` (浏览按键)。
5. 双击`imports`目录。
6. 选择唯一的`.cpp`文件，然后单击 **OK**。
7. 再次单击**OK**以生成RTL内核。
8. 成功生成RTL内核后，单击**Yes**退出Vivado Design Suite，然后返回SDAccel环境。

## 将RTL和C ++内核添加到主机代码中

在SDAccel环境中，您将返回到您开始的`rtl_c_prj`项目。展开项目浏览器的`src`目录，您将看到RTL内核已添加到`sdx_rtl_kernel`下的项目中。

内核源文件夹包含两个文件：
- `host_example.cpp` (示例主机代码)
- `.xo` 文件 (RTL内核)  
![Missing Image:KernelDir](images/mixing-c-rtl-kernels_kernel_dir_structure.PNG)

由于项目已包含主机代码，因此必须删除生成的`host_example.cpp`文件。要执行此操作，右击`host_example.cpp`文件并选择**delete**。

### 将新内核添加到二进制容器中

将新内核添加到项目中后，您现在必须将其添加到二进制容器中。

1. 在“SDx Application Project Settings”窗口中，从“硬件功能”区域中单击闪电图标。
2. 选择刚刚创建的RTL内核，然后单击 **OK**。  
这将RTL内核添加到二进制容器（已包含C ++内核）。
3. 更新主机代码（`host.cpp`）以合并新内核，因为更新已经完成。打开`host.cpp`文件，删除第49行的注释`//`以定义`ADD_RTL_KERNEL`：

  * 原代码:
  ```
  //#define ADD_RTL_KERNEL
  ```
  * 修改后代码:
  ```
  #define ADD_RTL_KERNEL
  ```

4. 保存文件。

  更新的`host.cpp`文件包含其他OpenCL调用，简要描述如下。实际的OpenCL调用与用于基于C ++的内核的调用相同，并且基于RTL的内核的参数已更改。 

  虽然第一组代码没有改变，但二进制容器现在包含C ++和RTL内核。
  
  ```
  cl::Program::Binaries bins;
  bins.push_back({buf,nb});
  devices.resize(1);
  cl::Program program(context, devices, bins);
  ```
  
  下面的代码从程序中获取`sdx_kernel_wizard_0`对象，并在第145行分配名称`krnl_const_add`。`sdx_kernel_wizard_0`对象名称将与RTL向导生成的名称匹配。
  
```
cl::Kernel krnl_const_add(program,"sdx_kernel_wizard_0");
```

  接下来，我们在第168行定义`krnl_const_add`内核参数。需要注意的是，在主机代码中，缓冲区'buffer_results'通过DDR直接从C内核传递到RTL内核，而不会移回主机内存。
  
```
krnl_const_add.setArg(0,buffer_result);
```

>**注意**: 在主机代码中，`buffer_results`缓冲区通过DDR直接从C内核传递到RTL内核，而不会移回主机内存。

  
在第173行启动`krnl_const_add`内核：

```
q.enqueueTask(krnl_vector_add);
```

将RTL内核添加到二进制容器和主机代码后，您可以重建项目并运行硬件仿真。

5. 确保将活动构建配置设置为**Emulation-HW**，然后单击运行按钮，如下所示。这将编译并运行硬件仿真。 
![Missing Image:RunButton](images/mixing-c-rtl-kernels_run_button.PNG)  
仿真完成后，您将在控制台窗口中看到“TEST WITH TWO KERNELS PASSED”。
再次打开“Application Timeline”报告，以查看两个内核是否正在运行。

6. 在“Assistant”窗口中，展开**Emulation-HW**和默认内核，然后双击“Application Timeline”文件将其打开。
7. 在**Device** - > **Binary Container**下，沿着时间轴遍历并放大。现在您将看到两个计算单元`krnl_vadd_1`和`rtl_kernel`正在运行。 
![Missing Image:Application Timeline 2](images/mixing-c-rtl-kernels_timeline_two_kernels.PNG)

## 总结

本教程演示了SDAccel环境应用程序如何使用任何内核组合，无论它们是在何种语言中开发。具体来说，我们展示了在同一应用程序中运行的基于C ++和RTL的内核。主机代码以相同的方式访问和运行内核。

<hr/>
<p align="center"><sup>Copyright&copy; 2019 Xilinx</sup></p>
