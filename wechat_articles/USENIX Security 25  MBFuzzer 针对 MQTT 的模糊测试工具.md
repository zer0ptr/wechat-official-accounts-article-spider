# USENIX Security 25 | MBFuzzer: 针对 MQTT 的模糊测试工具

> Title: MBFuzzer: A Multi-Party Protocol Fuzzer for MQTT Brokers
>
> Conference: USENIX Security 25
>
> PDF: https://www.usenix.org/system/files/usenixsecurity25-song-xiangpu.pdf
>
> Artifact: https://zenodo.org/records/14942763

## 引言

MQTT是一种广泛应用于IoT场景的多方通信协议。在通信过程中，MQTT brokers作为服务端与其他设备建立连接。  
当前针对MQTT broker的模糊测试存在输入空间不足的限制，因为它们主要采用two-party fuzzing 模型。在这种测试模型下，MQTT
broker中处理多方通信的代码不能得到有效检测。另外，现有的fuzzer专注于发现内存安全问题或逻辑问题，并没有考虑broker的代码实现是specification-
compliant（代码实现与协议规范一致）。

针对上述问题，本文提出了一种黑盒fuzz方法——MBFuzzer。首先，本文设计了一个包含两个Sender的multi-party
模糊测试框架，其输入空间能够覆盖broker中与处理多方通信相关的代码（路由和分发）。为提高测试效率，本文设计了一个message priority
scheduler，dependency rules和一个shared dependency
queue指导测试用例生成并协调两个Sender的消息发送。本文利用了差分测试的思想检测non-compliant
bugs，并且设计了一个基于LLM的non-compliant分析方法自动分析漏洞真实性和成因。

本文实现了一个MBFuzzer的原型，并且应用它测试了6个主流的MQTT
broker。测试中MBFuzzer成功发现73个bug（包含20个内存类bug和53个non-compliant
bug）。与最先进的fuzzers相比，MBFuzzer在代码覆盖率和漏洞发现能力上更为出色。

## 背景

MQTT使用P/S消息传输模型，主要包含4个重要组件：publisher，subscriber，broker，topic。  
下图展示了一个常见的MQTT通信过程：

![](/images/USENIX Security 25  MBFuzzer 针对 MQTT 的模糊测试工具/1758678528500.png)

## 动机

下图所示为CVE-2024-42655的原理图：

![](/images/USENIX Security 25  MBFuzzer 针对 MQTT 的模糊测试工具/1758678528958.png)

具体流程如下：

  1. 1\. broker读取了ACL config，配置禁止sub用户获取以$SYS开头的topic消息；
  2. 2\. subscriber使用用户名sub与broker建立连接
  3. 3\. subscrier订阅`+/+/+`主题（+是通配符）
  4. 4\. publisher与broker建立连接（CONNECT→CONNACK）
  5. 5\. publisher推送以$SYS开头的topic消息
  6. 6\. broker路由并将该消息转发给用户sub，此举违反了配置规则

漏洞的根本成因是没有遵守协议规范——’The server must not match topic filters starting a wildcard
character with topic names beginning with a $
character’。并且可推测，现有的fuzzer由于没有完整的publish/subscribe过程，不能触发broker的转发动作以及缺乏对non-
compliant bug的检测机制，无法发现这样的漏洞。

## 主要问题

  1. 1\. 如何协调两个不同的Sender之间的消息发送：不同于只有一个Sender的fuzzer，在一条测试用例生成之后，MBFuzzer需要确定该消息由哪个Sender发送。随机选择Sender发送忽视了两个Sender之间消息的依赖性
  2. 2\. 如何利用fuzzing的反馈提供漏洞发现效率：需要有合适的优化机制利用反馈指导black-box fuzzer分配更多资源发送更有可能触发漏洞的消息
  3. 3\. 如何有效精确地确认non-compliance bug并且得出成因

## 设计与实现

### Overview

![](/images/USENIX Security 25  MBFuzzer 针对 MQTT 的模糊测试工具/1758678529389.png)

