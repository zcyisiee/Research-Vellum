# GCG 主流程伪代码规范

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
   // 从 CSV 读取 n_train_data 条 (goal, target) 对，goal 为有害指令，target 为期望的有害回复开头

3. 加载模型和分词器到 GPU Worker 进程
   train_workers, test_workers = get_workers(params)
   // 每个 Worker 封装一个 LLM 实例，通过 multiprocessing 通信
   // 训练时 use_cache=False，评估时 use_cache=True

4. 根据 params.attack 动态导入攻击库
   attack_lib = importlib.import_module(f'llm_attacks.{params.attack}')
```

### 2b. 攻击实例创建（根据模式分支）

```
如果 params.transfer == True:
    // 跨模型迁移攻击
    attack = ProgressiveMultiPromptAttack(
        goals=train_goals, targets=train_targets,
        workers=train_workers, 
        progressive_goals=params.progressive_goals,
        progressive_models=params.progressive_models
    )
否则:
    // 单目标/单模型独立攻击
    attack = IndividualPromptAttack(
        goals=train_goals, targets=train_targets,
        workers=train_workers
    )
```

### 2c. 攻击主循环

```
对每个 goal_j, target_j in (train_goals, train_targets):

    1. 为当前目标创建 PromptManager，初始化对抗后缀
       control = control_init  // 初始后缀，如 20 个 "!" token
       manager = GCGPromptManager(goals=[goal_j], targets=[target_j],
                                   tokenizer, conv_template, control_init)

    2. 执行 GCG 优化循环
       循环 step = 0, 1, ..., n_steps - 1:

           2a. 计算梯度：对每个训练模型的 Worker，计算 token_gradients
               grads = [worker.grad(manager) for worker in train_workers[:num_train_models]]
               // 返回 shape: (n_control_toks, vocab_size) 的梯度张量
               // 如果多模型：先 L2 归一化每个 grad，再逐元素求和得到聚合梯度

           2b. 生成候选后缀：基于梯度采样 batch_size 个候选
               cand_toks = manager.sample_control(grad_agg, batch_size, topk, temp, allow_non_ascii)
               // 对后缀中随机一个位置，从 top-k 负梯度 token 中随机替换

           2c. 过滤候选：移除重新 tokenize 后长度变化的候选
               如果 filter_cand:
                   cand_toks = get_filtered_cands(cand_toks, tokenizer)

           2d. 评估候选：对每个候选计算目标损失
               对每个 worker:
                   logits = worker.logits(manager, cand_toks)
                   loss = target_weight * target_loss(logits, target_ids)
                        + control_weight * control_loss(logits, control_ids)
               总损失 = 所有 worker 上 loss 之和 / prompt 数
               选取 loss 最小的候选 → cand_loss_min, cand_control_best

           2e. 接受或拒绝候选
               如果 anneal == True:
                   以概率 P(loss_current, cand_loss_min, step) 接受
                   // P(e, e', k) = exp(-(e'-e)/T), T = 1 - (k+1)/(n_steps+anneal_from)
               否则:
                   如果 cand_loss_min < loss_current:
                       接受: control = cand_control_best, loss = cand_loss_min

           2f. 记录最优
               如果 loss < best_loss:
                   best_control = control
                   best_loss = loss

           2g. 周期性评估
               如果 step % test_steps == 0:
                   在测试集上评估越狱成功率 (JB%) 和精确匹配率 (EM%)
                   记录到日志

    3. 返回当前目标的最优对抗后缀
       result_j = (best_control, best_loss, n_steps)
```

### 2d. 渐进式多目标攻击循环

```
1. 初始化: goals_active = [goals[0]], models_active = [workers[0]]

2. 外层循环（直到收敛或达到总步数）:

    a. 创建 MultiPromptAttack(goals_active, models_active)
    b. 运行内层攻击循环 (同 2c 步骤 2)
    c. 如果所有 goal 均越狱成功:
         如果还有未加入的 model:
             models_active.append(下一个 model)
         否则 如果 incr_control:
             control_weight += 0.01
         否则:
             终止
    d. 否则（仍有 goal 未攻破）:
         如果 progressive_goals 且还有未加入的 goal:
             goals_active.append(下一个 goal)
         否则:
             继续内层循环
