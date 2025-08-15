# kvm
## 配置
kvm(linux): 6.6.30+

## kvm初始化
kvm初始化图参考如下：  
![kvm初始化](/picture/kvm_init.png)

### module_init(kvm_arm_init)
在 ARM64 的典型环境中，kvm_arm_init 是作为内置模块在内核启动时初始化的，确保虚拟化功能尽早可用。  

### kvm_arm_init
该函数负责检测硬件支持、设置关键虚拟化组件并初始化KVM子系统。  
通过源代码分析，我自动省略了一些不重要的部分。
```c
static __init int kvm_arm_init(void)
{
    if (!is_hyp_mode_available()) { ... }
    if (kvm_get_mode() == KVM_MODE_NONE) { ... }                             (1)

    err = kvm_sys_reg_table_init();                                          (2)

    in_hyp_mode = is_kernel_in_hyp_mode();                                   (3)

    if (cpus_have_final_cap(ARM64_WORKAROUND_DEVICE_LOAD_ACQUIRE) || ...)    (4)

    err = kvm_set_ipa_limit();                                              
    err = kvm_arm_init_sve();                                                
    err = kvm_arm_vmid_alloc_init();                                         (5)

    if (!in_hyp_mode) { err = init_hyp_mode(); }                             (6)

    err = kvm_init_vector_slots();                                           (7)

    err = init_subsystems();                                                 (8)

    err = kvm_init(sizeof(struct kvm_vcpu), 0, THIS_MODULE);                 (9)

    if (!in_hyp_mode) finalize_init_hyp_mode();                              (10)

    kvm_arm_initialised = true;                                              (11)
}
```
1. 硬件虚拟化支持检测  
Hyp模式检查：确认CPU支持ARM虚拟化扩展（EL2异常级别），否则无法运行虚拟机。  
命令行禁用检查：若内核启动参数禁用KVM（如kvm.arm=off），直接返回错误。 
2. 系统寄存器表初始化  
构建ARM64系统寄存器（如SCTLR_EL1、TTBR0_EL1）的虚拟化配置表，用于后续guest/host寄存器上下文切换。
3. 运行模式判断
检测当前内核是否已在Hyp模式（VHE模式）下运行：  
VHE模式（内核在EL2）：直接管理虚拟化资源。  
非VHE模式（内核在EL1）：需通过Hypervisor调用切换到EL2。  
4. 安全警告处理
打印CPU勘误警告：某些ARM芯片缺陷可能导致未经修补的Guest系统死锁，需仅运行受信任的虚拟机。
5. 关键资源初始化
(a) IPA地址空间限制  
设置Guest物理地址（IPA）空间大小（如40/48位），影响Stage2页表层级结构。  
(b) SVE支持初始化    
若CPU支持可伸缩向量扩展（SVE），初始化相关虚拟化支持（如保存/恢复SVE寄存器）。  
(c) VMID分配器   
初始化虚拟机标识符（VMID）分配器，用于TLB隔离（ARMv8.1+的VMID特性）。  
6. Hyp模式初始化（非VHE场景）  
动态加载Hyp代码：将KVM的Hyp部分（如异常向量表、EL2处理代码）加载到安全内存。  
配置EL2环境：设置Hyp模式的页表、定时器虚拟化等。  
7. 中断向量槽初始化  
为不同虚拟化事件（如Guest退出到Host）分配处理函数槽位，ARM64可能包括：  
系统寄存器访问退出  
内存访问异常（Stage2缺页）  
虚拟中断注入  
8. 子系统初始化  
启动KVM子模块：  
内存虚拟化：Stage2页表管理。  
虚拟设备：如GICv3中断控制器、Arch定时器。  
调试支持：处理Guest调试陷阱。  
9. 主KVM框架初始化  
注册KVM核心框架：  
分配kvm_vcpu结构体内存池（含ARM64特有的kvm_vcpu_arch字段）。  
创建设备节点/dev/kvm，暴露ioctl接口。  
10.    后期Hyp模式配置  
安全锁定：在非VHE模式下，完成Hyp代码的最终保护（如pKVM的内存隔离）。  
11.   状态标记与错误处理  
标记KVM已成功初始化，后续可创建虚拟机。若失败则逆向释放资源（如释放VMID、卸载Hyp代码）。  

### kvm_init






# 参考文献
[linux KVM原理分析](https://zhuanlan.zhihu.com/p/1923765014822643341)  
[QEMU/KVM源码分析之——虚拟机创建流程 ](https://leo-hou.github.io/2022/03/13/QEMU-KVM%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B9%8B%E2%80%94%E2%80%94%E8%99%9A%E6%8B%9F%E6%9C%BA%E5%88%9B%E5%BB%BA%E6%B5%81%E7%A8%8B/)  
[浅谈KVM源码——kvm_init](https://zhuanlan.zhihu.com/p/549823413)  
[Linux虚拟化](https://www.cnblogs.com/LoyenWang/category/1828942.html) 