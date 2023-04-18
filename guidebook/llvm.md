# LLVM指导手册
## 写在前面
从这一部分开始，我们将正式开始进入**代码生成**阶段。课程组给出了三种目标码，分别是生成到 **`PCode`** ，**`LLVM IR`** ，以及 **`MIPS`** 。写MIPS需要额外进行**代码优化**的操作。这里建议有时间的同学们可以去提前调研一下这几种代码再进行选择。由于理论课上，以及编译原理的教材上主要介绍了**四元式**，且在课本最后也介绍了PCode，所以采用PCode作为目标码的同学可以主要参考**编译原理教材**。

**`LLVM`** 可能看上去上手比较困难，毕竟我相信大部分同学是第一次接触，而在往年的编译原理课程中，LLVM的代码生成是软件学院的课程要求，指导书也是针对往届软件学院的编译原理实验。在2022年与计算机学院合并之后，课程组虽然也添加了LLVM的代码生成通道，但是由于课程合并后，例如**文法，实验过程，实现要求**等的不同，课程组同学在去年实验中收到了许许多多同届同学关于LLVM的问题，包括**看不懂指导书，无从下手**等问题，所以在今年的指导书中，我们将作出以下改进：
- 根据今年的实验顺序，**重新编排**每一个小实验部分的顺序，使得同学们在实验过程中更加顺畅。
- 对每一个部分进行**相关的说明**，帮助同学们更好地理解LLVM的代码生成过程。
- 会在指导书的每一个章节结束给出一些相对**较强的测试样例**，方便同学们做一个部分就测试一个部分。
- 由于LLVM本身就是一种很优秀的中间代码，所以对于想最终生成到MIPS的同学，今年的指导书中将新增LLVM的**中端优化部分**，帮助同学们更方便地从LLVM生成MIPS代码。

## 0. 简单介绍
### LLVM是什么

**`LLVM`** 最早叫底层虚拟机 (Low Level Virtual Machine) ，最初是伊利诺伊大学的一个研究项目，目的是提供一种现代的、基于SSA的编译策略，能够支持任意编程语言的静态和动态编译。从那时起，LLVM已经发展成为一个由多个子项目组成的伞式项目，其中许多子项目被各种各样的商业和开源项目用于生产，并被广泛用于学术研究。

现在，LLVM被用作实现各种静态和运行时编译语言的通用基础设施（例如，GCC、Java、.NET、Python、Ruby、Scheme、Haskell、D以及无数鲜为人知的语言所支持的语言族）。它还取代了各种特殊用途的编译器，如苹果OpenGL堆栈中的运行时专用化引擎和Adobe After Effects产品中的图像处理库。最后，LLVM还被用于创建各种各样的新产品，其中最著名的可能是OpenCL GPU编程语言。

> 一些参考资料：
> - https://aosabook.org/en/v1/llvm.html#footnote-1
> - https://llvm.org/

### 三端设计
传统静态编译器，例如大多数C语言的编译器，最主流的设计是**三端设计**，其主要组件是前端、优化器和后端。前端解析源代码，检查其错误，并构建特定语言的**抽象语法树 `AST`**(Abstract Syntax Tree)来表示输入代码。AST可以选择转换为新的目标码进行优化，优化器和后端在代码上运行。

**`优化器`** 的作用是**增加代码的运行效率**，例如消除冗余计算。 **`后端`** ，也即代码生成器，负责将代码**映射到目标指令集**，其常见部分包括指令选择、寄存器分配和指令调度。

当编译器需要支持**多种源语言或目标体系结构**时，使用这种设计最重要的优点就是，如果编译器在其优化器中使用公共代码表示，那么可以为任何可以编译到它的语言编写前端，也可以为任何能够从它编译的目标编写后端，如图 0-1 所示。

