---
title: HPC 入门笔记
date: 2024-04-23 20:06:21
tags: [HPC,超算]
categories: [笔记]
thumbnail: /images/hpc/0.png
---
## 前言
*不要因为走了太远，而忘记为什么出发*

高性能计算（HPC），简称为HPC，是一种技术，它利用并行工作的强大处理器集群来处理海量多维数据集（大数据），以极高的速度解决复杂问题。

此文记录了我早期入门超算的一些内容，主要以曾经写过的报告为主题，并加以修改和改进。有些部分因知识不足，或有欠缺之处，还请见谅。



## 基础知识
*The coming of the ship*
1. 编译器是一种计算机程序，负责把一种编程语言编写的源码转换成另外一种计算机代码，后者往往是以二进制的形式被称为目标代码 (object code)。这个转换的过程通常的目的是生成可执行的程序。 GCC/MSVC/Clang/ICC/NVCC
2. Frontier -US / Supercomputer Fugaku -JP
3. 拓扑结构：一个物理处理器（Package），每个处理器有4个核心（Core），每个核心有1个线程（Thread）。这台机器的总线程数是4。这台机器支持NUMA，有一个NUMA节点。
CPU 型号：CPU型号是Intel® Xeon® Gold 6248 CPU @ 2.50GHz，这是一款GenuineIntel的产品，CPU主频是2.5GHz，支持virtual-8086 mode enhancement等特性。
缓存大小：每个核心有32KB的一级数据缓存（L1d）和32KB的一级指令缓存（L1i），4096KB的二级缓存（L2），物理处理器有16MB的三级缓存（L3）。
内存：总内存是16GB，分布在一个NUMA节点上。

## 基础操作
*Empty and dark shall I raise my lantern*
### 1. 编译器基础使用
此处借用学长的话。

在 HPC 中，编译器的使用是基础中的基础，但也是很多时候的踩坑点：不同科学计算软件会使用各种各样的依赖库，而软件本身可能是由 Makefile 或者 CMake 等进行自动化构建；做移植时需要你替换依赖库实现更好的性能，但是随之而来的是各式各样的报错；多核 CPU 带来的多核计算程序、异构架构如 CUDA 等、新型国产硬件平台使得各式编译器层出不穷。这些问题在你没掌握基础前都是问题，所以我们希望你能先踩一些比较基础的坑，这样才能应对 “编译器的黄金时代” 。

### 2. 编译的简要流程  
预处理：处理代码，设置并行域，以便编译器使用。  
编译：将代码进行语法分析，优化和生成代码。  
汇编：将编译生成的代码翻译成机器指令，存储在.o中。  
链接：以静态库或者动态库的方式链接编译好的其他函数。

### 3. 静态库、动态库的区别  
静态库：在链接阶段，会将汇编生成的目标文件.o与引用到的库一起链接打包到可执行文件中。静态库对函数库的链接是放在编译时期完成的。  
动态库：动态库在程序编译时并不会被连接到目标代码中，而是在程序运行是才被载入。

### 4. 使用的编译参数的意义  
-m32：生成32位机器的汇编代码。  
-std=gnu++11：选择C语言编译标准。  
-Wall：该选项意思是编译后显示所有警告。  
-Wextra：检测更多的代码中的警告信息。  
-Bstatic：在静态库中查找符号而不是动态库。

#### 思考1：可不可以更换编译器如 Clang/LLVM 进行编译达到同样效果？
可以，但其对代码优化程度不同，所支持特性也不同。

#### 扩展1：Linux 中动态库和静态库的区别？
静态库编译时链接，动态库运行时链接，存储空间占用/更新部署方式不同。

#### 拓展2：哪些用于调试程序、分析二进制可执行文件的工具？
调试程序GDB，linux调试器；各种Debugger插件；分析二进制文件nm命令；IDA7，x64dbg常用于ctf中反编译程序；010 editor用来查看二进制信息。


## 自动化编译工具
### Makefile
#### 修改后的脚本源码
```makefile
CXX := g++
CXXFLAGS := -Wall -Wextra -std=c++17
LDFLAGS :=

SRC_DIR := ./src
INC_DIR := ./inc

CXXFLAGS += -I$(INC_DIR)

TARGET := main
HEADERS := $(wildcard $(INC_DIR)/*.hpp)
SRCS := $(wildcard $(SRC_DIR)/*.cpp)
OBJS := $(SRCS:.cpp=.o)

.PHONY: all
all: $(TARGET)

$(SRC_DIR)/%.o: $(SRC_DIR)/%.cpp $(HEADERS)
	$(CXX) -c -o $@ $< $(CXXFLAGS)

$(TARGET): $(OBJS)
	$(CXX) -o $@ $^ $(CXXFLAGS) $(LDFLAGS)

clean:
	@rm -f $(OBJS) $(TARGET)
```

#### 修改思路，对该 Makefile 构建程序流程进行解释
在Makefile 的开头，定义了一些变量，包括编译器（```CXX```）、编译器标志（```CXXFLAGS```）、链接器标志（```LDFLAGS```）、源代码目录（```SRC_DIR```）、头文件目录（```INC_DIR```）等。  
```makefile
CXXFLAGS += -I$(INC_DIR)
```  
这行命令将头文件目录添加到编译器标志中。  
```HEADERS```、```SRCS``` 和 ```OBJS```变量存储上面传下的参数，```wildcard```找到符合匹配的文件。
```makefile
.PHONY: all
all: $(TARGET)
```
定义命令集合all。
当运行make时，会生成 可执行文件$(TARGET)=main$。  
```makefile
$(SRC_DIR)/%.o: $(SRC_DIR)/%.cpp $(HEADERS)
```
表示每个 .o 目标文件都依赖于同名的 .cpp 源文件和所有的头文件。如果更新依赖或者目标文件不存在，则生成。  
```makefile
$(TARGET): $(OBJS) 
```
表示可执行文件依赖于所有的 .o 目标文件，包括CXX和LDF。  
clean删除生成的 .o文件和可执行文件。

