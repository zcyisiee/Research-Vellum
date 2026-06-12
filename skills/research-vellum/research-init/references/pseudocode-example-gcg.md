# 伪代码范例：GCG 主流程

填好的 `docs/PSEUDOCODE-gcg-attack.md` 应有的样子（黄金范例）。注意 YAML frontmatter
的元数据、§1a/§1b 的清晰区分、§2 的自然语言控制流叙事、§3 对核心算法公式的集中呈现。

---

```yaml
---
title: GCG 对抗后缀优化主流程
entry_scripts:
  - exp/gcg_attack.py
  - exp/config.yaml
created: 2024-01-15
modified: 2024-03-20
status: reviewed
---
```

【算法名称】：GCG (Greedy Coordinate Gradient) — Universal and Transferable Adversarial Attacks on Aligned Language Models (Zou et al., 2023)

---

## 【0. 目的与边界】

**做什么:** 对一组有害行为指令 (goals)，优化一个离散对抗性后缀 (adversarial suffix) $\mathbf{c}^* = (c_1, \ldots, c_L)$，使得对齐 LLM 在拼接 `[goal | c*]` 后，生成预设的有害目标回复 (target) $\mathbf{t}$。核心优化目标：

$$\mathbf{c}^* = \arg\min_{\mathbf{c} \in \mathcal{V}^L} \;\; \sum_{i=1}^{N} \mathcal{L}_{\text{CE}}\!\bigl(f_\theta(\text{prompt}_i \oplus \mathbf{c}),\; \mathbf{t}_i\bigr)$$

其中 $f_\theta$ 为目标 LLM，$\oplus$ 为 token 序列拼接，$\mathcal{L}_{\text{CE}}$ 为交叉熵损失（在 target token 上计算）。

**边界:** 本脚本完成一个完整的攻击实验单元——加载数据、加载模型、执行 GCG 优化循环、评估越狱成功率、记录结果。不涉及跨模型迁移评估（由 `evaluate.py` 完成）和 API 模型攻击（由 `api_experiments/` 完成）。

---

## 【1. 输入与超参】

### 1a. 可搜索超参

| 变量名 | 描述 | 默认值 |
|---|---|---|
| `n_steps` | 优化总步数 | 500 |
| `batch_size` | 每步生成的候选后缀数量 | 512 |
| `topk` | 梯度采样时每位置保留的 top-k 候选 token 数 | 256 |
| `temp` | 采样温度（当前未生效） | 1 |
| `lr` | 学习率（用于 GBDA 等其他攻击） | 0.01 |
| `target_weight` | 目标损失权重 | 1.0 |
| `control_weight` | 控制损失正则化权重 | 0.0 |
| `n_train_data` | 训练行为数 | 50 |
| `n_test_data` | 测试行为数 | 0 |
| `control_init` | 初始对抗后缀（空格分隔的 `!` token） | `"! ! ! ... !"` (20 tokens) |
| `test_steps` | 评估间隔步数 | 50 |

### 1b. 实验开关

| 变量名 | 描述 | 取值 |
|---|---|---|
| `attack` | 攻击算法类型 | `'gcg'` |
| `transfer` | 是否进行跨模型迁移攻击 | `False` |
| `progressive_goals` | 是否渐进式添加目标 | `False` |
| `progressive_models` | 是否渐进式添加模型 | `False` |
| `anneal` | 是否启用模拟退火接受策略 | `False` |
| `incr_control` | 是否逐步增大 control_weight | `False` |
| `stop_on_success` | 全部越狱成功时是否提前终止 | `False` |
| `allow_non_ascii` | 是否允许非 ASCII token 出现在后缀中 | `False` |
| `filter_cand` | 是否过滤重新 tokenize 后长度变化的候选 | `True` |
| `gbda_deterministic` | 确定性模式 | `True` |

### 1c. 运行时输入

| 变量名 | 描述 |
|---|---|
| `train_data` | 训练数据 CSV 路径（AdvBench 格式：goal, target 列） |
| `test_data` | 测试数据 CSV 路径（同格式，可选） |
| `model_paths` | 模型路径列表，如 `["lmsys/vicuna-7b-v1.3"]` |
| `tokenizer_paths` | 分词器路径列表（通常与模型路径一致） |
| `devices` | GPU 设备分配列表，如 `["cuda:0", "cuda:1"]` |
| `conv_template_name` | 对话模板名称，如 `"vicuna_v1.1"`, `"llama-2"` |

> 1a 与 1b 的区分是给第二阶段超参搜索用的硬边界：搜索 Agent 只能动 1a，碰 1b 即违规。