```

---

## 【3. 维护状态】

| 变量名 | 描述 | 初始值 |
|---|---|---|
| `control` | 当前对抗后缀 token 序列 | `control_init` 对应的 token IDs |
| `loss_current` | 当前后缀对应的目标损失 | 首次评估得到 |
| `best_control` | 历史最优后缀 | `control_init` |
| `best_loss` | 历史最低损失 | `+inf` |
| `step` | 当前优化步数 | `0` |
| `jb_success` | 越狱成功的 goal 列表（按 step 记录） | `[]` |
| `log` | 实验日志（每 test_steps 记录 loss、JB%、control 字符串） | `{}` |

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
  1. 获取嵌入权重矩阵 `E = model.get_input_embeddings().weight`，shape `(vocab_size, embed_dim)`
  2. 对当前控制 token 创建 one-hot 编码 `oh: Tensor[n_ctrl, vocab_size]`，设 `oh.requires_grad = True`
  3. 计算可微嵌入 `ctrl_embeds = oh @ E`，shape `(n_ctrl, embed_dim)`
  4. 拼接完整输入嵌入：`full_embeds = cat([embed(before_ctrl), ctrl_embeds, embed(after_ctrl)])`
  5. 前向传播：`logits = model(inputs_embeds=full_embeds).logits`
  6. 计算交叉熵损失：`loss = CrossEntropyLoss(logits[0, loss_slice, :], targets)`
  7. 反向传播：`loss.backward()`
  8. 返回 `oh.grad`
- **形式化:**

$$\nabla_{\mathbf{h}} \mathcal{L}_{\text{CE}}\bigl(f_\theta(\mathbf{e}_{<c} \oplus \mathbf{h} E \oplus \mathbf{e}_{>c}),\; \mathbf{t}\bigr)$$

其中 $\mathbf{h} \in \{0,1\}^{L \times |\mathcal{V}|}$ 为 one-hot 矩阵，$E$ 为嵌入矩阵。

---

### 4.2 基于梯度采样候选后缀

```python
def sample_control(
    grad: torch.Tensor,             # shape: (n_ctrl, vocab_size)
    batch_size: int,                # 候选后缀数量
    topk: int,                      # 每位置保留的 top-k token 数
    temperature: float,             # 采样温度
    allow_non_ascii: bool,          # 是否允许非 ASCII token
    not_allowed_ids: torch.Tensor,  # shape: (n_forbidden,) 禁止的 token ID
) -> torch.Tensor:                  # shape: (batch_size, n_ctrl)
```
- **输出:** 候选后缀 token ID 序列
- **逻辑:**
  1. 如果 `not allow_non_ascii`：将 `not_allowed_ids` 对应位置的梯度设为 `+inf`
  2. 取梯度最负的 topk 个 token：`topk_ids = topk(-grad, k=topk, dim=-1).indices`，shape `(n_ctrl, topk)`
  3. 对 `batch_size` 个候选：
     - 随机选一个位置 `pos ~ Uniform(0, n_ctrl-1)`
     - 在该位置的 topk token 中随机选一个 `tok ~ Uniform(topk_ids[pos])`
     - 复制当前 control，将 `control[pos] = tok`
  4. 返回所有候选

---

### 4.3 过滤 tokenize 不一致的候选

```python
def get_filtered_cands(
    control_toks: torch.Tensor,     # shape: (batch_size, n_ctrl)
    tokenizer: PreTrainedTokenizer, # 用于 decode → encode 验证
) -> torch.Tensor:                  # shape: (batch_size', n_ctrl), batch_size' <= batch_size
```
- **输出:** 过滤后的候选（仅保留 tokenize 后长度不变的）
- **逻辑:**
  1. 对每个候选：decode → encode，检查 token 长度是否仍为 `n_ctrl`
  2. 仅保留长度不变的候选（tokenize 一致性过滤）

---

### 4.4 计算目标回复上的交叉熵损失

```python
def target_loss(
    logits: torch.Tensor,           # shape: (batch_size, seq_len, vocab_size)
    target_ids: torch.Tensor,       # shape: (batch_size, target_len)
    target_slice: slice,            # 目标 token 区间
    loss_slice: slice,              # 损失计算区间 (target_slice 左移 1)
) -> torch.Tensor:                  # shape: (batch_size, target_len)
```
- **输出:** 每个 target token 上的交叉熵损失
- **逻辑:**
  1. `crit = CrossEntropyLoss(reduction='none')`
  2. `loss = crit(logits[:, loss_slice, :].transpose(1,2), target_ids[:, target_slice])`
  3. `loss_slice` = `target_slice` 向左偏移 1（next-token prediction 偏移）
- **形式化:**

$$\mathcal{L}_{\text{target}} = -\sum_{j=1}^{|\mathbf{t}|} \log P_\theta(t_j \mid \text{prompt} \oplus \mathbf{c},\; t_{<j})$$

---

### 4.5 计算对抗后缀上的正则损失

```python
def control_loss(
    logits: torch.Tensor,           # shape: (batch_size, seq_len, vocab_size)
    control_ids: torch.Tensor,      # shape: (batch_size, n_ctrl)
    control_slice: slice,           # 对抗后缀 token 区间
    loss_slice: slice,              # 损失计算区间 (control_slice 左移 1)
) -> torch.Tensor:                  # shape: (batch_size, n_ctrl)
```
- **输出:** 控制后缀 token 上的交叉熵损失（正则项）
- **逻辑:** 同 `target_loss`，但 `loss_slice` 和 `control_slice` 对应控制后缀区间

---

### 4.6 加载模型并创建 Worker 进程

```python
def get_workers(
    params: ml_collections.ConfigDict,  # 含 model_paths, tokenizer_paths, devices 等
) -> Tuple[List[ModelWorker], List[ModelWorker]]:
```
- **输出:** `(train_workers, test_workers)` — 训练用和测试用 ModelWorker 列表
- **逻辑:**
  1. 对每个 tokenizer_path：加载 `AutoTokenizer`，加载对话模板 `conv_template`
  2. 对每个 model_path：用 `AutoModelForCausalLM.from_pretrained(dtype=float16)` 加载模型
  3. 创建 `ModelWorker`，启动子进程，模型移入指定 GPU
  4. 划分 train/test workers（前 `num_train_models` 个用于梯度计算）

---

### 4.7 加载对抗行为数据集

```python
def get_goals_and_targets(
    params: ml_collections.ConfigDict,  # 含 train_data, n_train_data, n_test_data, data_offset
) -> Tuple[List[str], List[str], List[str], List[str]]:
```
- **输出:** `(train_goals, train_targets, test_goals, test_targets)`
- **逻辑:**
  1. 读取 CSV 文件（AdvBench 格式，含 `goal` 和 `target` 列）
  2. 从 `data_offset` 开始取 `n_train_data` 条作为训练数据
  3. 后续 `n_test_data` 条作为测试数据
  4. 可选：对目标字符串添加前缀空格

---

### 4.8 模拟退火接受判定

```python
def simulate_annealing_accept(
    e_current: float,               # 当前后缀对应的损失
    e_candidate: float,             # 候选后缀对应的损失
    step: int,                      # 当前优化步数
    n_steps: int,                   # 优化总步数
    anneal_from: int,               # 退火起始偏移
) -> bool:                          # 是否接受该候选
```
- **输出:** 是否接受该候选后缀
- **逻辑:**
  1. 如果 `e_candidate < e_current`：返回 `True`
  2. 否则：计算温度 $T = \max\bigl(1 - \frac{\text{step}+1}{\text{n\_steps} + \text{anneal\_from}},\; 10^{-7}\bigr)$
  3. 以概率 $\exp\bigl(-\frac{e_{\text{cand}} - e_{\text{curr}}}{T}\bigr)$ 接受
- **形式化:**

$$P(e, e', k) = \begin{cases} 1 & \text{if } e' < e \\ \exp\!\bigl(-\frac{e'-e}{T_k}\bigr) & \text{otherwise} \end{cases}, \quad T_k = 1 - \frac{k+1}{K + K_0}$$

---

## 【5. Gotcha 账本】

> 初始化时无需编写。随着实验迭代，在此记录踩过的坑。格式如下：

<!-- 示例：
- G1: KV Cache 与梯度计算不兼容
  - 表现: 开启 use_cache=True 后 token_gradients 返回 None 或 OOM
  - 原因: torch.enable_grad() 下 KV Cache 的中间状态不可对 one-hot 嵌入求导
  - 规则: 训练（梯度计算）时 use_cache=False，评估（logits/generate）时 use_cache=True
  - 对应自检: I1
  - 更新时间: 2024-01 initial GCG porting
-->