### cmake
#### 修改后的脚本源码
```cmake
cmake_minimum_required(VERSION 3.16)

project(compile_cmake)

SET(CMAKE_BUILD_TYPE "Debug")
SET(CMAKE_CXX_FLAGS_DEBUG "$ENV{CFLAGS} -DDEBUG -O0 -Wall -g -ggdb")
SET(CMAKE_CXX_FLAGS_RELEASE "$ENV{CFLAGS} -O3 -Wall")

include_directories(inc)
add_library(v3 SHARED src/v3.cpp)
add_library(particle SHARED src/particle.cpp)
add_executable(main src/main.cpp)
target_link_libraries(main PUBLIC particle v3)

add_custom_target(run
    COMMAND ./src/main
    DEPENDS main
    WORKING_DIRECTORY ${CMAKE_PROJECT_DIR}
)
```
#### 修改思路，对该 CMake 构建程序流程进行解释
```cmake
cmake_minimum_required(VERSION 3.16)
```
指定cmake版本。  
```cmake
project(compile_cmake)
```
指定项目名称。  
```cmake
SET(CMAKE_BUILD_TYPE "Debug")
SET(CMAKE_CXX_FLAGS_DEBUG "$ENV{CFLAGS} -DDEBUG -O0 -Wall -g -ggdb")
SET(CMAKE_CXX_FLAGS_RELEASE "$ENV{CFLAGS} -O3 -Wall")
```
设置编译类型，编译标志。  
```cmake
include_directories(inc)
```
添加头文件目录。  
```cmake
add_library(v3 SHARED src/v3.cpp)
add_library(particle SHARED src/particle.cpp)
```
添加库文件。  
```cmake
add_executable(main src/main.cpp)
```
添加可执行文件。  
```cmake
target_link_libraries(main PUBLIC particle v3)
```
这里一开始报错找不到v3，因此调换了位置，先链接依赖的库。  
```cmake
add_custom_target(run
...
)
```
执行make时，先构建main，然后在目录下运行./src/main。  

#### 拓展1：
##### 修改后的脚本源码
```cmake
cmake_minimum_required(VERSION 3.16)

project(compile_cmake)

SET(CMAKE_BUILD_TYPE "Debug")
SET(CMAKE_CXX_FLAGS_DEBUG "$ENV{CFLAGS} -DDEBUG -O0 -Wall -g -ggdb")
SET(CMAKE_CXX_FLAGS_RELEASE "$ENV{CFLAGS} -O3 -Wall")

include_directories(inc)

add_library(v3 SHARED ./step2/src/v3.cpp)
add_library(particle SHARED ./step2/src/particle.cpp)
add_executable(main ./step2/src/main.cpp)
target_link_libraries(main PUBLIC particle v3)

add_custom_target(m0
    COMMAND echo "========" && echo "单文件编译" && g++ hello.cpp -o ${CMAKE_BINARY_DI
R}/m0/hello
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/step1
)

add_custom_target(m1
    COMMAND echo "==============================" && echo "全部编译成目标文件，最后直
接进行链接" && g++ src/main.cpp -o ${CMAKE_BINARY_DIR}/m1/main.o -g -ggdb -O0 -std=c++
17 -I./inc -Wall -march=native -c && g++ src/particle.cpp -o ${CMAKE_BINARY_DIR}/m1/pa
rticle.o -g -ggdb -O0 -std=c++17 -I./inc -Wall -march=native -c && g++ src/v3.cpp -o $
{CMAKE_BINARY_DIR}/m1/v3.o -g -ggdb -O0 -std=c++17 -I./inc -Wall -march=native -c && g
++ ${CMAKE_BINARY_DIR}/m1/main.o ${CMAKE_BINARY_DIR}/m1/particle.o ${CMAKE_BINARY_DIR}
/m1/v3.o -o ${CMAKE_BINARY_DIR}/m1/main && echo "=============================="
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/step2
)


add_custom_target(m2
    COMMAND echo "=================" && echo "Statically linked" && g++ src/main.cpp -
o ${CMAKE_BINARY_DIR}/m2/main.o -g -ggdb -O0 -std=c++17 -I./inc -Wall -march=native -c
 && g++ src/particle.cpp -o ${CMAKE_BINARY_DIR}/m2/libparticle.so -shared -fPIC -g -gg
db -O0 -std=c++17 -I./inc -Wall -march=native && g++ src/v3.cpp -o ${CMAKE_BINARY_DIR}
/m2/libv3.so -shared -fPIC -g -ggdb -O0 -std=c++17 -Wall -I./inc -march=native && g++
${CMAKE_BINARY_DIR}/m2/main.o ${CMAKE_BINARY_DIR}/m2/libparticle.so ${CMAKE_BINARY_DIR
}/m2/libv3.so -o ${CMAKE_BINARY_DIR}/m2/main && echo "================="
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/step2
)

add_custom_target(m3
    COMMAND echo "===========" && echo "Shared Libs" && g++ src/main.cpp -o ${CMAKE_BI
NARY_DIR}/m3/main.o -g -ggdb -O0 -std=c++17 -I./inc -Wall -march=native -c && g++ src/
particle.cpp -o ${CMAKE_BINARY_DIR}/m3/particle.o -fPIC -shared -g -ggdb -O0 -std=c++1
7 -I./inc -Wall -march=native -c && g++ src/v3.cpp -o ${CMAKE_BINARY_DIR}/m3/v3.o -fPI
C -shared -g -ggdb -O0 -std=c++17 -Wall -I./inc -march=native -c && g++ ${CMAKE_BINARY
_DIR}/m3/v3.o -o ${CMAKE_BINARY_DIR}/m3/libv3.so -fPIC -shared && g++ ${CMAKE_BINARY_D
IR}/m3/particle.o -o ${CMAKE_BINARY_DIR}/m3/libparticle_rpath-link.so -fPIC -shared -W
l,-L${CMAKE_BINARY_DIR}/m3 -Wl,-rpath-link=${CMAKE_BINARY_DIR}/m3 -lv3 && g++ ${CMAKE_
BINARY_DIR}/m3/particle.o -o ${CMAKE_BINARY_DIR}/m3/libparticle_rpath.so -fPIC -shared
 -Wl,-L${CMAKE_BINARY_DIR}/m3 -Wl,-rpath=${CMAKE_BINARY_DIR}/m3 -lv3 && g++ ${CMAKE_BI
NARY_DIR}/m3/particle.o -o ${CMAKE_BINARY_DIR}/m3/libparticle.so -fPIC -shared -Wl,-L$
{CMAKE_BINARY_DIR}/m3 -lv3 && g++ ${CMAKE_BINARY_DIR}/m3/main.o -o ${CMAKE_BINARY_DIR}
/m3/main_rpath-link -Wl,-L${CMAKE_BINARY_DIR}/m3 -Wl,-rpath-link=${CMAKE_BINARY_DIR}/m
3 -lparticle_rpath-link -lv3 && g++ ${CMAKE_BINARY_DIR}/m3/main.o -o ${CMAKE_BINARY_DI
R}/m3/main_rpath -Wl,-L${CMAKE_BINARY_DIR}/m3 -Wl,-rpath=${CMAKE_BINARY_DIR}/m3 -lpart
icle_rpath -lv3 && g++ ${CMAKE_BINARY_DIR}/m3/main.o -o ${CMAKE_BINARY_DIR}/m3/main -W
l,-L${CMAKE_BINARY_DIR}/m3 -lparticle -lv3 && echo "==========="
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/step2
)

add_custom_target(m4
    COMMAND echo "===========" && echo "Static Libs" && g++ src/main.cpp -o ${CMAKE_BI
NARY_DIR}/m4/main.o -g -ggdb -O0 -std=c++17 -I./inc -Wall -march=native -c && g++ src/
particle.cpp -o ${CMAKE_BINARY_DIR}/m4/particle.o -g -ggdb -O0 -std=c++17 -I./inc -Wal
l -march=native -c && g++ src/v3.cpp -o ${CMAKE_BINARY_DIR}/m4/v3.o -g -ggdb -O0 -std=
c++17 -Wall -I./inc -march=native -c && ar crv ${CMAKE_BINARY_DIR}/m4/libv3.a ${CMAKE_
BINARY_DIR}/m4/v3.o && ar crv ${CMAKE_BINARY_DIR}/m4/libparticle.a ${CMAKE_BINARY_DIR}
/m4/particle.o && ranlib ${CMAKE_BINARY_DIR}/m4/libv3.a && ranlib ${CMAKE_BINARY_DIR}/
m4/libparticle.a && g++ ${CMAKE_BINARY_DIR}/m4/main.o -o ${CMAKE_BINARY_DIR}/m4/main -
static -Wl,-L${CMAKE_BINARY_DIR}/m4 -lparticle -lv3 && g++ ${CMAKE_BINARY_DIR}/m4/main
.o -o ${CMAKE_BINARY_DIR}/m4/main_1 -Wl,-L${CMAKE_BINARY_DIR}/m4 -lparticle -lv3 && ec
ho "==========="
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/step2
)

add_custom_target(m5
    COMMAND echo "===============================" && echo "Use both Shared and Static
 Libs" && g++ src/mixed_main.cpp -o ${CMAKE_BINARY_DIR}/m5/mixed_main.o -g -ggdb -O0 -
std=c++17 -I./inc -Wall -march=native -c && g++ src/mixed_a.cpp -o ${CMAKE_BINARY_DIR}
/m5/mixed_a.o -fPIC -shared -g -ggdb -O0 -std=c++17 -I./inc -Wall -march=native -c &&
g++ src/mixed_b.cpp -o ${CMAKE_BINARY_DIR}/m5/mixed_b.o -g -ggdb -O0 -std=c++17 -I./in
c -Wall -march=native -c && g++ ${CMAKE_BINARY_DIR}/m5/mixed_a.o -o ${CMAKE_BINARY_DIR
}/m5/libmixed_a.so -fPIC -shared && ar crv ${CMAKE_BINARY_DIR}/m5/libmixed_b.a ${CMAKE
_BINARY_DIR}/m5/mixed_b.o && ranlib ${CMAKE_BINARY_DIR}/m5/libmixed_b.a && g++ ${CMAKE
_BINARY_DIR}/m5/mixed_main.o -o ${CMAKE_BINARY_DIR}/m5/mixed_main -Wl,-L${CMAKE_BINARY
_DIR}/m5 -lmixed_a -lmixed_b && g++ ${CMAKE_BINARY_DIR}/m5/mixed_main.o -o ${CMAKE_BIN
ARY_DIR}/m5/mixed_main_v1 -Wl,-L${CMAKE_BINARY_DIR}/m5 -lmixed_a -Bstatic -lmixed_b &&
 g++ ${CMAKE_BINARY_DIR}/m5/mixed_main.o -o ${CMAKE_BINARY_DIR}/m5/mixed_main_v2 -Wl,-
L${CMAKE_BINARY_DIR}/m5 -lmixed_a -l:libmixed_b.a && echo "===========================
===="
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/step3
)
```

