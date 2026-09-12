---
type: concept
note_status: draft
maturity: seed
concept_kind: model
aliases:
  - 变换器
  - Transformer Architecture
  - Transformer模型
created: 2026-09-05
updated: 2026-09-11
---

# Transformer

> [!summary] 一句话理解
> Transformer 先让不同 Token 通过 Attention 交换信息，再用 FFN 对每个 Token 已获得的信息进行非线性加工；残差连接保留原表示并改善深层优化，位置编码补充顺序信息。在经典 Encoder-Decoder 架构中，Encoder 构建输入表示，Decoder 据此逐步生成输出。

## 我的原始疑惑

### 多头注意力与顺序

- 并行计算是否会在多头注意力最后拼接时打乱 Token 顺序？
- 输入 Tensor 本来就有第一行、第二行，为什么还说 Transformer 不知道顺序？

### Attention 与 FFN

- Attention 已经让句子内部交换信息，为什么还需要 FFN？
- FFN 对每个 Token 单独处理时，“自己加工”到底有什么用？

### 残差连接

- `x + F(x)` 中“直接加 x”具体是什么意思？
- 为什么把原来的表示加回来会让深层网络更容易训练？
- 残差连接是否只是保证这一层不要乱改或变差？

### Encoder、Decoder 与数据集

- Encoder 和 Decoder 分别负责什么？
- Decoder 是不是专门预测测试集，用来检查模型有没有学会？
- Encoder 输入和 Decoder 输出是否应该是同一段文本？

### Mask

- “遮挡后面的词”发生在 Encoder 还是 Decoder？
- 训练时已经有完整答案，为什么仍要遮住未来 Token？

## 我为什么会卡住

- 把“并行计算”和“缺乏天然顺序语义”误认为同一个问题。
- Attention 与 FFN 都会改变 Token 向量，没有先区分“跨 Token 交流”和“单 Token 特征变换”。
- 虽然残差公式简单，但没有建立“原表示加修正量”和梯度直接通路的直觉。
- Encoder、Decoder、Mask、训练集、验证集和测试集经常同时出现，因而把模型结构分工与数据用途划分混在了一起。

## 我的当前理解

当前最容易记忆的直觉是：

```text
Attention = 和其他 Token 交流，决定从哪里获取信息
FFN       = 每个 Token 自己加工已经获得的信息
Residual  = 保留原表示，并叠加本层学到的变化
Encoder   = 审题，构建输入的上下文表示
Decoder   = 参考输入表示和已生成内容，逐步写答案
```

这些都是教学类比，不是严格数学定义。Transformer 的关键不是由单个模块完成全部任务，而是把顺序、交流、加工、信息保留、输入理解和输出生成交给不同机制协作完成。

## 为什么需要它

RNN、LSTM 通过顺序递归传递信息，天然包含先后过程，但不同时间步之间存在串行依赖。Transformer 改变了信息交互方式：让一个 Token 可以直接对其他位置进行 Attention，从而支持更强的并行计算和全局信息交互。

并行计算不会自动打乱 Token，但不含位置输入的 Self-Attention 本身不会把数组位置解释为语言中的先后顺序，因此还需要显式的位置表示。

## 输入与输出

具体输入输出取决于任务。以经典翻译任务为例：

- Encoder输入：源语言 Token 序列，例如 `I love apples`。
- Encoder输出：每个源 Token 的上下文表示。
- Decoder输入：已经给出的或已经生成的目标序列前缀，以及 Encoder输出。
- Decoder输出：下一个 Token 的概率分布，重复执行后得到目标序列。

```text
I love apples
→ Encoder上下文表示
→ Decoder从 <BOS> 开始逐步生成
→ 我 → 我 喜欢 → 我 喜欢 苹果 → <EOS>
```

Encoder输入和 Decoder目标是否相同取决于任务：翻译、摘要通常不同，[[Autoencoder|AutoEncoder]] 重建任务则可以基本相同。

## 核心流程与组成

### 整体思维模型

```text
输入 Token
→ Embedding：把离散 Token 变成向量
→ Positional Encoding：补充位置信息
→ Attention：不同 Token 交换信息
→ Residual：原表示 + Attention带来的变化
→ FFN：每个 Token 独立进行非线性特征加工
→ Residual：继续保留原表示并叠加变化
→ 多层重复
→ 形成上下文化表示
```

### 各组成部分为什么存在