上图展示了MBFuzzer的总体架构，具体的工作流程如下：

  1. 1\. 手动提取MQTT协议的dependency rules，提供给Message Sending Coordinatation
  2. 2\. 在fuzzing循环中，Testcase Generation为每个Sender生成测试用例
  3. 3\. Message Sending Coordinatation通过影响测试用例生成协调两个Sender的消息发送
  4. 4\. Bug Detection通过监测socket status和差分测试，检测内存类漏洞和non-compliant bug
  5. 5\. LLM-based Bug Analyzer重发所有输出不一致的消息序列验证漏洞真实性并分析背后成因

### Dependency Extraction

> 为什么需要考虑依赖性？  
>  以CVE-2024-42655为例，如果pub没有推送$SYS消息，不会触发broker的转发动作
>
> 在测试中，为增强导向性，要考虑两个Sender之间的消息依赖性

本文考虑了两种依赖关系：

  1. 1\. message dependency：一对按特定顺序发送的能够影响broker和sender通信的消息
  2. 2\. field dependency：具有依赖的消息中存在的特定字段，作为broker路由和转发的依据

提取构造规则的步骤如下：

  1. 1\. 列举协议规范中定义的所有消息类型，分析某种类型的消息是否依赖于一种类型的消息（可以通俗地理解成**某种类型消息的发送必须在某种类型消息发送之后，才能触发建立新的通信或者中断通信** ）。如果有，认为存在Message Dependency。先发送的msg称为primary message，后发送的msg称为secondary message；
  2. 2\. 分析存在依赖关系的消息对，判断对于触发新的动作（建立通信/中断通信）是否需要特定的字段。比如为了触发broker的转发动作，pub必须推送一个和sub订阅主题topic字段一样的消息
  3. 3\. 综合message dependency和field dependency，得出下表

Affect Sender是Message Sending Coordination为测试用例选择Sender的依据

![](/images/USENIX Security 25  MBFuzzer 针对 MQTT 的模糊测试工具/1758678529821.png)

### Test Case Generation

在fuzzing循环中，MBFuzzer基于协议状态为每个Sender生成测试用例，算法设计如下：

![](/images/USENIX Security 25  MBFuzzer 针对 MQTT 的模糊测试工具/1758678530244.png)

其中协议状态的判断依据：response的固定头和响应码，具体代码实现为mbfuzzer_artifact/mbfuzzer/parsers/protocol_parser.py中的retQTableState()

从算法设计中可以看出，测试用例的消息类型有2个来源：shared dependency queue和message priority
scheduler。其中shared dependency queue是message sending
coordination部分用于控制消息生成的组件，将在下一部分介绍。而message priority
scheduler主要是在上一条发送消息在dependency
rules中没有“后续消息”时提供一个最有潜力的消息类型指导种子生成，以最大可能触发潜在bug。

scheduler的核心是Q-learning算法，基本思路是根据Q表为当前状态的下一行动提供指导。

###### Q-learning算法

Q表是Q-learning中记录在某个状态下采取某个动作所获得的**期望回报** 的二维表


| a1| a2  
---|---|---  
s1| -2| 1  
s2| 0| 2  

Q(s1, a1)表示在s1状态采取a1可获取的期望回报是-2（这里假设s1状态采取a1或a2状态都能到达s2）

**Q表的更新策略** ：在s1状态采取a2到达s2状态获得即时奖励R，Q(s1, a2)的目标值: ，估计值为Q(s1,a2)，差距为目标-
估计。新的Q(s1, a2)的估计值为为学习率为折扣率

最常见的选择策略是 -贪婪策略（以1- 的概率选择Q值最大的动作，以 的概率均匀选择）

在MBFuzzer的设计中，即时奖励R的给予遵守规则

MBFuzzer的选择策略采用softmax方法，利用
（T为温度参数，控制曲线平滑度）将Q值转换为概率分布，然后累加概率直到概率和超过随机值，将最后选择的行为作为指导建议。

