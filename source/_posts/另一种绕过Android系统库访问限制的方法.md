---
title: 另一种绕过Android系统库访问限制的方法
date: 2021-05-26 13:07:43
categories:
- Arm汇编
- Android
- 访问系统库
tags:
- Android
- 汇编
---


## 问题来源
从Android 7.0开始，Android系统开始阻止App中使用dlopen(), dlsym()等函数打开系统动态库。但是一些大型App在做性能监测和优化时，经常需要使用dl函数打开系统动态库。因此，有必要想办法绕过系统的这种限制。

## 限制App访问系统库原理
我们查阅Android11，看看dlopen()函数是怎样实现的：

<!-- more -->

```
// bionic/libdl/libdl.cpp
__attribute__((__weak__))
  void* dlopen(const char* filename, int flag) {
    const void* caller_addr = __builtin_return_address(0);
    return __loader_dlopen(filename, flag, caller_addr);
  }
```
通过源码可知，在真正调用__loader_dlopen()之前，先调用了__builtin_return_address(0), 获取caller_addr。这里，__builtin_return_address是Linux一个内建函数，__builtin_return_address(0)用于返回当前函数的返回地址。ARM架构里，LR寄存器里存的也就是当前函数的返回地址，因此__builtin_return_address(0)获取的就是当前LR寄存器的值。  

继续查看__loader_dlopen源码，最后执行到了do_dlopen：

```
// bionic/linker/linker.cpp
void* do_dlopen(const char* name, int flags,
                  const android_dlextinfo* extinfo,
                  const void* caller_addr) {
    std::string trace_prefix = std::string("dlopen: ") + (name == nullptr ? "(nullptr)" : name);
    ScopedTrace trace(trace_prefix.c_str());
    ScopedTrace loading_trace((trace_prefix + " - loading and linking").c_str());
    soinfo* const caller = find_containing_library(caller_addr);
    android_namespace_t* ns = get_caller_namespace(caller);
    
    ...
```
caller_addr传给了函数find_containing_library, 用于获取包含此地址的动态库的信息。  
find_containing_library函数的实现流程也比较简单，先遍历所有打开的动态库，在遍历每个动态库函数的地址段，对比caller_addr在不在此动态库中，在的话，则返回此动态库的信息。

```
// bionic/linker/linker.cpp
soinfo* find_containing_library(const void* p) {
    // Addresses within a library may be tagged if they point to globals. Untag
    // them so that the bounds check succeeds.
    ElfW(Addr) address = reinterpret_cast<ElfW(Addr)>(untag_address(p));
    for (soinfo* si = solist_get_head(); si != nullptr; si = si->next) {
      if (address < si->base || address - si->base >= si->size) {
        continue;
      }
      ElfW(Addr) vaddr = address - si->load_bias;
      for (size_t i = 0; i != si->phnum; ++i) {
        const ElfW(Phdr)* phdr = &si->phdr[i];
        if (phdr->p_type != PT_LOAD) {
          continue;
        }
        if (vaddr >= phdr->p_vaddr && vaddr < phdr->p_vaddr + phdr->p_memsz) {
          return si;
        }
      }
    }
    return nullptr;
  }
```
通过以上分析可知，系统限制app调用dlopen的方法，是通过检查执行dlopen函数时的LR寄存器值是不是系统库的地址。那应该如果绕过这种检查呢？下面介绍一种简单绕过方法。

## 绕过方法
根据以上分析，调用dlopen时，如果能把LR寄存器的值改成系统库的某个地址，应该就能骗过系统校验。但是，若改LR寄存器的值为任意系统库地址，这会导致函数调用结束后，无法回到调用dlopen函数后面的代码继续执行。因为在Arm32位处理器中，LR寄存器用于保存子程序返回地址， 在使用BL或BLX进行跳转时，跳转指令自动把返回地址放入LR寄存器中，子程序执行结束时，通过把LR复制到PC来实现程序的返回。

所以，在修改LR寄存器值时，我们还需要确保函数执行完能返回到原来LR寄存器存的地址开始执行。因此，修改LR之前，需要先保存原来LR寄存器中的值，函数执行完后再恢复回来，这样才能实现正确返回。 

下面以dlopen函数为例来详细介绍实现方案。

## 汇编实现
为了实现修改LR寄存器的值，我们不能直接调用dlopen函数，需要使用一个跳板函数来调用dlopen函数，并且还要确保跳板函数跳转到dlopen函数时，不修改LR寄存器的值。

