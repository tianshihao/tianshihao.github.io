+++
date = '2024-12-23T16:30:00+08:00'
draft = true
title = 'CUDA Graph'
tags = ['CUDA']
toc = true
+++

# CUDA Graph

## 为什么用 CUDA graph

### 多次启动 kernel 带来的性能下降

cuda 模型的典型模式下，会涉及很多迭代，每个步骤中有多个操作，如果将每个操作都单独提交 GPU 并启动计算，那么提交启动的开销汇总在一起可能导致明显性能下降。

### Cuda graph 减少 kernel 提交启动的开销

CUDA Graphs 将整个计算流程定义为一个图而不是单个操作列表。最后通过提供一种由单个 CPU 操作来启动图上的多个 GPU 操作的方式来减少 kernel 提交启动的开销，进而解决以上问题。

![计算图](/computation_graph.png)
图 1 计算图

一个操作在图中形成一个节点。 操作之间的依赖关系是边。 这些依赖关系限制了操作的执行顺序。

一个操作可以在它所依赖的节点完成后随时调度。 调度由 CUDA 系统决定。

CUDA graph 简介：

- 是一系列操作的图
- 图一次定义，多次重复加载。如在一个循环中使用图，往往第一次是创建图，会稍微慢些，后边的循环会利用创建好的图进行加速，带来加速收益。
- 一个 stream 对应一个 graph。

如下图所示，**利用 cuda graph 减少多次 launch 加载的开销，可以带来一定收益**。

![使用 cuda graph 带来的加速效果](/cuda_graph_acceleration.png)
图 2 使用 cuda graph 带来的加速效果

总体来说，使用 graph 的原因如下：

1. 通过一个 CPU 操作启动多个 GPU kernel 操作来减少重复加载。
2. 提升 CPU 和 GPU 的利用率。

## CUDA graph 怎么使用

1. 创建 cuda graph。

   1. 方法 1: graph API
   2. 方法 2:捕获 stream

2. 初始化 cuda graph
   `cudaGraphInstantiate(&instance, graph, NULL, NULL, 0);`
3. Launch CUDA Graph
   `cudaGraphLaunch(instance, stream);`

### 利用 API 创建图

```cpp
// 创建一个图
cudaGraphCreate(&graph, 0);

// 分别创建 cuda kernel 节点，支持的节点可以参考下方。之后再指定每个节点依赖的节点。
// 依赖节点的指定，也可以在创建节点的时候事先指定
cudaGraphAddKernelNode(&a, graph, NULL, 0, &nodeParams);
cudaGraphAddKernelNode(&b, graph, NULL, 0, &nodeParams);
cudaGraphAddKernelNode(&c, graph, NULL, 0, &nodeParams);
cudaGraphAddKernelNode(&d, graph, NULL, 0, &nodeParams);
// 设置节点间的互相依赖
cudaGraphAddDependencies(graph, &a, &b, 1); // A->B
cudaGraphAddDependencies(graph, &a, &c, 1); // A->C
cudaGraphAddDependencies(graph, &b, &d, 1); // B->D
cudaGraphAddDependencies(graph, &c, &d, 1); // C->D
```

### 使用流捕获创建图

流捕获提供了一种从现有的基于流的 API 创建图的机制。 将工作启动到流中的一段代码，包括现有代码，可以等同于用 `cudaStreamBeginCapture()` 和 `cudaStreamEndCapture()` 的调用。
在动态感知的工程代码里也多采用该种方式创建图，然后在代码中捕获图。

```cpp
cudaGraph_t graph;

cudaStreamBeginCapture(stream);

kernel_A<<< ..., stream >>>(...);
kernel_B<<< ..., stream >>>(...);
libraryCall(stream);
kernel_C<<< ..., stream >>>(...);

cudaStreamEndCapture(stream, &graph);
```

