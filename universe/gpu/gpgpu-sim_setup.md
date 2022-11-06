---
title: GPGPU-sim部署 + Ubuntu切换软件版本
author: 66RING
date: 2022-11-05
tags: 
- gpu
- gpgpu
mathjax: true
---

# 环境

- ubuntu18.04
- cuda11
- gcc5.5


# 流程

## 安装依赖, 安装gcc5

GPGPU-sim所需的依赖如下

GPGPU-Sim dependencies:

```
sudo apt-get install build-essential xutils-dev bison zlib1g-dev flex libglu1-mesa-dev
```

GPGPU-Sim documentation dependencies:

```
sudo apt-get install doxygen graphviz
```

AerialVision dependencies:

```
sudo apt-get install python-pmw python-ply python-numpy libpng12-dev python-matplotlib
```

CUDA SDK dependencies:

```
sudo apt-get install libxi-dev libxmu-dev libglut3-dev
```

安装gcc, 这里使用gcc-5和g++-5, 高版本可能会在执行时seg fault

```
sudo apt install gcc-5 g++-5
```

修改系统gcc版本, 添加到可用项, 最后的100表示优先级(越大优先级越高)适用于自动模式

```
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-5 100
sudo update-alternatives --install /usr/bin/g++ g++ /usr/bin/g++-5 100

# 删除用
sudo update-alternatives --remove /usr/bin/gcc gcc /usr/bin/gcc-5 100
sudo update-alternatives --remove /usr/bin/g++ g++ /usr/bin/g++-5 100
```

选择版本

```
sudo update-alternatives --config gcc
sudo update-alternatives --config g++
```

此时gcc和g++的默认版本应该是5: `gcc -v`


## 安装CUDA

下载CUDA安装包

```sh
wget https://developer.download.nvidia.com/compute/cuda/11.1.0/local_installers/cuda_11.1.0_455.23.05_linux.run
```

执行安装

```sh
chmod +x cuda_11.1.0_455.23.05_linux.run
sudo ./cuda_11.1.0_455.23.05_linux.run
```

键入`accept`后, 仅选择安装`Toolkit`即可。之后cuda会安装到`/usr/local/cuda`。不同版本cuda的安装路径不同, 可以使用`sudo find / -name "nvcc"`命令查找一下。


## 安装GPGPU-sim

下载GPGPU-sim源代码

```
git clone https://github.com/gpgpu-sim/gpgpu-sim_distribution
cd gpgpu-sim_distribution
```

配置环境, 让gpgpu找到cuda, 可以在`gpgpu-sim_distribution/setup_environment`文件的开头添加

```
# setup_environment
export CUDA_INSTALL_PATH=/usr/local/cuda
export PATH=$CUDA_INSTALL_PATH/bin:$PATH
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/local/cuda/lib64
```

执行安装

```
source setup_environment
make -j $(nproc)
```


## 测试程序

创建目录

```
mkdir test
vim test/test.cu
```

编辑测试程序`test.cu`: 对`0..31`逐个加1, 输出`1..32`

```c
// test.cu
#include<stdio.h>
#include<cuda.h>

typedef double FLOAT;
__global__ void sum(FLOAT *x) {
	int tid = threadIdx.x;
	x[tid] += 1;
}

int main() {
	int N = 32;
	int nbytes = N * sizeof(FLOAT);

	FLOAT *dx = NULL, *hx = NULL;
	int i;
	// 申请显存
	cudaMalloc((void**)&dx, nbytes);
	
	// 申请成功
	if (dx == NULL) {
		printf("GPU alloc fail");
		return -1;
	}

	// 申请CPU内存
	hx = (FLOAT*)malloc(nbytes);
	if (hx == NULL) {
		printf("CPU alloc fail");
		return -1;
	}

	// init: hx: 0..31
	printf("hx original:\n");
	for(int i=0;i<N;i++) {
		hx[i] = i;
		printf("%g\n", hx[i]);
	}

	// copy to GPU
	cudaMemcpy(dx, hx, nbytes, cudaMemcpyHostToDevice);

	// call GPU
	sum<<<1, N>>>(dx);

	// let gpu finish
	cudaThreadSynchronize();

	// copy data to CPU
	cudaMemcpy(hx, dx, nbytes, cudaMemcpyDeviceToHost);

	printf("hx after:\n");
	for(int i=0;i<N;i++) {
		printf("%g\n", hx[i]);
	}
	cudaFree(dx);
	free(hx);
	return 0;
}
```

拷贝gpgpu-sim配置文件到当前工程, 配置顾名思义是某些型号显卡的模拟

```
cd test
cp ../gpgpu-sim_distribution/configs/tested-cfgs/SM2_GTX480/* ./
```

编译运行

```
nvcc --cudart shared -o test test.cu
./test
```

这里指定`cudart`库是动态链接的, 有因为`sourcet setup_environment`配置了动态链接的方法, 使用ldd工具看到编译后的二进制使用的是gpgpu-sim提供的动态库:

```
        libcudart.so.11.0 => /home/ub/gpgpu-sim_distribution/lib/gcc-5.5.0/cuda-11010/release/libcudart.so.11.0 (0x00007f7d44ae4000)
```
TODO:

**之后运行测试程序前记得要`source setup_environment`**, 它会配置动态链接的一些设置


## 坑

- gcc-5.4编译不行❌
- gcc-9编译不行❌, 编译出来seg fault
- gcc-5.5编译行✅


## 其他

- 目前gpgpu-sim 4.0没有支持部分功能, 如`cudaMallocManaged`