##### 修改思路，构建流程的解释
指定版本，项目名，导入头文件目录，添加库文件，添加可执行文件copy。
```cmake
add_custom_target(m0 ...)
...
add_custom_target(m5 ...)
```
这些代码添加了六个自定义目标m0到m5，它们分别对应于compile.sh脚本中的m0到m5函数。每个目标都使用了一个自定义命令来执行相应的编译步骤。  
~~改完了发现其实可以用 m0R = ${CMAKE_BINARY_DIR}/m0 来代替，不看开头导致的。~~


## 并行基础
*Then he assigns you to his sacred fire*
### 过程


inc 目录存放了头文件，src 目录存放了源代码文件read_data.hpp。进入 src 目录，首先编译 datagen.cpp
```bash
g++ -I../inc datagen.cpp -o datagen
./datagen 1000 1000 1000
```
编译命令如下  
```bash
nvcc -I../inc matrix_cal_cuda.cu -o matrix_cal_cuda
nvc++ -I../inc matrix_cal_cuda.cu -o matrix_cal_cuda -std=c++17

g++ -I../inc matrix_cal_general.cpp -o matrix_cal_general

mpicxx -I../inc matrix_cal_mpi.cpp -o matrix_cal_mpi

#fatal error: mpi.h: No such file or directory #include <mpi.h>
nvcc -I../inc -c matrix_cal_mpi_cuda.cu -o matrix_cal_mpi_cuda.o -I/usr/lib/x86_64-linux-gnu/openmpi/include
#nvc++ -I../inc -c matrix_cal_mpi_cuda.cu -o matrix_cal_mpi_cuda.o -I/usr/lib/x86_64-linux-gnu/openmpi/include -std=c++17
mpicxx -I../inc matrix_cal_mpi_cuda.o -o matrix_cal_mpi_cuda -L/opt/nvidia/hpc_sdk/Linux_x86_64/23.9/cuda/lib64 -lcudart

nvcc -I../inc -c matrix_cal_mpi_cuda.cu -lmpi -I/usr/lib/x86_64-linux-gnu/openmpi/include -o matrix_cal_mpi_cuda


mpicxx -fopenmp -I../inc  matrix_cal_mpi_openmp.cpp -o matrix_cal_mpi_openmp -lstdc++

gcc -o matrix_cal_openblas matrix_cal_openblas.cpp -lopenblas -I../inc -lstdc++

gcc -fopenmp -I../inc  matrix_cal_openmp.cpp -o matrix_cal_openmp -lstdc++
```
运行命令如下  
```bash
!/bin/bash
#显然这台机器上没有显卡，所以关于cuda的都在我的笔记本上的wsl运行
./matrix_cal_cuda 2>&1 | tee logs/matrix_cal_cuda.log #2 pass

./matrix_cal_general 2>&1 | tee logs/matrix_cal_general.log #pass

mpirun -np 4 ./matrix_cal_mpi 2>&1 | tee logs/matrix_cal_mpi.log #pass

#sudo find / -name 'libcudart.so.12' export LD_LIBRARY_PATH=/:$LD_LIBRARY_PATH source ~/.bashrc sudo ldconfig
mpirun -np 4 ./matrix_cal_mpi_cuda 2>&1 | tee logs/matrix_cal_mpi_cuda.log #2 pass
#mpirun -np 1 ./matrix_cal_mpi_cuda 2>&1 | tee logs/matrix_cal_mpi_cuda np1.log

mpirun -np 4 ./matrix_cal_mpi_openmp 2>&1 | tee logs/matrix_cal_mpi_openmp.log #pass

./matrix_cal_openblas 2>&1 | tee logs/matrix_cal_openblas.log #pass

./matrix_cal_openmp 2>&1 | tee logs/matrix_cal_openmp.log #pass
```

