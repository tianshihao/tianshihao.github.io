+++
date = '2025-03-04T16:30:00+08:00'
draft = false
title = 'CUDA C++ Best Practices Guide Notes 2'
tags = ['CUDA']
toc = true
series = ["CUDA C++ Best Practices Guide Notes"]
+++

# CUDA C++ Best Practices Guide Notes 2

> [原文地址：https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/](https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/)

## 10. **\*内存优化**

内存优化是 CUDA 程序性能优化中的最关键部分。目标是通过最大化内存带宽以最大化地使用硬件。为了最大化带宽，应尽可能使用更多的快速内存，并减少对慢速访问内存的使用。

### 10.1. Host 和 Device 之间的数据传输

1. 带宽差异的背景
   - GPU 内部显存带宽：GPU 芯片与其显存（Device Memory，即 VRAM）之间的数据传输速度非常快，例如 Tesla V100 可达 898 GB/s。
   - 主机与设备之间的带宽：CPU 主机内存（Host Memory）和 GPU 显存之间通过 PCIe 总线连接，带宽相对较低，例如 PCIe x16 Gen3 只有 16 GB/s。
   - 可以把 GPU 内部显存想象成一个高速公路，而主机与 GPU 之间的 PCIe 总线则是一条窄路。数据在 GPU 内部流动很快，但一旦需要在 CPU 和 GPU 之间搬运数据，就会成为性能瓶颈。
2. 核心建议：最小化数据传输
   - 即使某些计算在 GPU 上运行并不比在 CPU 上快（甚至可能更慢），也应该尽量让它们在 GPU 上执行，以避免频繁的数据传输开销。
   - 因为数据传输的时间成本可能远大于计算本身的时间成本。
3. 中间数据应留在设备上
   - 中间结果（临时变量、中间计算数据）应该直接在 GPU 显存中创建、使用和销毁，不要传回主机内存。
   - 只有最终需要的结果才需要传回主机。
4. 批量传输优于多次小传输
   - 每次数据传输都有固定的启动开销（overhead），比如初始化传输、发送指令等。
   - 多次小传输会累积这些开销，效率很低。
   - 应该将多个小数据打包成一个大块一次性传输，即使需要额外的打包/解包操作。
5. 使用页锁定内存（Pinned Memory）提升带宽

#### 10.1.1 页锁定内存

为了消除数据传输的开销，CUDA 提供了 Pinned Memory（页锁定内存）机制。通过将一部分主机内存固定在物理 RAM 中，防止操作系统将其作为虚拟内存换出到磁盘，从而实现 Host 和 Device 之间的高带宽数据传输。这相当于一种用户可控的内存锁定机制。

Page-locked（锁页）或 Pinned（固定）Memory 可以实现 Host 和 Device 之间的最大传输带宽。Pinned Memory 是一种特殊的 Host Memory，其物理地址被锁定，不会被操作系统交换到磁盘。Pinned Memory 物理上位于主机（Host）的 RAM 中，具有以下两个核心优势：

1. 更高的带宽：Pinned Memory 可以通过 DMA（Direct Memory Access，直接内存访问） 直接与 Device Memory 进行数据传输，省去了中间拷贝步骤，减少了内存复制开销。而普通内存在传输前需要先拷贝到临时的页锁定缓冲区。
2. 更低的延迟：由于 Pinned Memory 的物理地址固定，GPU 的 DMA 控制器可以直接访问该内存区域，无需等待额外的拷贝操作，因此数据传输的延迟更低。

总结就是通过DMA复制Pinned memory实现更高带宽和更低延迟的复制。

