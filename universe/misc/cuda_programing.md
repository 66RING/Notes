---
title: CUDA编程小记
author: 66RING
date: 2023-12-28
tags: 
- cuda
- hpc
mathjax: true
---

# CUDA编程小记

[这个教程](https://www.bilibili.com/video/BV1sM4y1x7of)的小记/速查

## 核函数(kernel)

kernel的定义和启动形如，使用`__global__`修饰的函数就是kernel，由host端启动，在device端运行。如果使用`__host__`就是就是host端代码，cuda编译器不会编译到gpu上。使用`__device__`修饰就是设备端代码，只能在设备上运行

```cuda
__global__ void cuda_kernel(...);

// 启动核函数, 通过<<<Dg, Db, Ns, S>>>配置
cuda_kernel<<<grid_size, block_size, Ns, stream>>>(...)
```

CUDA程序的大体流程

1. cudaMalloc申请设备内存
2. host到device数据拷贝做初始化
3. kernel执行
4. GPU程序是**异步执行**的需要使用`cudaDeviceSynchronize()`等待gpu执行完成
5. device到host数据拷贝取回结果
6. cudaFree释放设备内存


## 线程模型

![Grid of Thread Blocks](https://docs.nvidia.com/cuda/cuda-c-programming-guide/_images/grid-of-thread-blocks.png)

一个kernel执行都对应有一个grid配置信息。grid如图所示，grid相当于一个Thread Block的数组，索引使用blockIdx来索引。每个Thread Block又相当于一个Thread的数组，用threadIdx来索引、**用blockDim来表示block的大小(即有多少各thread)**。

> ```cuda
> cuda_kernel<<<grid_size, block_size, Ns, stream>>>(...)
> ```

需要注意的是blockIdx和threadIdx都可以是一个三维的信息xyz，而层级顺序是xyz的(即先填满一层x再到下一层)，这在后续索引计算中需要注意。

```
如
     x0       x1       x2
y0: (x0, y0) (x1, y0) (x2, y0)
y1: (x0, y1) (x1, y1) (x2, y1)
y2: (x0, y2) (x1, y2) (x2, y2)
```

### 索引计算

知道blockIdx, threadIdx, blockDim的关系后就可以计算索引。所以一个二维是block可以这样计算二维索引。主要思路就先是`blockIdx * blockDim`得到之前的组有多少个thread, 再加上当前组的`threadIdx`得到在全局的idx。


```cuda
int i = blockIdx.x * blockDim.x + threadIdx.x;
int j = blockIdx.y * blockDim.y + threadIdx.y;
```

## nvcc编译流程和GPU算力

> ptx, cubin, fat cubin

一个cuda程序先会编译成ptx(parallel thread execution)，再由ptx编译成gpu可执行的二进制表示cubin。ptx则相当于二进制生成过程中的中间表示，与GPU硬件无关。这么是为兼容性做的一些设计，如你可以用一个ptx编译多种GPU支持的cubin,即fatbin。


### CUDA兼容性

兼容性分为两部分计算能力和GPU架构

指定虚拟架构能力：添加`-arch=compute_XY`参数进行编译产生对应PTX，X表示计算能力主版本号，Y表示次版本号。`-arch=compute_XY`编译除了的程序只能在计算的能力大于XY的GPU上运行。

指定真实架构能力：添加`-code=sm_XY`指定PTX编程成的目标二进制是怎样的，与GPU硬件相关。需要注意的是**二进制cubin代码在大版本之间是不兼容的**，且真实架构能力必须大于虚拟架构能力。

编译多GPU兼容的二进制文件fat cubin：使用多个`-gencode=arch=compute_XY,code=sm_XY`。如`-gencode=arch=compute_35,code=sm_35 -gencode=arch=compute_50,code=sm_50`。这样会生成一个fatcubin，可执行文件的大小也增加。

nvcc即时编译：`-gencode=arch=compute_XY,code=compute_XY`arch和code都用`compute_XY`，且版本号XY必须相同。

## CUDA调试



### CUDA错误检查

基本上每个CUDA runtime API都会返回一个`cudaError_t`错误码，可以使用`cudaGetErrorName`和`cudaGetErrorString`获取错误信息。再用上`__FILE__`和`__LINE__`可以获取文件名和行号信息。

```cuda
#define CUDA_CHECK(code) ErrorCheck(code, __FILE__, __LINE__)

cudaError_t ErrorCheck(cudaError_t error_code, const char* filename, int lineNumber)
{
    if (error_code != cudaSuccess)
    {
        printf("CUDA error:\r\ncode=%d, name=%s, description=%s\r\nfile=%s, line%d\r\n",
                error_code, cudaGetErrorName(error_code), cudaGetErrorString(error_code), filename, lineNumber);
        return error_code;
    }
    return error_code;
}
```

kernel运行时错误检查，GPU是异步的CPU没法直接获取CUDA运行过程中的错误，但是发生错误后kernel会停止，使用`cudaGetLastError()`获取上一次发生的错误。

```cuda
addFromGPU<<<grid, block>>>(fpDevice_A, fpDevice_B, fpDevice_C, iElemCount);    // 调用核函数
ErrorCheck(cudaGetLastError(), __FILE__, __LINE__);
ErrorCheck(cudaDeviceSynchronize(), __FILE__, __LINE__);
```


### CUDA性能检测

使用CUDA API记时

```cuda
cudaEvent_t start, stop;
ErrorCheck(cudaEventCreate(&start), __FILE__, __LINE__);
ErrorCheck(cudaEventCreate(&stop), __FILE__, __LINE__);
ErrorCheck(cudaEventRecord(start), __FILE__, __LINE__);
cudaEventQuery(start);

/************************************************************
需要记时间的代码
************************************************************/

ErrorCheck(cudaEventRecord(stop), __FILE__, __LINE__);
ErrorCheck(cudaEventSynchronize(stop), __FILE__, __LINE__);
float elapsed_time;
ErrorCheck(cudaEventElapsedTime(&elapsed_time, start, stop), __FILE__, __LINE__);
printf("Time = %g ms.\n", elapsed_time);

ErrorCheck(cudaEventDestroy(start), __FILE__, __LINE__);
ErrorCheck(cudaEventDestroy(stop), __FILE__, __LINE__);
```

使用单独的可执行程序[nvprof](https://docs.nvidia.com/cuda/profiler-users-guide/index.html)做profiling。简单使用可以直接`nvprof ./executable`


### 运行时GPU信息查询

使用`cudaSetDevice`设备目标GPU，使用`cudaGetDeviceProperties`查询目标GPU的信息。

```cuda
int main(void)
{
    int device_id = 0;
    ErrorCheck(cudaSetDevice(device_id), __FILE__, __LINE__);

    cudaDeviceProp prop;
    ErrorCheck(cudaGetDeviceProperties(&prop, device_id), __FILE__, __LINE__);

    printf("Device id:                                 %d\n",
        device_id);
    printf("Device name:                               %s\n",
        prop.name);
    printf("Compute capability:                        %d.%d\n",
        prop.major, prop.minor);
    printf("Amount of global memory:                   %g GB\n",
        prop.totalGlobalMem / (1024.0 * 1024 * 1024));
    printf("Amount of constant memory:                 %g KB\n",
        prop.totalConstMem  / 1024.0);
    printf("Maximum grid size:                         %d %d %d\n",
        prop.maxGridSize[0], 
        prop.maxGridSize[1], prop.maxGridSize[2]);
    printf("Maximum block size:                        %d %d %d\n",
        prop.maxThreadsDim[0], prop.maxThreadsDim[1], 
        prop.maxThreadsDim[2]);
    printf("Number of SMs:                             %d\n",
        prop.multiProcessorCount);
    printf("Maximum amount of shared memory per block: %g KB\n",
        prop.sharedMemPerBlock / 1024.0);
    printf("Maximum amount of shared memory per SM:    %g KB\n",
        prop.sharedMemPerMultiprocessor / 1024.0);
    printf("Maximum number of registers per block:     %d K\n",
        prop.regsPerBlock / 1024);
    printf("Maximum number of registers per SM:        %d K\n",
        prop.regsPerMultiprocessor / 1024);
    printf("Maximum number of threads per block:       %d\n",
        prop.maxThreadsPerBlock);
    printf("Maximum number of threads per SM:          %d\n",
        prop.maxThreadsPerMultiProcessor);

    return 0;
}
```


## TODO

...









