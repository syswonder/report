# QEMU Loongarch

wheatfox 2023.9
enkerewpo@hotmail.com

## QEMU环境配置

本机环境：

![Untitled](QEMU%20Loongarch/Untitled.png)

参考仓库：

[https://github.com/foxsen/qemu-loongarch-runenv](https://github.com/foxsen/qemu-loongarch-runenv)

缺少依赖：`libnettle.so.7`

apt检查libnettle8发现已安装，说明版本过高而缺少7版本的apt安装包（Ubuntu 22的仓库找不到libnettle7）,手动下载deb进行gdebi安装：

```bash
wget http://security.ubuntu.com/ubuntu/pool/main/n/nettle/libnettle7_3.5.1+really3.5.1-2ubuntu0.2_amd64.deb
sudo gdebi libnettle7_3.5.1+really3.5.1-2ubuntu0.2_amd64.deb
```

成功启动

![Untitled](QEMU%20Loongarch/Untitled%201.png)

```bash
Run a loongarch virtual machine.

Syntax: run_loongarch.sh [-b|c|d|D|g|h|i|k|m|q]
options:
b <bios>    use this file as BIOS
c <NCPU>    simulated cpu number
d use -s to listen on 1234 debug port
D use -s -S to listen on 1234 debug port and stop waiting for attach
g     Use graphic UI.
h     Print this Help.
i <initrd> use this file as initrd
k <kernel> use this file as kernel
m <mem> specify simulated memory size
q <qemu> use this file as qemu
```

使用图形启动：

![Untitled](QEMU%20Loongarch/Untitled%202.png)

CPU: Loongson3A5000

qemu目前实现了virtio设备、PCIe控制器、UART口、实时时钟和电源管理端口。未实现：

1. Address Routing Registers
2. IOCSR只能用iocsr指令访问，不能通过MMIO
3. 部分IOCSR未实现：所有Chip Configuration Register和Legacy Interrupts（手册第4章和第11章）
4. 未实现GPIO
5. 未实现温度传感器
6. 未实现DDR4 SDRAM控制器
7. 未实现HyperTransport控制器
8. 未实现UART1, SPI, I2C

## 中断控制

中断路径: sources -> level3 -> level 2 -> level 1

```
- level 1: CPU core interrupt controller(HWI0-HWI7)
- level 2: extended Interrupt controller(extioi_pic in qemu, 256 irqs)
- level 3: LS7A1000 interrupt controller(pch_pic in qemu, 64 irqs; pch_msi, 192 irqs, refer to 7A1000 user manual)
- interrupt sources:
    - 4 PCIE interrupt output connect to pch_pic 16,17,18,19
    - LS7A UART/RTC/SCI connect to pch_pic 2/3/4
```

## 有关CPU核中断控制器

1. 有些CPU产生的中断直接连接本控制器，如Stable Timer、Performance Counter、IPI等。

## 有关拓展IO中断控制

1. 每个拓展IO中断irq被映射到CPU核中断控制器HWI0-7
    - 默认所有256个irq被路由到HWI0(CPU int2)
    - 默认情况下所有irq路由到node 0 core 0
    - 不支持轮转分发
2. qemu实现情况：
    - 表11-6. iocsr 0x420 只实现了读，写无效. EXT_INT_en默认打开
    - 表11-7. 基本上总是enabled
    - 表11-8. 可读写，但不会生效
    - 表11-9, 未实现
    - 表11-10, 实现
    - 表11-12, 实现
    - 表11-13/14/15 基本实现，但不支持轮转

## LS7A1000中断控制器

1. 所有pch-pic irq被映射到extioi_pic irq，使用的是HT Message Interrupt向量（offset `0x200-0x238`），默认所有irq映射到extioi input 0
2. qemu实现情况：
    - offset 0/0x20/0x60/0x80/0x380/0x3a0/0x3e0 实现
    - offset 0x40/0xc0/0xe0/0x100-0x13f, 可读写，但不会生效
    - offset 0x200-0x23f, 默认全零
    - offset 0x300/0x320, 未实现

## 如何打开UART中断

io port at `0x1fe001e0`

1. LS7A1000中断控制器：
    - unmask input pin2 (bit 2 of reg at `0x20`)
    - set edge trigger or level trigger mode(`0x60`)
    - set ht message interrupt vector (byte at offset `0x202` equal the target extioi irq number)
2. 设置extioi
    - enable mapped extioi irq
3. 设置cpu core ieq
4. 打开global irq

## 当UART irq触发时的操作

1. 响应cpu中断控制器
    - 需要清除中断源，即下一步的动作
2. 响应extioi中断控制器
    - iocsr `0x1800-`写入1
3. 响应LS7A1000中断控制器
    - 写`1<<irq`到intclr寄存器
4. 接收串口字符

# Loongarch手册笔记

[loongsonlab/qemu](https://gitee.com/loongsonlab/qemu)

[龙芯架构文档](https://loongson.github.io/LoongArch-Documentation/README-CN.html)

[在x86平台体验龙芯LoongArch--使用Qemu-7.2安装LoongArch版的ArchLinux_小菜刀_的博客-CSDN博客](https://blog.csdn.net/mxcai2005/article/details/129631722)

[qemu运行虚拟机无反应，只输出一行提示信息:VNC server running on 127.0.0.1:5900_Imagine Miracle的博客-CSDN博客](https://blog.csdn.net/qq_36393978/article/details/118353939?ydreferer=aHR0cHM6Ly93d3cuZ29vZ2xlLmNvbS8=)

编译安装支持Loongarch64的LLVM16+Clang16（Ubuntu22只能apt安装到14）

![Untitled](QEMU%20Loongarch/Untitled%203.png)

[https://github.com/sunhaiyong1978/CLFS-for-LoongArch](https://github.com/sunhaiyong1978/CLFS-for-LoongArch)

[国产loongarch64（龙芯）GCC的初体验](http://www.taodudu.cc/news/show-6268324.html?action=onClick)

之后我下载了官方的cross compile工具包，编译简单的loongarch64程序成功，这里使用的qemu是我自己编译的版本（Github最新release版src），编译时只打开了`qemu-loongarch64`的target。

![Untitled](QEMU%20Loongarch/Untitled%204.png)

![Untitled](QEMU%20Loongarch/Untitled%205.png)

查看生成的loongarch汇编：

![Untitled](QEMU%20Loongarch/Untitled%206.png)

## 要点整理

1. **龙芯架构组成**
    1. 龙芯基础指令集（Loongson Base）
    2. 二进制翻译扩展（LBT）
    3. 向量扩展（LSX）
    4. 高级向量扩展（LASX）
    5. 虚拟化扩展（LVZ）
    
2. 控制状态寄存器（Control and Status Register, **CSR**）
  在实现虚拟化拓展的情况下，处理器中会有两套CSR，一套属于Host，一套属于Guest。

3. **龙芯存储访问类型**
    1. 一致可缓存（Coherent Cached）**CC**
    *最终存储对象+缓存*
    2. 强序非缓存（Strongly-ordered UnCached）**SUC**
    *最终存储对象*
    满足顺序一致性，访存操作依次执行
    3. 弱序非缓存（Weakly-ordered UnCached）**WUC**
    *最终存储对象*
    读访问允许推测执行，写访问则可以内部合并为更大规模，如一个Cache行，之后采用Burst方式写入
    
    其类型在页表项的MAT（Memory Access Type）中标记，龙芯架构要求只有SUC类型的访存指令不能用副作用，即不可推测地执行，通常用于访问IO设备。但是，龙芯架构允许SUC的**取指**操作有副作用。
    
    WUC通常用于加速非缓存内存数据，如显存。
    
4. **原子访存指令**
  AM*, LL, SC
  原子地进行“读+修改+写”操作

5. **同步操作指令**
  龙芯的存储一致性为弱一致性（Weakly Consistency）WC
  使用同步操作保护写共享单元，保证多个处理器核对写共享单元的访问是互斥的
  `dbar`,`ibar`,带有dbar功能的AM原子访存指令,`LL-SC`指令对

6. **龙芯特权架构**
    
    1. 特权等级
    `PLV0-PLV3`，其中`PLV0`权限最高，可以执行特权指令
    `CSR.CRMD.PLV`域指示当前运行在哪个特权等级
    2. CSR访问指令
        1. `csrrd` 将CSR的值写入rd
        2. `csrwr` 将rd的值写入CSR，同时将CSR的旧值写入rd
        3. `csrxchg` 根据rj中的掩码信息，只将rd写入CSR对于掩码为1的位置，其余值不变，同时将CSR的旧值写入rd
    3. 上述指令中需要`csr_num`由14位立即数给出。（0号CSR的csr_num=0，依此类推）
    4. IOCSR访问指令
        1. `iocsrrd` IOCSR采用直接地址映射方式，地址来自寄存器rj
        2. `iocsrwr`
    5. IOCSR寄存器通常可以被多个处理器core同时访问，多个处理器核上的IOCSR访问指令满足顺序一致性（[https://blog.laisky.com/p/sequential-consistent/](https://blog.laisky.com/p/sequential-consistent/)）*sequential consistency*。
    [http://zhangtielei.com/posts/blog-distributed-strong-weak-consistency.html](http://zhangtielei.com/posts/blog-distributed-strong-weak-consistency.html)
    6. CACHE维护指令
    7. TLB维护指令
        1. `tlbsrch` 若未实现LVZ，则使用`CSR.ASID`和`CSR.TLBEHI`的信息去查TLB，如果命中，则将索引写入`CSR.TLBIDX.Index`，将`CSR.TLBIDX`的NE=0，若未命中则NE=1
        2. `tlbrd` 若未实现LVZ，则将`CSR.TLBIDX.Index`作为索引去读TLB
        3. `tlbwr` 若未实现LVZ，将CSR中存放的页表项信息依据`CSR.TLBIDX.Index`存入TLB（CSR.TLBEHI, CSR.TBLELO0, CSR.TLBLO1, CSR.TLBIDX.PS）
        4. `tlbfill`
        5. `tlbclr` 维持TLB与内存页表数据的一致性，依据`CSR.TLBIDX.Index`，当其落入STLB范围时，执行一条tlbclr，将STLB中由Index低位所指示的那一组中的所有路中G=0且ASID=`CSR.ASID.ASID`的页表项无效掉。
        6. `tlbflush` 当Index落在MTLB范围内时，执行tlbflush，将MTLB中所有页表置无效；若落在STLB中则将Index低位所指示的那一组所有路的页表无效掉
        7. `invtlb [op,rj,rk]` ，主要用于无效TLB中的内容，详情见《LoongArch-Vol1-v1.02-CN》P75页操作表格，op=0x0-0x6对应不同的操作。
    8. 软件遍历页表指令
       
       
        | level | CSR |
        | --- | --- |
        | 1 | `CSR.PWCL.PT` |
        | 2 | `CSR.PWCL.Dir1` |
        | 3 | `CSR.PWCL.Dir2` |
        | 4 | `CSR.PWCL.Dir3` |
        1. `lddir [rd,rj,level]` level指定当前访问的是哪一级页表
            1. 若rj[6]=0，此时rj中是level级页表的基址的物理地址，并根据当前的TLB重填地址（？）访问level页表，取得level+1级的页表基址。
            2. 若rj[6]=1，则这时一个大页（Huge Page）的页表项，rj的值直接写入rd。
        2. `ldpte [rj.req]` 若rj[6]=0则rj内容是PTE那一级页表的基址的物理地址，根据当前TLB重填地址访问PTE级页表，取回页表项写入对应CSR中，若rj[6]=1则直接将rj中的值转换为最终的页表项格式写入CSR。
    9. 其他指令
        1. `ertn` 从例外处理返回
        2. `dbcl` 进入调试模式
        3. `idle` 处理器核等待，直至被中断唤醒或被复位
    
7. **虚拟地址空间**
    1. 直接地址翻译模式
    物理地址默认等于虚拟地址`[PALEN-1:0]`位
    2. 映射地址翻译模式
        1. 直接映射地址翻译模式
        配置映射窗口
        `CSR.DMW0`-`CSR.DMW3`寄存器
        虚地址的`[PALEN-1:0]`位与物理地址一致，但高位需要和配置窗口寄存器中的`VSEG`域相等。
        2. 页表映射地址翻译模式
    
8. **页表映射存储管理**
    1. 包含两种TLB：STLB和MTLB，前者页大小一致（`CSR.STLBPS`的PS域配置），后者每一个页表项对应的页可以不一样大。
    2. STLB多路组相联、MTLB全相联。
       
        ![Untitled](QEMU%20Loongarch/Untitled%207.png)
        
    3. 页表项奇偶共用，即不保存奇偶位置，由虚页号最低位判断。
    4. TLB相关例外：
        1. **TLB重填例外**
        访存的虚地址在TLB中没找到，进行TLB重填
        2. **load操作页无效例外**
        load操作找到了但V=0
        3. **store操作页无效例外**
        store操作找到了但V=0
        4. **取指操作无效例外**
        取指找到了但V=0
        5. **页特权等级不合规例外**
        V=1但特权不合规
        6. **页修改例外**
        store且V=1，特权合规，但D=0
        7. **页不可读例外**
        load且V=1，特权合规，但NR=1
        8. **页不可执行例外**
        取指且V=1，特权合规，但NX=1
    5. TLB的初始化 - invtlb r0.r0
       
        ![Untitled](QEMU%20Loongarch/Untitled%208.png)
        
        ![Untitled](QEMU%20Loongarch/Untitled%209.png)
        
    
9. **例外与中断**
    1. 中断
        1. 中断类型
        1个核间中断（IPI），1个定时器中断（TI），1个性能检测计数溢出中断（PMI），8个硬中断（HWI0-HWI7），2个软中断（SWI0-SWI1）。
        2. 中断号越大优先级越高
        3. 中断入口
        在计算入口地址时，中断对应的例外号=自身的中断号+64，中断信号被采样至CSR.ESTAT.IS域。
    2. 例外
        1. TLB重填例外入口来自`CSR.TLBRENTRY`
        2. 机器错误例外入口来自`CSR.MERRENTRY`
        3. 其他例外称为普通例外，入口地址为“入口页号|页内偏移“的计算方式（按位或），入口页号来自`CSR.EENTRY`
        入口偏移=$2^{\text{CSR.ECFG.VS}+2}\times (\text{ecode}+64)$
        4. 例外优先级：中断大于例外、取指阶段产生的优先级最高、译码次之、执行次之。
    
10. **控制状态寄存器一览表**

![Untitled](QEMU%20Loongarch/Untitled%2010.png)

![Untitled](QEMU%20Loongarch/Untitled%2011.png)

![Untitled](QEMU%20Loongarch/Untitled%2012.png)

1. 虚拟化LVZ拓展部分暂未公开文档

## 龙芯架构手册发现的一些问题

1. 在小节2.1.7存储访问类型开头，P10，第四段，”即此类指令不可推测的执行“，”的“应为”地“。（LoongArch-Vol1-v1.02-CN）
2. 在小节4.2.4.5中，“所有路中等于G=0”应去掉”等于“。
3. 在小节4.2.5.2 LDPTE中， 指令格式处的 req 应为 seq。

# 龙芯CPU手册（LS3A5000）