---

## 【2. 主控制流】

### 2a. 初始化

```
1. 从 config 加载所有超参和运行时路径
2. 从 CSV 读取 n_train_data 条 (goal, target) 对作为训练集，n_test_data 条作为测试集
3. 将每个目标模型和分词器加载到对应 GPU，创建 Worker 进程
   训练时 Worker 关闭 KV Cache（需要计算梯度），评估时开启
4. 根据 attack 类型动态导入对应的攻击模块
```

### 2b. 攻击实例创建

```
如果 transfer == True:
    创建 ProgressiveMultiPromptAttack（支持渐进式添加目标和模型）
否则:
    创建 IndividualPromptAttack（单 prompt 攻击）
```

### 2c. 攻击主循环

```
对每个 (goal, target) 训练对:
    1. 初始化对抗后缀为 control_init，创建 PromptManager 管理 prompt 拼接
    2. 循环 step = 0 .. n_steps-1:
        a. 计算梯度：对每个训练模型，算出 loss 对后缀每个位置 one-hot 编码的梯度
           多模型时：先 L2 归一化每个模型的梯度，再逐元素求和
        b. 采样候选：从 -grad 方向取 topk 个 token，随机选位置替换，生成 batch_size 个候选后缀
           若 allow_non_ascii == False：禁止采样的 token 位置梯度置为 +inf
        c. 过滤候选（若 filter_cand）：每个候选 decode→encode，丢弃长度变化的
        d. 评估候选：对每个候选计算 loss = target_weight × 目标损失 + control_weight × 后缀损失
           选出 loss 最小的候选
        e. 接受判定：若 anneal == True，用模拟退火策略决定是否接受（见 §3）
           否则：仅当候选 loss < 当前 loss 时接受
        f. 更新全局最优：若当前 loss < best_loss，记录为最优
        g. 每隔 test_steps 步：在测试集上评估越狱成功率 (JB%) 和精确匹配率 (EM%)，写日志
    3. 输出该 goal 的最优后缀、最优 loss、总步数
```

---

## 【3. 关键算法细节】

### 3.1 坐标梯度的计算

GCG 的核心 trick：不直接对离散 token 求梯度，而是对后缀位置的 one-hot 嵌入求梯度，再用梯度近似离散优化的方向。

$$\nabla_{\mathbf{h}} \mathcal{L}_{\text{CE}}\bigl(f_\theta(\mathbf{e}_{<c} \oplus \mathbf{h} E \oplus \mathbf{e}_{>c}),\; \mathbf{t}\bigr)$$

其中 $\mathbf{h}$ 为后缀 one-hot 矩阵，$E$ 为模型 embedding 权重。梯度矩阵的 $(i, v)$ 元素表示：将位置 $i$ 替换为 token $v$ 后 loss 的变化趋势。取 $-\text{grad}$ 的 top-k 即为最有潜力的替换方向。

### 3.2 模拟退火接受策略

当 `anneal == True` 时，不仅接受更优的候选，也以递减概率接受较差候选，帮助跳出局部最优：

$$P(e, e', k) = \begin{cases} 1 & e' < e \\ \exp\!\bigl(-\tfrac{e'-e}{T_k}\bigr) & \text{otherwise} \end{cases}, \quad T_k = 1 - \frac{k+1}{K + K_0}$$

温度 $T_k$ 随步数线性衰减到 0，后期退化为贪心接受。

---

## 【4. 维护状态】

| 变量名 | 描述 | 初始值 |
|---|---|---|
| `control` | 当前对抗后缀 token 序列 | `control_init` 对应 token IDs |
| `loss_current` | 当前后缀对应的目标损失 | 首次评估得到 |
| `best_control` | 历史最优后缀 | `control_init` |
| `best_loss` | 历史最低损失 | `+inf` |
| `step` | 当前优化步数 | `0` |
| `jb_success` | 越狱成功的 goal 列表 | `[]` |

---

## 【5. Gotcha 账本】

> 初始化时无需编写；随实验迭代追加。示例：

<!--
- G1: KV Cache 与梯度计算不兼容
  - 表现: 开启 use_cache=True 后梯度返回 None 或 OOM
  - 原因: enable_grad() 下 KV Cache 中间状态不可对 one-hot 嵌入求导
  - 规则: 梯度计算时 use_cache=False，评估时 use_cache=True
  - 对应自检: assert 模型在 grad 路径上 config.use_cache == False
  - 更新时间: 2024-01 initial GCG porting
-->