Arm32中，实现指令的跳转有两种方法：
 1. 使用专门的跳转指令：B, BX, BL, BLX
 2. 直接向程序计数器PC写入跳转地址值： MOV PC, R0; POP {R4, PC}等

使用跳转指令跳转到目标地址，是一种短跳转，最多只能实现向前或向后32MB的地址空间跳转，也就是说这种一般都是模块内的局部跳转。而通过向程序计数器PC写入跳转地址值的方式，是一种长跳转，可以实现在4GB的地址空间中的任意跳转，并且这种跳转不会修改LR寄存器的值。另外，bl register, blx register这类指令也可以在全地址空间范围内跳转。


因此，我们选择通过修改PC寄存器的值跳转到dlopen函数。

假如我们已知一个系统库的地址是sys_addr, 实现修改LR为sys_addr并跳转到dlopen的汇编实现就是：
```
    mov lr, sys_addr   // 修改lr寄存器的值为系统库地址
    mov pc, dlopen     // 跳转到dlopen函数
```
单纯地这样处理会存在一个问题，就是dlopen函数执行完后，无法返回到调用处继续执行后面的代码。因此，我们需要将原来的LR寄存器的值保存起来，dlopen函数执行完成后，再恢复原来LR寄存器的值，并跳到对应的地址开始执行。

Arm汇编中，局部对象一般是保存在栈上。因此，我们使用push指令将LR寄存器的值保存到栈上，dlopen执行完后，再使用pop指令将保存在栈上的LR值恢复到PC寄存器中，这样就能返回到原来的位置开始执行。
指令如下：
```
    push {r4, lr}      // 将原lr保存到栈上。这里，r4可以是r0-r7中的任意一个，这是push指令的必选参数，为了指令对齐
    mov lr, sys_addr   // 修改lr寄存器的值为系统库地址
    mov pc, dlopen     // 跳转到dlopen函数
```
通过这跳板指令，dlopen函数执行完后，跳到lr寄存器中的地址开始执行，也就是sys_addr这个地址上。所以，我们希望sys_addr这个位置对应的指令是：
```
     pop {r4, pc}    // 将栈上存的原lr寄存器的值恢复到pc寄存器中
```
这样，就能将原来的LR寄存器的地址从栈上弹出到PC寄存器中。从而能回到原来调用代码的位置后面开始执行接下来的指令。

其实，熟悉Arm汇编的人应该都知道，push {r0-r7, lr}跟与之对应的pop {r0-r7, pc}, 是大多数函数的第一条汇编指令和最后一条汇编指令。分别对应汇编中函数的序言准备(Prologue)和结束收尾(Epilogue)。序言的目的是为了保存函数执行之前的状态(通过存储LR以及R0-R7到栈上)。收尾的目的主要是用来恢复序言中保存程序寄存器的值以及回到函数调用发生之前的状态。

## 获取系统库地址

上面的汇编代码中，保存到LR寄存器中的sys_addr目前还是未知的，假如能取到这个地址，就能完美解决问题。通过上述分析，这个地址只要满足这两个条件就行，第一，是系统库中的地址，第二，地址对应的指令是`pop {r4, pc}`。

这里，有两种方法能取到这样的地址：  
1. 从系统库的so文件代码区中搜索出一个指令为`pop {r4, pc}`的地址；
2. 修改系统库某个已知地址对应的指令为`pop {r4, pc}`；

这里，我们采用第一种搜索的方式。分以下几个步骤来进行：
1. 遍历`/proc/self/maps`文件，找到so文件在内存中的基地址；
2. 将so文件通过mmap映射到内存中；
3. 通过映射到内存中的elf header，读出section header的偏移(e_shoff)，每个section header的size(e_shentsize)以及section header的数量(e_shnum);
4. 根据offset, size和number遍历section header, 找到name为.text对应的节区，此节区中包含程序的可执行指令；
5. 遍历.text节区中的所有指令，找到`pop {r4, pc}`指令(0xBD10)对应的偏移量；
6. 偏移量加上so文件的基地址就是指令对应的内存地址。

将搜索到的地址替换为汇编代码中的sys_addr即可。

## 最后
按照上面的思路，完整实现代码已上传到Github，欢迎star.  
[bypass_dlfunctions](https://github.com/WindySha/bypass_dlfunctions)
