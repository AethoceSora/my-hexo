---
title: 高性能计算-基准测试程序Linpack（HPL）
cover: https://s2.loli.net/2022/12/12/SH7Yhl2dIbEpzcm.jpg
tags: High Performance Computing
---


自通用计算机时代开始以来，就出现了各种用于评估计算机性能的基准测试程序。这些程序的性质通常反映了构建计算机的预期目的，同时还提供了可以与制造商的理论性能估计进行比较的经验性能测量。

高性能计算中最广泛使用的基准之一就是Linpack基准，它的起源是一个线性代数运算包，后来被Lapack库和其他竞争对手取代。但Linpack的基准测试程序在以后的日子里继续发挥强大的影响力。

HPL，即(High-Performance *Linpack*)是早期Linpack的衍生产品，其高度并行化的设计，使得它用于评估TOP500超级计算机的性能排名。

## 编译安装HPL

### 环境依赖

在安装HPL之前，系统上需要安装好支持C语言和Fortran77语言的编译器，我个人选择使用gcc与gfortran ，我将用他们来安装BLAS、MPICH以及HPL本体。

#### BLAS

到[BLAS官网](http://www.netlib.org/blas/)下载好源码，使用tar指令解压。进入BLAS文件夹，执行编译指令：

```shell
gfortran -c -O3 *.f 
# 编译所有的 .f 文件，生成 .o文件
 
ar rv libblas.a *.o  
# 链接所有的 .o文件，生成.a 文件
 
cp libblas.a /usr/local/lib  
#将库文件复制到系统库目录
```

#### MPICH

到[MPICH官网](https://www.mpich.org/downloads/)下载源码包，使用tar指令解压。进入mpich文件夹进行配置：

```
./configure --prefix=/home/software/mpich-4.0.3
```

然后`make&&make install`完成安装。

记得编辑`.bashrc`以配置环境变量。可参考下方配置：

```
# mpich

export MPI_HOME=/home/software/mpich-4.0.3

export PATH=$MPI_HOME/bin:$PATH

export PATH=$PATH:$MPI_HOME/include

export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$MPI_HOME/lib

export MANPATN=$MANPATH:$MPI_HOME/man
```

最后激活环境变量即可使用：

```
source .bashrc
```

#### HPL

于[HPL官网](https://netlib.org/benchmark/hpl/)下载源码包，使用`tar`指令解压后进入其`setup`目录。

手写Make文件是一件痛苦的事情，好在这里有一些针对各种体系结构的编译设置实例。对于此实例，我们需要执行`make_generic`脚本以生成用于创建编译设置的模板。

```
cd hpl-2.3/setup
sh make_generic
cp Make.UNKNOWN ..Make.linux
cd ..
```

现在，我们需要修改`Make.linux`文件，以声明hpl2.3目录以及BLAS库的位置.

```
97 LAlib	= -lblas
```

使用-L标志为编译器提供库位置，例如我的BLAS库安装在`/usr/local/lib`中，则将97行更改为：

```
97 LAlib	= -L/usr/local/lib -lblas
```

另外在第70行指定`hpl-2.3`目录的位置，在64行将架构体系的名称从UNKNOWN改为linux

准备好`Make.linux`文件后，发出以下指令来编译HPL。

`make arch=linux`

这将在`bin/linux`目录里创建HPL的可执行文件xhpl。

### 调试HPL参数文件

伴随`xhpl`可执行文件的是一个参数文件(HPL.dat)，用于调整HPL的计算参数。

下面给出一个示例：

```
HPLinpack benchmark input file
Innovative Computing Laboratory, University of Tennessee
HPL.out      output file name (if any)
6            device out (6=stdout,7=stderr,file)
1            # of problems sizes (N)
143360 256000 1000         Ns  
1            # of NBs
384 192 256      NBs 
1            PMAP process mapping (0=Row-,1=Column-major)
1            # of process grids (P x Q)
1 2          Ps  
1 2          Qs  
16.0         threshold
1            # of panel fact
2 1 0        PFACTs (0=left, 1=Crout, 2=Right)
1            # of recursive stopping criterium
2            NBMINs (>= 1)
1            # of panels in recursion
2            NDIVs
1            # of recursive panel fact.
1 0 2        RFACTs (0=left, 1=Crout, 2=Right)
1            # of broadcast
0            BCASTs (0=1rg,1=1rM,2=2rg,3=2rM,4=Lng,5=LnM)
1            # of lookahead depth
0            DEPTHs (>=0)
0            SWAP (0=bin-exch,1=long,2=mix)
1            swapping threshold
1            L1 in (0=transposed,1=no-transposed) form
1            U  in (0=transposed,1=no-transposed) form
0            Equilibration (0=no,1=yes)
8            memory alignment in double (> 0)
```

测试过程中您可以根据节点的硬件配置，调整HPL.dat文件中相关参数，参数的说明如下所示。

- 第5~6行内容。

  ```shell
  1            # of problems sizes (N)，N表示求解的矩阵数量与规模
  143360 256000 1000         Ns  
  ```

  N表示求解的矩阵数量与规模。矩阵规模N越大，有效计算所占的比例也越大，系统浮点处理性能也就越高。但矩阵规模越大会导致内存消耗量越多，如果系统实际内存空间不足，使用缓存、性能会大幅度降低。矩阵占用系统总内存的80%左右为最佳，即N×N×8=系统总内存×80%（其中总内存的单位为字节）。

- 第7~8行内容。

  ```shell
  1            # of NBs，NB表示求解矩阵过程中矩阵分块的大小
  384 192 256      NBs 
  ```

  求解矩阵过程中矩阵分块的大小。分块大小对性能有很大的影响，NB的选择和软硬件许多因素密切相关。NB值的选择主要是通过实际测试得出最优值，一般遵循以下规律：

  - NB不能太大或太小，一般小于384。
  - NB×8一定是缓存行的倍数。
  - NB的大小和通信方式、矩阵规模、网络、处理器速度等有关系。

  一般通过单节点或单CPU测试可以得到几个较好的NB值，但当系统规模增加、问题规模变大，有些NB取值所得性能会下降。因此建议在小规模测试时选择3个性能不错的NB值，再通过大规模测试检验这些选择。

- 第10~12行内容。

  ```shell
  1            # of process grids (P x Q)，P表示水平方向处理器个数，Q表示垂直方向处理器个数
  1 2          Ps  
  1 2          Qs  
  ```

  P表示水平方向处理器个数，Q表示垂直方向处理器个数。P×Q表示二维处理器网格。P×Q=系统CPU数=进程数。一般情况下一个进程对应一个CPU，可以得到最佳性能。对于Intel ® Xeon ®，关闭超线程可以提高HPL性能。P和Q的取值一般遵循以下规律：

  - P≤Q，一般情况下P的取值小于Q，因为列向通信量（通信次数和通信数据量）要远大于横向通信。
  - P建议选择2的幂。HPL中水平方向通信采用二元交换法（Binary Exchange），当水平方向处理器个数P为2的幂时性能最优。