### 分析
（以general为例）  
源程序进行计算的流程  
- 检查命令行参数，获取路径，获取文件
- 对每个文件中的矩阵进行如下操作：
  - 初始化结果矩阵
  - 循环计算矩阵
  - 串行计算
  - 验证 A B -> C


（优化实现）  
前言：应该说每个有优化的程序必须有调库和调用的区域  
例如，对于mpi来说，就是调用<mpi.h>，对openmp就是<omp.h>  
然后，对openmp来说就是有一个并行域用来执行任务划分/调度 制导同步/解决数据竞争和cache冲突  
对mpi来说就是调用头文件 编译执行/通信 同步 /广播 /分散 收集/归约这些操作（在注释里也不难看出，给大佬跪了orz）  

- mpi：首先由主进程将任务划分到各个进程，每个进程持有一部分矩阵数据进行计算；  
但结果矩阵并没有再划分，而是将各个线程归约，将数据集中在一起比较，这里理解是通信是需要时间的，显然传两次比传一次效率高  

- openmp：直接划分线程  
- cuda：cudaMalloc 分配内存 cudaMemcpy 从内存cp到显存  
定义gird大小根据mp也就是矩阵大小计算 ，定义block包含16*16个线程  
速度很快，不愧是矩阵专业户，计算返回  
- openblas：不会，爆了（x） 还是看一眼怎么使用的吧  
cblas_dgemm(CblasRowMajor, CblasNoTrans, CblasNoTrans, m, p, n, 1.0, m1, n, m2, p, 0.0, answer, p);  
但实际上官方有用户手册，理解就简单多了  
https://github.com/OpenMathLib/OpenBLAS/wiki/User-Manual  
  - CblasRowMajor 行优先存储
  - CblasNoTrans 不转置
  - m,p,n指定两个输入矩阵和结果矩阵的维度 m1(m*n) m2(n*p) answer(m*p)
  - 1.0 计算结果=1.0*(m1*m2)
  - m1,n,m2,p 指定输入矩阵和其列数
  - 0.0 answer p 结果矩阵行p初始值0.0  
经典blas三级运算 查了一下发现是“在O(n2)时间复杂度实现O(n3)量级的浮点运算，能充分发挥现代处理器的性能，并且能为用户提供透明的并发机制。”  啊……懂了 ~~但实际上仍然是白痴（指向自己）~~

### OPENMP改写
修改的内容如下  
```c
#include <cstdlib>
// ...
int main(int argc, char* argv[]) {
    // ...
    // 读取环境变量 OMP_NUM_THREADS
    char* env_threads = getenv("OMP_NUM_THREADS");
    int num_threads = env_threads ? std::stoi(env_threads) : 16;
    #pragma omp parallel for shared(answer, m1, m2, m, p, n) private(i, j, k) num_threads(num_threads)
}
```
```bash
gcc -fopenmp -I../inc  matrix_cal_openmp.cpp -o matrix_cal_openmp
#g++ -fopenmp -I../inc matrix_cal_openmp.cpp -o matrix_cal_openmp -lstdc++fs
export OMP_NUM_THREADS=64
./matrix_cal_openmp
```

比较如下：  
|  实验次数/线程数   | 1 | 2 | 4 | 8 | 16 | 32 | 64 |
|  ----  | ----  | ----  | ----  | ----  | ----  | ----  | ----  |
|  1  | 7.66 | 4.02 | 1.96 | 1.96 | 1.91 | 1.87 | 1.93 |
|  2  | 7.38 | 3.93 | 1.94 | 1.93 | 1.86 | 1.88 | 1.85 |
|  3  | 7.36 | 3.91 | 1.91 | 1.86 | 1.88 | 1.92 | 1.87 |
|  4  | 7.35 | 3.91 | 1.94 | 1.89 | 1.85 | 1.87 | 1.89 |

明显可以看出，在4线程前，运行时间都是减少的；4线程后，运行时间的变化就不明显了  
很显然，这是cpu核心数导致的

### MPI测试
```bash
mpicxx -I../inc matrix_cal_mpi.cpp -o matrix_cal_mpi
mpirun -np 4 ./matrix_cal_mpi
mpirun --oversubscribe -np 8 ./matrix_cal_mpi
```
比较如下：  
|  实验次数/进程数   | 1 | 2 | 4 | 8 |
|  ----  | ----  | ----  | ----  | ----  |
|  1  | 11.137 | 5.68647 | 3.29428 | 3.64855 |
|  2  | 11.0721 | 5.7994 | 3.09677 | 3.68822 |
|  3  | 10.9346 | 5.68283 | 3.03596 | 3.68843 |
|  4  | 11.1715 | 5.72521 | 3.00175 | 3.62148 |

多了得申请超进程，不然会报错（然而超进程也不能超多了，不然会有无法整除的问题）  
但很明显超进程会导致性能下降，推测原因是**模拟逻辑核心**导致资源消耗/通信时间增长

### MPI+OPENMP 测试
```bash
#代码更改如上4
mpicxx -fopenmp -I../inc  matrix_cal_mpi_openmp.cpp -o matrix_cal_mpi_openmp -lstdc++
export OMP_NUM_THREADS=1
mpirun -np 1 ./matrix_cal_mpi_openmp
```
比较如下：  
|  实验次数/进程数(MPI)*线程数(OPENMP)   | 1*1 | 1*2 | 2*1 | 2*2 | 4*1 | 4*2 | 4*4 | 4*64 |
|  ----  | ----  | ----  | ----  | ----  | ----  | ----  | ----  | ----  |
|  1  | 16.2246 | 15.9681 | 8.36887 | 8.39439 | 4.42203 | 4.58054 | 4.60151 | 4.40564 |
|  2  | 15.9925 | 16.248 | 8.25618 | 8.26528 | 4.25193 | 4.39034 | 4.62932 | 4.33078 |
|  3  | 16.2064 | 15.9181 | 8.12511 | 8.35172 | 4.32775 | 4.40731 | 4.64253 | 4.45745 |
|  4  | 16.0636 | 16.0229 | 8.05308 | 8.29208 |  4.37209 | 4.49706 | 4.66099 | 4.3982 |

