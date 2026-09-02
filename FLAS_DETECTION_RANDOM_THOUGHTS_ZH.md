# 关于让 FLAS 学会“读”模型内部状态的一些零散想法

> 这不是一份 research plan，也不是对下一篇论文结构的承诺。它更像一个临时思想容器：记录目前看起来有趣的直觉、可能的连接、尚未区分的假设，以及一些也许很快就会被实验否定的方向。这里的重点不是排出优先级，而是把问题空间摊开。

## 一个起点：为什么 detection 和 steering 总是错位

SAE 很吸引人的一点，是它看起来同时提供了 detection 和 steering：一个 feature 的 activation 可以被当作概念是否存在的证据，对同一个 feature 做 intervention 又可以改变模型行为。但现实似乎经常是，SAE 做 detection 不如 task-specific probe，做 steering 又不如 affine map 或其他直接为行为变化优化的 operator。

这也许不是某个 SAE 训练目标没有调好，而是一个更根本的错位。Detection 想要的是最好的 decision boundary；steering 想要的是以最小副作用改变模型计算的最佳 actuator。一个追求 reconstruction、sparsity 和某种程度 monosemanticity 的通用坐标系，没有理由同时成为这两个问题的最优解。

因此，FLAS 也许不应该试图寻找一个“既能读又能写”的固定 feature basis。更自然的统一方式可能是：让 detection 和 steering 共享自然语言定义的语义接口，并通过因果一致性连接起来，但允许它们使用不同的 operator。

## 把 FLAS 看成一种 perturbation instrument

目前的 FLAS 通常被描述为一个 concept-conditioned steering controller：给定内部状态 (h)、时间 (t) 和自然语言概念 (c)，它产生 velocity

\[
v_\theta(h,t,c),
\]

再通过多步更新得到一条 steering trajectory。另一个可能的视角是，FLAS 不只是改变状态的工具，也是一个主动测量状态的工具。

如果对同一个概念 (c) 施加干预，不同的初始状态可能表现出不同的响应：有些状态需要很大的位移，有些只需要很小的修正；有些 trajectory 很快收敛，有些不断旋转；有些 hidden-state 位移很大，但下游 logits 几乎不变。于是，可以把

\[
R(h,c)=\text{FLAS 对状态 }h\text{ 和概念 }c\text{ 的完整响应}
\]

当作一种 concept-conditioned spectroscopy。我们不是直接问“能不能从 (h) 线性解码出 (c)”，而是问：“当我试图把模型推向 (c) 时，它如何响应？”

这个问题可能同时包含 observability 和 controllability 的信息。

## 关于“饱和”的直觉

一个很自然的猜想是：如果模型已经准备回答某个概念，再向内部状态加入同一概念，边际效果会变小。换句话说，当前状态已经接近 FLAS 为概念 (c) 所诱导的某种目标区域，因此再次 steering 会出现 saturation。

值得注意的是，当前 FLAS 的训练方式确实让这个猜想具有一定合理性。Teacher forcing 时，FLAS 不只作用于尚未出现概念的 prompt states，也作用于已经逐渐形成乃至明确表达概念的 output-token states。因此，它可能已经隐式接触过“这个概念已经存在时还应当做什么”的问题。

但 fluency 和 instruction following 保持得好，并不自动意味着 velocity norm 会在概念已存在时变小。至少还存在几种不同可能：

- **Fixed-point saturation**：概念已经存在时，velocity 和 endpoint displacement 都变小。
- **Trajectory contraction**：初始仍有响应，但后续 velocity 很快衰减。
- **Behavioral saturation**：hidden state 仍被明显移动，但 logits 或生成行为几乎不再改变。
- **Reinforcement resonance**：概念越强，FLAS 的响应反而越强、更一致，类似 concept-conditioned amplifier。
- **Overshoot 或 conflict**：概念已经存在时继续 steering 会破坏表达、引发重复，或者把状态推离原有的良好解。