Pinned Memory 是通过 Runtime API 中的`cudaHostAlloc()`函数分配的。CUDA 示例程序[bandwidthTest](https://github.com/NVIDIA/cuda-samples/tree/master/Samples/1_Utilities/bandwidthTest)展示了如何使用这些函数以及如何测量内存传输性能。

对于已经预先分配的系统内存区域（Host Memory），可以使用`cudaHostRegister()`动态地将其固定住，而无需分配单独的缓冲区并将数据复制到其中。

Pinned Memory 不应被过度使用。过度使用可能会降低整体系统性能，因为 Pinned Memory 是一种稀缺资源，但很难提前知道多少才算过多。此外，与大多数常规系统内存分配相比，固定系统内存是一项开销较大的操作，因此与所有优化一样，应测试应用程序及其运行的系统以确定最佳性能参数。

#### 10.1.2. 异步传输与计算的重叠

一些背景知识，GPU 具备同时进行数据拷贝和计算的硬件能力：

- Copy Engine：负责 Host ↔ Device 数据传输
- SMs（流式多处理器）：负责执行 kernel 计算

两者是独立的硬件单元，可以并行工作。

使用`cudaMemcpyAsync()`可以异步传输数据，控制权会立即返回主机线程，从而允许数据传输与CPU计算重叠。异步的复制需要pinned meomory（否则会回退成隐式的同步拷贝，CPU 会阻塞一段时间（拷贝到临时缓冲区）），以及一个额外的参数stream id，stream就是设备上按顺序执行的一系列操作序列。不同流中的操作可以**交错执行**，在某些情况下甚至可以重叠执行。这个特性可以用来隐藏数据传输延迟、提高GPU利用率。

这里有一个例子[Asynchronous and Overlapping Transfers with Computation](https://docs.nvidia.com/cuda/cuda-c-best-practices-guide/#asynchronous-transfers-and-overlapping-transfers-with-computation)：

##### 计算和数据传输的重叠

```cpp
cudaMemcpyAsync(a_d, a_h, size, cudaMemcpyHostToDevice, 0);
Kernel<<<Grid, Block>>>(a_d);
cpuFunction();
```

例如，在执行`MemcpyAsync()`之后，控制权立即返回给 Host 线程，由于 Kernel 也使用 Default Stream，二者使用的stream相同，所以 Kernel 会在数据复制之后执行，因此，不需要显式地同步。因为 Kernel 的启动也是异步的，因此Kernel也会将控制权返回给 Host 线程，然后`cpuFunction()`开始执行，这里，CPU的计算与GPU的复制、计算重叠了。

这个例子中，数据传输和 Kernel 执行是按顺序执行的。在能够并发复制和计算的设备上，还可以将设备的 Kernel 执行与 Host 和 Device 之间的数据传输重叠。设备是否具有此功能由`cudaDeviceProp`结构的`asyncEngineCount`字段指示（或列在 DeviceQuery CUDA Sample 的输出中）。这个功能要求是 Pinned Memory 并且数据传输和 Kernel 需要使用不同的、非默认的 Stream。（之所以需要非默认的 Stream，是因为 Default Stream 是独占的，只有在先前所有 Stream 中的调用完成后，Default Stream 中的操作才能开始，第 8 章提到过。）

> 就是非默认的 Stream 之间可以并行。这个在实践中是基础，这个文档里面说得有点绕，不过也很全面。

##### 并发复制和计算

```cpp
cudaStreamCreate(&Stream1);
cudaStreamCreate(&Stream2);
cudaMemcpyAsync(a_d, a_h, size, cudaMemcpyHostToDevice, Stream1);
Kernel<<<Grid, Block, 0, Stream2>>>(otherData_d);
```

复制和计算使用了不同的stream，是并发的。Copy engine和SMs的存在提供了GPU计算和复制并行的可能。

##### 顺序复制和计算

```cpp
cudaMemcpy(a_d, a_h, N*sizeof(float), dir);
Kernel<<<N/nThreads, nThreads>>>(a_d);
```

一个大数组，使用阻塞的复制，复制和计算串行。那么如何优化呢？把数据分块，让复制和计算重叠。

即使用异步复制。

##### 分块并发复制和计算

什么意思呢？因为CUDA

```cpp
size = N * sizeof(float) / nStreams;
for (i = 0; i < nStreams; i++) {
  offset = i * N / nStreams;
  cudaMemcpyAsync(a_d + offset, a_h + offset, size, dir, Stream[i]);
  Kernel<<<N / (nThreads * nStreams), nThreads, 0, Stream[i]>>>(a_d + offset);
}
```

为什么可以通过分块加速呢？原因是GPU有独立的数据拷贝引擎（Copy Engine），因此数据的拷贝和计算（SM负责核函数的执行），在硬件层面上可以同时进行，互不阻塞。

再假设数据和计算任务可以分块处理，即数据之间互不影响，那么就可以采用多stream并发，分块传输、分块计算，这样每一块的计算与下一块的数据传输自然可以实现高度重叠。

上述代码展示了如何将传输和执行核函数分解为 $nStreams$ 个阶段。其中，假设 $N$ 能被 $nThreads \times nStreams$ 整除，即把要处理的$N$个数据，平均分给$nStreams$个 Stream，每个 Stream 要处理的数据数量为：

$$
Elements \ processed \ by \ every \ Stream = \frac{N}{nStreams}
$$

而每个 Stream 中的数据又被划分成若干 block，每个 block 包含的线程数为$n Threads$，每个线程处理一个数据，block 的数量乘以线程的数量即每个 Stream 要处理的数据数量：

$$
Elements \ processed \ by \ every \ Stream = \frac{N}{nStreams} = Block \ number \times nThreads
$$

由此 block 的数量应该为：

$$
Block \ number = \frac{\frac{N}{nStreams}}{nThreads}= \frac{N}{nThreads \times nStreams}
$$

所以上述代码中`Kernel`的`gridSize`是`N / (nThreads * nStreams)`，代表一个 grid 里面有`N / (nThreads * nStreams)`个 block。

由于每一个 Stream 内的操作是顺序执行的，因此在各自 Stream 中的数据传输完成之前，kernel 不会启动（复制和 kernel 函数使用同一个`Stream[i]`），但是不同 Stream 之间的复制和 kernel 函数执行是可以并行的，效果见下图：

![图1：复制和kernel执行的时间线比较](/timeline-comparison-for-copy-and-kernel-execution.png)
**图 1：复制和 Kernel 执行的时间线比较**

上面是顺序复制和执行，下面是并发复制和执行，可以看到并发执行的时候，第一个绿色（`Stream[0]`的复制）和第一个红色（`Stream[0]`的 kernel 函数执行）是并行的，但是第二个绿色（`Stream[1]的复制`）和第一红色是并行的，从而提高了利用率，节省了时间。

对于这个示例，假设数据传输时间和内核执行时间相当。在这种情况下，当执行时间（$tE$）超过传输时间（$tT$）时，分阶段版本的总体时间估计为 $tE + \frac{tT}{nStreams}$，而顺序版本的总体时间估计为 $tE + tT$。如果传输时间超过执行时间，则总体时间的估计为 $tT + \frac{tE}{nStreams}$。

另外，有“单个复制引擎（copy engine）” 的 GPU 可以执行一次异步数据传输并执行 kernel，而具有两个复制引擎的 GPU 可以同时执行一次从 Host 到 Device 的异步数据传输、一次从 Device 到 Host 的异步数据传输以及执行 kernel。GPU 上的复制引擎数量由`cudaDeviceProp`结构中的`asyncEngineCount`字段给出，也可以在`DeviceQuery` CUDA 示例的输出中查看。另外，在 Default Stream 上的 Blocking transfer （`cudaMemcpy()`）是不能与 computation overlap 的，还记得么？Default Stream 是独占的。

#### 10.1.3. 零拷贝

零拷贝允许 GPU 线程直接访问 Host 内存。为此，它需要映射的 Pinned Memory。如果 GPU 是集成的（即 CUDA 设备属性中`integrated`为 1），则映射的 Pinned Memory 始终会带来性能提升，因为集成 GPU 和 CPU 内存在物理上是相同的，从而避免了多余的复制。在独立的 GPU 上，映射的 Pinned Memory 应仅被读取或者写入一次，并且读取和写入该内存的全局加载和存储操作应该是合并的。零拷贝可以替代 Stream 的使用，因为 Kernel 发起的数据传输会自动与 Kernel 的执行重叠，而无需设置和确定最优 Stream 的开销。

所谓零拷贝就是可以让GPU上的线程直接访问主机内存。怎么做到呢？就是在主机内存上分配pinned memory，并且映射到gpu。映射的前提是主机内存锁定，不然换页之后白映射了。这样，GPU就可能直接访问主机内存上的数据了。

- 对于集成式GPU，因为CPU和GPU使用的是同一块物理内存，所以这么做能够节省多余的内存复制。这就是真正的零拷贝！
- 对于独立式GPU，GPU和CPU使用的是两块物理内存，映射的pinned memory只是让gpu可以远程“访问”主机内存，但是速度受到PCIe/NVLINK上限，通过比拷贝到显存再计算要慢，除非是那种只读一次、写一次，没有办法提前搬运的情况。
  - 这个和普通的pinned还不一样，普通的pinned，GPU通过DMA直接实现对主机内存更高带宽、更低延迟的数据“拷贝”，仅仅是拷贝。
  - 但是map之后，这段主机内存还被映射到了GPU的虚拟空间，相当于直接访问了，不复制，不搬运数据，走的是PCIe总线。
  - 这么看来，如果独立式GPU的情况，在主机内存有一大块数据，但是只需要修改其中的一小部分的话，mapped pinned memory（零拷贝）更合适，不用全部复制。如果还是需要大规模处理，还是用普通pinned然后复制好。
  - 对于集成式GPU，比如某一个feature，每次在CPU写入，然后模型只读取一次，显然适合pinned memory。如果写写一次，然后多次读，更适合h2d，拷贝到device里面，这样每次读最快。

零拷贝可以替代stream，因为kernel直接访问主机的内存了，访问和计算是同时发生的，没必要分配stream安排复制再计算了，直接就原地搞定了。

##### 零拷贝的使用

```cpp
float *a_h, *a_map;
...
cudaGetDeviceProperties(&prop, 0);
if (!prop.canMapHostMemory)
    exit(0);
cudaSetDeviceFlags(cudaDeviceMapHost);
cudaHostAlloc(&a_h, nBytes, cudaHostAllocMapped);
cudaHostGetDevicePointer(&a_map, a_h, 0);
Kernel<<<GridSize, BlockSize>>>(a_map);
```

上述代码中，使用`cudaGetDeviceProperties()`返回的结构中的`canMapHostMemory` 字段来检查设备是否支持将 Host 内存映射到设备的地址空间。通过调用`cudaSetDeviceFlags()`并传入`cudaDeviceMapHost`，可以启用页锁定内存映射。需要注意的是，`cudaSetDeviceFlags()`必须在设置设备或进行需要状态的 CUDA 调用之前调用（即基本上在创建上下文之前）。使用`cudaHostAlloc()`分配页锁定的映射 Host 内存，并通过`cudaHostGetDevicePointer()`函数获取映射的设备地址空间指针。在零拷贝 Host 代码中，`Kernel()`可以使用指针`a_map`引用映射的固定 Host 内存，就像`a_map`指向设备内存中的位置一样。

> 项目中零拷贝位于业务上游，业务直接使用了上游提供的地址，对于业务自身而言，不需要关心零拷贝的存在，只是换了一个地方访问复制数据。

这个如何理解？

> 映射的固定 Host 内存允许你在避免使用 CUDA 流的情况下，将 CPU-GPU 内存传输与计算重叠。但由于对此类内存区域的任何重复访问都会导致重复的 CPU-GPU 传输，因此可以考虑在 Device 内存中创建第二个区域，手动缓存之前读取的 Host 内存数据。

#### 10.1.4. 统一虚拟地址

计算能力 2.0 及更高版本的设备在 64 位 Linux 和 Windows 上支持一种称为统一虚拟地址空间（Unified Virtual Addressing, UVA）的特殊寻址模式。在 UVA 下，Host Memory 和所有已安装支持设备的内存共享一个单一的虚拟地址空间。

在 UVA 之前，应用程序必须跟踪哪些指针指向 Device Memory（以及哪个设备），哪些指针指向 Host Memory，这需要为每个指针单独存储元数据（或在程序中硬编码信息）。而在 UVA 下，通过使用`cudaPointerGetAttributes()`检查指针的值，可以简单地确定指针指向的物理内存空间。

在 UVA 下，使用`cudaHostAlloc()`分配的固定 Host Memory 将具有相同的 Host 和 Device 指针，因此无需为此类分配调用`cudaHostGetDevicePointer()`。然而，通过`cudaHostRegister()`事后固定的 Host Memory 分配将继续具有与 Host 指针不同的 Device 指针，因此在这种情况下仍然需要调用`cudaHostGetDevicePointer()`。

UVA 还是在支持的配置中启用点对点（Peer-to-Peer, P2P）数据传输的必要前提条件，P2P 允许数据直接通过 PCIe 总线或 NVLink 在支持的 GPU 之间传输，而无需经过 Host Memory。

有关 UVA 和 P2P 的进一步解释和软件要求，请参阅《CUDA C++ 编程指南》。

### 10.2. Device 内存空间

![图2：CUDA上的内存空间](/memory-spaces-on-cuda-device.png)
**图 2：CUDA 上的内存空间**

CUDA 设备使用多个内存空间，这些内存空间具有不同的特性，反映了它们在 CUDA 应用程序中的不同用途。

在这些不同的存储空间中，global Memory 是最丰富的；有关每个计算能力级别的每个存储空间中可用的内存量，请参阅《CUDA C++编程指南》的功能和技术规格。global、local 和 texture 内存的访问延迟最大，其次是 constant Memory、shared Memory 和 regiter file。

| Memory   | Location on/off chip | Cached  | Access | Scope                | Lifetime        |
| -------- | -------------------- | ------- | ------ | -------------------- | --------------- |
| Register | On                   | n/a     | R/W    | 1 thread             | Thread          |
| Local    | Off                  | Yes[^1] | R/W    | 1 thread             | Thread          |
| Shared   | On                   | n/a     | R/W    | All threads in Block | Block           |
| Gloal    | Off                  | [^2]    | R/W    | All threads + Host   | Host allocation |
| Constant | Off                  | Yes     | R      | All threads + Host   | Host allocation |
| Texture  | Off                  | Yes     | R      | All threads + Host   | Host allocation |

[^1]: 默认情况下缓存在 L1 和 L2 中，计算能力为 5.x 的设备除外；计算能力为 5.x 的设备仅在 L2 中缓存本地。
[^2]: 默认情况下，在计算能力为 6.0 和 7.x 的设备上缓存在 L1 和 L2 中；默认情况下，在计算能力较低的设备上，仅在 L2 中缓存，尽管有些设备也允许通过编译标志选择加入 L1 中的缓存。

在纹理访问的情况下，如果纹理引用绑定到全局内存中的线性数组，则设备代码可以写入底层数组。绑定到 CUDA 阵列的纹理引用可以通过将曲面绑定到相同的底层 CUDA 阵列存储来通过曲面写入操作写入）。应避免在同一内核启动中写入纹理的底层全局内存数组时读取纹理，因为纹理缓存是只读的，在修改相关全局内存时不会失效。

#### 10.2.1. 对全局内存的合并访问

在为支持 CUDA 的 GPU 架构编程时，一个非常重要的性能考虑因素是全局内存访问的合并（Coalescing）。设备会将一个 Warp 中线程的全局内存加载和存储操作合并为尽可能少的事务。

> 确保全局内存访问尽可能合并。

对于计算能力大于等于 6.0 的设备，一句话概括：“一个 Warp 中线程（一般为 32 个）的并发访问将合并成一系列的事务，一个事务访问 32 个字节，事务的数量足以服务该 Warp 中的所有线程。”

对于某些设备计算能力 5.2 的能力的设备，可以选择启用全局内存访问的 L1 缓存。如果在这些设备上启用了 L1 缓存，所需的事务数量等于所于的 128 字节对齐段的数量。对于计算能力大于等于 6.0 的设备，L1 缓存是默认启用的。然而，无论全局加载是否缓存在 L1 中，数据访问单元但是 32 字节。

在配备了 GDDR 内存的设备上，当 ECC（纠错码）启用时，以合并方式访问内存变得更加重要。分散的访问会增加 ECC 内存传输的开销，尤其在向全局内存写入数据时。

##### 10.2.1.1. 最简单的访问模式

在计算能力大于等于 6.0 的设备上最简单的合并可以这样实现：一个 Warp 中的第 k 个线程访问 以 32 字节对齐的数组中的第 k 个字。不需要 Warp 中的所有线程都访问。（就是以 Warp 为单位访问，集体行动）

例如，如果一个 Warp 中的线程访问相邻的 4 字节的字（例如相邻的 `float` 值，4 字节），那么就需要 4 次 32 字节对齐的、合并的事务。因为一个 Warp 中共 32 个线程，所以对于整个 Warp 而言，实际要访问一段 128 字节的内存，图 3 展示了这种访问模式。

![图3：合并访问](/coalesced-access.png)
**图 3：合并访问**

如果从 4 个 32 字节段中的任何一个段中只请求了部分字（例如，如果多个线程访问了同一个字，或者某些线程没有参与访问），仍然会获取整个段。此外，如果 Warp 中的线程的访问在 4 个段内或跨段进行了重新排列，对于计算能力 6.0 或更高的设备，仍然只会执行 4 个 32 字节事务。

##### 10.2.1.2 顺序但是非对齐的访问模式

如果 Warp 中的线程顺序访问内存，但内存地址没有和 32 字节段对齐，则会请求 5 个 32 字节段。

![图4：非对齐的顺序地址，跨越5个32字节段](/misaligned-sequential-addresses.png)
**图 4：非对齐的顺序地址，跨越 5 个 32 字节段**

最右边的箭头表示访问的地址未对齐 32 字节。

通过 CUDA Runtime API（例如`cudaMalloc()`）分配的内存保证至少对齐到 256 字节。因此，选择合理的线程块大小（例如 Warp 大小的倍数，即当前 GPU 上的 32），有助于确保 Warp 的内存访问正确对齐。（例如，考虑如果线程块大小不是 Warp 大小的倍数，第二个、第三个及后续线程块访问的内存地址会发生什么情况。）

##### 10.2.1.3. 未对齐访问的影响

使用一个简单的复制 Kernel 为例：

###### 展示未对齐访问的复制 Kernel

```cpp
__global__ void offsetCopy(float *odata, float* idata, int offset)
{
    int xid = BlockIdx.x * BlockDim.x + threadIdx.x + offset;
    odata[xid] = idata[xid];
}
```

在上述代码中，数据从输入数据`idata`复制到输出数组`odata`，两者都存在于 global meomry 中。kernel 在 Host 代码中的一个循环中执行，循环中参数`offset`从 0 变化到 32，以 NVIDIA Tesla V100 为例，不同偏移量下复制的有效带宽如下：

![图5：offsetCopy的性能](/performance-of-offsetcopy-kernel.png)
**图 5：offsetCopy 的性能**

即没有偏移量或偏移量为 8 的倍数的全局内存访问会导致 4 个 32 字节事务，带宽约为 790GB/s。否则，每个 Warp 会加载 5 个 32 字节事务，带宽约为无偏移量的$\frac{4}{5}$。

即未对齐访问会增加内存事务的数量，代之带宽浪费和性能下降。

##### 10.2.1.4. 跨步访问

在上面的例子中，顺序但是未对齐的访问情况下，使用缓存有助于提高性能。然而，对于非单位步长访问，情况可能不同，而这种情况在处理多维数据或者矩阵时很常见。下面是展示非单位步长数据复制的代码，该 Kernel 函数以步长`stride`为单位将数据从`idata`复制到`odata`。

###### 展示非单位步长数据复制的 Kernel

```cpp
__global__ void strideCopy(float *odata, float* idata, int stride)
{
    int xid = (BlockIdx.x*BlockDim.x + threadIdx.x)*stride;
    odata[xid] = idata[xid];
}
```

图 6 展示了这种情况，本例中，Warp 中的线程以步长 2 访问内存。这种行为导致在 Tesla V100 上每个 Warp 加载 8 个 L2 缓存段。

![图6：相邻线程以2为步长访问内存](/adjacent-threads-accessing-memory-with-stride-of-2.png)
**图 6：相邻线程以 2 为步长访问内存**

步长为 2 导致 50%的加载/存储效率，因为事务中的一半元素没有被使用，代表了带宽的浪费。随着步长的增加，有效带宽会进一步降低，直到每个 Warp 中的 32 个线程都需要加载 32 个 32 字节段。

![图7：非单位步长数据复制的性能](/performance-of-stridecopy-kernel.png)
**图 7：非单位步长数据复制的性能**

如图 7 中所示，应尽可能避免非单位步长的全局内存访问。实现这一点的一种方法是利用共享内存，这将在下一节中讨论。

#### 10.2.2. L2 缓存

CUDA 11.0 之后设备可以影响 L2 缓存。因为 L2 是 On-Chip 的，所以它能提供更高的带宽和更低的延迟来访问全局内存。

##### 10.2.2.1. L2 缓存访问窗口

当一个 CUDA Kernel 不断访问全局内存的同一个区域时，这些数据可以被认为是持久的（Persisting）。而如果数据只被访问了一次，那么这些数据就是流式的（Streaming）。L2 缓存的一部分区域可以被预留出来（Set-aside）用于对全局内存中的数据进行持久访问。如果这部分被留出来的区域没有被使用，其它的访问仍然能够正常的使用它。

> 类似 L2 中的 Pinned Memory！

用于持久访问的 L2 缓存预留大小可以在限制范围内进行调整：

```cpp
cudaGetDeviceProperties(&prop, Device_id);
cudaDeviceSetLimit(cudaLimitPersistingL2CacheSize, prop.persistingL2CacheMaxSize); /* Set aside max possible size of L2 cache for persisting accesses */
```

用户数据到 L2 预留部分的映射可以使用 CUDA Stream 或 CUDA graph Kernel 节点上的访问策略窗口进行控制。下面的示例显示了如何在 CUDA 流上使用访问策略窗口。

```cpp
// Stream level attributes data structure
cudaStreamAttrValue Stream_attribute;
// Global Memory data pointer
Stream_attribute.accessPolicyWindow.base_ptr = reinterpret_cast<void*>(ptr);
// Number of bytes for persisting accesses.
// (Must be less than cudaDeviceProp::accessPolicyMaxWindowSize)
Stream_attribute.accessPolicyWindow.num_bytes = num_bytes;

// Hint for L2 cache hit ratio for persisting accesses in the num_bytes region
Stream_attribute.accessPolicyWindow.hitRatio = 1.0;
// Type of access property on cache hit
Stream_attribute.accessPolicyWindow.hitProp = cudaAccessPropertyPersisting;
// Type of access property on cache miss.
Stream_attribute.accessPolicyWindow.missProp = cudaAccessPropertyStreaming;

// Set the attributes to a CUDA Stream of type cudaStream_t
cudaStreamSetAttribute(Stream, cudaStreamAttributeAccessPolicyWindow,
                        &Stream_attribute);
```

访问策略窗口需要`hitRaio`和`num_bytes`的值。根据`num_bytes`参数的值和 L2 缓存的大小，可能需要调整`hitRaio`的值，以避免 L2 缓存的抖动。

> 项目中倒是没用到这个。

##### 10.2.2.2. 微调访问窗口的命中率

`hitRatio`参数可以用来指定接收`hitProp`属性的访问比例。如果`hitRatio`的值是 0.6，那么全局内存区域[ptr..ptr_num_bytes)中的 60%的内存访问具有持久化属性，40%的内存访问具有流式属性。下面用一个 benchmark 来说明。

假设我们使用 GPU 全局内存中大小为 1024 MB 的区域。首先，我们使用`cudaDeviceSetLimit()`把 L2 缓存中的 30 MB 预留给持久化访问。然后，如下图所示，我们指定对于内存区域前`freqSize * sizeof(int)`字节的访问是持久化的。因此，这些数据将使用 L2 缓存的预留部分。在我们的实验当中，我们将 1024 MB 全局内存中持久化数据区域的大小从 10 MB 调整到 60 MB，以模拟数据适合或超出可用 L2 预留部分（本例中固定为 30 MB）的各种场景。注意，NVIDIA Tesla A100 GPU 的 L2 缓存总容量是 40 MB。对剩余内存区域剩余数据（即流式数据）的访问仍将被视为正常或流式访问，因此，将使用剩余的 10 MB 非预留部分。

![图8：在滑动窗口试验中将持久化数据访问映射到L2的预留部分](/sliding-window-l2.png)
**图 8：在滑动窗口试验中将持久化数据访问映射到 L2 的预留部分**

下面是上述 benchmark 代码的实现：

```cpp
__global__ void Kernel(int* data_persistent, int* data_Streaming, int dataSize,
                       int freqSize) {
  int tid = BlockIdx.x * BlockDim.x + threadIdx.x;

  /*Each CUDA thread accesses one element in the persistent data section
    and one element in the Streaming data section.
    Because the size of the persistent Memory region (freqSize * sizeof(int)
    bytes) is much smaller than the size of the Streaming Memory region
    (dataSize * sizeof(int) bytes), data in the persistent region is accessed
    more frequently*/

  data_persistent[tid % freqSize] = 2 * data_persistent[tid % freqSize];
  data_Streaming[tid % dataSize] = 2 * data_Streaming[tid % dataSize];
}

Stream_attribute.accessPolicyWindow.base_ptr =
    reinterpret_cast<void*>(data_persistent);
Stream_attribute.accessPolicyWindow.num_bytes =
    freqSize *
    sizeof(int);  // Number of bytes for persisting accesses in range 10-60 MB
Stream_attribute.accessPolicyWindow.hitRatio =
    1.0;  // Hint for cache hit ratio. Fixed value 1.0
```

上述 Kernel 代码的性能如下图所示。当持久化数据区域的大小正好等于 L2 缓存的 30 MB 预留部分时，就可以观察到性能提升了 50%。然而，一旦持久化数据区域的大小超过了 L2 缓存的 30 MB 预留部分，就有出现因为抖动导致的 10%左右的性能下降。

![图9：hit-ratio固定为1的滑动窗口benchmark的性能表现](/l2-hitratio-before.png)
**图 9：hit-ratio 固定为 1 的滑动窗口 benchmark 的性能**

为了更加优化性能，当持久化数据区域的大小超过 L2 缓存预留部分的大小时，我们把`num_bytes`和`hitRatio`的值做如下调整。

```cpp
Stream_attribute.accessPolicyWindow.base_ptr =
    reinterpret_cast<void*>(data_persistent);
Stream_attribute.accessPolicyWindow.num_bytes = 20 * 1024 * 1024;  // 20 MB
Stream_attribute.accessPolicyWindow.hitRatio =
    (20 * 1024 * 1024) /
    ((float)freqSize *
     sizeof(int));  // Such that up to 20MB of data is resident.
```

我们把访问窗口中的`num_bytes`固定为 20 MB，然后调整`hitRatio`，使总的持久化数据的随机 20 MB 驻留在 L2 的预留部分。而剩余的持久化数据将使用流式访问。这样做可以减少抖动。结果如下图所示，无论持久化数据是否适合预留的 L2，我们都可以看到良好的性能表现。

![图10：调整过hit-ratio的滑动窗口benchmark的性能表现](/l2-hitratio-after.png)
**图 10：调整过 hit-ratio 的滑动窗口 benchmark 的性能表现**

> 总结：通过调整`hitRatio`的值，可以优化持久化数据访问的性能，避免 L2 缓存的抖动。

#### 10.2.3. 共享内存（Shared Memory）

由于共享内存是 On-Chip 的，因此相比于全局内存，共享内存有更高的带宽和更低的延迟，前提是线程之间没有 Bank Conflict。

##### 10.2.3.1. 共享内存和 Meomry Banks

为了实现高带宽的并发访问，共享内存被划分成了多个大小相同的 Memory Bank，这些 Bank 可以被同时访问。因此，任何跨越 n 个不同 Bank 的 n 个地址的内存加载或者存储操作都可以同时进行，从而使得有效带宽是单个 Bank 的 n 倍。

然而，如果一个内存请求的多个地址映射到了同一个 Memory bank，这些访问仍将是串行的。硬件会把有 Bank 冲突的内存访问拆分成多个无冲突的独立请求，从而将有效带宽降低为拆分后的请求数量的倒数。唯一的例外是当一个 Warp 中的多个线程访问同一个共享内存地址时，会触发广播机制。在这种情况下，来自不同 Bank 的多个广播会被合并为从请求的共享内存位置到线程的单次多播。

为了最小化 Bank 冲突，理解内存地址如何映射到 Bank 以及如何优化调度内存请求非常重要。

在计算能力大于等于 5.x 的设备上，每一个 Bank 在每个时钟周期的带宽为 32 bit，连续的 32 bit 字被分配到连续的 Bank 中。Warp 中有 32 个线程，bank 的数量也是 32 个，因此，Warp 中的任何线程之间都可能发生 Bank 冲突。

##### 10.2.3.2. 矩阵乘法（$C=AB$）中的共享内存

共享内存使得 Block 中的线程可以合作。当 Block 中的多个线程使用来自全局内存中的相同数据时，共享内存可以用于从全局内存中仅访问一次数据。共享内存还可以通过以合并模式从全局内存加载和存储数据，然后在共享内存中重新排序，来避免非合并的内存访问。除了 Bank 冲突外，Warp 中的线程在共享内存从的非顺序或未对齐访问不会带来性能上的损失。

下面通过矩阵乘法$C=AB$来展示共享内存的使用，其中$A$的维度是$M \times w$，$B$的维度是$w \times N$，$C$的维度是$M \times N$。其中，$w$的值是 32，$M$和$N$都是 32 的倍数，因为每个 Warp 中有 32 个线程，这样可以保证例子中的 Kernel 足够简单。

这个问题中，可以将矩阵分解为$w \times w$的 Block。这样从$w\times w$的 Block 的角度来看，这个这个问题就会被简化为一个列矩阵$A$和一个行矩阵$B$的乘法（$M$和$N$是$w$的倍数），而$C$是它们的外积。如图 11 所示。而完成整个计算任务，则需要$(N/w) \times (M/w)$个 Block，即 Grid 的大小是$(N/w) \times (M/w)$。（矩阵的分解需要一点点线代知识）

![图11：块列矩阵$A$乘以块行矩阵$B$，结果是矩阵$C$](/matrix-multiplication-block-column-by-block-row.png)
**图 11：块列矩阵$A$乘以块行矩阵$B$，结果是矩阵$C$**

为此，`simpleMultiply`Kernel 的实现如下：

###### 未优化的矩阵乘法

```cpp
__global__ void simpleMultiply(float* a, float* b, float* c, int N) {
  int row = BlockIdx.y * BlockDim.y + threadIdx.y;
  int col = BlockIdx.x * BlockDim.x + threadIdx.x;
  float sum = 0.0f;
  for (int i = 0; i < TILE_DIM; i++) {
    sum += a[row * TILE_DIM + i] * b[i * N + col];
  }
  c[row * N + col] = sum;
}
```

上述代码中，`a`、`b` 和 `c` 分别指向矩阵$A$、$B$和$C$的全局内存指针。`BlockDim.x`、`BlockDim.y`和`TILE_DIM`都等于$w$。$w\times w$的 Block 中每个线程计算$C$中一个子矩阵中的一个元素。`row`和`col`是由特定线程计算的$C$中元素的行和列。`for`循环遍历`i`，将$A$的一行与$B$的一列相乘，然后将结果写入$C$。

> 上面示例的代码有点问题，应该是`sum += a[row * A_Cols + i] * b[i * B_Cols + col];`。只不过在这个例子里面 A 的列数恰好就是`TILE_DIM`，所以没有问题。
>
> “子矩阵”原文是“tile”，直译是“瓦片”。矩阵就好比房顶，被一片片瓦片覆盖，相当于矩阵被分割成了很多小块，每一块就是一个“tile”。我忘了线性代数里面有没有这个概念了，就先叫子矩阵吧。

这个 Kernel 在 NVIDIA Tesla V100 上的有效带宽是 119.9 GB/s。为了分析性能，我们必须知道在`for`循环里面 Warp 如何访问全局内存。每一个 Warp 各自计算$C$中子矩阵的一行，而这需要访问$A$子矩阵的一行和$B$的整个子矩阵。图 12 展示了需要访问的数据。

![图12：计算子矩阵的一行。使用$A$的一行和$B$的整个子矩阵计算$C$中子矩阵的一行](/computing-row-of-tile.png)
**图 12：计算子矩阵的一行。使用$A$的一行和$B$的整个子矩阵计算$C$中子矩阵的一行**

对于`for`循环的每一次迭代$i$，Warp 中的线程会读取$B$子矩阵中的一行，这种访问方式在所有的计算能力下都是顺序且合并的。

然而，对于每次迭代$i$，Warp 中的线程会从全局内存中读取矩阵$A$的同样的值，这是因为索引`row * TILE_DIM + i`对于 Warp 中的所有线程来说是相同的（`i`从 0 取到 31，而事务一次访问 32 字节，所以对于矩阵$A$来说，无论`i`取什么值，访问的内存都是一样的。且本例为了简单，矩阵$A$、$B$的维度都是 32 的倍数，所以可以假定所有数据都是 32 字节对齐的[^3]）。尽管这种访问在计算能力 2.0 或者更高的设备上仅需要一次事务，但事务中存在带宽浪费，因为在 32 字节缓存行中的 8 个字中，仅使用了 1 个 4 字节字（对于每次迭代$i$，只计算一次，也就是只使用读取到的$A$的第`row`行数据中的第`i`个）。

为了避免这种浪费，我们可以在循环后续的迭代中重复使用这一 32 字节缓存行，并最终使用其中的全部数据。然而，一般情况下，当许多 Warp 在同一 Multiprocessor 上同时执行时，缓存行可能会在循环的第$i$和$i+1$之间被换出缓存。因此在第$i+1$次迭代中，Warp 中的线程就会因为缓存未命中而不得不重新访问全局内存，结果还是浪费了带宽。

解决的办法就是共享内存。手动把数据读取到片上的[Shared Memory](#92-device-内存空间)进行管理。

###### 使用共享内存提高矩阵乘法中加载全局内存的效率

```cpp
__global__ void coalescedMultiply(float* a, float* b, float* c, int N) {
  __shared__ float aTile[TILE_DIM][TILE_DIM];

  int row = BlockIdx.y * BlockDim.y + threadIdx.y;
  int col = BlockIdx.x * BlockDim.x + threadIdx.x;
  float sum = 0.0f;
  aTile[threadIdx.y][threadIdx.x] = a[row * TILE_DIM + threadIdx.x];
  __syncWarp();
  for (int i = 0; i < TILE_DIM; i++) {
    sum += aTile[threadIdx.y][i] * b[i * N + col];
  }
  c[row * N + col] = sum;
}
```

在上述代码中，矩阵$A$中的每一个元素只从全局内存中读取一次，并以一种完全合并的方式（没有带宽浪费）加载到共享内存中。在`for`循环的每次迭代中，共享内存中的值可以通过广播的方式共享给 Warp 中的每一个线程。

使用`__syncWarp()`同步屏蔽（Barrier）调用，因为只有 Warp 内写入数据到共享内存中的线程才会读取这些数据。在 NVIDIA Tesla V100 上，这个 Kernel 的有效带宽为 144 GB/s。

> 1. `__syncWarp()` 用于同步一个 Warp 中的线程，它可以保证一个 Warp 中的线程都执行完`__syncWarp()`前面的操作，即在 Warp 中所有的线程都到达`__syncWarp()`之前，已经到达的线程会被阻塞。在所有的线程都到达`__syncWarp()`之后，所有的线程才会继续执行后面的代码。线程集结点，集解之后，发起冲锋。
> 2. 而`__syncthreads()`会同步 Block 中的所有线程，作用范围更大，开销也更大。两者使用时都需要注意，不要把它们放到分支语句中，否则会导致死锁。

这说明当硬件 L1 缓存的替换策略与应用的需求不匹配，或者 L1 缓存未用于从全局内存读取时，共享内存可以作为“用户管理的缓存”来提高性能。

我们假设上述代码的上下文中，Block 大小是 32 * 32，那么`aTile`实际上缓存的就是一个 Block 的线程会访问的矩阵$A$的元素。`threadIdx.y`和`threadIdx.x`分别代表在 Block 中 Thread 的纵横坐标，`row *TILE_DIM + threadIdx.x`即该线程需要读取的矩阵$A$的元素的全局内存地址。有了这些信息之后，就可以把 Thread 要读取的数据提前读取到共享内存当中了。

而`__syncWarp()`的作用范围只是 Warp，Block 的大小是 32 \* 32，而一个 Warp 中有 32 个线程，因此，一旦 Warp 中的所有线程在共享内存中准备好了资源，它们就可以进行对应部分的计算。问题，一个 Warp 中的线程是否对应 Block 中的一行呢？

更深入的优化是把矩阵$B$的子矩阵也用共享内存缓存起来。

###### 通过将额外数据读入共享内存进行改进

```cpp
__global__ void sharedABMultiply(float* a, float* b, float* c, int N) {
  __shared__ float aTile[TILE_DIM][TILE_DIM], bTile[TILE_DIM][TILE_DIM];
  int row = blockIdx.y * blockDim.y + threadIdx.y;
  int col = blockIdx.x * blockDim.x + threadIdx.x;
  float sum = 0.0f;
  aTile[threadIdx.y][threadIdx.x] = a[row * TILE_DIM + threadIdx.x];
  bTile[threadIdx.y][threadIdx.x] = b[threadIdx.y * N + col];
  __syncthreads();
  for (int i = 0; i < TILE_DIM; i++) {
    sum += aTile[threadIdx.y][i] * bTile[i][threadIdx.x];
  }
  c[row * N + col] = sum;
}
```

这样做，$A$只需要读取$B\_ cols / TILE\_ DIM$次全局内存访问，而不是$B\_ cols$次。$B$只需要读取$A\_ rows / TILE\_ DIM$次全局内存，而不是$A\_ rows$次。

有一点不同，这里使用的是`__syncthreads()`，而不是`__syncWarp()`。这是因为在计算$C$的时候，会用到$B$的整个子矩阵，也就是说，一个 Warp 的线程会从共享内存中读取由不同的 Warp 写入的数据。因此需要使用`__syncthreads()`来同步 Block 中的所有线程。下面是 3 种矩阵乘法的性能对比：

| Optimization                                                    | NVIDIA Tesla V100 | NVIDIA 3080 （自测） |
| --------------------------------------------------------------- | ----------------- | -------------------- |
| No optimization                                                 | 119.9 GB/s        |                      |
| Coalesced using shared memory to store a tile of A              | 144.4 GB/s        | 77.4 GB/s            |
| Using shared memory to eliminate redundant reads of a tile of B | 195.5 GB/s        |                      |

##### 10.2.3.3. 矩阵乘法（$C=AA^T$）中的共享内存

另一个可以是说明全局内存的跨步访问和共享内存的 Bank 冲突的例子是用$A$的转置替换$B$得到的矩阵乘法$C=AA^T$。

###### 未优化的全局内存跨步访问方式

```cpp
__global__ void simpleMultiply(float *a, float *c, int M)
{
    int row = blockIdx.y * blockDim.y + threadIdx.y;
    int col = blockIdx.x * blockDim.x + threadIdx.x;
    float sum = 0.0f;
    for (int i = 0; i < TILE_DIM; i++) {
        sum += a[row*TILE_DIM+i] * a[col*TILE_DIM+i];
    }
    c[row*M+col] = sum;
}
```

在这个转置的例子中，每一次迭代`i`，`a[col * TILE_DIM + i]`中的`col`表示$A^T$连续的列，因此，`col * TILE_DIM`表示以步长`TILE_DIM`访问全局内存，导致带宽的浪费。

[^3]: 但例子中不是 float 么？