很有意思的结果，但不出所料。多线程负优化在之前的一些实验中也能发现，但总归来说多线程还是能起到一定作用的（特定数量下）  
为了探究这个问题，关于openmp和mpi混合编译，我找到了以下资料  
https://lab.cs.tsinghua.edu.cn/hpc/doc/faq/binding/#mpi-openmp  
其中特别提到：
- *在运行计算密集的程序时，通常需要将进程、线程与 CPU 核心进行绑定（binding / pinning），即控制进程与 CPU 核心的亲和性（affinity），消除上述的各类影响* **（PS:综合前文，这里应该指上述提到的进程迁移，冷启动开销，性能波动等）**
- *在使用 MPI + OpenMP 混合编程时，进程绑定对性能的影响尤为关键。*

我猜这就是为什么下一项就是关于亲和度绑定，由此诞生的问题或许能得到解答

### MPI+OPENMP 亲和度绑定

进程/线程亲和度应该指其在指定某个CPU核上尽量长时间运行而不被迁移。比如，linux内核调度器会倾向于减少进程/线程迁移；或者将进程/线程直接绑定到CPU核上。  

机器的NUMA结构如下：  
![Alt text](numa.png)

Intel的openmp有一个特殊的api实现绑定  
https://www.intel.com/content/www/us/en/docs/cpp-compiler/developer-guide-reference/2021-8/thread-affinity-interface.html  
在科学计算软件中使用openmp+mpi混合也有参考样例  
https://www.vasp.at/wiki/index.php/Combining_MPI_and_OpenMP  
在之前我使用openmp+mpi时，出现了线程全部分布在奇数位cpu核上的情况，是因为：
- *在使用 MPI + OpenMP 混合编程时，进程绑定对性能的影响尤为关键。每个 MPI 进程需要绑定在一组核心上（通常属于同一个 NUMA domain），并把它的 OpenMP 线程绑定在其中的每个核心上。*
- *OpenMP 线程只能绑定于其“可见”的核心上，也就是父进程被绑定的核心。*


```bash
mpirun -np 4 -x OMP_NUM_THREADS=4 -x OMP_PROC_BIND=close -x  OMP_PLACES=cores  --bind-to core  --report-bindings  ./matrix_cal_mpi_openmp

#export OMP_NUM_THREADS=1
#mpirun -H localhost --report-bindings -np 1 ./matrix_cal_mpi_openmp 
#一个插槽并要求一个进程，因此该进程绑定到单个内核

#mpirun -H localhost,localhost --report-bindings -np 2 ./matrix_cal_mpi_openmp
#mpirun -H localhost,localhost,localhost,localhost --report-bindings -np 4 ./matrix_cal_mpi_openmp
#[head:08805] MCW rank 0 bound to socket 0[core 0[hwt 0]]: [B/././.]
#[head:08805] MCW rank 1 bound to socket 0[core 1[hwt 0]]: [./B/./.]
```
比较如下：  
|  实验次数/进程数(MPI)*线程数(OPENMP)   | 1*1 | 1*2 | 2*1 | 2*2 | 4*4 |
|  ----  | ----  | ----  | ----  | ----  | ----  |
|  1  | 15.9717 | 16.1274 | 8.10164 | 8.14833 | 4.68596 |
|  2  | 16.0037 | 16.0947 | 8.12477 | 8.11571 | 4.51245 |
|  3  | 16.1317 | 15.9645 | 8.18908 | 8.13617 | 4.47034 |
|  4  | 16.0732 | 15.9364 | 8.1226  | 8.11981 | 4.46143 |

![Alt text](bound.png)

在测试中，每个线程绑定到一个 core，线程在 socket 上连续分布（分别绑定到 core 0,1,2,3）  
根据结果所示，其耗时均值均略小于未绑定

### CUDA分析
从之前对代码的分析来看，应该是在第一次运行中加载库/分配空间/将数据从内存
复制到显存这些操作耗时较长，但在之后的运行中则不需要重复这些操作  
其实从结果来看，任何并行库加载都会耗时，但没有CUDA这么明显

### 不同MPI实现，Blas库
- openmpi
- mpich
- intel mpi

- openblas
- intel mkl
- cublas

~~这项先跳了，如果还有时间再回来做~~

### OPENMP MPI MPI+OPENMP的缓存利用
```bash
perf stat -e cache-misses mpirun -np 4 matrix_cal_mpi
perf record -e cache-misses mpirun -np 4 matrix_cal_mpi
perf report
sudo perf report -i perf.data > perf.txt
export OMP_NUM_THREADS=4
perf record -e cache-misses mpirun -np 4 matrix_cal_mpi_openmp

# mpi
 Performance counter stats for 'mpirun -np 4 matrix_cal_mpi':
         332531131      cache-misses
      33.672167156 seconds time elapsed
# Samples: 342K of event 'cache-misses'
# Event count (approx.): 353917664
    83.46%  matrix_cal_mpi  matrix_cal_mpi                 [.] std::vector<double, std::allocator<double> >::operator[]
    12.82%  matrix_cal_mpi  matrix_cal_mpi                 [.] main
     1.97%  matrix_cal_mpi  [kernel.kallsyms]              [k] copy_user_enhanced_fast_string
     0.41%  matrix_cal_mpi  [kernel.kallsyms]              [k] clear_page_erms

# openmp
# Samples: 165K of event 'cache-misses'
# Event count (approx.): 53612667
    44.44%  matrix_cal_open  matrix_cal_openmp    [.] main._omp_fn.0
    31.55%  matrix_cal_open  [kernel.kallsyms]    [k] copy_user_enhanced_fast_string
     9.60%  matrix_cal_open  matrix_cal_openmp    [.] main
     3.93%  matrix_cal_open  matrix_cal_openmp    [.] std::abs
     3.49%  matrix_cal_open  [kernel.kallsyms]    [k] clear_page_erms
     1.13%  matrix_cal_open  [kernel.kallsyms]    [k] page_fault
     0.84%  matrix_cal_open  [kernel.kallsyms]    [k] get_page_from_freelist
     0.63%  matrix_cal_open  [kernel.kallsyms]    [k] page_remove_rmap

# mpi+openmp
# Samples: 338K of event 'cache-misses'
# Event count (approx.): 351509793
    85.77%  matrix_cal_mpi_  matrix_cal_mpi_openmp                [.] std::vector<double, std::allocator<double> >::operator[]
     8.21%  matrix_cal_mpi_  matrix_cal_mpi_openmp                [.] main._omp_fn.0
     1.99%  matrix_cal_mpi_  [kernel.kallsyms]                    [k] copy_user_enhanced_fast_string
     1.25%  matrix_cal_mpi_  matrix_cal_mpi_openmp                [.] main
     1.02%  matrix_cal_mpi_  matrix_cal_mpi_openmp                [.] std::__fill_a1<double*, double>
     0.40%  matrix_cal_mpi_  [kernel.kallsyms]                    [k] clear_page_erms
```

