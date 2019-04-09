<table>
 <tr>
   <td align="center"><img src="https://www.xilinx.com/content/dam/xilinx/imgs/press/media-kits/corporate/xilinx-logo.png" width="30%"/><h1>2018.3 SDAccel™开发环境教程</h1>
   <a href="https://github.com/Xilinx/SDAccel-Tutorials/branches/all">查看其他版本</a>
   </td>
 </tr>
 <tr>
 <td align="center"><h3>Host Code Optimization</h3>
 </td>
 </tr>
</table>

## 简介

本教程重点介绍与FPGA加速应用程序相关的主机代码的性能调优。主机代码优化只是性能优化的一个方面，它还包括以下原则：
* 主机程序优化
* 内核代码优化
* 拓扑优化
* 执行优化

在本教程中，您将使用简单，单一，通用的C ++内核实现。这允许您从主机代码实现的分析中消除内核代码修改，拓扑优化和实现选择的任何方面。
>本教程中显示的主机代码优化技术仅限于优化加速器集成的方面。允许在主机代码上使用多个CPU内核或内存管理的其他常见技术不是本讨论的一部分。有关更多信息，请参阅 _SDAccel Profiling and Optimization Guide_ ([UG1207](https://www.xilinx.com/support/documentation/sw_manuals/xilinx2018_3/ug1207-sdaccel-optimization-guide.pdf)).

以下部分重点介绍以下特定主机代码优化问题：
* 软件流水线/事件队列
* 内核和主机代码同步
* 缓冲区大小

## 模型

本示例中使用的内核仅用于主机代码优化。它在整个教程中都是静态的，它允许您查看优化对主机代码的影响。
<!--this could be better suited as a table-->

C ++内核有一个输入和一个输出端口。这些端口为512位宽，以最佳地利用AXI带宽。每次执行内核消耗的元素数量可通过`numInputs`参数进行配置。类似地，`processDelay`参数可用于改变内核的延迟。该算法将输入值增加`ProcessDelay`的值。但是，这个增量是通过循环执行`processDelay`次来实现的，每次递增输入值一次。因为这个循环存在于内核实现中，所以每次迭代最终都需要一个恒定的循环量，可以乘以`processDelay`数。

内核还设计用于启用AXI突发传输。内核包含一个读取和写入过程，在进程结束时与实际的内核算法（`exec`）并行执行。
读取和写入过程在一个简单的循环中启动AXI事务，并将接收到的值写入内部FIFO，或从内部FIFO读取并写入AXI输出。 Vivado高级综合（HLS）将这些块实现为并发并行进程，因为DATAFLOW编译指示是在周围的`pass_dataflow`函数上设置的。

## 构建内核
>**注意**: 本教程中的所有指令都是从`reference-files`目录运行的。

虽然一些主机代码优化在硬件仿真中表现良好，但准确的运行时信息和大型测试向量的运行将要求内核在实际系统上执行。通常，在主机代码优化期间，内核不会发生变化;这是一次性命中，可以在硬件模型最终确定之前轻松执行。

在本教程中，设置了一个示例内核，通过发出以下命令一次构建硬件比特流：<!--ThomasB: Would be nice to add the specific string for U200 as an example.-->
```
make TARGET=hw DEVICE=<device> kernel
```
将`device`替换为已安装的Xilinx®加速卡的设备文件（`.xpfm`）。
>**注意**: 此构建过程将花费几个小时，并且必须先完成内核编译，然后才能分析主机代码影响。

## 主机代码

在检查主机代码的不同实现选项之前，请查看代码的结构。主机代码文件旨在让您专注于主机代码优化的关键方面。通过公共源目录（`srcCommon`）中的头文件提供了三个类：

* `srcCommon/AlignedAllocator.h`: `AlignedAllocator`是一个有两种方法的小结构。此结构作为辅助类提供，以支持测试向量的内存对齐分配。内存对齐的数据块可以更快地传输，如果传输的数据不是内存对齐的，OpenCL™库将产生警告。

* `srcCommon/ApiHandle.h`: 该类封装了主要的OpenCL对象：
  * 内容
  * 程序
  * `device_id`
  * 执行内核
  * `command_queue`  
这些结构由构造函数填充，构造函数逐步执行OpenCL函数调用的默认序列。构造函数只有两个配置参数：
      * 一个字符串，包含用于编程FPGA的比特流名称（`xclbin`）。
      * 一个布尔值，用于确定是否应创建无序队列或顺序执行队列。

  该类为生成缓冲区和加速器上的任务调度所需的队列，上下文和内核提供附件函数。当调用`ApiHandle`析构函数时，该类还会自动释放已分配的OpenCL对象。

* `srcCommon/Task.h`:类`Task`的对象表示要在加速器上执行的工作负载的单个实例。无论何时构造此类的对象，都会根据每个任务调用要传输的缓冲区大小来分配和初始化输入和输出向量。类似地，析构函数将取消分配在任务执行期间生成的任何对象。
>**注意**:这种用于调用模块的单个工作负载的封装允许该类还包含输出验证器函数（`outputOk`）。

   该类的构造函数包含两个参数：
    * `bufferSize`: 确定执行此任务时传输的512位值。
    * `processDelay`: 提供类似命名的内核参数，并在验证期间使用它。

  这个类最重要的成员函数是`run`-函数。此函数将OpenCL排列为三个不同的步骤来执行算法：
    1. 将数据写入FPGA加速器
    2. 设置内核并运行加速器
    3. 从FPGA上的DDR存储器读取数据

  要执行此任务，将在DDR上分配缓冲区以进行通信。此外，事件用于表示不同任务之间的依赖关系（在读取之前执行之前写入）。

除了ApiHandle对象之外，`run`-函数还有一个条件参数。此参数允许任务依赖于先前生成的事件。这允许主机代码建立任务顺序依赖关系，如本教程后面所述。

在本教程中，任何这些头文件中的代码都不会被修改。所有关键概念都将显示在不同的`host.cpp`文件中，如下所示：
* `srcBuf`
* `srcPipeline`
* `srcSync`

但是，即使`host.cpp`文件中的main函数也遵循特定的结构，这将在下一节中介绍。

### host.cpp主要功能
主函数包含相应标记在源中的以下部分。

1. 环境/使用检查
2. 常用参数：
   * `numBuffers`: 预计不会被修改。此参数用于确定执行的内核调用次数。
   * `oooQueue`: 如果为true，则此布尔值用于声明在ApiHandle内生成的OpenCL事件队列的类型。
   * `processDelay`: 该参数可用于人为地延迟内核所需的计算时间。此版本的教程不会使用此参数。
   * `bufferSize`: 此参数用于声明每个内核调用要传输的512位值的数量。
   * `softwarePipelineInterval`: 此参数用于确定在同步发生之前允许预先安排的操作数。
3. 设置：为确保您了解配置变量的状态，本节将打印出最终配置。
4. 执行：在本节中，您将能够模拟几个不同的主机代码性能问题。这些是您将在本教程中关注的内容。
5. 测试：执行完成后，本节将对输出执行简单检查。
6. 性能统计：如果模型在实际加速卡上运行（未模拟），主机代码将根据系统时间测量计算并打印性能统计信息。

>**注意:** 设置以及其他部分可以打印记录系统状态的其他消息，以及运行的总体 `PASS` 或 `FAIL` 。

### 使用乱序事件队列的流水线内核执行

在第一个练习中，您将看到流水线内核执行。
>**注意:** 您正在处理单个计算单元（内核的实例），因此在每个点上，只有一个内核可以在硬件中实际运行。然而，如上所述，内核的运行还需要向计算单元和从计算单元传输数据。应重叠这些活动以最小化使用主机应用程序的内核的空闲时间。

首先编译并运行主机代码 (`srcPipeline/host.cpp`):

```
make TARGET=hw DEVICE=<device> pipeline
```


同样，``<device>``应替换为可用加速卡的实际设备文件（`.xpfm`）。与内核编译时间相比，这似乎是一个瞬间动作。

查看主机代码中的执行循环：

```
  // -- Execution -----------------------------------------------------------

  for(unsigned int i=0; i < numBuffers; i++) {
    tasks[i].run(api);
  }
  clFinish(api.getQueue());
```
<!--ThomasB: Formatting should be cpp, not bash-->
在这种情况下，代码会调度所有缓冲区并让它们执行。只有在最后它才会实际同步并等待完成。

构建完成后，您可以使用以下命令运行主机可执行文件：

```
make TARGET=hw DEVICE=<device> pipelineRun
```

此脚本设置为运行应用程序，然后生成SDaccel GUI。 GUI将自动填充收集的运行时数据。

使用`sdaccel.ini`文件生成运行时数据，该文件包含以下内容：

```
[Debug]
profile=true
timeline_trace=true
data_transfer_trace=coarse
stall_trace=all
```

`sdaccel.ini`文件的详细信息可以在 _SDAccel Environment User Guide_ 中找到([UG1023](https://www.xilinx.com/support/documentation/sw_manuals/xilinx2018_2/ug1023-sdaccel-user-guide.pdf)).

“Application Timeline”查看器说明了可执行文件的完整运行。时间表的三个主要部分是：
* OpenCL API Calls
* Data Transfer section
* Kernel Enqueues

放大说明实际加速器执行的部分，并选择一个内核入队以查看类似于以下内容的图像：
![](images/OrderedQueue.PNG)

蓝色箭头标识依赖关系，您可以看到每个写入/执行/读取任务执行都依赖于先前的写入/执行/读取操作集。这有效地序列化了执行。

回顾主机代码中的执行循环，在Write / Execute / Read运行之间没有指定依赖关系。对特定任务的**run**的每次调用仅依赖于apiHandle，否则完全封装。

在这种情况下，通过使用有序队列创建依赖关系。在参数部分中，`oooQueue`参数设置为`false`：<!--ThomasB: Formatting should be cpp, not bash-->

```
  bool         oooQueue                 = false;
```

您可以通过将无序参数更改为`true`来打破此依赖关系：<!--ThomasB: Formatting should be cpp, not bash-->

```
  bool         oooQueue                 = true;
```

重新编译并执行：

```
make TARGET=hw DEVICE=<device> pipeline
make TARGET=hw DEVICE=<device> pipelineRun
```

放大应用程序时间轴并单击任何内核排队结果类似于下图：

![](images/OutOfOrderQueue.PNG)

如果选择其他传递内核入队，您将看到它们中的所有10个现在仅在Write/Execute/Read组中显示依赖关系。这允许读写操作与执行重叠，并且您正在有效地管道化软件的写入，执行和读取。这可以显着提高整体性能，因为通信开销与加速器的执行同时发生。


## 内核和主机代码同步

在这一步中，查看`srcSync`（`srcSync/host.cpp`）中的源代码，并检查执行循环。这与教程上一节中的相同：<!--ThomasB: Formatting should be cpp, not bash-->

```
  // -- Execution -----------------------------------------------------------

  for(unsigned int i=0; i < numBuffers; i++) {
    tasks[i].run(api);
  }
  clFinish(api.getQueue());
```

在此示例中，代码实现了自由运行的管道。直到结束时才执行同步，此时在事件队列上执行对`clFinish`的调用。虽然这会创建一个有效的管道，但此实现存在与缓冲区分配以及执行顺序相关的问题。

例如，如果`numBuffer`变量增加到一个大数或者它是一个未知数（如处理视频流时的情况），可能会出现问题。在这些情况下，缓冲区分配和内存使用可能会成为问题。在此示例中，主机存储器已预先分配并与FPGA共享，因此此示例可能会耗尽内存。

类似地，由于执行加速器的每个调用都是独立且不同步的（无序队列），因此不同调用之间的执行顺序可能与排队顺序不一致。因此，如果主机代码正在等待特定块完成，则可能要比预期晚得多。这有效地禁用了加速器运行时的任何主机代码并行性。

为了缓解这些问题，OpenCL提供了两种同步方法：
* `clFinish`call
* `clWaitForEvents` call

首先，看看使用`clFinish`调用。要说明该行为，您必须修改执行循环，如下所示：<!--ThomasB: Formatting should be cpp, not bash-->

```
  // -- Execution -----------------------------------------------------------

  int count = 0;
  for(unsigned int i=0; i < numBuffers; i++) {
    count++;
    tasks[i].run(api);
    if(count == 3) {
	  count = 0;
	  clFinish(api.getQueue());
    }
  }
  clFinish(api.getQueue());
```

重新编译并执行：

```
make TARGET=hw DEVICE=<device> sync
make TARGET=hw DEVICE=<device> syncRun
```

如果放大应用程序时间轴，将显示类似于以下内容的图像：
![](images/clFinish.PNG)

这个图中的关键元素是名为`clFinish`的红色框，内核之间的较大间隙将加速器的每三次调用排入队列。

对`clFinish`的调用会在完整的OpenCL命令队列上创建一个同步点。这意味着在`clFinish`将控制权返回给主机程序之前，必须完成排队到给定队列的所有命令。因此，在下一组3个加速器调用可以恢复之前，必须完成所有活动（包括缓冲区通信）。这实际上是屏障同步。

虽然这使得可以释放缓冲区并且保证所有进程都已完成的同步点，但它也可以防止同步点处的重叠。

查看备用同步方案，其中基于先前执行对加速器的调用的完成来执行同步。编辑`host.cpp`文件以更改执行循环，如下所示：

```
  // -- Execution -----------------------------------------------------------

  for(unsigned int i=0; i < numBuffers; i++) {
    if(i < 3) {
      tasks[i].run(api);
	} else {
	  tasks[i].run(api, tasks[i-3].getDoneEv());
    }
  }
  clFinish(api.getQueue());
```

重新编译并执行：

```
make TARGET=hw DEVICE=<device> sync
make TARGET=hw DEVICE=<device> syncRun
```

如果放大应用程序时间轴，将显示类似于以下内容的图像：
![](images/clEventSync.PNG)

在时间轴的后半部分，执行了五次执行，没有任何不必要的间隙。然而，更有说服力的是标记点处的数据传输。此时，已经发送了三个包以由加速器处理，并且已经收到一个包。因为在第一次加速器调用完成时已经同步了下一次写入/执行/读取调度，所以现在在接收到任何其他包之前会观察到另一个写入操作。这清楚地标识了重叠执行。

在这种情况下，通过在类任务的`run`方法中使用以下事件同步，您可以在执行预定的三个调用完成时同步完整的下一个加速器执行：<!--ThomasB: Formatting should be cpp, not bash-->

```
    if(prevEvent != nullptr) {
      clEnqueueMigrateMemObjects(api.getQueue(), 1, &m_inBuffer[0],
				 0, 1, prevEvent, &m_inEv);
    } else {
      clEnqueueMigrateMemObjects(api.getQueue(), 1, &m_inBuffer[0],
				 0, 0, nullptr, &m_inEv);
    }
```

虽然这是OpenCL中排队对象之间的通用同步方案，但也可以通过调用以下方式同步主机代码：<!--ThomasB: Formatting should be cpp, not bash-->

```
  clWaitForEvents(1,prevEvent);
```

这允许在加速器在先前排队的任务上操作时进行额外的主机代码计算。这里没有进一步探讨，而是留给读者作为额外的练习。


## OpenCL API缓冲区大小

在本教程的最后一节中，您将研究缓冲区大小对总体性能的影响。为此，您将专注于`srcBuf / host.cpp`中的主机代码。执行循环与上一节的结尾完全相同。

但是，在此主机代码文件中，要处理的任务数已增加到100.此更改的目标是获得100个加速器调用以传输100个缓冲区并读取100个缓冲区。这使该工具能够获得每次传输更准确的平均吞吐量估算。

此外，还添加了第二个命令行选项（`SIZE =`）以指定特定运行的缓冲区大小。在单次写入或读取期间要传输的实际缓冲区大小是通过计算指定参数（`pow（2，argument）`）乘以512位的幂来确定的。

您可以通过调用以下命令编译主机代码：

```
make TARGET=hw DEVICE=<device> buf
```

使用以下命令运行可执行文件：

```
make TARGET=hw DEVICE=<device> SIZE=14 bufRun
```

参数`SIZE`用作主机代码可执行传递的第二个参数。
>**注意**: 如果不包括`SIZE`，则默认设置为`SIZE = 14`。这允许代码以不同的缓冲区大小执行实现，并通过监视总计算时间来测量吞吐量。此数字在Testbench中计算，并通过FPGA吞吐量输出报告。

为了简化不同缓冲区大小的扫描，创建了一个额外的Makefile目标，它可以通过以下命令执行：

```
make TARGET=hw DEVICE=<device> bufRunSweep
```

>**注意**: 扫描脚本（`auxFiles / run.py`）需要python安装，这在大多数系统中都可用。执行扫描将运行并记录FPGA吞吐量，缓冲区大小参数为8到19.测量的吞吐量值与`runBuf / results.csv`文件中的每次传输的实际字节数一起记录，该文件打印在makefile执行结束。

分析这些数字时，应观察到类似于下图的阶梯函数：  
![](images/stepFunc.PNG)

此图像显示缓冲区大小明显影响性能并开始达到大约2 MB的水平。
>**注意**: 该图像是通过`results.csv`文件中的gnuplot创建的，如果在您的系统上找到它，它将在您运行扫描后自动显示。

关于主机代码性能，此步骤功能识别缓冲区大小和总执行速度之间的关系。如此示例所示，当默认实现基于少量输入数据时，很容易采用算法并更改缓冲区大小。它不必像这里执行的那样是动态的和运行时确定性的，但原理保持不变。您可以传输多个输入值，并在单次调用加速器时重复算法执行，而不是为一次调用算法传输单个值集。

## 总结

本教程说明了主机代码优化的三个特定领域：
   * 使用乱序事件队列的流水线内核执行
   * 内核和主机代码同步
   * OpenCL API缓冲区大小

在尝试创建有效的加速实现时，您应该考虑这些方面。该教程展示了如何分析这些性能瓶颈，并展示了如何改进这些瓶颈的方法。

有许多方法可以实现主机代码并提高性能。这适用于提高主机到加速器性能以及缓冲区管理等其他方面。关于主机代码优化的所有方面，本教程并不完整。


有关可用于分析应用程序性能的工具和流程的更多信息，请参阅 _SDAccel Profiling and Optimization Guide_ ([UG1207](https://www.xilinx.com/support/documentation/sw_manuals/xilinx2018_3/ug1207-sdaccel-optimization-guide.pdf)).

<hr/>
<p align="center"><sup>Copyright&copy; 2019 Xilinx</sup></p>
