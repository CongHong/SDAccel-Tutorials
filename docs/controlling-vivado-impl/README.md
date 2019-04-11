<table>
 <tr>
   <td align="center"><img src="https://www.xilinx.com/content/dam/xilinx/imgs/press/media-kits/corporate/xilinx-logo.png" width="30%"/><h1>2018.3 SDAccel™开发环境教程</h1>
   <a href="https://github.com/Xilinx/SDAccel-Tutorials/branches/all">查看其他版本</a>
   </td>
 </tr>
 <tr>
 <td align="center"><h3>Controlling Vivado Implementation</h3>
 </td>
 </tr>
</table>

Xilinx®开放代码编译器（XOCC）是一个命令行编译器，它可以获取源代码并通过Vivado®实现功能运行它;此功能生成编程基于FPGA的加速器板所需的比特流（和其他文件）。有时，必须使用先进的Vivado综合和实现选项来实现所需的结果，包括时序收敛。
本节介绍如何在实现项目时控制Vivado流，并指导您完成GUI流程中的各个步骤。  
>**注意**: 本章适用于System Run。编译应用程序可能需要一个多小时。

SDAccel™环境提供两种控制Vivado流的方法：
* 在启动系统运行的编译之前，您可以使用`--xp` XOCC编译器开关传递Vivado特定的综合和实现选项。
* 如果您的应用程序已经为系统运行编译，那么您可以：
  1. 打开Vivado工具GUI。
  2. 优化设计。
  3. 生成一个新的检查点。
  4. 将此检查点导回SDAccel环境以完成新实现。

## 使用`--xp` XOCC编译器转换
此方案将描述如何：
 * 在RTL合成期间完全展平层次结构（即，指定 _FLATTEN_HIERARCHY = full_ ）。
 * 在路由步骤中使用NoTimingRelaxation指令： _DIRECTIVE=NoTimingRelaxation_

要使用`--xp` XOCC编译器转换将这些选项传递给Vivado流，必须使用以下语法（每个选项需要使用单独的`--xp`开关）：
* `--xp vivado_prop:run.my_rm_synth_1.{STEPS.SYNTH_DESIGN.ARGS.FLATTEN_HIERARCHY}={full}`
* `--xp vivado_prop:run.impl_1.{STEPS.ROUTE_DESIGN.ARGS.DIRECTIVE}={NoTimingRelaxation}`


必须为SDAccel GUI中的XOCC链接器指定这些选项。
1. 通过选择**File** - > **Import...**菜单导入`Prj_5_Pipes`项目。<!--ThomasB: the project name is non-descriptive and should be changed. Also, it is best to create projects from scratch rather than load existing ones.-->
2. 在提示窗口中，选择**Xilinx/SDx Project**。
3. 在下一个窗口中，选择 **SDx project exported zip file**。
4. 单击**Next**，浏览到项目目录，然后选择`Prj_5_Pipes.zip`。
5. 单击**OK**，您将看到**Application Projects/Prj_5_Pipes**已被选中。单击 **Finish**。  
6. 在“Assistant”选项卡中，选择* **System**->**binary_container_1**, 然后从右键单击菜单中单击 **Settings**:

  ![](images/vivado-implementation_snap1.PNG)

  为XOCC链接器选项指定了`--xp`选项：
  ![](images/vivado-implementation_snap2.PNG)
</br>
7. 单击 **Cancel** 关闭窗口。  
8. 右击 `Prj_5_Pipes/System`, 然后选择 **Build**。  

  在设计实现过程中，您可以观察到这些选项已正确传播到Vivado流程。 Vivado RTL Synthesis生成的日志消息位于`Prj_5_Pipes/System/binary_container_1/logs/link/syn/my_rm_synth_1_runme.log`.  
  ![](images/vivado-implementation_snap4.PNG)

9. 打开 `Prj_5_Pipes/System/binary_container_1/logs/link/syn/my_rm_synth_1_runme.log`, 然后搜索`flatten_hierarchy`字符串。你应该找到： 
```
Command: synth_design -top pfm_dynamic -part xcvu9p-fsgd2104-2-i -flatten_hierarchy full -mode out_of_context  
```

  Vivado实现功能生成的日志消息位于`Prj_5_Pipes/System/binary_container_1/logs/link/impl/impl_1_runme.log`  
![](images/vivado-implementation_snap5.PNG)
</br>
10. 打开 `Prj_5_Pipes/System/binary_container_1/logs/link/impl/impl_1_runme.log`, 然后搜索 `NoTimingRelaxation` string. 字符串。你应该找到：  
```
Command: route_design -directive NoTimingRelaxation
```

  您可以查看日志文件以验证SDAccel环境中的指定选项是否已成功传播到Vivado流。

## 从SDAccel GUI运行Vivado
如果已为系统运行编译了应用程序，则可以打开Vivado GUI，加载合成或实现的设计，交互式地进行多项更改，生成新的检查点，然后将此检查点导出回SDAccel环境以完成新的实现。

>**重要**: 如果您的计算机内存小于32 GB，请仅查看以下步骤而不实际运行它们。否则，可能存在内存不足问题。

1. 打开Prj_5_Pipes项目，选择 **Xilinx**->**Vivado Integration**->**Open Vivado Project**启动Vivado工具：:  
![](images/vivado-implementation_snap6.PNG)  
2. 在打开的Vivado项目中，选择综合运行（`my_rm_synth_1`）。
3. 在“Synthesis Run Properties”部分中，选中“Options”选项卡。 `flatten_hierarchy`选项设置为`full`。这是使用`--xp`开关传播`flatten_hierarchy`选项的结果。
![](images/vivado-implementation_snap7.PNG)  
同样，检查Vivado实现功能设置是否按预期显示。

  假设您要调整Vivado实现结果中的时序。

4. 打开 **IMPLEMENTATION**, 然后打开 **Open Implemented Design**.
5. 要运行路径后物理综合，请在控制台窗口中进入：  
```
phys_opt_design -directive AggressiveExplore
```  
![](images/vivado-implementation_snap8.PNG)  
 如果只有轻微的降低，时间会得到改善。
6. 完成此过程后，将其写回以路由`.dcp`文件：
```
write_checkpoint -force prj/prj.runs/impl_1/pfm_top_wrapper_routed.dcp
```
>**注意**: 您可能需要更改`.dcp`文件的路径。使用`pwd`命令检查控制台中的工作目录。
7. 优化完成后，关闭Vivado工具。

## 将Vivado检查点导回SDAccel GUI

1. 在SDx™环境中，通过选择**Xilinx**->**Vivado Integration**->**Import Vivado Checkpoint...**导入更新的`.dcp`。  
完成后，会出现一个窗口，提示您确认导入`Vivado Settings`文件。  
![](images/vivado-implementation_snap9.PNG)  
2. 单击 **Yes** 以在提示窗口中导入Vivado设置文件（`xocc.ini`）。确认导入了`.dcp`和`.ini`文件，如下所示： 
![](images/vivado-implementation_snap10.PNG)
3. 在“Assistant”窗口中， 右击 **Prj_5_Pipes/System**, 然后选择 **Build**。  

您将看到增量流继续到Vivado实现流并生成比特流。

<hr/>
<p align="center"><sup>Copyright&copy; 2019 Xilinx</sup></p>