所以，“饱和”最好暂时被理解为一个开放的 response-regime 问题，而不是预先规定低 velocity 就等于 concept presence。

## 另一个缺口：FLAS 目前基本是单向的

当前 FLAS 比较自然地做 concept promotion，却似乎不擅长 concept suppression。这也许不说明架构本身只能单向控制，更直接的原因可能是训练信号本来就是单向的：训练数据只保留 concept-positive outputs，flow time 也取正值，模型从未被明确要求“保留原任务，但让某个概念停止影响输出”。

简单把 (T) 取负不一定能够解决这个问题。当前 velocity field 是在正向、有限时间区间和 positive LM targets 上学到的；负时间既是时间条件的分布外输入，也暗含“正向 transformation 的逆就是 suppression”这一很强的假设。Concept promotion 的逆操作可能不是 concept removal：它可能导致语言质量退化、激活进入陌生区域，或者删除与该概念纠缠的其他能力。

一个更自然的扩展是给 FLAS 显式加入 control polarity 或 target state：

\[
v_\theta(h,t,c,m),
\qquad
m\in\{\text{promote},\text{suppress}\},
\]

甚至不把它限制成二元模式，而是给定一个连续 setpoint (r)：

\[
v_\theta(h,t,c,r),
\qquad
r\in[-1,1].
\]

这里 (r) 不是简单乘在 velocity 上的正负系数，而是描述希望 concept 在后续计算中具有怎样的目标强度。Controller 可以根据当前状态和目标之间的误差，学习不同的非线性路径。

“抑制概念”本身也可能包含几种不同目标：

- **Lexical suppression**：不要生成某个词或短语。
- **Semantic suppression**：不要在内容中表达某个概念，即使换用同义表达。
- **Behavioral suppression**：概念可以被模型理解，但不应驱动当前回答或工具行为。
- **Representational erasure**：希望内部状态本身不再包含该信息。

前三者逐渐变难，第四个目标又与前三者不完全相同。工业控制通常更需要 behavioral suppression，而不是让模型“忘掉”知识。模型可以知道某种危险操作是什么，只是不允许这部分知识在当前权限和上下文中推动行动。

训练数据方面，已有 AxBench-style 数据中包含当前训练代码主动丢弃的 negative category；这些样本也许可以作为最初的 suppress targets。对于只有 positive exemplars 的 Concept16k/46k，也可以考虑构造同一 prompt 下的 counterfactual pair：一个输出自然表达概念 (c)，另一个输出仍然完整回答原 instruction，但不让 (c) 影响内容。后者不能只是删除关键词，否则 controller 很容易把 suppression 学成 lexical censorship。

如果 observer 能够给出 concept score，promote 和 suppress 可以统一为一个很简洁的排序约束。设

\[
h_d^-=M_{s+1:d}(\operatorname{FLAS}_{\text{suppress}}(h_s,c)),
\]

\[
h_d^0=M_{s+1:d}(h_s),
\]

\[
h_d^+=M_{s+1:d}(\operatorname{FLAS}_{\text{promote}}(h_s,c)),
\]

那么理想情况是

\[
D(h_d^-,c)<D(h_d^0,c)<D(h_d^+,c).
\]

这使 detector 不只是 FLAS 的附加功能，还可能成为双向控制的 learned value function。Promote controller 使 (D) 上升，suppress controller 使 (D) 下降；当状态已经达到目标区间时，两者都应趋向 no-op。之前关于 saturation 的直觉也因此变得更具体：saturation 不一定需要从 velocity norm 猜测，而可以定义为 observer 已经到达 setpoint 后，继续 intervention 不再带来 score gain。