```bash
mpicxx -I../inc matrix_cal_mpi_change.cpp -o matrix_cal_mpi_change
perf stat -e cache-misses mpirun -np 4 matrix_cal_mpi_change
mpirun -np 4 ./matrix_cal_mpi_change
```

我有一个思路是将块划分的大小改为适合cache的大小
```c
int L1d_cache_size = 32 * 1024; // 32KB
int double_size = sizeof(double); // 8 bytes
int block_size_L1d = L1d_cache_size / double_size;
// ...
int block_size_i = block_size_L1d / n; // 计算每个块的行数
int start_i = rank * block_size_i; // 计算需要处理的块的范围
int end_i = (rank + 1) * block_size_i;
// ...
// 局部乘法
for (size_t i = start_i; i < end_i; i++) {
    for (size_t j = 0; j < p; j++) {
        for(size_t k = 0; k < n; k++) {
            answer_local[i * p + j] += m1_local[i * n + k] * m2[k * p + j];
        }
    }
}
```
但是呃失败了  
我猜可能是数组越界了，因为没有考虑block_size可能不能被m整除的情况，但这方面我不是很熟悉

```bash
e测试矩阵规模：1000 1000 1000
每个数据测量 1 次
[head:23304] *** Process received signal ***
[head:23304] Signal: Segmentation fault (11)
[head:23304] Signal code: Invalid permissions (2)
[head:23304] Failing at address: 0x7fccb18cf000
[head:23304] [ 0] /lib/x86_64-linux-gnu/libpthread.so.0(+0x12980)[0x7fccbfe2d980]
[head:23304] [ 1] /lib/x86_64-linux-gnu/libc.so.6(+0x18e9df)[0x7fccbfbb89df]
[head:23304] [ 2] /usr/local/openmpi/lib/libopen-pal.so.40(+0x4e73c)[0x7fccbf4a873c]
[head:23304] [ 3] /usr/local/openmpi/lib/libmpi.so.40(ompi_datatype_sndrcv+0x54f)[0x7fccc0a7a78f]
[head:23304] [ 4] /usr/local/openmpi/lib/libmpi.so.40(ompi_coll_base_scatter_intra_basic_linear+0x1aa)[0x7fccc0abfffa]
[head:23304] [ 5] /usr/local/openmpi/lib/openmpi/mca_coll_tuned.so(ompi_coll_tuned_scatter_intra_dec_fixed+0x65)[0x7fccb2d80dc5]
[head:23304] [ 6] /usr/local/openmpi/lib/libmpi.so.40(PMPI_Scatter+0x1a7)[0x7fccc0aa2f97]
[head:23304] [ 7] ./matrix_cal_mpi_change[0x406876]
[head:23304] [ 8] /lib/x86_64-linux-gnu/libc.so.6(__libc_start_main+0xe7)[0x7fccbfa4bc87]
[head:23304] [ 9] ./matrix_cal_mpi_change[0x4051da]
[head:23304] *** End of error message ***
--------------------------------------------------------------------------
Primary job  terminated normally, but 1 process returned
a non-zero exit code. Per user-direction, the job has been aborted.
--------------------------------------------------------------------------
--------------------------------------------------------------------------
mpirun noticed that process rank 0 with PID 0 on node head exited on signal 11 (Segmentation fault).
--------------------------------------------------------------------------
```

我另一个想法是将矩阵循环划分给各个进程，使其适应L1 cache，但仍然不知其可实现性如何  
std::vector 应该也有优化方法，比如说使用其它类型的存储方式  
也许使用bcc的cachestat和cachetop，以及pcstat会有更多的发现

后记：看HPC那篇。

### SIMD

SIMD 是 Single Instruction, Multiple Data 的缩写，意为“单指令，多数据”。这是一种在现代微处理器中广泛使用的并行计算技术。在 SIMD 模式下，处理器可以在一个时钟周期内对多个数据元素执行相同的操作。


### Roofline Model
Roofline Model应该是用于计算理论性能上限，即程序在某计算平台上可以达到最快的浮点计算速度。

则题目中的浮点运算次数是 1999 * 1000 * 1000 = 1.999 * 10^9 FLOPs
访问的内存量是 2000 * 1000 * 1000 * 8 = 16 * 10^9 Bytes
假设内存带宽DDR4 2666：21.3 GB/s
假设平台峰值计算能力为 P FLOPs/s
理论性能上限（FLOPs） = min(P FLOPs，21.3 GB/s * 1.999 * 10^9 FLOPs / 16 * 10^9 字节) = min(P FLOPs，2.665625 GFLOPs)

## CPU性能分析
*A little while, a moment of rest upon the wind*  

### CPU 微架构性能分析与优化

#### perf
```bash
apt install linux-tools-4.15.0-151-generic linux-cloud-tools-4.15.0-151-generic linux-cloud-tools-generic

perf stat ./matrix_cal_general
perf record ./matrix_cal_general
sudo perf report -i perf.data > perf.txt
```
```bash
Performance counter stats for './matrix_cal_general':

   351615.883768      task-clock (msec)         #  f  1.000 CPUs utilized
             447      context-switches          #    0.001 K/sec
               0      cpu-migrations            #    0.000 K/sec
          389249      page-faults               #    0.001 M/sec
   1114263653464      cycles                    #    3.169 GHz
   2104440335203      instructions              #    1.89  insn per cycle
     50689096703      branches                  #  144.160 M/sec
        53960798      branch-misses             #    0.11% of all branches

   351.642911157 seconds time elapsed
```

#### vtune
```bash
wget https://registrationcenter-download.intel.com/akdlm/IRC_NAS/dfae6f23-6c90-4b9f-80e2-fa2a5037fe36/l_oneapi_vtune_p_2023.2.0.49485.sh

sudo sh ./l_oneapi_vtune_p_2023.2.0.49485.sh
#好，走到这一步寄了，因为这个cpu不支持vtune
#所以在这里换了机器
#然后暂时先用的是远程gui，所以没命令
#但其实命令也差不多，最后还得导入gui看
```
居家必用👍

