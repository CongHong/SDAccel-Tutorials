<table>
 <tr>
   <td align="center"><img src="https://www.xilinx.com/content/dam/xilinx/imgs/press/media-kits/corporate/xilinx-logo.png" width="30%"/><h1>2018.3 SDAccel™开发环境教程</h1>
   <a href="https://github.com/Xilinx/SDAccel-Tutorials/branches/all">查看其他版本</a>
   </td>
 </tr>
 <tr>
 <td align="center"><h3>Using Multiple DDR Banks</h3>
 </td>
 </tr>
</table>

## 简介

默认情况下，在SDAccel™环境中，内核和DDR之间的数据传输是使用单个DDR存储区实现的。在某些应用程序中，数据移动是性能瓶颈。如果内核需要在全局内存（DDR）和FPGA之间移动大量数据，则可以使用多个DDR存储区。这使内核能够同时访问各种存储体。结果，应用程序性能提高。

使用`--sp`转换的系统端口映射选项允许设计人员将内核端口映射到特定的全局存储器组，例如DDR或可编程逻辑RAM（PLRAM）。本教程介绍如何将内核端口映射到多个DDR bank。

## 示例描述

这是添加矢量的简单示例。它显示了`vadd`内核从`in1`和`in2`读取数据并产生结果`out`。

在本教程中，您将使用三个DDR库实现向量添加应用程序。

因为XOCC编译器的默认行为是使用单个DDR存储区在内核和全局内存之间进行数据交换，所有通过端口`in1`，`in2`和`out`的数据访问都将通过默认的DDR存储区来完成该平台。  
![](./images/mult-ddr-banks_fig_01.png)

假设在应用程序中，您想要访问：
* `in1` 通过 `Bank0`
* `in2` 通过 `Bank1`
* `out` 通过 `Bank2`

![](./images/mult-ddr-banks_fig_02.png)

为了实现所需的映射，必须进行以下更新： 
* 修改主机代码以使用带有内核和参数索引的`cl_mem_ext_ptr`。
* 指示SDAccel工具将每个内核参数连接到所需的bank。

本教程中的示例使用C ++内核;但是，对于RTL和OpenCL™内核，所描述的步骤也是相同的。

## 教程设置

1. 启动SDx™环境GUI。
	1. 创建工作区目录：
		* `cd mult-ddr-banks`
		* `mkdir workspace`
	2. 在GUI模式下打开SDx™环境:
```
sdx -workspace workspace
```
2. 创建一个新项目:
	1. 从菜单中选择**SDX Application Project**。项目名称应为：**Prj_DDRs**
	2. 选择**xilinx_u200_xdma_201830_1**作为平台。
	3. 选择 **Empty Application**, 然后单击 **Finish**。
	4. 将本实验的源文件导入到项目的`src`目录中：  
![](./images/mult-ddr-banks_img_01.png)
3. 确保已将以下源文件导入`src`目录。展开`src`目录后，它将如下所示： 
![](./images/mult-ddr-banks_img_02.png)
4. 在SDx应用程序项目设置中添加**vadd**硬件功能：  
![](./images/mult-ddr-banks_img_03.png)
5. 选择**Emulation-HW**作为Active build配置。
6. 从菜单中选择 **Run Configurations**。  
![](./images/mult-ddr-banks_img_04.png)
7. 在Arguments选项卡中，选择 **Automatically add binary container to arguments**。  
![](./images/mult-ddr-banks_img_05.png)
8. 应用设置，并构建硬件仿真。 
如前所述，该设计的默认实现使用单个DDR Bank。在链接步骤中观察控制台窗口中的消息，您应该看到类似下面显示的消息。
![](./images/mult-ddr-banks_img_06.png)  
这确认了在没有指定明确的`--sp`选项的情况下，SDx环境为每个内核参数自动推断映射。
9. 单击绿色的“Run”按钮运行**HW-Emulation**。模拟完成后，您可以在Guidance选项卡下看到内核数据传输的内存连接，如下所示： 
![](./images/mult-ddr-banks_img_07.png)

现在，您将探索如何跨以下方式拆分数据传输：
* `DDR Bank 0`
* `DDR Bank 1`
* `DDR Bank 2`