| 组成部分 | 主要作用 | 为什么需要 |
|---|---|---|
| Embedding | Token变成向量 | 神经网络不能直接处理离散文字标记 |
| Positional Encoding | 加入位置信息 | Self-Attention本身没有天然的先后顺序语义 |
| Self-Attention | Token之间交换信息 | 单个Token需要利用其他位置的上下文 |
| Multi-Head Attention | 在多个特征子空间学习关系 | 单一Attention模式可能不足以表达多类关系 |
| FFN | 按位置进行非线性特征变换 | Attention取回信息后仍需重组和提炼特征 |
| Residual Connection | 原表示与模块输出逐元素相加 | 保留信息并为深层优化、梯度传播提供更直接的路径 |
| Encoder | 构建输入上下文表示 | 生成前需要形成对源输入的表示 |
| Masked Self-Attention | Decoder只能使用当前位置及之前的目标信息 | 防止训练时读取未来答案 |
| Cross-Attention | Decoder读取Encoder输出 | 生成目标时需要参考源输入 |
| Decoder | 逐步生成目标序列 | 将内部表示转化为输出 |

### Multi-Head Attention 拼接什么

设输入形状为 `[seq_len, d_model] = [3, 512]`，8个 Head 可以直观理解为得到 `[8, 3, 64]`。最终拼接的是每个 Token 在不同 Head 中得到的特征维度：

```text
8 × 64 → 512
```

它不会把“我／喜欢／苹果”这些 Token 的位置重新排列。多头拼接不破坏 Token 顺序，位置编码解决的是模型如何利用顺序语义，而不是修复被打乱的数组。

### Attention 与 FFN 的分工

Attention回答：

> 当前 Token 应该从哪些 Token 获取什么信息？

FFN回答：

> 当前 Token 拿到这些上下文信息后，应该怎样进行非线性加工？

FFN通常对每个位置独立应用同一组参数：

$$
FFN(x)=W_2\,\sigma(W_1x+b_1)+b_2
$$

所以“FFN是总结”只能作为初步类比，更准确的说法是“按位置进行非线性特征变换和组合”。

### 残差连接

残差连接执行逐元素相加：

$$
y=x+F(x)
$$

例如：

```text
x    = [1, 2, 3]
F(x) = [0.2, 0.5, -0.3]
y    = [1.2, 2.5, 2.7]
```

“每层在原表示上学习修正量”是有用的直觉，但不能把内部变化简单解释成每层都会加入一个人类可命名的语义。残差连接也不保证增加一层后损失一定降低；它只是让网络更容易表示接近恒等映射的行为，并给梯度提供更直接的传播路径：

$$
\frac{\partial y}{\partial x}=1+\frac{\partial F(x)}{\partial x}
$$

### Encoder、Decoder 与 Mask

经典 Encoder-Decoder Transformer 中：

- Encoder Self-Attention通常可以查看完整输入。
- Decoder Masked Self-Attention只能查看当前位置及过去位置。
- Decoder Cross-Attention通常可以查看完整 Encoder输出。

训练时为了并行计算，可以一次输入完整目标序列；Causal Mask 仍会阻止当前位置读取未来目标 Token，从逻辑上保持自回归生成约束。

Encoder和Decoder是模型结构；Training、Validation、Test是数据用途。三类数据都可能经过Decoder，只有训练集通常参与参数更新。

## 容易混淆的概念

- 并行计算不等于 Token 被重新排列。
- Multi-Head 拼接的是不同 Head 的特征，不是拼接不同 Token。
- FFN不负责 Token之间继续交流；跨 Token交流主要发生在 Attention。
- Residual改善信息和梯度通路，但不保证每增加一层模型一定更好。
- Decoder不是测试模块；模型结构与数据集划分属于不同维度。
- 经典 Encoder 的双向可见性与 Decoder 的因果 Mask不能混为一谈。
- Encoder-Decoder只是 Transformer 的一种架构，还存在 Encoder-only 和 Decoder-only。

## 本次纠正的错误理解

1. **原理解：** 并行计算导致 Token顺序在拼接时丢失。  
   **修正：** 并行不会打乱 Token；位置编码用于提供顺序语义。
2. **原理解：** 多头注意力最后把不同位置拼在一起。  
   **修正：** 拼接的是多个 Head 对同一 Token得到的不同特征。
3. **原理解：** FFN只是简单总结，没有独立作用。  
   **修正：** FFN对各 Token已经获得的上下文特征进行非线性加工。
4. **原理解：** 残差连接保证这一层不要变差。  
   **修正：** 残差提供信息与优化通路，但不保证逐层性能单调提升。
5. **原理解：** Decoder专门预测测试集。  
   **修正：** Decoder是生成结构，训练、验证和测试阶段都可以运行。
6. **原理解：** Encoder和Decoder应该看到相同文本。  
   **修正：** 是否相同由任务决定，翻译与重建任务不同。
7. **原理解：** 未来信息遮挡可能主要由Encoder完成。  
   **修正：** 当前讨论的因果遮挡主要发生在Decoder Masked Self-Attention。

## 与其他概念和研究任务的关系

