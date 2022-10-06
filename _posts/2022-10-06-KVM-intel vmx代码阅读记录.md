---
layout: post
read_time: true
show_date: true
title: "vmx 代码阅读记录"
date: 2018-10-06
img: posts/20210420/post8-rembrandt.jpg
tags: [KVM]
category: opinion
author: Bobo Du
description: "KVM-intel vmx代码阅读记录"
---

### 整体执行流程

![KVM流程图](./assets/img/posts/kvm/KVM流程图.jpg)

### Linux KVM Intel 入口函数？
入口函数：vmx_init()
所在文件：arch/x86/kvm/vmx.c

1. 读取扩展特性寄存器(MSR_EFER)，保存到全局静态变量host_efer中；
MSR_EFER说明：
   
/* x86-64 specific MSRs */

\#define MSR_EFER                0xc0000080 /* extended feature register */

2. 初始化保存全局共享的msr寄存器数组
代码如下：
   
```
for (i = 0; i < NR_VMX_MSR; ++i)
                kvm_define_shared_msr(i, vmx_msr_index[i]);
```   

3. 申请8个空闲页作为位图，所有位全部置1，并且将vmx_io_bitmap_a的0x80置0

```
/*
         * Allow direct access to the PC debug port (it is often used for I/O
         * delays, but the vmexits simply slow things down).
         */
        memset(vmx_io_bitmap_a, 0xff, PAGE_SIZE);
        clear_bit(0x80, vmx_io_bitmap_a);
```



4. 调用kvm_init(), 将vmx_x86_ops注册进去并初始化kvm模块，同时传入的参数还有x86架构下vcpu结构体大小和对齐方式。

```
r = kvm_init(&vmx_x86_ops, sizeof(struct vcpu_vmx),
                     __alignof__(struct vcpu_vmx), THIS_MODULE);
        if (r)
                goto out7;
```

这里之所以将vmx_x86_ops传入kvm_init是因为x86架构下的虚拟化实现有两种，一种是vmx(intel)，一种是svm(amd)，该参数会传给x86.c中的kvm_x86_ops变量，

vmx和svm只能选择一种，kvm_arch_init()中对kvm_x86_ops做了检查。

5. kvm_init()调用特定CPU实现的kvm_arch_init()去做初始化（x86.c）。

6. 接着是调用kvm_arch_init()

   检查硬件是否支持虚拟化、BIOS是否禁用虚拟化。

   调用kvm_mmu_module_init初始化mmu, 创建了3个mem_cache，这里还注册了一个mmu_shrinker用于内存不足时的回收（以后再看）。