## 更改主机代码
在主机代码中，您需要将`cl_mem_ext_ptr`与内核和参数索引一起使用。这是使用Xilinx®供应商扩展实现的。

打开`host.cpp`文件，然后寻找到以下部分，其中显示了代码更改：

```
// #define USE_MULT_DDR_BANKs
...
	// ------------------------------------------------------------------
	// Create Buffers in Global Memory to store data
	//             GlobMem_BUF_in1 - stores in1
	// ------------------------------------------------------------------
#ifndef USE_MULT_DDR_BANKs
#else
	cl_mem_ext_ptr_t  GlobMem_BUF_in1_Ext;
	GlobMem_BUF_in1_Ext.param = krnl_vector_add.get();
	GlobMem_BUF_in1_Ext.flags = 0; // 0th Argument
	GlobMem_BUF_in1_Ext.obj   = source_in1.data();

	cl_mem_ext_ptr_t  GlobMem_BUF_in2_Ext;
	GlobMem_BUF_in2_Ext.param = krnl_vector_add.get();
	GlobMem_BUF_in2_Ext.flags = 1; // 1st Argument
	GlobMem_BUF_in2_Ext.obj   = source_in2.data();

	cl_mem_ext_ptr_t  GlobMem_BUF_output_Ext;
	GlobMem_BUF_output_Ext.param = krnl_vector_add.get();
	GlobMem_BUF_output_Ext.flags = 2; // 2nd Argument
	GlobMem_BUF_output_Ext.obj   = source_hw_results.data();
#endif
```

要将内核端口分配给多个DDR存储区，您将使用`cl_mem_ext_ptr_t`结构。  
Where:
* `param`: 与arugument索引相关的内核。
* `flags`: 与有效内核关联的参数索引。
* `obj`: 分配的主机内存用于数据传输。

例如，`in2`的内存指针被分配给全局内存中的`GlobMem_BUF_in2_Ext`缓冲区。 
>**注意**: 如果您没有明确地将存储区分配给缓冲区，那么它将转到默认的DDR存储区。

然后，在缓冲区分配期间，使用`cl :: Buffer API`进行下面代码中显示的更改（与不使用Xilinx扩展时相比）：

```
#ifndef USE_MULT_DDR_BANKs
    OCL_CHECK(err, cl::Buffer buffer_in1   (context,CL_MEM_USE_HOST_PTR | CL_MEM_READ_ONLY,
            vector_size_bytes, source_in1.data(), &err));
    OCL_CHECK(err, cl::Buffer buffer_in2   (context,CL_MEM_USE_HOST_PTR | CL_MEM_READ_ONLY,
            vector_size_bytes, source_in2.data(), &err));
    OCL_CHECK(err, cl::Buffer buffer_output(context,CL_MEM_USE_HOST_PTR | CL_MEM_WRITE_ONLY,
            vector_size_bytes, source_hw_results.data(), &err));
#else
    OCL_CHECK(err, cl::Buffer buffer_in1   (context,CL_MEM_USE_HOST_PTR | CL_MEM_READ_ONLY | CL_MEM_EXT_PTR_XILINX,
            vector_size_bytes, &GlobMem_BUF_in1_Ext, &err));
    OCL_CHECK(err, cl::Buffer buffer_in2   (context,CL_MEM_USE_HOST_PTR | CL_MEM_READ_ONLY | CL_MEM_EXT_PTR_XILINX,
            vector_size_bytes, &GlobMem_BUF_in2_Ext, &err));
    OCL_CHECK(err, cl::Buffer buffer_output(context,CL_MEM_USE_HOST_PTR | CL_MEM_WRITE_ONLY | CL_MEM_EXT_PTR_XILINX,
            vector_size_bytes, &GlobMem_BUF_output_Ext, &err));
#endif
```

要在主机代码中进行所有更改，请取消注释主机代码开头的`#define USE_MULT_DDR_BANKs`行并保存。

## 设置XOCC链接器选项

接下来，您需要指示XOCC内核链接器将内核参数连接到相应的库。使用`--sp`选项映射内核端口或内核参数。