### RNN与LSTM

RNN、LSTM依靠顺序递归传递历史信息；Transformer不再只依靠一步步传递，而允许 Token直接关注其他位置。二者对顺序的利用机制和并行能力不同。

### 点云

文本中“狗咬人”和“人咬狗”因顺序不同而语义不同；普通点云中点的输入排列通常不应改变物体本身。这个对比说明：模型是否应利用顺序，取决于数据自身的语义。

### AI + CAD与AutoEncoder

Transformer知识可作为后续理解[[AI for CAD 领域地图]]中序列式 CAD 表示与生成方法的基础。[[Autoencoder|AutoEncoder]] 与 Transformer 不在同一层级：前者描述以重构目标学习表示的框架，后者可以作为 Encoder 的具体网络结构。讨论中提到的 WHUCAD AutoEncoder 使用“Encoder压缩表示、Decoder重建序列”的抽象框架；本次没有核验其论文原文，因此暂不建立论文级事实关系。

## 来源与证据

### 已验证来源

- 本次讨论没有直接提供或逐段核验论文、教材、课程页面或官方资料，因此暂时没有可列为已验证的外部来源。

### AI辅助解释／待验证

- 本笔记依据2026-09-05提供的 ChatGPT知识点讨论总结整理。
- Self-Attention、Multi-Head Attention、Positional Encoding、FFN、Residual、经典 Encoder-Decoder、Causal Mask、Cross-Attention和 Teacher Forcing 等内容目前属于通用知识解释，尚未在本笔记中逐项绑定可靠来源。
- “Attention = 交流”“FFN = 自己消化”“Residual = 原表示 + 修正”“Encoder = 审题”“Decoder = 写答案”仅为教学类比，不应当作严格数学定义。
- WHUCAD关联只是帮助理解的学习上下文，不代表已经核验对应论文。

## 尚未理解

- Encoder最终输出 Tensor 的具体形状和语义是什么？
- Decoder每一步具体接收哪些输入？
- Cross-Attention中 Q、K、V 分别来自哪里？
- Causal Mask矩阵长什么样，为什么加入负无穷后 Softmax对应位置变成0？
- Teacher Forcing下的训练过程与自回归推理具体有什么差异？
- 正弦／余弦位置编码、可学习位置嵌入、相对位置和 RoPE 有什么关系？
- LayerNorm做什么，为什么常与Residual一起出现？Pre-Norm和Post-Norm有什么区别？
- Encoder-only、Decoder-only、Encoder-Decoder如何系统区分？
- GPT为什么只有Decoder结构仍能理解输入并生成文本？

## 自我检查

1. Transformer并行计算时，Token的物理排列真的会被打乱吗？
2. Multi-Head Attention最后拼接的是Token还是各Head得到的特征？
3. 为什么Self-Attention仍然需要位置编码？
4. Attention与FFN的任务有什么本质区别？
5. 为什么可以暂时把Attention理解为“交流”，FFN理解为“自己加工”？
6. `F(x) + x` 中的 `x` 和 `F(x)` 分别是什么？
7. 残差结构为何有助于深层网络优化？
8. 残差连接是否保证增加一层后模型一定更好？
9. Encoder在翻译任务中负责什么？
10. Decoder在翻译任务中负责什么？
11. Encoder输入与Decoder目标是否一定相同？
12. 为什么Decoder Self-Attention要遮挡未来Token？
13. 为什么经典Encoder通常不需要同样的因果Mask？
14. Decoder的Cross-Attention读取什么？
15. Training、Validation、Test和Encoder、Decoder属于同一种划分吗？
16. AutoEncoder重建与翻译任务的Encoder／Decoder有什么共同点和区别？
17. 为什么文本重视Token顺序，而普通点云通常不应依赖点的输入排列？

## 学习记录

### 2026-09-05：第一次系统讨论

- 讨论主题：从多头注意力与位置顺序出发，进一步理解FFN、残差连接、Encoder／Decoder和Causal Mask。
- 理解变化：从“多个模块都在处理信息”的模糊印象，发展为“跨Token交流、单Token加工、保留与增量修改、输入表示、受约束生成”的分工模型。
- 当前状态：已经建立整体因果链，但具体数据流、数学实现、架构类型和训练／推理差异仍待补齐。
- 证据状态：主要来自AI辅助讨论，等待论文、教材或官方资料验证。

### 2026-09-11：区分网络结构与训练框架

- 通过 [[Autoencoder]] 进一步区分：Transformer 描述可承担 Encoder 角色的网络结构，Autoencoder 描述 Encoder、潜在表征、Decoder 与重构目标组成的表示学习框架。
- 当前仍未核验具体 CAD AutoEncoder 论文，不据此建立论文级方法事实。