当然，只优化 detector score 会产生新的 Goodhart 问题。Controller 可能学会欺骗 observer，而不真正改变输出行为。因此 suppression 至少需要同时满足几件事：后层 observer score 下降；最终输出对 concept 的实际依赖下降；原 instruction、fluency 和无关能力尽量保持；在换 detector、换自然语言 paraphrase 或换 readout layer 时仍然成立。最理想的目标不是最大限度改变 activation，而是完成 suppression 所需的最小 intervention。

这里可能出现一种有趣的闭环形态：observer 先判断 concept 是否存在以及是否正在影响行为；如果低于风险阈值，suppress controller 不动作；如果高于阈值，controller 做一次最小修正，后层 observer 再检测；只有仍高于阈值时，下一 token 才继续干预。这样 suppression 不再是始终施加一个负方向，而是 event-driven 的 feedback control。

如果这种双向控制真的可以通过相对小的训练修改得到，它会显著改变 FLAS 的含义。FLAS 不再只是“让模型更多谈论某个概念”，而开始接近一个可以设定、提高、降低和维持内部行为变量的 runtime controller。

## 前层 intervention，后层 detection

一个很干净的架构是把 actuator 和 observer 放在不同深度。设 FLAS 在较早层 (s) 做干预：

\[
\tilde h_s=\operatorname{FLAS}(h_s,c),
\]

然后让冻结的 base model 继续传播到较晚层 (d>s)：

\[
\tilde h_d=M_{s+1:d}(\tilde h_s),
\]

最后训练一个 concept-conditioned detector：

\[
D_\phi(\tilde h_d,c)\rightarrow \text{concept score}.
\]

Concept16k 在这里提供了一种很便宜的因果监督。对同一个 prompt 分别注入一批概念 (c_1,\ldots,c_B)，得到后层状态 \(\tilde h_d^{(1)},\ldots,\tilde h_d^{(B)}\)，然后构造

\[
S_{ij}=D_\phi(\tilde h_d^{(i)},c_j).
\]

Detector 的任务是判断哪一个自然语言概念造成了当前 downstream state。这和普通 probe 有一点不同：标签来自已知的 intervention，而不是仅仅来自文本与 activation 的共现。

Detector 本身可以从零训练。一个直观结构是让 concept tokens 作为 query、较晚层 activation 作为 key/value。FLAS 是 activation queries concept；detector 则反过来是 concept queries activation。两者共享的可以只是自然语言 concept encoder 或语义空间，不必共享核心参数。

## intervention 产生的状态，能否迁移到自然状态

这里最重要的困难也很明显：detector 可能只是在读取 FLAS 留下的 watermark。

即使 intervention 和 detection 错开多个层，FLAS 仍可能在某些 downstream-nullspace 方向留下很容易解码的 concept fingerprint。Detector 可以准确识别这个 fingerprint，但模型自然形成相同概念时未必会产生它；更糟的情况是，这个 fingerprint 与最终行为没有任何关系。

因此，一个有意义的 observer 应当同时把两类状态识别为相同概念：

\[
D(h_d^{\text{natural }c},c)\rightarrow1,
\]

\[
D(h_d^{\text{steered }c},c)\rightarrow1.
\]

这也是为什么最初冻结 base model 和已有 FLAS、只训练 detector 可能更安全。如果一开始联合训练，FLAS 和 detector 很容易发展出一种隐藏通信协议：FLAS 编码 concept ID，detector 解码 concept ID，而 base model 的真实行为并没有相应变化。

从另一个角度看，natural/steered transfer 本身可能就是一个有趣问题：由 intervention 产生的 concept state 和模型自然形成的 concept state，究竟是不是同一种内部对象？如果不是，它们在哪些层开始靠近，又在哪些层保持可区分？

## 一种可能的 response fingerprint

对状态–概念对 ((h,c))，可以暂时记录如下量，而不急着认定其中哪一个是 detector：

\[
R(h,c)=
\left[
\|v_0\|,\ldots,\|v_{N-1}\|,
\|h_N-h_0\|,
\sum_k\|h_{k+1}-h_k\|,
\cos(v_0,v_1),\ldots,
D_{\mathrm{KL}}(p_N\Vert p_0)
\right].
\]