![](https://github.com/echo17666/BUAA-Compiler2023-llvm-pro/raw/master/image/0-1.png)

##### <p align="center">图 0-1 三端设计示意图</p>

官网对这张设计图的描述只有一个单词 **Retargetablity**，译为**可重定向性**或**可移植性**。通俗理解，如果需要移植编译器以支持新的源语言，只需要实现一个新的前端，但现有的优化器和后端可以重用。如果不将这些部分分开，实现一种新的源语言将需要从头开始，因此，不难发现，支持 $N$ 个目标和 $M$ 种源语言需要 $N×M$ 个编译器，而采用三端设计后，中端优化器可以复用，所以只需要 $N+M$ 个编译器。例如我们熟悉的Java中的 **`JIT`** ， **`GCC`** 都是采用这种设计。

**`IR`** (Intermediate Representation) 的翻译即为中间表示，在基于LLVM的编译器中，前端负责解析、验证和诊断输入代码中的错误，然后将解析的代码转换为LLVM IR，中端（此处即LLVM优化器）对LLVM IR进行优化，后端则负责将LLVM IR转换为目标语言。

### 工具介绍
> 讲道理这一节应该配合下一节一起食用，但是下一节需要用到这些工具，就先放前面写了。

我们的实验目标是将C语言的程序生成为LLVM IR的中间代码，尽管我们会给指导书，但不可避免地，同学们还会遇到很多不会的情况。所以这里给出一个能够自己进行代码生成测试的工具介绍，帮助大家更方便地测试和完成实验。

这里着重介绍 **`Ubuntu`** (20.04或更新) 的下载与操作，一是方便，二是感觉大家或多或少有该系统的Vmware或者云服务器等等。
> 如果真的没有，腾讯云学生优惠有Ubuntu 20.04的云服务器，9.9RMB一年，如果实在不想花钱，也可以在Windows或MacOS上直接装。MacOS和Windows安装Clang和LLVM的方法请自行搜索。

首先安装 **`LLVM`** 和 **`Clang`** 
```bash
$ sudo apt-get install llvm
$ sudo apt-get install clang
```
安装完成后，输入指令查看版本。如果出现版本信息则说明安装成功。
```bash
$ clang -v 
$ lli --version 
```
> **注意：** 请务必保证llvm版本至少是**10.0.0 及以上**，否则**会影响正确性！**

如果使用apt无法安装，则将下列代码加入到 `/etc/apt/sources.list` 文件中
```bash
deb http://apt.llvm.org/focal/ llvm-toolchain-focal-10 main
deb-src http://apt.llvm.org/focal/ llvm-toolchain-focal-10 main
```
然后在终端执行
```bash
wget -O - https://apt.llvm.org/llvm-snapshot.gpg.key|sudo apt-key add -
apt-get install clang-10 lldb-10 lld-10
```
**`MacOS`** 上的安装也稍微提一嘴，需要安装XCode或XCode Command Line Tools, 其默认自带Clang
```bash
xcode-select --install
brew install llvm
```
安装完成后，需要添加LLVM到$PATH
```bash
echo 'export PATH="/usr/local/opt/llvm/bin:$PATH"' >> ~/.bash_profile
```
这时候可以仿照之前查看版本的方法，如果显示版本号则证明安装成功。

我们安装的 **`Clang`** 是 LLVM 项目中 C/C++ 语言的前端，其用法与 GCC 基本相同。
 **`lli`** 会解释.bc 和 .ll 程序。

具体如何使用上述工具链，我们将在下一章介绍。
### LLVM IR示例
LLVM IR 具有三种表示形式，一种是在**内存中**的数据结构格式，一种是在磁盘二进制 **位码 (bitcode)** 格式 **`.bc`** ，一种是**文本格式** **`.ll`** 。生成目标代码为LLVM IR的同学要求输出的是 **`.ll`** 形式的 LLVM IR。

作为一门全新的语言，与其讲过于理论的语法，不如直接看一个实例来得直观，也方便大家快速入门。

例如，我们的源程序 `main.c` 如下
```c
int a=1;
int add(int x,int y){
    return x+y;
}
int main(){
    int b=2;
    return add(a,b);
}
```
现在，我们想知道其对应的LLVM IR长什么样。这时候我们就可以用到Clang工具。下面是一些常用指令
```bash
$ clang main.c -o main # 生成可执行文件
$ clang -ccc-print-phases main.c # 查看编译的过程
$ clang -E -Xclang -dump-tokens main.c # 生成 tokens
$ clang -fsyntax-only -Xclang -ast-dump main.c # 生成语法树
$ clang -S -emit-llvm main.c -o main.ll -O0 # 生成 llvm ir (不开优化)
$ clang -S main.c -o main.s # 生成汇编
$ clang -c main.c -o main.o # 生成目标文件
```
输入 `clang -S -emit-llvm main.c -o main.ll` 后，会在同目录下生成一个 `main.ll` 的文件。在LLVM中，注释以';'打头。
```llvm
; ModuleID = 'main.c'     
source_filename = "main.c"  
target datalayout = "e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128"
target triple = "x86_64-pc-linux-gnu"


; 从下一行开始，是实验需要生成的部分，注释不要求生成。
@a = dso_local global i32 1, align 4 

; Function Attrs: noinline nounwind optnone uwtable 
define dso_local i32 @add(i32 %0, i32 %1) #0 {
  %3 = alloca i32, align 4
  %4 = alloca i32, align 4
  store i32 %0, i32* %3, align 4
  store i32 %1, i32* %4, align 4
  %5 = load i32, i32* %3, align 4
  %6 = load i32, i32* %4, align 4
  %7 = add nsw i32 %5, %6
  ret i32 %7
}

; Function Attrs: noinline nounwind optnone uwtable 
define dso_local i32 @main() #0 {
  %1 = alloca i32, align 4
  %2 = alloca i32, align 4
  store i32 0, i32* %1, align 4
  store i32 2, i32* %2, align 4
  %3 = load i32, i32* @a, align 4
  %4 = load i32, i32* %2, align 4
  %5 = call i32 @add(i32 %3, i32 %4)
  ret i32 %5
}

; 实验要求生成的代码到上一行即可

attributes #0 = { noinline nounwind optnone uwtable ...}
; ...是我自己手动改的，因为后面一串太长了

!llvm.module.flags = !{!0}
!llvm.ident = !{!1}

!0 = !{i32 1, !"wchar_size", i32 4}
!1 = !{!"clang version 10.0.0-4ubuntu1 "}
```
粗略一看，LLVM IR很长很麻烦，但仔细一看，在我们需要生成的代码部分，像是一种特殊的三元式。事实上，LLVM IR使用的是**三地址码**。我们对上述代码进行简要注释。