#### 解读

比如，以上文perf结果为例
- task-clock：程序运行了约351615.88毫秒。
- CPUs utilized：程序使用了1个CPU。
- context-switches：程序发生447次上下文切换。
- cpu-migrations：没有发生CPU迁移。
- page-faults：程序执行中有389249次页错误。
- cycles：程序执行中有1114263653464个CPU周期。
- instructions：程序运行中执行了大约2104440335203条指令。
- insn per cycle：每个CPU周期内执行了大约1.89条指令。
- branches：程序运行期间有大约50689096703条分支指令。
- branch-misses：程序运行期间大约0.11%的分支指令预测错误。
- seconds time elapsed：程序运行了大约351.64秒。


### 优化
~~每个人都有一颗想看懂源码的心~~  
先尝试编译器优化吧（

#### 火焰图
perf和vtune都有火焰图，vtune的自带在hotspot结果中，所以这里试一下perf  
这里以matrix_cal_mpi_openmp为例  
（lamegraph读到栈损坏,程序不运行就没法分析了）
```bash
export OMP_NUM_THREADS=4
perf record  mpirun -np 4 matrix_cal_mpi_openmp
perf report
perf script -i perf.data &> perf.unfold
/root/FlameGraph-master/stackcollapse-perf.pl perf.unfold &> perf.folded
/root/FlameGraph-master/flamegraph.pl perf.folded > perf_o_m.svg
perf script | /root/FlameGraph-master/stackcollapse-perf.pl | /root/FlameGraph-master/flamegraph.pl > process.svg


ps -ef | grep matrix_cal_general
perf record -F 50 -a -p 4869 -g -- sleep 60
perf report
perf script -i perf.data &> perf.unfold
/root/FlameGraph-master/stackcollapse-perf.pl perf.unfold &> perf.folded
/root/FlameGraph-master/flamegraph.pl perf.folded > perf_o_m.svg
```
![Alt text](perf/perf_o_m.svg)

#### 编译器相关优化
```bash
wget https://registrationcenter-download.intel.com/akdlm/IRC_NAS/5c8e686a-16a7-4866-b585-9cf09e97ef36/l_dpcpp-cpp-compiler_p_2024.0.0.49524_offline.sh
sh ./l_dpcpp-cpp-compiler_p_2024.0.0.49524_offline.sh
source /opt/intel/oneapi/setvars.sh intel64

#无力吐槽clang，换机器重新实验
#wget https://apt.llvm.org/llvm.sh
#chmod +x llvm.sh
#sudo ./llvm.sh 16

g++ -I../inc matrix_cal_general.cpp -o matrix_cal_general -O0
#确实不能轻易用icx，但冒险一下是值得的
icpx  -I../inc matrix_cal_general.cpp -o matrix_cal_general -Wincompatible-compiler
#icpx matrix_cal_general.cpp -o matrix_cal_general -Wincompatible-compiler
clang++  matrix_cal_general.cpp -o matrix_cal_general -std=c++17

```
-O0	不进行优化处理。  
-O 或 -O1	优化生成代码。  
-O2	进一步优化。  
-O3	比 -O2 更进一步优化，包括 inline 函数。