这里包含了初始响应强度、总位移、路径长度、方向变化和 downstream behavioral effect。也许真正稳定的 detection signal 不是某一个标量，而是整个 trajectory fingerprint。

还有一个更偏控制论的量：从当前状态出发，使模型表现出概念 (c) 所需的最小控制能量

\[
E^*(h,c)=
\min_{\{a_t\}}
\sum_t\|a_t\|^2
\quad
\text{s.t. output satisfies }c.
\]

如果这个量能够被一个 critic 近似，它可能同时扮演 detector/value function：能量很低表示模型已经接近该行为区域，能量很高表示概念虽然也许可以被 probe 解码，但当前计算并没有准备使用它。

## 绝对 detection 与 intervention gain

假设已经有了后层 detector，可以比较干预前后的分数：

\[
p_0=D(h_d,c),
\]

\[
p_1=D\!\left(M_{s+1:d}(\operatorname{FLAS}(h_s,c)),c\right),
\]

并定义

\[
\Delta_c(h)=p_1-p_0.
\]

这给出一个比单独 concept probability 更有意思的二维描述：

| 原始分数 (p_0) | 干预增益 \(\Delta_c\) | 一种可能的解释 |
|---:|---:|---|
| 高 | 小 | 概念已经存在并接近饱和 |
| 低 | 大 | 概念尚不存在，但容易被控制 |
| 低 | 小 | 概念不存在，且当前状态难以到达 |
| 高 | 大 | 持续 reinforcement，或 detector/controller 不稳定 |
| 高 | 负 | overshoot、冲突，或干预破坏了自然表示 |

这里 detection 不再只回答“概念有没有”，而开始回答：“它有没有、它是否正在被使用、它是否容易被改变，以及继续干预是否危险。”这可能比一个更高 AUROC 的 probe 更接近实际运行时需要的状态估计。

## 最有趣的测试可能发生在概念说出口之前

如果 prompt 或已有输出已经明确包含概念词，detector 很可能只是做文本识别。更有意义的状态是：模型已经形成了随后要表达的概念，但第一处显式语言证据尚未出现。

Concept16k 的 positive outputs 可以被切成多个 response prefixes。找到第一次明确表达 concept 的位置，然后检查此前各 token 的 hidden state。对于真实概念 (c^+) 和其他候选概念 (c^-)，分别运行 FLAS response 或后层 detector。

如果在显式 concept token 出现之前已经有

\[
R(h_{\text{prefix}},c^+)\neq R(h_{\text{prefix}},c^-),
\]

或者 detector 已能稳定识别 (c^+)，那就更接近 latent intention detection，而不是从已经生成的文本恢复标签。

同样值得观察的是这个信号沿模型深度的传播。概念可能在中层短暂出现又被删除，可能逐层增强，也可能只在接近 logits 时才形成。把 detector 放在多个 (d>s) 的位置，也许会得到某种 concept propagation profile。

## 自然语言解释可能怎样进入这里