> Q-learning算法实现在mbfuzzer_artifact/mbfuzzer/fuzzer/q_learning.py

### Message Sending Coordination

shared dependency queue是该部分的重要数据结构，管理dependency rules中的secondart
message并通过控制测试用例生成协调消息发送（通俗理解就是sdq是消息类型选择指南，如果上一条发送的消息在sdq中能对应上某条规则的primary
message，sdp就会指导生成相应的secondary message）

具体步骤如下：

  1. 1\. 根据response确定协议状态
  2. 2\. 查找rules表判断是否存在对应的secondary message
  3. 3\. 若存在，则复制rules表中限定的字段，并将其存放到队列中（高优先级）

![](/images/USENIX Security 25  MBFuzzer 针对 MQTT 的模糊测试工具/1758678530789.png)

上图为基于shared dependency queue的两个并行Sender通信行为的Petri net
model（方框表示事件，圆圈表示状态），下表为模型的参考解释

代号| 说明  
---|---  
T1| client与broker建立连接  
P1| client与broker建立连接后等待发送消息  
T2| 选择消息类型，生成测试用例并发送  
P2| client发送消息后等待broker处理请求后返回的response消息  
T3| client接收并处理response（发生错误到P3状态，终止连接；正常返回P1状态）  
P4| 过渡状态  
T4| 检查是否存在依赖关系；若有，则将对应的secondary message存储到共享队列中  
Q| 等待选择共享队列中的消息  

### Bug Detection

##### Differential Checker

设计上，为了能够应用差分测试，MBFuzzer同时对多个broker进行测试。秉承**按照协议规范实现的软件对同一输入应该有同一输出**
的思路，checker对brokers的返回信息逐个字段比较分析一致性，具体步骤如下：

  1. 1\. 将response消息分类：publish msg和其他消息（因为subscribe能够触发publish和suback两类消息发送，其中suback是一定的）
  2. 2\. 使用topic和payload字段的组合作为摘要，对forwording message进行排序和对齐
  3. 3\. 对两类消息按序逐个字段进行比较（区分字段类型为redundant和non-redundant）

##### Crash Oracles

MBFuzzer要检测的漏洞类型为non-compliant bug和内存安全bug，oracle是监听socket connection
state和检测内存错误

### LLM-based Bug Analyzer

为节省人力资源和时间成本，MBFuzzer使用了LLM自动验证和分析在不一致性下的协议违反情况，具体流程如下图所示。

![](/images/USENIX Security 25  MBFuzzer 针对 MQTT 的模糊测试工具/1758678531219.png)

为减轻LLM幻觉，MBFuzzer使用了background-augmented
prompting方法，同时为了克服上下文限制，从MQTT协议规范中提取违规描述，并按“协议版本|消息类型”的格式存储，分析违规时会将对应类型的描述合并到Prompt
B中一起发送。

为了确保LLM分析的稳定性，MBFuzzer采用了prompt
chaining策略，将分析过程分成了两部分：首先重播不一致的测试用例，并将通信消息转换为可读文本格式；然后使用prompt
A利用LLM对重播的请求和响应信息进行分析总结。分析结果会包含在Prompt B中。随后MBFuzzer通过增强的LLM判断当前测试用例是否存在Prompt
B中描述的违规行为。若发现则将其交由人工进一步验证。

![](/images/USENIX Security 25  MBFuzzer 针对 MQTT
的模糊测试工具/1758678531659.png)![](/images/USENIX Security 25  MBFuzzer 针对 MQTT
的模糊测试工具/1758678532126.png)

## Evaluation

本文从3个问题出发，设计相应实验评估MBFuzzer有效性：

  1. 1\. MBFuzzer是否能够挖掘出MQTT broker真实存在的漏洞，这些漏洞的影响如何？
  2. 2\. 相比目前最先进的Fuzzer，MBFuzzer是否能够拥有更好的表现?
  3. 3\. 每个组件对于MBFuzzer的作用如何？