#### 编译器版本，下载地址
- [gcc (GCC) 12.3.0](https://mirrors.lzu.edu.cn/gnu/gcc/gcc-12.3.0/)
- [Intel(R) oneAPI DPC++/C++ Compiler for applications running on Intel(R) 64, Version 2024.0.0 Build 20231017](https://registrationcenter-download.intel.com/akdlm/IRC_NAS/5c8e686a-16a7-4866-b585-9cf09e97ef36/l_dpcpp-cpp-compiler_p_2024.0.0.49524_offline.sh)
- [clang version 6.0.0-1ubuntu2 (tags/RELEASE_600/final)](http://mirror.lzu.edu.cn/ubuntu/)

换个机器  
- gcc (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0
- Intel(R) oneAPI DPC++/C++ Compiler 2024.0.0 (2024.0.0.20231017)
- Ubuntu clang version 14.0.0-1ubuntu1.1


#### 开启-O0 -O3 后的性能分析比较

设备A

gcc 12.3.0

|  实验次数/优化等级   | 无 | -O0 | -O3 |
|  ----  | ----  | ----  | ----  |
|  1  | 7.46082 | 7.56615 | 2.42238 |
|  2  | 7.59183 | 7.45967 | 2.5212  |
|  3  | 7.59043 | 7.45407 | 2.51571 |
|  4  | 7.5156  | 7.47651 | 2.52421 |

icpx

|  实验次数/优化等级   | 无 | -O0 | -O3 |
|  ----  | ----  | ----  | ----  |
|  1  | 3.0166  | 6.34876 | 2.86272 |
|  2  | 3.03364 | 6.42262 | 2.66049 |
|  3  | 3.03413 | 6.52243 | 2.65275 |
|  4  | 3.09894 | 6.47761 | 2.65943 |

设备B

gcc 11.4.0

|  实验次数/优化等级   | 无 | -O0 | -O3 |
|  ----  | ----  | ----  | ----  |
|  1  | 3.50859 | 4.43146 | 1.08519 |
|  2  | 3.51196 | 4.46978 | 1.14748 |
|  3  | 4.14652 | 4.68874 | 1.08495 |
|  4  | 3.84136 | 4.75892 | 1.10398 |

icpx

|  实验次数/优化等级   | 无 | -O0 | -O3 |
|  ----  | ----  | ----  | ----  |
|  1  | 1.45927 | 2.91811 | 1.03595  |
|  2  | 1.14828 | 2.91811 | 1.00384  |
|  3  | 1.07161 | 2.87911 | 1.02894  |
|  4  | 1.04854 | 3.19374 | 0.994133 |

clang

|  实验次数/优化等级   | 无 | -O0 | -O3 |
|  ----  | ----  | ----  | ----  |
|  1  | 3.36724 | 3.428   | 0.996193 |
|  2  | 3.31483 | 3.63713 | 0.990838 |
|  3  | 3.31882 | 3.7065  | 1.05766  |
|  4  | 3.43748 | 3.69653 | 1.00913  |

#### 不同编译器的性能分析比较
```bash
icpx  -I../inc matrix_cal_general.cpp -o matrix_cal_general -Wincompatible-compiler -O3 -qopt-report
clang++  matrix_cal_general.cpp -o matrix_cal_general -std=c++17 -O3 -Rpass 2>&1| tee >clang_log.txt
```
见附件 matrix_cal_general.opt.yaml 和 clang_log

#### 更多编译优化参数参数
对于intel c++来说，比如：  
-fast：最大化整个程序的速度，相当于：-ipo, -O3, -no-prec-div, -static,
和-xHos  
-Ofast：设置某些激进参数优化程序速度（实际上启用这项速度对比-O3反而略有降低

### 拓展 GPU性能分析优化
####  Nsight Systems 使用的版本
- NsightSystems-2023.4.1.97-3355750
#### 生成分析报告
```bash
nvcc -I../inc matrix_cal_cuda.cu -o matrix_cal_cuda -arch sm_86
./matrix_cal_cuda
/home/
```
#### 对分析报告的简短理解

有cpu的使用和时长，gpu显存的使用timeline  
对于cudaapi，可以看到红色的cudamalloc  
对于接下来的矩阵，每段cudamalloc，然后cudamemcpy  
计算结束后，cudafree所有数据

简短来看，报告展示了程序运行时cpu和gpu的情况，以及cuda的行为

### 拓展 MPI优化

```bash
wget https://registrationcenter-download.intel.com/akdlm/IRC_NAS/2c45ede0-623c-4c8e-9e09-bed27d70fa33/l_mpi_oneapi_p_2021.11.0.49513_offline.sh
sh l_mpi_oneapi_p_2021.11.0.49513_offline.sh

mpiicpx -I../inc matrix_cal_mpi.cpp -o matrix_cal_mpi
mpirun -np 4 ./matrix_cal_mpi

mpiicx --version
#Intel(R) oneAPI DPC++/C++ Compiler 2024.0.0 (2024.0.0.20231017)
# Target: x86_64-unknown-linux-gnu
# Thread model: posix
# InstalledDir: /opt/intel/oneapi/compiler/2024.0/bin/compiler
# Configuration file: /opt/intel/oneapi/compiler/2024.0/bin/compiler/../icx.cfg

mpicc --version
# gcc (GCC) 12.3.0
# Copyright (C) 2022 Free Software Foundation, Inc.
# This is free software; see the source for copying conditions.  There is NO
# warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```
#### OPENMPI 和 Intel MPI 的版本
- mpirun (Open MPI) 4.1.6
- Intel® MPI Library (version 2021.11.0)

#### 不同 MPI 实现编译的性能比较

前面有openmpi的

比较如下：  
|  实验次数/进程数   | 1 | 2 | 4 | 8 |
|  ----  | ----  | ----  | ----  | ----  |
|  1  | 11.137 | 5.68647 | 3.29428 | 3.64855 |
|  2  | 11.0721 | 5.7994 | 3.09677 | 3.68822 |
|  3  | 10.9346 | 5.68283 | 3.03596 | 3.68843 |
|  4  | 11.1715 | 5.72521 | 3.00175 | 3.62148 |

intel mpi

比较如下：  
|  实验次数/进程数   | 1 | 2 | 4 | 8 |
|  ----  | ----  | ----  | ----  | ----  |
|  1  | 2.55897 | 1.48034 | 0.779943 | 1.0831   |
|  2  | 2.54911 | 1.44536 | 0.771445 | 0.974259 |
|  3  | 2.54947 | 1.44413 | 0.802421 | 1.01117  |
|  4  | 2.55782 | 1.44145 | 0.780028 | 0.981472 |

intel赢麻了

#### IPM 工具
```bash
mpicxx -I../inc matrix_cal_mpi.cpp -o matrix_cal_mpi -L$PREFIX/lib -lipm
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/root/IPM/src/.libs/
mpirun -np 4 ./matrix_cal_mpi

#openmpi
##IPMv2.0.6###########################
#
# command   : ./matrix_cal_mpi
# start     : Mon Dec 04 18:28:51 2023   host      : head
# stop      : Mon Dec 04 18:29:29 2023   wallclock : 37.26
# mpi_tasks : 4 on 1 nodes               %comm     : 9.95
# mem [GB]  : 0.40                       gflop/sec : 0.00
#
#           :       [total]        <avg>          min          max
# wallclock :        149.04        37.26        37.26        37.26
# MPI       :         14.83         3.71         1.07         5.70
# %wall     :
#   MPI     :                       9.95         2.88        15.31
# #calls    :
#   MPI     :           304           76           76           76
# mem [GB]  :          0.40         0.10         0.09         0.12
#
#################################

mpiicpx -I../inc matrix_cal_mpi.cpp -o matrix_cal_mpi -L$PREFIX/lib -lipm
#intelmpi
##IPMv2.0.6###########################
#
# command   : ./matrix_cal_mpi
# start     : Mon Dec 04 18:32:29 2023   host      : head
# stop      : Mon Dec 04 18:32:38 2023   wallclock : 8.85
# mpi_tasks : 4 on 1 nodes               %comm     : 9.96
# mem [GB]  : 0.37                       gflop/sec : 0.00
#
#           :       [total]        <avg>          min          max
# wallclock :         35.38         8.84         8.84         8.85
# MPI       :          3.53         0.88         0.36         1.84
# %wall     :
#   MPI     :                       9.96         4.07        20.75
# #calls    :
#   MPI     :           304           76           76           76
# mem [GB]  :          0.37         0.09         0.08         0.11
#
#################################

```

#### mpi多机
```bash
mpirun -n 10 -hosts client,master ./matrix_cal_mpi
```

#### 不得不品鉴之mpi
https://en.wikipedia.org/wiki/Message_Passing_Interface  
https://docs.open-mpi.org/en/v5.0.x/mca.html  
https://www.open-mpi.org/  
https://stackoverflow.com/questions/66228038/concurrent-communications-in-mpi-with-mpi-thread-multiple-access-level

OPENMPI 的主要功能是通过**模块化组件架构（MCA）**来实现。  
OPENMPI 的软件体系结构包括以下几个层次：
- OPENMPI 层（OMPI）：顶级 MPI API 和支持逻辑
- 开放运行时环境（ORTE）：后端运行时系统的接口
- 开放可移植性访问层（OPAL）：操作系统/实用程序代码（列表、引用计数等）

在处理多种网络通信方面，MPI 保证了一对等级之间的消息是非超越的，这限制了什么和何时可以通信。这并不排除同时传输多个消息，只要网络允许，无论请求是顺序发布的还是来自多个线程。OPENMPI 利用多个网络连接（如果有多个接口提供连接性）。

## 后记
尽管有许多遗憾，未能入选今年的ASC队伍，但学习之路未必不令人欣喜。此处一并感谢老师，学长的教诲和指导，使我有幸一窥门中真知。要学的还有很多，未知在等待我们，

每章节副标题出自纪伯伦《先知》


## 参考链接
[什么是高性能计算 (HPC)？](https://www.ibm.com/cn-zh/topics/hpc)