对 `cudaStreamBeginCapture()` 的调用，将流置于捕获模式。 捕获流时，启动到流中的工作不会排队执行。 相反，它被附加到正在逐步构建的内部图中。 然后通过调用 `cudaStreamEndCapture()` 返回此图，这也结束了流的捕获模式。 由流捕获主动构建的图称为捕获图(capture graph)。
流捕获可用于除 `cudaStreamLegacy`（“NULL 流”）之外的任何 CUDA 流。

### CUDA Graph 的 node 类型

支持的 node 类型有：

- kernel 函数
- CPU 函数 call
- 内存拷贝
- Memset
- 空节点
- event 等待
- Record event
- 等待外部信号
- 子图：执行一个分离的图

## CUDA graph 测试

### 官方示例

如下一段 cuda 代码，在 2 重循环中多次启动 kernel 函数，但是每次 kernel 启动完都需要同步线程，只有前一个 kernel 执行完才执行下一个 kernel。

```cpp
#define NSTEP 1000
#define NKERNEL 20

// start CPU wallclock timer
for(int istep=0; istep<NSTEP; istep++){
  for(int ikrnl=0; ikrnl<NKERNEL; ikrnl++){
    shortKernel<<<blocks, threads, 0, stream>>>(out_d, in_d);
    cudaStreamSynchronize(stream); // 阻塞 host，等待 device 中该 stream 的所有线程执行完成
}
}
//end CPU wallclock time
```

用 nsys 导出上方代码的执行细节，可以看出 kernel 函数是顺序执行的，并且每个 kernel 会由 cudaStreamSynchronize 阻塞等待，每两个 kernel 函数的执行间有很大 gap。

![CUDA Graph Nsys](/cuda_graph_nsys.png)
图 3 CUDA Graph Nsys

#### 改进版

改进上方的代码，将 `cudaStreamSynchronize()` 同步操作放到内层循环外部，每两个 kernel 间的 gap 缩小，有一定加速效果。

```cpp
// start wallclock timer
for(int istep=0; istep<NSTEP; istep++){
  for(int ikrnl=0; ikrnl<NKERNEL; ikrnl++){
    shortKernel<<<blocks, threads, 0, stream>>>(out_d, in_d);
  }
  cudaStreamSynchronize(stream);
}
//end wallclock timer
```

![改进版 CUDA Graph Nsys](/cuda_graph_nsys_v2.png)
图 4 改进版 CUDA Graph Nsys

#### 使用 cuda graph 来进行加速

典型的使用 cuda graph 的是一系列 kernel 函数连续执行，在一个循环中先创建 graph 相关的变量，再捕获 stream，将该 stream 中的 kernel 函数加入到一个 graph，然后统一进行 launch。

下方 cudaGraph_t 类型的 graph 包含了图的结构和图的内容信息；`cudaGraphExec_t` 类型的 instance 是可执行的图，表示可以以同样方式 launch 和执行的图。

```cpp
bool graphCreated=false;
cudaGraph_t graph;
cudaGraphExec_t instance;
for(int istep=0; istep<NSTEP; istep++){
  if(!graphCreated){
    cudaStreamBeginCapture(stream, cudaStreamCaptureModeGlobal);
    for(int ikrnl=0; ikrnl<NKERNEL; ikrnl++){
      shortKernel<<<blocks, threads, 0, stream>>>(out_d, in_d);
    }
    cudaStreamEndCapture(stream, &graph);
    cudaGraphInstantiate(&instance, graph, NULL, NULL, 0);
    graphCreated=true;
  }
  cudaGraphLaunch(instance, stream);
  cudaStreamSynchronize(stream);
}
```

首先定义出一个 graph，抓取在 `cudaStreamBeginCapture` 和 `cudaStreamEndCapture` 之间的已经提交在 stream 上的 GPU 上活动的信息来实现创建。然后，我们必须通过 `cudaGraphInstantiate` 调用实例化 graph，它创建并预初始化所有内核工作描述符，以便它们可以尽可能快地重复启动。

重要的是，只需要捕获和实例化一次（在第一次循环时），并在所有后续循环中重复使用相同的实例。