本文选择了6个主流MQTT broker作为benchmark

![](/images/USENIX Security 25  MBFuzzer 针对 MQTT 的模糊测试工具/1758678532608.png)

##### Discovered Real-world Bugs

MBFuzzer在测试中发现了73个bug（包括20个内存安全类漏洞和53个non-compliant bug），如下所示：  
![](/images/USENIX Security 25  MBFuzzer 针对 MQTT 的模糊测试工具/1758678533070.png)

其中内存安全类漏洞的情况如下图所示，其中存在漏洞具备中等以上的危害性（如M2被认为是中等危害）

![](/images/USENIX Security 25  MBFuzzer 针对 MQTT 的模糊测试工具/1758678533523.png)

##### Comparison to Existing MQTT Fuzzers

在与目前主流先进的Fuzzer对比测试中，MBFuzzer拥有更高的代码覆盖率和更好的漏洞发现能力

![](/images/USENIX Security 25  MBFuzzer 针对 MQTT
的模糊测试工具/1758678534086.png)![](/images/USENIX Security 25  MBFuzzer 针对 MQTT
的模糊测试工具/1758678534571.png)

##### Ablation Study

在消融实验中，本文设置了3个测试项，分别是

  1. 1\. !d：禁用denpendency rules
  2. 2\. !r：禁用message priority scheduler，在不存在消息依赖的情况下随机选择消息类型
  3. 3\. !m：使用Markov chain替代Q-learning

![](/images/USENIX Security 25  MBFuzzer 针对 MQTT
的模糊测试工具/1758678535080.png)![](/images/USENIX Security 25  MBFuzzer 针对 MQTT
的模糊测试工具/1758678535550.png)

从上述测试结果看，本文提出设计的各个组件对MBFuzzer都有一定的积极作用

## Summary and Harvest

本工作从常见的fuzzer架构出发，指出在采用P/S模型的 MQTT 协议中，传统的 two-party fuzzing model对 broker
的测试覆盖不足。为此，作者提出了一种新的 **multi-party 模型** ，包含两个 Sender 与一个
Server，以更真实地模拟协议交互。测试用例生成过程中，特别考虑了两个 Sender 之间的消息依赖性，并设计了 **message sending
coordination** 与 **message priority scheduler** 两个关键机制来协调和优化 Senders
的消息发送。此外，工作还引入了一种新的 **Oracle** ，结合差分测试方法，用于检测软件实现中的 **non-compliance bug** 。

同时本文也说明了工作的不足：

  1. 1\. 仍需人工提取协议规则，过程繁琐且效率不高；
  2. 2\. 系统实现尚不完善，在工程层面存在改进空间；
  3. 3\. 基于 LLM 的漏洞分析依赖协议规范描述，可能遗漏规范未覆盖的异常行为；
  4. 4\. 由于不足[3]的限制，差分测试在多个broker实现同时输出一致性错误时，可能导致潜在漏洞被忽略。

通过学习这篇优秀的工作，有以下几点收获：

  1. 1\. **研究思路** ：在构思新方法时，要优先思考现有工具在某类漏洞上的检测盲点，并思考“为什么效果不好？”。这种从不足出发的思路能自然引出改进点。
  2. 2\. **工具评估** ：在评估实验效果时，要设计高质量的问题（如：是否能发现真实漏洞、发现的漏洞价值如何、对比其他类似工具是否表现更为优异），从多角度凸显工作的价值。
  3. 3\. **跨领域借鉴** ：面对协议测试中的复杂性，可以主动借鉴其他方向的方法，例如强化学习的调度思想、差分测试的 oracle 设计，把不同领域的技术结合到自己的研究场景中

## 参考

  1. 1\. MBFuzzer: A Multi-Party Protocol Fuzzer for MQTT Brokers
  2. 2\. 强化学习——Q-learning算法
  3. 3\. OpenAI. (2025). ChatGPT (September 21version) Large language model

  

