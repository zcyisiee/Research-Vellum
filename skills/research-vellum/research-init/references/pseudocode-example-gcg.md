# 伪代码范例：GCG 主流程

填好的 `docs/PSEUDOCODE.md` 应有的样子（黄金样例）。注意 §1a/§1b 的清晰区分、§2 的
控制流分支、§4 函数 signature 的 shape 注释与形式化表达。

---

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

---

## 【2. 主控制流】

### 2a. 初始化阶段

```
1. 加载实验配置
   params = load_config(config_path)
2. 加载目标行为数据集
   train_goals, train_targets, test_goals, test_targets = get_goals_and_targets(params)
   // 从 CSV 读取 n_train_data 条 (goal, target) 对
3. 加载模型和分词器到 GPU Worker 进程
   train_workers, test_workers = get_workers(params)
   // 训练时 use_cache=False，评估时 use_cache=True
4. 根据 params.attack 动态导入攻击库
   attack_lib = importlib.import_module(f'llm_attacks.{params.attack}')
```

### 2b. 攻击实例创建（根据模式分支）

```
如果 params.transfer == True:
    attack = ProgressiveMultiPromptAttack(goals, targets, workers,
        progressive_goals=params.progressive_goals,
        progressive_models=params.progressive_models)
否则:
    attack = IndividualPromptAttack(goals, targets, workers)
```

### 2c. 攻击主循环

```
对每个 goal_j, target_j in (train_goals, train_targets):
    1. control = control_init
       manager = GCGPromptManager([goal_j], [target_j], tokenizer, conv_template, control_init)
    2. 循环 step = 0 .. n_steps-1:
        2a. grads = [worker.grad(manager) for worker in train_workers]   # (n_ctrl, vocab)
            多模型：先 L2 归一化每个 grad，再逐元素求和
        2b. cand_toks = manager.sample_control(grad_agg, batch_size, topk, temp, allow_non_ascii)
        2c. 如果 filter_cand: cand_toks = get_filtered_cands(cand_toks, tokenizer)
        2d. 对每个候选算 loss = target_weight*target_loss + control_weight*control_loss
            选 loss 最小的候选 → cand_loss_min, cand_control_best
        2e. 如果 anneal: 以 P(loss, cand_loss_min, step) 接受
            否则若 cand_loss_min < loss_current: 接受
        2f. 如果 loss < best_loss: best_control, best_loss = control, loss
        2g. 如果 step % test_steps == 0: 评估 JB% / EM%，写日志
    3. result_j = (best_control, best_loss, n_steps)
```

---

## 【3. 维护状态】

| 变量名 | 描述 | 初始值 |
|---|---|---|
| `control` | 当前对抗后缀 token 序列 | `control_init` 对应 token IDs |
| `loss_current` | 当前后缀对应的目标损失 | 首次评估得到 |
| `best_control` | 历史最优后缀 | `control_init` |
| `best_loss` | 历史最低损失 | `+inf` |
| `step` | 当前优化步数 | `0` |
| `jb_success` | 越狱成功的 goal 列表 | `[]` |

---

## 【4. 核心函数】

### 4.1 计算对抗后缀的坐标梯度

```python
def token_gradients(
    model: PreTrainedModel,
    input_ids: torch.Tensor,        # shape: (1, seq_len)
    input_slice: slice,             # 完整输入区间
    target_slice: slice,            # 目标 token 区间
    loss_slice: slice,              # 损失计算区间 (target_slice 左移 1)
    control_slice: slice,           # 对抗后缀 token 区间
) -> torch.Tensor:                  # shape: (n_control_toks, vocab_size)
```
- **输出:** 损失对后缀每个位置 one-hot 编码的梯度
- **逻辑:**
  1. `E = model.get_input_embeddings().weight`  # (vocab, embed_dim)
  2. 对控制 token 建 one-hot `oh`，`oh.requires_grad = True`
  3. `ctrl_embeds = oh @ E`
  4. `full_embeds = cat([embed(before), ctrl_embeds, embed(after)])`
  5. `logits = model(inputs_embeds=full_embeds).logits`
  6. `loss = CE(logits[0, loss_slice], targets)`；`loss.backward()`
  7. 返回 `oh.grad`
- **形式化:**

$$\nabla_{\mathbf{h}} \mathcal{L}_{\text{CE}}\bigl(f_\theta(\mathbf{e}_{<c} \oplus \mathbf{h} E \oplus \mathbf{e}_{>c}),\; \mathbf{t}\bigr)$$

### 4.2 基于梯度采样候选后缀

```python
def sample_control(
    grad: torch.Tensor,             # shape: (n_ctrl, vocab_size)
    batch_size: int,
    topk: int,
    temperature: float,
    allow_non_ascii: bool,
    not_allowed_ids: torch.Tensor,  # shape: (n_forbidden,)
) -> torch.Tensor:                  # shape: (batch_size, n_ctrl)
```
- **输出:** 候选后缀 token ID 序列
- **逻辑:**
  1. 若 `not allow_non_ascii`：把 `not_allowed_ids` 位置的梯度设为 `+inf`
  2. `topk_ids = topk(-grad, k=topk, dim=-1).indices`
  3. 每个候选：随机选位置 `pos`，从 `topk_ids[pos]` 随机取 token 替换
  4. 返回所有候选

### 4.3 过滤 tokenize 不一致的候选

```python
def get_filtered_cands(
    control_toks: torch.Tensor,     # shape: (batch_size, n_ctrl)
    tokenizer: PreTrainedTokenizer,
) -> torch.Tensor:                  # shape: (batch_size', n_ctrl)
```
- **逻辑:** 每个候选 decode→encode，仅保留长度仍为 `n_ctrl` 的候选。

### 4.4 计算目标回复上的交叉熵损失

```python
def target_loss(
    logits: torch.Tensor,           # shape: (batch_size, seq_len, vocab_size)
    target_ids: torch.Tensor,       # shape: (batch_size, target_len)
    target_slice: slice,
    loss_slice: slice,              # target_slice 左移 1
) -> torch.Tensor:                  # shape: (batch_size, target_len)
```
- **形式化:**

$$\mathcal{L}_{\text{target}} = -\sum_{j=1}^{|\mathbf{t}|} \log P_\theta(t_j \mid \text{prompt} \oplus \mathbf{c},\; t_{<j})$$

### 4.5 模拟退火接受判定

```python
def simulate_annealing_accept(
    e_current: float, e_candidate: float,
    step: int, n_steps: int, anneal_from: int,
) -> bool:
```
- **逻辑:** `e_candidate < e_current` 直接接受；否则以 $\exp(-(e'-e)/T)$ 概率接受。
- **形式化:**

$$P(e, e', k) = \begin{cases} 1 & e' < e \\ \exp\!\bigl(-\tfrac{e'-e}{T_k}\bigr) & \text{otherwise} \end{cases}, \quad T_k = 1 - \frac{k+1}{K + K_0}$$

---

## 【5. Gotcha 账本】

> 初始化时无需编写；随实验迭代追加。示例：

<!--
- G1: KV Cache 与梯度计算不兼容
  - 表现: 开启 use_cache=True 后 token_gradients 返回 None 或 OOM
  - 原因: enable_grad() 下 KV Cache 中间状态不可对 one-hot 嵌入求导
  - 规则: 梯度计算时 use_cache=False，评估时 use_cache=True
  - 对应自检: assert 模型在 grad 路径上 config.use_cache == False
  - 更新时间: 2024-01 initial GCG porting
-->
