# ETH Zurich 披露 VMSCAPE 漏洞：AMD 与 Intel CPU 存在虚拟化隔离缺陷

发布时间: 

  

> 论文标题：VMScape: Exposing and Exploiting Incomplete Branch Predictor Isolation
> in Cloud Environments.  
> 论文作者：Jean-Claude Graf; Sandro Rüegge; Ali Hajiabadi; and Kaveh Razavi  
> 原文链接：https://comsec-files.ethz.ch/papers/vmscape_sp26.pdf  
> 发表会议：S&P 2025

苏黎世联邦理工学院（ETH Zurich）计算机安全实验室的最新研究，把 Spectre-BTI
在“云+虚拟化”场景里构建了可实战的端到端攻击——**VMScape** 。研究团队不仅系统勘测了在不同“保护域”（host user(HU)/guest
user(GU)、host supervisor (HS)/guest supervisor (GS)）之间的分支预测隔离是否可靠，还在 **AMD Zen
1–5** 与 **Intel Coffee Lake** 等平台上找到了**新的跨虚拟化分支目标注入（vBTI）原语**
，最终在不修改任何超管代码的前提下，从恶意 KVM Guest 对 QEMU 发起内存泄漏，在AMD Zen 1达到了 **32 B/s**
的泄露速率（示例为磁盘加/解密密钥），完整利用链平均 **1092 s** 可跑通。

![](/images/ETH Zurich 披露 VMSCAPE 漏洞：AMD 与 Intel CPU
存在虚拟化隔离缺陷/1758678522300.png)

攻击场景及受影响微架构

## 论文概述

研究首先**系统化测试** 了 BTB 与
IBP/ITA（间接分支预测/目标阵列）在虚拟化与特权边界上的隔离性，给出了“从哪儿训练→在哪儿被误导”的**vBTI 记号体系**
（vBTIX→Y），并据此绘出了不同 CPU 代际在各个方向上的有效/失效组合：**除了 Intel Raptor Cove/Gracemont
外，其他微架构都对若干 vBTI 原语脆弱** ；其中 **Zen 1–5 全系对 vBTI GU→HU 与 vBTIHU→GU 脆弱**，Zen 1–4
与 Coffee Lake 还额外受 vBTIGS→HU、vBTIHS→GU 影响。

* * *

## 攻击链构造

**选边信道与构造利用链。** 团队观察到 **QEMU 会把 Guest 内存整体映射进其用户态进程地址空间** ，这意味着可以直接利用共享页做
**FLUSH+RELOAD** 侧信道作为“回显板”（论文记为 Observation O11）。随后在 QEMU
原生代码里定位到可重复触发的**推测执行 gadget** 与**披露 gadget** ，拼成泄漏链路（论文给出了具体源码/反汇编片段与
Listing）。

**打破 ASLR。** 由于 AMD Zen 4/5 的 BTB 在误判时并不总是覆盖旧目标，而是把新目标先插入 ITA
保留旧条目，**常见“按地址遍历找受害分支”的办法在这里会爆炸** 。论文改用更契合 Zen4/5 行为的策略，两步解随机：先找推测 gadget
进而得披露 gadget，再找 QEMU 映射的“重载缓冲区”虚拟地址。实测**定位重载缓冲区的中位时间 224 s，成功率 96%** （100 次）。

**延长推测窗口。** Guest 无法像本机攻击那样直接 `clflush` 受害指针，因此研究**反向工程了 AMD Zen 4/5 的非包容
LLC** ，提出了首个针对 Zen4/5 的 **LLC 驱逐集构造** ，并利用 **XOR 索引** 特性把 2 MiB 页内的 L2
集合**一次定位、批量合并** ，**一次成功率达 100%** 。这一步既拉长了推测窗口，也把端到端成功率提上来（论文记为 Observation
O10）。

**触发误预测：利用“缓解开关”的反作用力。** AMD 文档显示，历史（BHB/IBP/ITA）只有在间接分支见过**多个** 目标后才参与预测；如果
Guest 在训练前**执行 IBPB 刷新 BPU** ，可确保受害分支只“见过”训练目标，从而在不精准重放完整历史的情况下也能稳定误导到攻击者目标（论文
Observation O13）。这从侧面说明：**给 Guest 的缓解控制，反过来可被用来操纵Host的预测条目可见性。**

把上述要素拼起来，VMSCAPE 的**完整流程** 就是：用 vBTIGU→HU 劫持 QEMU 用户态的受害分支→在长推测窗口里跑披露 gadget→借
FLUSH+RELOAD 回显泄漏位→先泄出结构信息定位密钥，再按字节读取密钥/秘密；**端到端中位用时 1092 s** ，其中 **326 s**
花在“找分支+找重载缓冲区”，**泄露速率 32 B/s，字节准确率 99.78%** 。

* * *

## 影响

  * • **跨域原语覆盖面：** Zen 1–5 与 Coffee Lake 在多条 vBTI 方向上都有信号；**Raptor Cove/Gracemont** （较新的 Intel 客户端/小核）在表中表现为“充分隔离”。不过**Coffee Lake 的 STIBP 默认并不总开** ，这会额外暴露跨 SMT 的攻击面。
  * • **SEV-SNP/TDX 场景：** 论文报告 **Zen 3** 默认下 **SEV-SNP 不隔离 BTB** ；而 **Zen 5** 上看起来 Host 与 SEV-SNP Guest 之间预测条目已正确隔离（实验条件受限，Zen 4 服务器未测）。
  * • **BHI（分支历史注入）家族的新影子：** 即便 Intel 新 CPU 的 **eIBRS** 在虚拟化场景里能把 BTB 挡住，**BHB 历史** 仍可能被来宾部分操控；Intel 已确认**针对 HU（ Host User）的 vBHI 原语** 在近期 CPU 上**潜在可行** （作者未复现，但得到厂商确认）。

* * *

## “怎么补”：内核与业界采取了什么缓解？

论文提出的基线缓解是 **IBPB-on-VMEXIT** ：**每次从 Guest 退回 Host（VMEXIT）后进入用户态超管前，执行一次 IBPB
把 BPU 清空** 。这在 TMG→H 模型下强制了 guest↔host 之间的 BTB 隔离，但**粗暴版开销很大** 。作者用 UnixBench
评测其几何平均**开销 57%（Zen 5）** 。Linux 内核社区据此做了重要优化：**仅在将要“退出到用户态”（KVM_EXIT→QEMU）时才发
IBPB** （“IBPB before exit to userspace”），把 IBPB
挪出热路径，同时保持等价的安全语义；在计算型负载上**额外开销约 1%（Zen 4）** ，而针对频繁出入用户态的 I/O 压测（fio，virtio
10× 10 GB 随机读写）时，**优化后也有 ~51% 开销** ，接近“重锤方案”的 60%。该补丁**已面向受影响的 AMD Zen 5 与
Intel 新平台（如 Lunar Lake、Granite Rapids）推广。**

* * *

##  时间线

**披露与跟进：** 研究于 **2025-06-07** 通知 AMD/Intel PSIRT，并在 **2025-09-11** 解禁；Linux
维护者基于作者建议合入补丁；漏洞编号 **CVE-2025-40300** 。项目主页提供了**实验/利用代码** 与更多细节。

  

  