后续计算时，第一次循环启动并实例化 graph；剩余的循环中启动 graph 并执行。

#### 自测

根据以上代码，编写测试代码，测试使用 cuda graph 前后的区别。在使用 graph 前后的时延差别较大。
环境：

- RTX3080
- CUDA：11.7
  This content is only supported in a Feishu Docs
  main.cu

|         | 两层循环次数 | Block thread | with graph | no graph |
| ------- | ------------ | ------------ | ---------- | -------- |
| latency | [1000,20]    | 128ms        | 50ms       | 70ms     |

## Reference

CUDA graph Link：https://developer.nvidia.com/blog/cuda-graphs/
CUDA graph doc: TSE Training\_ TensorRT CUDA Graph.pdf
CUDA graph Sample：https://github.com/NVIDIA/cuda-samples/tree/master/Samples/3_CUDA_Features/simpleCudaGraphs
CUDA graph Pytorch Accelarate：https://pytorch.org/blog/accelerating-pytorch-with-cuda-graphs/

## main.cu

```cpp
#include "cuda_runtime.h"
#include "device_launch_parameters.h"
// #include "call_imple.cu"
#include <iostream>
#include <chrono>

#define NSTEP 1000
#define NKERNEL 20
#define N 5000
#define NUM_THREAD 128
#define USE_CUDA_GRAPH 0

using default_high_resolution_clock = std::chrono::time_point<std::chrono::high_resolution_clock>;

void get_latency(default_high_resolution_clock begin){
    auto end = std::chrono::high_resolution_clock::now();
    auto elapsed = std::chrono::duration_cast<std::chrono::milliseconds>(end - begin).count();
    printf("Time measured: %d ms.\n", elapsed);
}

__global__ void shortKernel(float * out_d, float * in_d){
  int idx=blockIdx.x*blockDim.x+threadIdx.x;
  if(idx<N) out_d[idx]=1.23*in_d[idx];
}


int main(){
  float *h_a = new float[N];
  for (size_t i = 0; i < N; i++)
  {
    h_a[i] = float(i/100);
  }
  float *in_d = nullptr;
  cudaMalloc((void**)&in_d, N*sizeof(float));
  float *out_d = nullptr;
  cudaMalloc((void**)&out_d, N*sizeof(float));
  cudaMemcpy(in_d, h_a, N*sizeof(float), cudaMemcpyHostToDevice);
  // start CPU wallclock timer
  // uint32_t blocks_num = get_blocks((uint32_t)N, NUM_THREAD);
  uint32_t blocks_num = (N + NUM_THREAD - 1) / NUM_THREAD;
  dim3 blocks{blocks_num};
  dim3 threads{NUM_THREAD};
  auto start = std::chrono::high_resolution_clock::now();
  if (USE_CUDA_GRAPH) {
    cudaStream_t stream=0;
    cudaStreamCreate (&stream) ;
    bool graphCreated=false;
    cudaGraph_t graph;
    cudaGraphExec_t instance;
    for(int istep=0; istep<NSTEP; istep++){
      if(!graphCreated){
        cudaStreamBeginCapture(stream, cudaStreamCaptureModeGlobal);
        for(int ikrnl=0; ikrnl<NKERNEL; ikrnl++){
          shortKernel<<<blocks, threads, 0, stream>>>(out_d, in_d);
        }
        cudaStreamEndCapture(stream, &graph);
        cudaGraphInstantiate(&instance, graph, NULL, NULL, 0);
        graphCreated=true;
      }
      cudaGraphLaunch(instance, stream);
      cudaStreamSynchronize(stream);
    }
  } else {
    for(int istep=0; istep<NSTEP; istep++){
      for(int ikrnl=0; ikrnl<NKERNEL; ikrnl++){
        shortKernel<<<blocks, threads, 0, 0>>>(out_d, in_d);
      }
      cudaStreamSynchronize(0);
    }
  }


  get_latency(start);

  cudaFree(in_d);
  cudaFree(out_d);
}
```