内核端口：
```
--sp <kernel_cu_name>.<kernel_port>:<sptag>
```
内核args：
```
--sp <kernel_cu_name>.<kernel_arg>:<sptag>
```
你也可以做范围：``<sptag> [min：max]``。为方便起见，还支持单个索引（例如，DDR [2]）。

* `<kernel_cu_name\>`: 计算单元基于内核名称，后跟`_`和`index`，从值“1”开始。例如，vadd内核的计算机单元名称将为“vadd_1”
* `<kernel_arg\>`: 计算单元的内存接口或函数参数。对于`vadd`内核，可以在`vadd.cpp`文件中找到内核参数。
* `<sptag\>`: 表示目标平台上可用的内存资源。有效的sptag名称包括DDR和PLRAM。在本教程中，目标是`DDR[0]`，`DDR[1]`和`DDR[2]`。


查看`vadd`内核的`--sp`参数的定义。
* **Kernel**: `vadd`
* **Kernel instance name**: `vadd_1`.

考虑到这是一个C / C +内核，参数在`vadd.cpp`文件中指定。这个内核参数（`in1`，`in2`和`out`）应该连接到`DDR [0]`，`DDR [1]`和`DDR [2]`。
因此`--sp`选项应该是：
```
--sp vadd_1.in1:DDR[0]
--sp vadd_1.in2:DDR[1]
--sp vadd_1.out:DDR[2]
```
参数`in1`访问DDR Bank0，参数`in2`访问DDR Bank1，参数`out`访问DDR Bank2。

>**注意**: 建议在分配DDR存储区时使用`--slr`选项将计算单元分配给特定的SLR。  
>该选项的语法是：`--slr <COMPUTE_UNIT>：<SLR_NUM>`，其中`COMPUTE_UNIT`是计算单元的名称，`SLR_NUM`是计算单元所分配的SLR编号。
>例如，`xocc ... --slr vadd_1：SLR2`将名为`vadd_1`的计算机单元分配给SLR2。

当你把所有`--sp`选项放在一起时，你会看到：
```
--sp vadd_1.in1:DDR[0] --sp vadd_1.in2:DDR[1] --sp vadd_1.out:DDR[2]
```

>**重要**：为了避免在SDAccel GUI中指定所有这些选项时出错，请从`host.cpp`文件的开头复制这些选项。选择以`--sp`开头的字符串的一部分：

```
// #define USE_MULT_DDR_BANKs
// Note: set XOCC options for Link: --sp vadd_1.in1:DDR[0] --sp vadd_1.in2:DDR[1] --sp vadd_1.out:DDR[2]
```

要为XOCC链接器指定这些选项，请从“Assistant ”窗口中单击 **Settings**:  
![](./images/mult-ddr-banks_img_09.png)

在“Project Settings”对话框中，通过粘贴先前复制的`--sp`选项来指定**XOCC linker options**：  
![](./images/mult-ddr-banks_img_10.png)

单击**Apply**后，关闭Project Settings窗口。通过单击**Project** - > **Clean...**，在**HW Emulation**模式下完成项目的清除。
![](./images/mult-ddr-banks_img_11.png)

再次，在链接步骤中观察控制台窗口中的消息;您现在应该看到类似于下图的消息： 
![](./images/mult-ddr-banks_img_12.png)

这确认了SDAccel环境已经从提供的`--sp`选项正确地将内核参数映射到指定的DDR存储区。

现在，通过单击绿色的“Run”按钮运行HW-Emulation，然后验证设计的正确性。模拟完成后，单击**Guidance**以查看内核数据传输的内存连接，如下所示。  
![](./images/mult-ddr-banks_img_13.png)

## 总结

本教程向您展示了如何将内核`vadd`的端口`in1`，`in2`和`out`的默认映射从单个DDR存储区更改为多个DDR存储区。您还学会了如何：

* 定义主机指针，并将主机指针分配给`cl：Buffer` API。
* 使用`--sp`开关设置XOCC链接器选项，将内核参数绑定到多个DDR存储区。
* 构建应用程序并验证DDR映射。

<hr/>
<p align="center"><sup>Copyright&copy; 2019 Xilinx</sup></p>