[LatentQA](https://arxiv.org/abs/2412.08686) 把 activation 当作一种新模态，让 LLM 回答关于内部状态的开放式自然语言问题；[Activation Oracles](https://arxiv.org/abs/2512.15674) 则进一步探索 general-purpose activation explainer 和 OOD generalization。[SelfIE](https://arxiv.org/abs/2403.10949) 与后来的 [Training Language Models to Explain Their Own Computations](https://arxiv.org/abs/2511.08579) 也指向“模型以自然语言读取自身或其他模型内部状态”的可能性。

这些工作给 FLAS 的启发也许不只是增加一个会说话的 decoder。更有意思的是，自然语言可以成为共享的 latent address：

- 用问题描述想读取的内部状态；
- 用同一段语言描述希望模型进入的目标状态；
- observer 给出当前状态、证据和不确定性；
- FLAS 根据 observer–target error 做 intervention；
- intervention 以后重新读取状态，检查解释是否经得起 counterfactual verification。

自然语言解释最大的风险是 plausibility without faithfulness。FLAS 可能提供一个特别合适的验证闭环：如果 observer 声称某个内部状态正在因果性地影响输出，那么基于该解释进行 ablation 或 steering，结果应当按照解释的预测变化。解释不再只靠另一个语言模型打分，而要能够预测 intervention outcome。

## 也许最终的对象不是 detector，而是 observer

“Detector”听起来像一个二分类器，但真实模型状态可能不是简单的 present/absent。一个更完整的 observer 也许应当输出：

- concept 是否可被解码；
- concept 是否正在影响当前计算；
- concept 是否将出现在后续输出；
- 当前状态对该 concept 的 steering susceptibility；
- 当前状态距离某个目标行为还有多远；
- observer 对自己的判断是否处于 OOD；
- 哪些 token、层或 activation component 构成证据。

于是 FLAS 可能从一个单向 steering method 变成一种 language-conditioned observer–controller：前层 actuator 改变状态，后层 observer 读取结果；在 autoregressive generation 中，当前 token 的后层 observation 可以决定下一 token 是否还需要在前层继续干预。

## 一个最小 thought experiment

如果现在只允许做一个很小的探索，我会倾向于冻结 base model 和当前 FLAS，在一个早期层注入 Concept16k concepts，并在几个较晚层从零训练自然语言条件 detector。同时混入未经 steering 的 natural positive states，检查由 intervention 学到的 detector 是否能迁移到自然生成。

真正让我兴奋的结果不会只是“detector 能识别我们刚刚注入了哪个 concept”。更关键的是以下几种现象是否存在：

- 在概念第一次被说出之前，detector 已经能够读到它；
- detector score 能预测稍后的生成行为；
- intervention gain 能区分“已存在”“缺失但可控”和“缺失且不可达”；
- natural 与 steered concept states 在某些后层形成共同表示；
- observer 的判断能决定 FLAS 什么时候应该停止，从而形成真正的 saturation-aware closed loop。

如果这些都不存在，这条路也仍可能告诉我们一个重要事实：FLAS 产生的是一种外加行为偏置，而不是模型自然内部状态的可读写接口。如果其中一部分稳定存在，那么 FLAS 的研究对象就可能从 steering trajectory 扩展为模型内部状态的主动测量、因果解释与闭环控制。

## 暂时没有答案的问题

- FLAS trajectory 中 concept-specific 的部分，多少与最终行为有关，多少只是 downstream-nullspace motion？
- 一个由 intervention 训练的 detector，能否识别完全自然形成的 concept state？
- “模型理解了一个概念”“模型正在使用它”和“模型即将说出它”是否对应不同层或不同 response regime？
- Saturation 是 velocity 变小、trajectory 收缩、output effect 变小，还是另一种我们尚未命名的几何现象？
- Detector 应读取单层 activation、跨层状态，还是 FLAS trajectory 本身？
- 自然语言 query 是否真的能泛化到未见 concepts，还是最终仍需要一个有限 ontology？
- 如果 observer 和 controller 共享 concept encoder，它们会形成共同语义空间，还是互相牵制？
- 能否让 observer 学习一个有实际意义的 value function，而不只是另一个高容量 classifier？
- 模型版本变化后，observer 是否比 controller 更容易迁移，或者反过来？
- 对工业系统而言，最有价值的可能不是“检测概念”，而是检测是否需要 intervention。后者是否应该成为真正的监督目标？

这些问题目前不需要被组织成一条单一路线。它们更像围绕同一个核心直觉的不同切面：**自然语言条件的 intervention 可能不仅能控制模型，也能作为理解模型当前内部状态的主动实验。**

