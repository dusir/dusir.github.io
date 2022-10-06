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

### 整体架构

![KVM整体架构流程图](./assets/img/posts/kvm/KVM整体架构流程图.jpg)

### 整体执行流程

![KVM流程图](./assets/img/posts/kvm/KVM流程图.jpg)

### Linux KVM Intel 入口函数？
入口函数：vmx_init()
所在文件：arch/x86/kvm/vmx.c

1. 读取扩展特性寄存器(MSR_EFER)，保存到全局静态变量host_efer中；
MSR_EFER说明：
   
```
/* x86-64 specific MSRs */

\#define MSR_EFER                0xc0000080 /* extended feature register */
```

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

6. kvm_arch_init()

   检查硬件是否支持虚拟化、BIOS是否禁用虚拟化。

   调用kvm_mmu_module_init初始化mmu, 创建了3个mem_cache，这里还注册了一个mmu_shrinker用于内存不足时的回收。
```
arm的：
/**
 * Initialize Hyp-mode and memory mappings on all CPUs.
 */
int kvm_arch_init(void *opaque)
{
        int err;
        int ret, cpu;

        if (!is_hyp_mode_available()) {  //确定hypervisor模式是否可用
                kvm_err("HYP mode not available\n");
                return -ENODEV;
        }

        for_each_online_cpu(cpu) {
                smp_call_function_single(cpu, check_kvm_target_cpu, &ret, 1);//在smp系统中检查每个cpu是否支持虚拟化。
                if (ret < 0) {
                        kvm_err("Error, CPU %d not supported!\n", cpu);
                        return -ENODEV;
                }
        }

        err = init_hyp_mode(); //初始化hypervisor，本函数的主要目的就是为hypervisor分配页表pgd，该函数将与finalize_hyp_mode后边一起介绍
        if (err)
                goto out_err;

        err = register_cpu_notifier(&hyp_init_cpu_nb);
        if (err) {
                kvm_err("Cannot register HYP init CPU notifier (%d)\n", err);
                goto out_err;
        }

        kvm_coproc_table_init();
        return 0;
out_err:
        return err;
}
```

```
x86的：
int kvm_arch_init(void *opaque)
{
        int r;
        struct kvm_x86_ops *ops = (struct kvm_x86_ops *)opaque;

        if (kvm_x86_ops) {
                printk(KERN_ERR "kvm: already loaded the other module\n");
                r = -EEXIST;
                goto out;
        }

        if (!ops->cpu_has_kvm_support()) {
                printk(KERN_ERR "kvm: no hardware support\n");
                r = -EOPNOTSUPP;
                goto out;
        }
        if (ops->disabled_by_bios()) {
                printk(KERN_ERR "kvm: disabled by bios\n");
                r = -EOPNOTSUPP;
                goto out;
        }

        r = -ENOMEM;
        shared_msrs = alloc_percpu(struct kvm_shared_msrs);
        if (!shared_msrs) {
                printk(KERN_ERR "kvm: failed to allocate percpu kvm_shared_msrs\n");
                goto out;
        }

        r = kvm_mmu_module_init();
        if (r)
                goto out_free_percpu;

        kvm_set_mmio_spte_mask();
        kvm_init_msr_list();

        kvm_x86_ops = ops;
        kvm_mmu_set_mask_ptes(PT_USER_MASK, PT_ACCESSED_MASK,
                        PT_DIRTY_MASK, PT64_NX_MASK, 0);

        kvm_timer_init();
```

7.kvm_irqfd_init
创建工作队列，用于处理vm的shudown事件。
为eventfd创建一个全局的工作队列，它用于在虚拟机被关闭时，关闭所有与其相关的irqfd，并等待该操作完成。
```
/*
 * create a host-wide workqueue for issuing deferred shutdown requests
 * aggregated from all vm* instances. We need our own isolated single-thread
 * queue to prevent deadlock against flushing the normal work-queue.
 */
int kvm_irqfd_init(void)
{
        irqfd_cleanup_wq = create_singlethread_workqueue("kvm-irqfd-cleanup");
        if (!irqfd_cleanup_wq)
                return -ENOMEM;

        return 0;
}
```

8.kvm_vcpu_cache
创建用于分配kvm vcpu的特定的slab缓存
```
      kvm_vcpu_cache = kmem_cache_create("kvm_vcpu", vcpu_size, vcpu_align,
                                           0, NULL);
```

9.kvm_async_pf_init
创建用于分配kvm_async_pf特定的slab缓存
```
        r = kvm_async_pf_init();
        if (r)
                goto out_free;
                
        ------->具体实现如下：
        static struct kmem_cache *async_pf_cache;

        int kvm_async_pf_init(void)
        {
                async_pf_cache = KMEM_CACHE(kvm_async_pf, 0);
                
                if (!async_pf_cache)
                        return -ENOMEM;
        
                return 0;
        }
```

10.misc_register
misc_register用于注册字符设备驱动，在kvm_init函数中调用此函数完成注册，以便上层应用程序来使用kvm模块：

```
        kvm_chardev_ops.owner = module;
        kvm_vm_fops.owner = module;
        kvm_vcpu_fops.owner = module;

        r = misc_register(&kvm_dev);
        if (r) {
                printk(KERN_ERR "kvm: misc device register failed\n");
                goto out_unreg;
        }
```
具体的流程：
![kvm字符设备流程图](./assets/img/posts/kvm/kvm字符设备流程图.jpg)

字符设备的注册分为三级，分别代表kvm, vm, vcpu，上层最终使用底层的服务都是通过ioctl函数来操作；

kvm：代表kvm内核模块，可以通过kvm_dev_ioctl来管理kvm版本信息，以及vm的创建等；
vm：虚拟机实例，可以通过kvm_vm_ioctl函数来创建vcpu，设置内存区间，分配中断等；
vcpu：代表虚拟的CPU，可以通过kvm_vcpu_ioctl来启动或暂停CPU的运行，设置vcpu的寄存器等；
以Qemu的使用为例：

```
1.打开/dev/kvm设备文件；
2.ioctl(xx, KVM_CREATE_VM, xx)创建虚拟机对象；
3.ioctl(xx, KVM_CREATE_VCPU, xx)为虚拟机创建vcpu对象；
4.ioctl(xx, KVM_RUN, xx)让vcpu运行起来；
```

11.register_syscore_ops
注册挂起、恢复时的操作函数

12.kvm sched
```
        kvm_preempt_ops.sched_in = kvm_sched_in;
        kvm_preempt_ops.sched_out = kvm_sched_out;
```

13.kvm debugfs
```
        r = kvm_init_debug();
        if (r) {
                printk(KERN_ERR "kvm: create debugfs files failed\n");
                goto out_undebugfs;
        }

        return 0;
```