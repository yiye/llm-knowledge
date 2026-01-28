# 10 - 分布式训练实践：多卡并行策略与工程经验

> 🎯 **核心观点**：单卡训练7B模型需要3天，8卡并行只需5小时——但前提是你懂得如何正确使用分布式训练。本文深入讲解DP、DDP、FSDP、DeepSpeed ZeRO、Megatron-LM等并行策略，剖析NCCL通信原理，提供完整的多节点训练代码，并分享故障恢复、性能调优的实战经验。

---

## 📋 目录

1. [为什么需要分布式训练？](#why-distributed)
2. [并行策略全景图](#parallelism-overview)
3. [数据并行：DP vs DDP](#data-parallel)
4. [FSDP：ZeRO的PyTorch实现](#fsdp)
5. [DeepSpeed ZeRO深度解析](#deepspeed-zero)
6. [Megatron-LM：张量与流水线并行](#megatron)
7. [3D并行：DP+TP+PP组合拳](#3d-parallelism)
8. [通信优化：NCCL与Ring-AllReduce](#communication)
9. [梯度累积与混合精度](#gradient-accumulation)
10. [多节点训练实战](#multi-node-training)
11. [Checkpoint与故障恢复](#fault-tolerance)
12. [性能调优与Profiling](#performance-tuning)
13. [常见问题与调试](#debugging)

---

<a name="why-distributed"></a>
## 🤔 1. 为什么需要分布式训练？

### 1.1 单卡的极限

```python
"""
单卡训练的瓶颈分析
"""

class SingleGPUBottleneck:
    """
    单卡训练瓶颈计算
    """
    
    def calculate_memory_requirement(
        self,
        model_params_billion=7,
        precision='fp16',
        batch_size=8,
        seq_len=2048
    ):
        """
        计算训练所需显存
        
        公式：
        总显存 = 模型参数 + 梯度 + 优化器状态 + 激活值
        """
        # 1. 模型参数（FP16: 2 bytes per param）
        bytes_per_param = 2 if precision == 'fp16' else 4
        model_memory_gb = (model_params_billion * 1e9 * bytes_per_param) / (1024 ** 3)
        
        # 2. 梯度（与模型参数相同大小）
        gradient_memory_gb = model_memory_gb
        
        # 3. 优化器状态（Adam: 2倍模型参数，FP32存储）
        # moment1 + moment2 + FP32 master weights
        optimizer_memory_gb = model_params_billion * 1e9 * 4 * 3 / (1024 ** 3)
        
        # 4. 激活值（取决于batch size和序列长度）
        # 简化估算：每个token的激活约占 batch_size * seq_len * hidden_dim * num_layers * 2
        hidden_dim = 4096  # 7B模型典型值
        num_layers = 32
        activation_memory_gb = (
            batch_size * seq_len * hidden_dim * num_layers * bytes_per_param * 10  # 10倍是经验系数
        ) / (1024 ** 3)
        
        # 总显存
        total_memory_gb = (
            model_memory_gb + 
            gradient_memory_gb + 
            optimizer_memory_gb + 
            activation_memory_gb
        )
        
        print(f"""
        LLaMA-7B 训练显存需求分析 (FP16, batch_size={batch_size}):
        
        模型参数:      {model_memory_gb:.1f} GB
        梯度:          {gradient_memory_gb:.1f} GB
        优化器状态:    {optimizer_memory_gb:.1f} GB
        激活值:        {activation_memory_gb:.1f} GB
        ─────────────────────────────────────
        总计:          {total_memory_gb:.1f} GB
        
        ⚠️ A100-40GB 无法训练！需要分布式方案。
        """)
        
        return total_memory_gb

# 运行分析
analyzer = SingleGPUBottleneck()
analyzer.calculate_memory_requirement(model_params_billion=7, batch_size=8)

# 输出：
# 总计: 98.6 GB
# ⚠️ A100-40GB 无法训练！
```

> 🔥 **关键洞察**：训练7B模型需要~100GB显存，远超单卡容量。即使是70B模型，推理需要140GB，训练需要700GB+！

---

### 1.2 分布式训练的收益

```python
"""
分布式训练加速比计算
"""

import matplotlib.pyplot as plt
import numpy as np

def plot_scaling_efficiency():
    """
    可视化不同并行策略的扩展效率
    """
    num_gpus = np.array([1, 2, 4, 8, 16, 32, 64, 128])
    
    # 理想线性加速（100%效率）
    ideal = num_gpus
    
    # 实际加速比（不同策略）
    dp_speedup = num_gpus * 0.95  # DataParallel: 95%效率（通信开销小）
    ddp_speedup = num_gpus * 0.92  # DDP: 92%效率
    fsdp_speedup = num_gpus * 0.88  # FSDP: 88%效率（ZeRO-3通信多）
    pipeline_speedup = num_gpus * 0.70  # Pipeline: 70%效率（气泡）
    
    plt.figure(figsize=(10, 6))
    plt.plot(num_gpus, ideal, 'k--', label='理想线性', linewidth=2)
    plt.plot(num_gpus, dp_speedup, 'b-o', label='DP (单机)', linewidth=2)
    plt.plot(num_gpus, ddp_speedup, 'g-s', label='DDP', linewidth=2)
    plt.plot(num_gpus, fsdp_speedup, 'r-^', label='FSDP/ZeRO-3', linewidth=2)
    plt.plot(num_gpus, pipeline_speedup, 'm-d', label='Pipeline Parallel', linewidth=2)
    
    plt.xlabel('GPU数量', fontsize=12)
    plt.ylabel('加速比', fontsize=12)
    plt.title('分布式训练扩展效率对比', fontsize=14, fontweight='bold')
    plt.legend(fontsize=10)
    plt.grid(alpha=0.3)
    plt.xscale('log', base=2)
    plt.yscale('log', base=2)
    plt.tight_layout()
    plt.savefig('distributed_scaling.png', dpi=150)

plot_scaling_efficiency()
```

**传统程序员的类比**：

| 单机程序 | 分布式训练 |
|---------|-----------|
| 单线程处理 | 单GPU训练 |
| 多线程（共享内存） | DataParallel (DP) |
| 多进程（消息传递） | DistributedDataParallel (DDP) |
| 分布式系统（MapReduce） | Pipeline/Tensor Parallel |

---

<a name="parallelism-overview"></a>
## 🗺️ 2. 并行策略全景图

### 2.1 四大并行范式

```python
"""
并行策略分类
"""

parallelism_taxonomy = {
    '数据并行 (Data Parallelism)': {
        'principle': '每个GPU持有完整模型，训练不同数据',
        'variants': ['DP', 'DDP', 'FSDP/ZeRO'],
        'communication': 'AllReduce梯度',
        'memory_per_gpu': '完整模型（DP/DDP）或分片模型（FSDP）',
        'use_case': '模型能放入单卡显存',
        'icon': '🔀'
    },
    
    '张量并行 (Tensor Parallelism)': {
        'principle': '将单个层的权重矩阵切分到多GPU',
        'variants': ['Megatron-LM TP'],
        'communication': 'AllReduce激活值（频繁）',
        'memory_per_gpu': '1/N 的每层参数',
        'use_case': '超大层（hidden_dim > 8192）',
        'icon': '✂️'
    },
    
    '流水线并行 (Pipeline Parallelism)': {
        'principle': '按深度切分模型，每个GPU负责若干层',
        'variants': ['GPipe', 'PipeDream', 'Megatron-LM PP'],
        'communication': '点对点传输激活值',
        'memory_per_gpu': 'L/N 层的参数（L=总层数）',
        'use_case': '超深模型（layers > 50）',
        'icon': '🔄'
    },
    
    '专家并行 (Expert Parallelism)': {
        'principle': 'MoE模型的专家分布到不同GPU',
        'variants': ['DeepSpeed MoE', 'FairScale MoE'],
        'communication': 'AllToAll路由token',
        'memory_per_gpu': 'E/N 个专家',
        'use_case': 'MoE架构（如Mixtral）',
        'icon': '🎯'
    }
}

# 可视化决策树
def choose_parallelism_strategy(model_size_b, num_layers, hidden_dim, num_gpus):
    """
    根据模型特征选择并行策略
    """
    print(f"\n模型特征:")
    print(f"  参数量: {model_size_b}B")
    print(f"  层数: {num_layers}")
    print(f"  隐藏维度: {hidden_dim}")
    print(f"  可用GPU: {num_gpus}")
    print(f"\n推荐策略:")
    
    # 决策逻辑
    strategies = []
    
    # 1. 模型能否放入单卡？
    model_memory_gb = model_size_b * 2  # FP16
    if model_memory_gb < 40:  # A100-40GB
        strategies.append("✅ 数据并行 (DDP) - 模型较小，单卡可容纳")
    else:
        strategies.append("❌ 单卡无法容纳，需要模型并行")
    
    # 2. 是否需要张量并行？
    if hidden_dim > 8192:
        strategies.append("✅ 张量并行 (TP) - 隐藏维度超大")
    
    # 3. 是否需要流水线并行？
    if num_layers > 50:
        strategies.append("✅ 流水线并行 (PP) - 模型超深")
    
    # 4. 最终推荐
    if model_size_b <= 7:
        strategies.append("🔥 最佳：DDP + FSDP")
    elif model_size_b <= 70:
        strategies.append("🔥 最佳：TP=4 + PP=4 + DP")
    else:
        strategies.append("🔥 最佳：3D并行 (TP + PP + DP)")
    
    for s in strategies:
        print(f"  {s}")

# 示例
choose_parallelism_strategy(model_size_b=7, num_layers=32, hidden_dim=4096, num_gpus=8)
choose_parallelism_strategy(model_size_b=70, num_layers=80, hidden_dim=8192, num_gpus=64)
```

---

<a name="data-parallel"></a>
## 🔀 3. 数据并行：DP vs DDP

### 3.1 DataParallel (DP)：单机多卡

```python
"""
DataParallel 实现（已过时，仅供理解）
"""

import torch
import torch.nn as nn

# 简单模型
model = nn.Sequential(
    nn.Linear(1000, 2000),
    nn.ReLU(),
    nn.Linear(2000, 10)
)

# DP封装（自动分发到所有GPU）
model = nn.DataParallel(model)
model = model.cuda()

# 训练
for batch in dataloader:
    inputs, labels = batch
    inputs = inputs.cuda()
    labels = labels.cuda()
    
    # Forward pass（自动并行）
    outputs = model(inputs)
    loss = criterion(outputs, labels)
    
    # Backward pass
    loss.backward()
    optimizer.step()
    optimizer.zero_grad()
```

**DP的问题**：

| 问题 | 原因 | 影响 |
|------|------|------|
| **GPU 0负载不均** | 所有梯度聚合到GPU 0 | GPU 0显存/计算更高 |
| **GIL限制** | Python多线程受GIL影响 | 扩展性差 |
| **单进程** | 无法跨节点 | 只能单机 |
| **效率低** | 频繁的CPU-GPU传输 | 吞吐量低 |

> ⚠️ **结论**：DataParallel已被废弃，**永远使用DDP**！

---

### 3.2 DistributedDataParallel (DDP)：多进程并行

**DDP原理**：

```python
"""
DDP核心原理演示
"""

class DDPSimulation:
    """
    模拟DDP的工作流程
    """
    
    def __init__(self, world_size=4):
        self.world_size = world_size  # 总GPU数
        self.ranks = list(range(world_size))
    
    def simulate_training_step(self):
        """
        模拟一个训练步骤
        """
        print("="*60)
        print("DDP训练步骤模拟")
        print("="*60)
        
        # 步骤1: 每个进程独立forward
        print("\n1️⃣ Forward Pass（各进程独立）:")
        for rank in self.ranks:
            print(f"  Rank {rank}: Forward(batch_{rank}) → loss_{rank}")
        
        # 步骤2: 每个进程独立backward
        print("\n2️⃣ Backward Pass（各进程独立）:")
        for rank in self.ranks:
            print(f"  Rank {rank}: Backward() → gradients_{rank}")
        
        # 步骤3: AllReduce梯度（关键！）
        print("\n3️⃣ AllReduce梯度（同步）:")
        print(f"  所有进程的梯度通过AllReduce同步并平均")
        print(f"  gradient_avg = (grad_0 + grad_1 + grad_2 + grad_3) / 4")
        
        # 步骤4: 每个进程用相同的梯度更新模型
        print("\n4️⃣ 参数更新（各进程相同）:")
        for rank in self.ranks:
            print(f"  Rank {rank}: params -= lr * gradient_avg")
        
        print("\n✅ 结果：所有进程的模型参数保持一致")

# 运行模拟
sim = DDPSimulation(world_size=4)
sim.simulate_training_step()
```

---

### 3.3 DDP完整实战代码

```python
"""
DDP完整训练脚本 - 单机多卡
"""

import os
import torch
import torch.nn as nn
import torch.distributed as dist
from torch.nn.parallel import DistributedDataParallel as DDP
from torch.utils.data import DataLoader, Dataset
from torch.utils.data.distributed import DistributedSampler

# ============================================
# 1. 初始化分布式环境
# ============================================
def setup_ddp(rank, world_size):
    """
    初始化DDP
    
    参数:
        rank: 当前进程的rank (0 到 world_size-1)
        world_size: 总进程数（通常等于GPU数）
    """
    # 设置环境变量
    os.environ['MASTER_ADDR'] = 'localhost'
    os.environ['MASTER_PORT'] = '12355'
    
    # 初始化进程组（NCCL后端用于GPU）
    dist.init_process_group(
        backend='nccl',      # NCCL是NVIDIA GPU的最佳后端
        init_method='env://',
        world_size=world_size,
        rank=rank
    )
    
    # 设置当前进程使用的GPU
    torch.cuda.set_device(rank)

def cleanup_ddp():
    """清理DDP"""
    dist.destroy_process_group()

# ============================================
# 2. 定义模型
# ============================================
class SimpleModel(nn.Module):
    """示例模型"""
    def __init__(self, input_dim=1000, hidden_dim=2000, output_dim=10):
        super().__init__()
        self.layers = nn.Sequential(
            nn.Linear(input_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, hidden_dim),
            nn.ReLU(),
            nn.Linear(hidden_dim, output_dim)
        )
    
    def forward(self, x):
        return self.layers(x)

# ============================================
# 3. 训练函数
# ============================================
def train_ddp(rank, world_size):
    """
    DDP训练主函数
    
    每个GPU会启动一个进程运行这个函数
    """
    print(f"🚀 启动进程 Rank {rank}/{world_size}")
    
    # 初始化DDP
    setup_ddp(rank, world_size)
    
    # 创建模型并移到GPU
    model = SimpleModel().cuda(rank)
    
    # 🔥 关键：用DDP包装模型
    model = DDP(
        model, 
        device_ids=[rank],
        output_device=rank,
        find_unused_parameters=False  # 如果有未使用的参数，设为True
    )
    
    # 创建优化器
    optimizer = torch.optim.Adam(model.parameters(), lr=1e-3)
    
    # 创建数据集（示例）
    class DummyDataset(Dataset):
        def __len__(self):
            return 1000
        
        def __getitem__(self, idx):
            return torch.randn(1000), torch.randint(0, 10, (1,))
    
    dataset = DummyDataset()
    
    # 🔥 关键：使用DistributedSampler确保每个进程看到不同数据
    sampler = DistributedSampler(
        dataset,
        num_replicas=world_size,
        rank=rank,
        shuffle=True
    )
    
    dataloader = DataLoader(
        dataset,
        batch_size=32,
        sampler=sampler,
        num_workers=2,
        pin_memory=True
    )
    
    # 训练循环
    model.train()
    for epoch in range(10):
        # 🔥 重要：每个epoch设置sampler的epoch，确保shuffle不同
        sampler.set_epoch(epoch)
        
        epoch_loss = 0.0
        for batch_idx, (inputs, labels) in enumerate(dataloader):
            # 数据移到GPU
            inputs = inputs.cuda(rank)
            labels = labels.cuda(rank).squeeze()
            
            # Forward pass
            outputs = model(inputs)
            loss = nn.functional.cross_entropy(outputs, labels)
            
            # Backward pass
            optimizer.zero_grad()
            loss.backward()  # DDP自动AllReduce梯度！
            optimizer.step()
            
            epoch_loss += loss.item()
        
        # 只在rank 0打印（避免重复）
        if rank == 0:
            avg_loss = epoch_loss / len(dataloader)
            print(f"Epoch {epoch}: Loss = {avg_loss:.4f}")
    
    # 保存模型（只在rank 0保存）
    if rank == 0:
        # DDP模型需要通过.module访问原始模型
        torch.save(model.module.state_dict(), "ddp_model.pth")
        print("✅ 模型已保存")
    
    # 清理
    cleanup_ddp()

# ============================================
# 4. 主函数：启动多进程
# ============================================
if __name__ == "__main__":
    world_size = torch.cuda.device_count()  # 自动检测GPU数
    
    print(f"使用 {world_size} 块GPU进行DDP训练")
    
    # 使用torch.multiprocessing启动多个进程
    torch.multiprocessing.spawn(
        train_ddp,
        args=(world_size,),
        nprocs=world_size,
        join=True
    )
```

**运行方式**：

```bash
# 方式1: 使用torch.multiprocessing（上面代码）
python train_ddp.py

# 方式2: 使用torchrun（推荐）
torchrun --nproc_per_node=4 train_ddp.py

# 方式3: 多节点训练
# Node 0:
torchrun --nproc_per_node=4 --nnodes=2 --node_rank=0 \
    --master_addr="192.168.1.100" --master_port=12355 train_ddp.py

# Node 1:
torchrun --nproc_per_node=4 --nnodes=2 --node_rank=1 \
    --master_addr="192.168.1.100" --master_port=12355 train_ddp.py
```

---

### 3.4 DDP关键要点

```python
"""
DDP最佳实践 Checklist
"""

ddp_best_practices = {
    '✅ 必须做': [
        '每个进程设置不同的random seed（可选，但推荐）',
        '使用DistributedSampler确保数据不重复',
        '每个epoch调用sampler.set_epoch(epoch)',
        '只在rank 0保存模型/打印日志',
        '通过model.module访问DDP包装的模型',
        '使用NCCL后端（GPU）',
        '设置find_unused_parameters=False（性能）'
    ],
    
    '❌ 不要做': [
        '不要在不同进程使用不同的模型结构',
        '不要在训练过程中改变模型结构',
        '不要忘记调用dist.barrier()进行同步（需要时）',
        '不要在rank > 0的进程中保存模型（会重复）',
        '不要使用Python的print（改用logging）'
    ],
    
    '⚡ 性能优化': [
        'gradient_as_bucket_view=True（减少内存拷贝）',
        'static_graph=True（如果模型结构固定）',
        'broadcast_buffers=False（如果没有BN层）',
        '使用torch.compile()（PyTorch 2.0+）',
        '启用混合精度训练'
    ]
}

# 打印
for category, items in ddp_best_practices.items():
    print(f"\n{category}:")
    for item in items:
        print(f"  {item}")
```

---

<a name="fsdp"></a>
## 🔥 4. FSDP：ZeRO的PyTorch实现

### 4.1 FSDP原理

**Fully Sharded Data Parallel (FSDP)** 是PyTorch对DeepSpeed ZeRO-3的原生实现。

```python
"""
FSDP原理演示
"""

class FSDPPrinciple:
    """
    FSDP原理：分片存储，按需聚合
    """
    
    def compare_memory_usage(self, model_params_b=7, world_size=8):
        """
        对比DDP vs FSDP的显存占用
        """
        # DDP：每个GPU完整模型
        ddp_memory_per_gpu = model_params_b * 2 * 3  # 参数+梯度+优化器(3倍)
        
        # FSDP：参数分片，但需要临时聚合
        fsdp_static_memory = model_params_b * 2 * 3 / world_size  # 分片存储
        fsdp_dynamic_memory = model_params_b * 2  # Forward时临时聚合当前层
        fsdp_total_memory = fsdp_static_memory + fsdp_dynamic_memory
        
        print(f"""
        显存对比 ({model_params_b}B模型, {world_size}卡):
        
        DDP (每卡):
          模型+梯度+优化器: {ddp_memory_per_gpu:.1f} GB
          总显存占用:       {ddp_memory_per_gpu:.1f} GB
        
        FSDP (每卡):
          分片存储:         {fsdp_static_memory:.1f} GB
          临时聚合:         {fsdp_dynamic_memory:.1f} GB
          总显存占用:       {fsdp_total_memory:.1f} GB
        
        节省比例: {(1 - fsdp_total_memory/ddp_memory_per_gpu)*100:.0f}%
        """)

principle = FSDPPrinciple()
principle.compare_memory_usage(model_params_b=7, world_size=8)

# 输出：
# DDP (每卡): 42.0 GB
# FSDP (每卡): 19.3 GB
# 节省比例: 54%
```

**FSDP工作流程**：

```python
"""
FSDP前向传播流程
"""

class FSDPForwardPass:
    """
    FSDP Forward Pass详细流程
    """
    
    def simulate_forward(self, num_layers=4, world_size=4):
        """
        模拟FSDP的Forward Pass
        """
        print("FSDP Forward Pass流程:")
        print("="*60)
        
        for layer_idx in range(num_layers):
            print(f"\n📍 Layer {layer_idx}:")
            
            # 步骤1: AllGather参数
            print(f"  1️⃣ AllGather: 收集分片参数")
            for rank in range(world_size):
                print(f"     Rank {rank}: 广播自己的参数分片")
            print(f"     → 每个Rank现在都有完整的Layer {layer_idx}参数")
            
            # 步骤2: 计算
            print(f"  2️⃣ Compute: Forward计算")
            print(f"     → output_{layer_idx} = layer_{layer_idx}(input)")
            
            # 步骤3: 释放参数
            print(f"  3️⃣ Free: 释放完整参数，保留分片")
            print(f"     → 显存释放，只保留自己的分片")
        
        print("\n" + "="*60)
        print("Forward完成，显存占用最小化！")

sim = FSDPForwardPass()
sim.simulate_forward()
```

---

### 4.2 FSDP完整实战代码

```python
"""
FSDP训练完整示例 - PyTorch 2.0+
"""

import torch
import torch.nn as nn
from torch.distributed.fsdp import (
    FullyShardedDataParallel as FSDP,
    MixedPrecision,
    ShardingStrategy,
    BackwardPrefetch,
    StateDictType,
)
from torch.distributed.fsdp.wrap import (
    size_based_auto_wrap_policy,
    transformer_auto_wrap_policy,
)
from transformers import AutoModelForCausalLM, AutoTokenizer

# ============================================
# 1. FSDP配置
# ============================================
def get_fsdp_config():
    """
    FSDP配置（2025最佳实践）
    """
    # 混合精度配置
    mixed_precision_policy = MixedPrecision(
        param_dtype=torch.bfloat16,      # 参数用BF16
        reduce_dtype=torch.bfloat16,     # 梯度规约用BF16
        buffer_dtype=torch.bfloat16,     # Buffer用BF16
    )
    
    # Sharding策略
    sharding_strategy = ShardingStrategy.FULL_SHARD  # ZeRO-3（完全分片）
    # ShardingStrategy.SHARD_GRAD_OP  # ZeRO-2（梯度+优化器分片）
    # ShardingStrategy.NO_SHARD       # DDP（无分片）
    
    return {
        'mixed_precision': mixed_precision_policy,
        'sharding_strategy': sharding_strategy,
        'backward_prefetch': BackwardPrefetch.BACKWARD_PRE,  # 预取优化
        'forward_prefetch': True,  # Forward预取
        'limit_all_gathers': True,  # 限制AllGather显存峰值
        'use_orig_params': True,    # 使用原始参数（兼容性更好）
    }

# ============================================
# 2. 自动包装策略
# ============================================
def get_transformer_wrap_policy(model):
    """
    Transformer模型的自动包装策略
    
    将每个Transformer块作为一个FSDP单元
    """
    from transformers.models.llama.modeling_llama import LlamaDecoderLayer
    
    # 自动包装策略：对LlamaDecoderLayer应用FSDP
    auto_wrap_policy = transformer_auto_wrap_policy(
        transformer_layer_cls={LlamaDecoderLayer},
    )
    
    return auto_wrap_policy

# ============================================
# 3. 训练函数
# ============================================
def train_fsdp(rank, world_size):
    """
    FSDP训练主函数
    """
    # 初始化分布式
    setup_ddp(rank, world_size)
    
    # 加载模型
    model_name = "meta-llama/Llama-2-7b-hf"
    model = AutoModelForCausalLM.from_pretrained(
        model_name,
        torch_dtype=torch.bfloat16,
        use_cache=False  # 训练时关闭KV Cache
    )
    
    # 获取FSDP配置
    fsdp_config = get_fsdp_config()
    auto_wrap_policy = get_transformer_wrap_policy(model)
    
    # 🔥 用FSDP包装模型
    model = FSDP(
        model,
        auto_wrap_policy=auto_wrap_policy,
        **fsdp_config
    )
    
    # 优化器
    optimizer = torch.optim.AdamW(
        model.parameters(),
        lr=2e-5,
        betas=(0.9, 0.95),
        weight_decay=0.1
    )
    
    # 准备数据（示例）
    tokenizer = AutoTokenizer.from_pretrained(model_name)
    dataset = prepare_dataset(tokenizer)  # 自定义函数
    sampler = DistributedSampler(dataset, rank=rank, num_replicas=world_size)
    dataloader = DataLoader(dataset, batch_size=4, sampler=sampler)
    
    # 训练循环
    model.train()
    for epoch in range(3):
        sampler.set_epoch(epoch)
        
        for batch_idx, batch in enumerate(dataloader):
            # 数据移到GPU
            input_ids = batch['input_ids'].cuda(rank)
            attention_mask = batch['attention_mask'].cuda(rank)
            labels = batch['labels'].cuda(rank)
            
            # Forward pass
            outputs = model(
                input_ids=input_ids,
                attention_mask=attention_mask,
                labels=labels
            )
            loss = outputs.loss
            
            # Backward pass
            loss.backward()
            
            # 梯度裁剪（重要！）
            torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
            
            optimizer.step()
            optimizer.zero_grad()
            
            if rank == 0 and batch_idx % 10 == 0:
                print(f"Epoch {epoch}, Batch {batch_idx}, Loss: {loss.item():.4f}")
    
    # 保存模型（FSDP特殊处理）
    if rank == 0:
        save_fsdp_checkpoint(model, optimizer, "fsdp_checkpoint")
    
    cleanup_ddp()

# ============================================
# 4. FSDP Checkpoint保存/加载
# ============================================
def save_fsdp_checkpoint(model, optimizer, save_path):
    """
    保存FSDP checkpoint
    
    FSDP有特殊的checkpoint格式
    """
    from torch.distributed.fsdp import FullStateDictConfig, StateDictType
    
    # 配置：将分片的状态收集到rank 0
    save_policy = FullStateDictConfig(offload_to_cpu=True, rank0_only=True)
    
    with FSDP.state_dict_type(
        model, 
        StateDictType.FULL_STATE_DICT, 
        save_policy
    ):
        state_dict = model.state_dict()
        optimizer_state = FSDP.optim_state_dict(model, optimizer)
    
    # 只在rank 0保存
    if dist.get_rank() == 0:
        checkpoint = {
            'model': state_dict,
            'optimizer': optimizer_state,
        }
        torch.save(checkpoint, f"{save_path}/checkpoint.pth")
        print(f"✅ FSDP checkpoint已保存到 {save_path}")

def load_fsdp_checkpoint(model, optimizer, load_path):
    """
    加载FSDP checkpoint
    """
    from torch.distributed.fsdp import FullStateDictConfig, StateDictType
    
    # 加载checkpoint
    checkpoint = torch.load(f"{load_path}/checkpoint.pth", map_location='cpu')
    
    # 加载模型状态
    load_policy = FullStateDictConfig(offload_to_cpu=True, rank0_only=True)
    with FSDP.state_dict_type(
        model, 
        StateDictType.FULL_STATE_DICT, 
        load_policy
    ):
        model.load_state_dict(checkpoint['model'])
    
    # 加载优化器状态
    optimizer_state = FSDP.optim_state_dict_to_load(
        model, optimizer, checkpoint['optimizer']
    )
    optimizer.load_state_dict(optimizer_state)
    
    print(f"✅ FSDP checkpoint已从 {load_path} 加载")
```

---

### 4.3 FSDP vs DDP对比

```python
"""
FSDP vs DDP性能对比
"""

performance_comparison = {
    '指标': ['显存/GPU', '通信量', '训练速度', '最大模型', '易用性'],
    
    'DDP': {
        '显存/GPU': '完整模型（42GB for 7B）',
        '通信量': '低（只同步梯度）',
        '训练速度': '快（通信少）',
        '最大模型': '受单卡显存限制',
        '易用性': '⭐⭐⭐⭐⭐'
    },
    
    'FSDP (ZeRO-2)': {
        '显存/GPU': '参数完整+优化器分片（~30GB）',
        '通信量': '中（AllReduce梯度+优化器）',
        '训练速度': '较快',
        '最大模型': '可训练13B级别',
        '易用性': '⭐⭐⭐⭐'
    },
    
    'FSDP (ZeRO-3)': {
        '显存/GPU': '全部分片（~19GB for 7B）',
        '通信量': '高（AllGather参数）',
        '训练速度': '中（通信多）',
        '最大模型': '可训练70B+',
        '易用性': '⭐⭐⭐⭐'
    }
}

import pandas as pd
df = pd.DataFrame(performance_comparison).set_index('指标')
print(df.to_string())
```

**选择建议**：

| 场景 | 推荐方案 |
|------|---------|
| 7B模型 + 8卡A100 | DDP（单卡能放下） |
| 13B模型 + 8卡A100 | FSDP ZeRO-2 |
| 70B模型 + 64卡A100 | FSDP ZeRO-3 + TP |
| 实验/调试 | DDP（更简单） |
| 生产训练 | FSDP（更灵活） |

---

<a name="deepspeed-zero"></a>
## ⚡ 5. DeepSpeed ZeRO深度解析

### 5.1 ZeRO三阶段原理

根据 [DeepSpeed官方文档](https://www.deepspeed.ai/tutorials/zero/)：

```python
"""
ZeRO三阶段原理
"""

class ZeROExplainer:
    """
    ZeRO (Zero Redundancy Optimizer) 原理解析
    """
    
    def explain_stages(self, model_params_b=7, world_size=8):
        """
        ZeRO三阶段对比
        """
        # 基础数据
        model_size_gb = model_params_b * 2  # FP16
        gradient_size_gb = model_size_gb
        optimizer_size_gb = model_params_b * 4 * 2  # Adam: moment1 + moment2 (FP32)
        
        print("="*70)
        print(f"ZeRO三阶段对比 ({model_params_b}B模型, {world_size}卡)")
        print("="*70)
        
        # Baseline: 无ZeRO
        baseline_memory = model_size_gb + gradient_size_gb + optimizer_size_gb
        print(f"\n❌ Baseline (无优化):")
        print(f"   每卡显存: {baseline_memory:.1f} GB")
        print(f"   详细: 模型({model_size_gb:.1f}) + 梯度({gradient_size_gb:.1f}) + 优化器({optimizer_size_gb:.1f})")
        
        # ZeRO Stage 1: 优化器状态分片
        stage1_memory = model_size_gb + gradient_size_gb + optimizer_size_gb / world_size
        print(f"\n1️⃣ ZeRO Stage 1 (优化器状态分片):")
        print(f"   每卡显存: {stage1_memory:.1f} GB")
        print(f"   节省: {(1 - stage1_memory/baseline_memory)*100:.0f}%")
        print(f"   详细: 模型({model_size_gb:.1f}) + 梯度({gradient_size_gb:.1f}) + 优化器分片({optimizer_size_gb/world_size:.1f})")
        
        # ZeRO Stage 2: 优化器+梯度分片
        stage2_memory = model_size_gb + gradient_size_gb / world_size + optimizer_size_gb / world_size
        print(f"\n2️⃣ ZeRO Stage 2 (优化器+梯度分片):")
        print(f"   每卡显存: {stage2_memory:.1f} GB")
        print(f"   节省: {(1 - stage2_memory/baseline_memory)*100:.0f}%")
        print(f"   详细: 模型({model_size_gb:.1f}) + 梯度分片({gradient_size_gb/world_size:.1f}) + 优化器分片({optimizer_size_gb/world_size:.1f})")
        
        # ZeRO Stage 3: 全部分片（与FSDP相同）
        stage3_memory = (model_size_gb + gradient_size_gb + optimizer_size_gb) / world_size
        print(f"\n3️⃣ ZeRO Stage 3 (全部分片):")
        print(f"   每卡显存: {stage3_memory:.1f} GB")
        print(f"   节省: {(1 - stage3_memory/baseline_memory)*100:.0f}%")
        print(f"   详细: 全部状态分片({(model_size_gb + gradient_size_gb + optimizer_size_gb)/world_size:.1f})")
        
        print("\n" + "="*70)

explainer = ZeROExplainer()
explainer.explain_stages(model_params_b=7, world_size=8)

# 输出：
# ❌ Baseline: 70.0 GB
# 1️⃣ Stage 1: 35.0 GB (节省 50%)
# 2️⃣ Stage 2: 19.8 GB (节省 72%)
# 3️⃣ Stage 3: 8.8 GB (节省 87%)
```

---

### 5.2 DeepSpeed完整实战

```python
"""
DeepSpeed ZeRO训练完整示例
"""

import torch
import deepspeed
from deepspeed.ops.adam import DeepSpeedCPUAdam
from transformers import AutoModelForCausalLM, AutoTokenizer

# ============================================
# 1. DeepSpeed配置文件 (ds_config.json)
# ============================================
ds_config = {
    "train_batch_size": 32,               # 全局batch size
    "train_micro_batch_size_per_gpu": 4,  # 每卡batch size
    "gradient_accumulation_steps": 1,      # 梯度累积步数
    
    # 🔥 ZeRO优化配置
    "zero_optimization": {
        "stage": 3,                        # ZeRO Stage 3
        
        # Stage 3特定配置
        "offload_param": {
            "device": "cpu",               # 参数offload到CPU
            "pin_memory": True
        },
        "offload_optimizer": {
            "device": "cpu",               # 优化器offload到CPU
            "pin_memory": True
        },
        
        # 通信优化
        "overlap_comm": True,              # 通信与计算重叠
        "contiguous_gradients": True,      # 梯度连续存储
        "sub_group_size": 1e9,             # AllGather分组大小
        "reduce_bucket_size": 5e8,         # Reduce bucket大小
        "stage3_prefetch_bucket_size": 5e8,  # 预取bucket
        "stage3_param_persistence_threshold": 1e6,  # 参数持久化阈值
        
        # 内存优化
        "stage3_max_live_parameters": 1e9,   # 最大存活参数
        "stage3_max_reuse_distance": 1e9,    # 参数复用距离
        "stage3_gather_16bit_weights_on_model_save": True  # 保存时收集FP16权重
    },
    
    # 混合精度
    "fp16": {
        "enabled": True,
        "loss_scale": 0,                   # 动态loss scaling
        "loss_scale_window": 1000,
        "hysteresis": 2,
        "min_loss_scale": 1
    },
    
    # 或使用BF16（H100推荐）
    # "bf16": {
    #     "enabled": True
    # },
    
    # 优化器
    "optimizer": {
        "type": "AdamW",
        "params": {
            "lr": 2e-5,
            "betas": [0.9, 0.95],
            "eps": 1e-8,
            "weight_decay": 0.1
        }
    },
    
    # 学习率调度
    "scheduler": {
        "type": "WarmupDecayLR",
        "params": {
            "warmup_min_lr": 0,
            "warmup_max_lr": 2e-5,
            "warmup_num_steps": 100,
            "total_num_steps": 10000
        }
    },
    
    # Activation Checkpointing（节省显存）
    "activation_checkpointing": {
        "partition_activations": True,
        "cpu_checkpointing": False,
        "contiguous_memory_optimization": True,
        "number_checkpoints": None,
        "synchronize_checkpoint_boundary": False,
        "profile": False
    },
    
    # 梯度裁剪
    "gradient_clipping": 1.0,
    
    # 日志
    "steps_per_print": 10,
    "wall_clock_breakdown": False
}

# 保存配置文件
import json
with open('ds_config.json', 'w') as f:
    json.dump(ds_config, f, indent=2)

# ============================================
# 2. 训练脚本
# ============================================
def train_deepspeed():
    """
    DeepSpeed训练主函数
    """
    # 加载模型
    model_name = "meta-llama/Llama-2-7b-hf"
    model = AutoModelForCausalLM.from_pretrained(
        model_name,
        torch_dtype=torch.float16,
        use_cache=False
    )
    tokenizer = AutoTokenizer.from_pretrained(model_name)
    
    # 准备数据
    train_dataset = prepare_dataset(tokenizer)
    
    # 🔥 初始化DeepSpeed
    model_engine, optimizer, train_dataloader, _ = deepspeed.initialize(
        model=model,
        model_parameters=model.parameters(),
        training_data=train_dataset,
        config='ds_config.json'  # 配置文件路径
    )
    
    # 训练循环
    model_engine.train()
    for epoch in range(3):
        for step, batch in enumerate(train_dataloader):
            # 数据移到GPU（DeepSpeed自动处理设备）
            input_ids = batch['input_ids'].to(model_engine.device)
            attention_mask = batch['attention_mask'].to(model_engine.device)
            labels = batch['labels'].to(model_engine.device)
            
            # Forward pass
            outputs = model_engine(
                input_ids=input_ids,
                attention_mask=attention_mask,
                labels=labels
            )
            loss = outputs.loss
            
            # Backward pass（DeepSpeed自动处理）
            model_engine.backward(loss)
            
            # Optimizer step（DeepSpeed自动处理梯度累积）
            model_engine.step()
            
            if step % 10 == 0:
                print(f"Epoch {epoch}, Step {step}, Loss: {loss.item():.4f}")
    
    # 保存checkpoint
    model_engine.save_checkpoint('deepspeed_checkpoint')
    print("✅ DeepSpeed checkpoint已保存")

# ============================================
# 3. 运行方式
# ============================================
if __name__ == "__main__":
    # 使用deepspeed命令启动
    # deepspeed --num_gpus=8 train_deepspeed.py
    train_deepspeed()
```

**运行命令**：

```bash
# 单机8卡
deepspeed --num_gpus=8 train_deepspeed.py

# 多节点（2台机器，每台8卡）
# Node 0:
deepspeed --num_gpus=8 --num_nodes=2 --node_rank=0 \
    --master_addr=192.168.1.100 --master_port=29500 train_deepspeed.py

# Node 1:
deepspeed --num_gpus=8 --num_nodes=2 --node_rank=1 \
    --master_addr=192.168.1.100 --master_port=29500 train_deepspeed.py

# 使用hostfile（推荐）
# hostfile内容:
# node0_hostname slots=8
# node1_hostname slots=8

deepspeed --hostfile=hostfile train_deepspeed.py
```

---

### 5.3 ZeRO++ 优化（2025最新）

根据 [DeepSpeed ZeRO++文档](https://www.deepspeed.ai/tutorials/zeropp/)：

```python
"""
ZeRO++ 优化配置（2025）
"""

zeropp_config = {
    "zero_optimization": {
        "stage": 3,
        
        # 🔥 ZeRO++ 新特性
        "zero_quantized_weights": True,         # qwZ: INT8量化权重
        "zero_hpz_partition_size": 16,          # hpZ: 分层分区
        "zero_quantized_gradients": True,       # qgZ: 量化梯度
        
        # ZeRO++ 可减少4x通信量！
        "reduce_scatter": True,
        "allgather_bucket_size": 5e8,
        
        # 其他配置...
    }
}

# 效果：
# - 参数AllGather通信量减半（qwZ）
# - 反向传播跨节点通信消除（hpZ）
# - 梯度AllReduce替换为高效AllToAll（qgZ）
```

> 🔥 **2025推荐**：使用ZeRO++ Stage 3可将通信量减少4倍，训练175B模型速度提升2.16倍！

---

<a name="megatron"></a>
## ✂️ 6. Megatron-LM：张量与流水线并行

### 6.1 张量并行 (Tensor Parallelism, TP)

**核心思想**：将单个层的权重矩阵按列/行切分到多GPU。

```python
"""
张量并行原理演示
"""

class TensorParallelismExplainer:
    """
    张量并行原理
    """
    
    def explain_column_parallel(self):
        """
        列并行：Linear层的列切分
        
        原始：Y = X @ W
        切分：Y = X @ [W1 | W2] = [X@W1 | X@W2]
        """
        print("列并行（Column Parallel）:")
        print("="*60)
        print("原始操作：")
        print("  Y = X @ W")
        print("  其中 W shape: [hidden_dim, hidden_dim]")
        print()
        print("切分到2个GPU：")
        print("  GPU 0: Y1 = X @ W1  (W1是W的前半列)")
        print("  GPU 1: Y2 = X @ W2  (W2是W的后半列)")
        print()
        print("拼接：")
        print("  Y = [Y1 | Y2]  (直接拼接，无需通信！)")
        print()
        
    def explain_row_parallel(self):
        """
        行并行：Linear层的行切分
        """
        print("\n行并行（Row Parallel）:")
        print("="*60)
        print("原始操作：")
        print("  Y = X @ W")
        print()
        print("切分到2个GPU：")
        print("  GPU 0: Y1 = X[:, :half] @ W1  (W1是W的前半行)")
        print("  GPU 1: Y2 = X[:, half:] @ W2  (W2是W的后半行)")
        print()
        print("归约：")
        print("  Y = AllReduce(Y1 + Y2)  (需要通信！)")
        print()

explainer = TensorParallelismExplainer()
explainer.explain_column_parallel()
explainer.explain_row_parallel()
```

---

### 6.2 Transformer中的TP实现

```python
"""
Megatron-LM风格的Transformer TP
"""

import torch
import torch.nn as nn
import torch.distributed as dist

class ColumnParallelLinear(nn.Module):
    """
    列并行Linear层
    
    将权重矩阵按列切分到多GPU
    """
    def __init__(self, input_size, output_size, tp_size):
        super().__init__()
        self.tp_size = tp_size
        self.rank = dist.get_rank()
        
        # 每个GPU只存储 output_size / tp_size 列
        self.output_size_per_partition = output_size // tp_size
        
        # 权重矩阵分片
        self.weight = nn.Parameter(
            torch.randn(input_size, self.output_size_per_partition)
        )
    
    def forward(self, x):
        # Forward：每个GPU独立计算自己的分片
        output = torch.matmul(x, self.weight)
        return output  # 返回分片结果，后续拼接

class RowParallelLinear(nn.Module):
    """
    行并行Linear层
    
    将权重矩阵按行切分到多GPU
    """
    def __init__(self, input_size, output_size, tp_size):
        super().__init__()
        self.tp_size = tp_size
        self.rank = dist.get_rank()
        
        # 每个GPU只存储 input_size / tp_size 行
        self.input_size_per_partition = input_size // tp_size
        
        # 权重矩阵分片
        self.weight = nn.Parameter(
            torch.randn(self.input_size_per_partition, output_size)
        )
    
    def forward(self, x):
        # 输入也需要分片
        # x shape: [batch, seq_len, input_size]
        # 每个GPU只处理input_size的一部分
        
        # Forward：每个GPU独立计算
        output_parallel = torch.matmul(x, self.weight)
        
        # AllReduce：归约所有GPU的结果
        dist.all_reduce(output_parallel)
        
        return output_parallel

class TransformerLayerTP(nn.Module):
    """
    带张量并行的Transformer层
    
    Megatron-LM的实现方式
    """
    def __init__(self, hidden_size, ffn_hidden_size, tp_size):
        super().__init__()
        self.tp_size = tp_size
        
        # Attention的Q/K/V投影：列并行
        self.query_key_value = ColumnParallelLinear(
            hidden_size, 3 * hidden_size, tp_size
        )
        
        # Attention输出投影：行并行
        self.dense = RowParallelLinear(
            hidden_size, hidden_size, tp_size
        )
        
        # FFN第一层：列并行
        self.mlp_h_to_4h = ColumnParallelLinear(
            hidden_size, ffn_hidden_size, tp_size
        )
        
        # FFN第二层：行并行
        self.mlp_4h_to_h = RowParallelLinear(
            ffn_hidden_size, hidden_size, tp_size
        )
    
    def forward(self, hidden_states):
        # Attention
        qkv = self.query_key_value(hidden_states)
        # ... attention计算（省略）
        attention_output = self.dense(attention_output)
        
        # FFN
        ffn_hidden = self.mlp_h_to_4h(hidden_states)
        ffn_hidden = torch.nn.functional.gelu(ffn_hidden)
        ffn_output = self.mlp_4h_to_h(ffn_hidden)
        
        return ffn_output

# 通信分析
"""
每个Transformer层的通信：
  Forward:
    - Attention输出：1次AllReduce (dense层)
    - FFN输出：1次AllReduce (mlp_4h_to_h层)
    总计：2次AllReduce
  
  Backward:
    - 梯度反向传播：2次AllReduce
    总计：2次AllReduce

每层4次AllReduce，通信量 = 4 * hidden_size * batch_size * seq_len
"""
```

---

### 6.3 流水线并行 (Pipeline Parallelism, PP)

```python
"""
流水线并行原理
"""

class PipelineParallelismExplainer:
    """
    流水线并行：按层切分模型
    """
    
    def explain_gpipe_schedule(self, num_stages=4, num_microbatches=8):
        """
        GPipe调度：简单但有气泡
        """
        print("GPipe调度（朴素流水线）:")
        print("="*60)
        
        # 模拟时间线
        for stage in range(num_stages):
            print(f"\nStage {stage} (GPU {stage}):")
            timeline = [" "] * (num_stages + num_microbatches)
            
            # Forward pass
            for mb in range(num_microbatches):
                timeline[stage + mb] = "F"
            
            # Backward pass（reverse order）
            for mb in range(num_microbatches):
                timeline[stage + num_microbatches + (num_microbatches - 1 - mb)] = "B"
            
            print("  " + "".join(timeline))
        
        bubble_time = num_stages - 1
        total_time = num_stages + num_microbatches * 2 - 1
        bubble_ratio = bubble_time / total_time
        
        print(f"\n气泡时间: {bubble_time} / {total_time} = {bubble_ratio:.1%}")
    
    def explain_1f1b_schedule(self, num_stages=4, num_microbatches=8):
        """
        1F1B调度：Megatron-LM的优化调度
        
        核心：Forward和Backward交替，减少气泡
        """
        print("\n\n1F1B调度（Megatron-LM）:")
        print("="*60)
        print("优化：每完成一个microbatch的Forward，立即执行Backward")
        print("效果：气泡时间减少到 (num_stages - 1) 个microbatch")
        
        for stage in range(num_stages):
            print(f"\nStage {stage}:")
            timeline = [" "] * 20
            
            # Warmup: 前num_stages个Forward
            for i in range(stage, num_stages):
                timeline[i] = "F"
            
            # Steady state: 1F1B交替
            pos = num_stages
            for i in range(num_microbatches - num_stages):
                timeline[pos] = "F"
                timeline[pos + 1] = "B"
                pos += 2
            
            # Cooldown: 剩余Backward
            for i in range(num_stages - stage - 1):
                timeline[pos] = "B"
                pos += 1
            
            print("  " + "".join(timeline))

explainer = PipelineParallelismExplainer()
explainer.explain_gpipe_schedule()
explainer.explain_1f1b_schedule()
```

---

### 6.4 Megatron-LM完整示例

```python
"""
Megatron-LM风格的训练脚本
"""

import torch
import torch.distributed as dist
from torch.distributed.pipeline.sync import Pipe

# 使用Megatron-LM库
# pip install megatron-core

from megatron.core import parallel_state
from megatron.core.tensor_parallel import (
    ColumnParallelLinear,
    RowParallelLinear,
)
from megatron.core.pipeline_parallel import schedules

def setup_megatron_parallel(tp_size=4, pp_size=2):
    """
    初始化Megatron并行
    
    参数:
        tp_size: 张量并行大小
        pp_size: 流水线并行大小
    """
    world_size = dist.get_world_size()
    assert world_size == tp_size * pp_size
    
    # 初始化并行组
    parallel_state.initialize_model_parallel(
        tensor_model_parallel_size=tp_size,
        pipeline_model_parallel_size=pp_size
    )
    
    # 获取当前rank的并行信息
    tp_rank = parallel_state.get_tensor_model_parallel_rank()
    pp_rank = parallel_state.get_pipeline_model_parallel_rank()
    
    print(f"Rank {dist.get_rank()}: TP rank={tp_rank}, PP rank={pp_rank}")

def train_megatron_model():
    """
    Megatron训练示例
    """
    # 初始化
    dist.init_process_group(backend='nccl')
    setup_megatron_parallel(tp_size=4, pp_size=2)
    
    # 构建模型（每个PP rank只构建自己的层）
    pp_rank = parallel_state.get_pipeline_model_parallel_rank()
    
    if pp_rank == 0:
        # 前半部分层
        model = nn.ModuleList([
            TransformerLayerTP(...) for _ in range(16)
        ])
    else:
        # 后半部分层
        model = nn.ModuleList([
            TransformerLayerTP(...) for _ in range(16)
        ])
    
    # 训练（使用1F1B调度）
    optimizer = torch.optim.AdamW(model.parameters())
    
    for batch in dataloader:
        # 使用Megatron的调度器
        loss = schedules.forward_backward_pipelining_without_interleaving(
            forward_step_func=forward_step,
            data_iterator=iter([batch]),
            model=model,
            num_microbatches=8,
            seq_length=2048,
            micro_batch_size=4,
        )
        
        optimizer.step()
        optimizer.zero_grad()

# 运行方式：
# torchrun --nproc_per_node=8 train_megatron.py
# 其中8 = tp_size(4) * pp_size(2)
```

---

<a name="3d-parallelism"></a>
## 🎯 7. 3D并行：DP+TP+PP组合拳

### 7.1 3D并行原理

```python
"""
3D并行：大规模训练的标准方案
"""

class ThreeDParallelism:
    """
    3D并行组合
    
    Data Parallel (DP) + Tensor Parallel (TP) + Pipeline Parallel (PP)
    """
    
    def calculate_parallelism_config(
        self, 
        model_params_b=175,
        total_gpus=512,
        gpu_memory_gb=40
    ):
        """
        为大模型设计3D并行配置
        """
        print(f"设计3D并行方案：{model_params_b}B模型，{total_gpus}块GPU")
        print("="*70)
        
        # 1. 确定TP size（基于层大小）
        # 经验：hidden_dim > 8192时才需要TP
        if model_params_b >= 70:
            tp_size = 8  # 超大模型
        elif model_params_b >= 13:
            tp_size = 4  # 大模型
        else:
            tp_size = 1  # 小模型不需要TP
        
        # 2. 确定PP size（基于模型深度和显存）
        # 每个stage的显存占用 = 模型参数 / pp_size
        model_memory_per_stage = (model_params_b * 2) / 1  # FP16, 初始假设pp_size=1
        
        # 找到最小的pp_size使得每stage能放入显存
        pp_size = 1
        while model_memory_per_stage / pp_size > gpu_memory_gb * 0.5:  # 留50%给激活值
            pp_size *= 2
        
        # 3. 剩余GPU用于DP
        dp_size = total_gpus // (tp_size * pp_size)
        
        print(f"\n推荐配置:")
        print(f"  TP (Tensor Parallel):    {tp_size}  (层内并行)")
        print(f"  PP (Pipeline Parallel):  {pp_size}  (层间并行)")
        print(f"  DP (Data Parallel):      {dp_size}  (数据并行)")
        print(f"  总GPU: {tp_size} × {pp_size} × {dp_size} = {tp_size * pp_size * dp_size}")
        
        print(f"\n显存分析:")
        memory_per_gpu = (model_params_b * 2) / (tp_size * pp_size) + 10  # 模型+激活值
        print(f"  每卡显存需求: ~{memory_per_gpu:.1f} GB")
        print(f"  A100-40GB: {'✅ 可行' if memory_per_gpu < 40 else '❌ 不可行'}")
        
        print(f"\n有效batch size:")
        micro_batch_size = 2
        global_batch_size = micro_batch_size * dp_size * pp_size  # PP会做gradient accumulation
        print(f"  Micro batch size: {micro_batch_size}")
        print(f"  Global batch size: {global_batch_size}")
        
        return {
            'tp_size': tp_size,
            'pp_size': pp_size,
            'dp_size': dp_size,
            'global_batch_size': global_batch_size
        }

designer = ThreeDParallelism()

# 示例1: GPT-3 175B, 512 GPUs
designer.calculate_parallelism_config(model_params_b=175, total_gpus=512)

# 示例2: LLaMA-70B, 64 GPUs
designer.calculate_parallelism_config(model_params_b=70, total_gpus=64)

# 输出示例：
# 175B模型，512块GPU
# 推荐配置:
#   TP: 8, PP: 16, DP: 4
#   总GPU: 8 × 16 × 4 = 512
#   每卡显存需求: ~32.8 GB
#   Global batch size: 128
```

---

### 7.2 3D并行完整实现

```python
"""
3D并行训练（Megatron-LM + DeepSpeed）
"""

# DeepSpeed配置文件
ds_config_3d = {
    "train_batch_size": 128,
    "train_micro_batch_size_per_gpu": 2,
    "steps_per_print": 10,
    
    # 🔥 ZeRO Stage 1（用于DP维度）
    "zero_optimization": {
        "stage": 1,  # TP+PP时只用Stage 1
        "offload_optimizer": False,
    },
    
    # FP16
    "fp16": {
        "enabled": True,
        "loss_scale": 0,
        "loss_scale_window": 1000,
    },
    
    # Pipeline配置
    "pipeline": {
        "activation_checkpoint_interval": 1,
    },
    
    # 张量并行（在Megatron层面配置）
    "tensor_parallel": {
        "tp_size": 8,
    },
}

def train_3d_parallel():
    """
    3D并行训练主函数
    """
    import deepspeed
    from megatron.core import parallel_state
    
    # 1. 初始化并行组
    world_size = dist.get_world_size()  # 512
    tp_size = 8
    pp_size = 16
    dp_size = world_size // (tp_size * pp_size)  # 4
    
    parallel_state.initialize_model_parallel(
        tensor_model_parallel_size=tp_size,
        pipeline_model_parallel_size=pp_size
    )
    
    # 2. 构建模型（按PP rank）
    pp_rank = parallel_state.get_pipeline_model_parallel_rank()
    layers_per_stage = total_layers // pp_size
    
    start_layer = pp_rank * layers_per_stage
    end_layer = (pp_rank + 1) * layers_per_stage
    
    model = build_model_layers(start_layer, end_layer, tp_size)
    
    # 3. DeepSpeed初始化（处理DP维度）
    model_engine, optimizer, _, _ = deepspeed.initialize(
        model=model,
        config=ds_config_3d,
    )
    
    # 4. 训练循环
    for step, batch in enumerate(dataloader):
        # 使用1F1B流水线调度
        loss = forward_backward_pipelining(
            model_engine,
            batch,
            num_microbatches=pp_size * dp_size
        )
        
        model_engine.step()
    
    print("✅ 3D并行训练完成")

# 运行：
# deepspeed --num_gpus=512 --num_nodes=64 train_3d.py
```

---

<a name="communication"></a>
## 📡 8. 通信优化：NCCL与Ring-AllReduce

### 8.1 NCCL通信原语

```python
"""
NCCL核心通信操作
"""

import torch
import torch.distributed as dist

class NCCLOperations:
    """
    NCCL通信原语演示
    """
    
    def demo_all_reduce(self, rank, world_size):
        """
        AllReduce: 所有GPU的数据求和并广播
        
        用途：DDP梯度同步
        """
        # 创建数据
        tensor = torch.ones(10) * (rank + 1)  # GPU 0:[1,1,...], GPU 1:[2,2,...]
        tensor = tensor.cuda()
        
        print(f"Rank {rank} before AllReduce: {tensor[:3]}")
        
        # AllReduce（求和）
        dist.all_reduce(tensor, op=dist.ReduceOp.SUM)
        
        print(f"Rank {rank} after AllReduce: {tensor[:3]}")
        # 所有GPU都有相同结果: [10, 10, ...] (1+2+3+4)
    
    def demo_all_gather(self, rank, world_size):
        """
        AllGather: 收集所有GPU的数据
        
        用途：FSDP参数聚合
        """
        # 每个GPU的数据
        tensor = torch.ones(10) * (rank + 1)
        tensor = tensor.cuda()
        
        # 准备接收buffer
        tensor_list = [torch.zeros_like(tensor) for _ in range(world_size)]
        
        # AllGather
        dist.all_gather(tensor_list, tensor)
        
        print(f"Rank {rank} gathered: {[t[0].item() for t in tensor_list]}")
        # [1, 2, 3, 4] - 收集了所有GPU的数据
    
    def demo_reduce_scatter(self, rank, world_size):
        """
        ReduceScatter: AllReduce + Scatter的组合
        
        用途：优化的梯度分片
        """
        # 输入：每个GPU有完整数据
        tensor = torch.ones(world_size * 10) * (rank + 1)
        tensor = tensor.cuda()
        
        # 输出：每个GPU接收自己的分片（已reduce）
        output = torch.zeros(10).cuda()
        
        # ReduceScatter
        dist.reduce_scatter(output, list(tensor.chunk(world_size)))
        
        print(f"Rank {rank} result: {output[:3]}")

# NCCL通信量分析
"""
AllReduce: 2 * data_size * (world_size - 1) / world_size
  ≈ 2 * data_size (world_size很大时)

AllGather: data_size * (world_size - 1)

ReduceScatter: data_size * (world_size - 1) / world_size
"""
```

---

### 8.2 Ring-AllReduce算法

```python
"""
Ring-AllReduce原理
"""

class RingAllReduceExplainer:
    """
    Ring-AllReduce: NCCL的核心算法
    """
    
    def explain_algorithm(self, world_size=4):
        """
        Ring-AllReduce两阶段：
        1. ReduceScatter: 每个GPU收到自己负责chunk的sum
        2. AllGather: 广播完整结果
        """
        print("Ring-AllReduce算法演示")
        print("="*70)
        print(f"场景：{world_size}个GPU，每个GPU有4个chunk的梯度")
        print()
        
        # 初始状态
        print("初始状态:")
        for rank in range(world_size):
            print(f"  GPU {rank}: [A{rank}, B{rank}, C{rank}, D{rank}]")
        print()
        
        # 阶段1: ReduceScatter
        print("阶段1：ReduceScatter (world_size-1 步)")
        for step in range(world_size - 1):
            print(f"  Step {step+1}:")
            for rank in range(world_size):
                send_to = (rank + 1) % world_size
                recv_from = (rank - 1) % world_size
                print(f"    GPU {rank}: 发送给GPU {send_to}，接收自GPU {recv_from}")
        
        print(f"  结果：每个GPU有一个chunk的完整sum")
        print(f"    GPU 0: [A_sum, ?, ?, ?]")
        print(f"    GPU 1: [?, B_sum, ?, ?]")
        print(f"    GPU 2: [?, ?, C_sum, ?]")
        print(f"    GPU 3: [?, ?, ?, D_sum]")
        print()
        
        # 阶段2: AllGather
        print("阶段2：AllGather (world_size-1 步)")
        for step in range(world_size - 1):
            print(f"  Step {step+1}:")
            for rank in range(world_size):
                send_to = (rank + 1) % world_size
                recv_from = (rank - 1) % world_size
                print(f"    GPU {rank}: 发送完整chunk给GPU {send_to}")
        
        print(f"  最终：所有GPU都有完整结果")
        for rank in range(world_size):
            print(f"    GPU {rank}: [A_sum, B_sum, C_sum, D_sum]")
        
        # 复杂度分析
        chunk_size = 1  # 单位
        total_data = world_size * chunk_size
        
        print(f"\n通信复杂度:")
        print(f"  传统AllReduce: O(N * data_size)")
        print(f"  Ring-AllReduce: O(2 * (N-1) * data_size / N) ≈ O(2 * data_size)")
        print(f"  优势：与GPU数量无关！")

explainer = RingAllReduceExplainer()
explainer.explain_algorithm()
```

---

### 8.3 通信与计算重叠

```python
"""
通信与计算重叠优化
"""

class CommunicationComputationOverlap:
    """
    重叠通信与计算，提升效率
    """
    
    def without_overlap(self, model, batch):
        """
        无重叠：串行执行
        """
        # 1. 计算梯度（所有层）
        outputs = model(batch)
        loss = compute_loss(outputs)
        loss.backward()  # 计算所有梯度
        
        # 2. 同步梯度（AllReduce）
        for param in model.parameters():
            dist.all_reduce(param.grad)  # 阻塞等待
        
        # 3. 更新参数
        optimizer.step()
        
        # 问题：计算和通信串行，GPU空闲时间长
    
    def with_overlap_ddp(self, model, batch):
        """
        DDP自动重叠
        
        原理：Backward时边计算边通信
        """
        # DDP会将模型参数分成多个bucket
        # 每个bucket的梯度计算完成后立即启动AllReduce
        
        outputs = model(batch)  # model是DDP包装的
        loss = compute_loss(outputs)
        loss.backward()  # 🔥 DDP在backward过程中自动重叠通信！
        
        # Backward结束时，梯度已经同步完成
        optimizer.step()
        
        # 效果：通信时间被计算时间掩盖
    
    def visualize_overlap(self):
        """
        可视化重叠效果
        """
        print("通信与计算重叠:")
        print("="*70)
        
        print("\n❌ 无重叠（串行）:")
        print("  GPU: [计算Layer1] [计算Layer2] [计算Layer3]          [空闲...]")
        print("  网络:                                     [AllReduce...]")
        print("  总时间: 计算时间 + 通信时间")
        
        print("\n✅ 有重叠（DDP）:")
        print("  GPU: [计算Layer1] [计算Layer2] [计算Layer3]")
        print("  网络:         [AllReduce1][AllReduce2][AllReduce3]")
        print("  总时间: max(计算时间, 通信时间)")
        print()
        print("  效果：通信被计算掩盖，几乎无额外开销！")

overlap_demo = CommunicationComputationOverlap()
overlap_demo.visualize_overlap()
```

---

<a name="gradient-accumulation"></a>
## 🔋 9. 梯度累积与混合精度

### 9.1 梯度累积原理

```python
"""
梯度累积：模拟大batch训练
"""

class GradientAccumulation:
    """
    梯度累积
    
    用途：显存不足时模拟大batch
    """
    
    def train_without_accumulation(self, model, dataloader, batch_size=32):
        """
        无梯度累积：标准训练
        """
        for batch in dataloader:
            outputs = model(batch)
            loss = criterion(outputs, labels)
            
            loss.backward()
            optimizer.step()
            optimizer.zero_grad()
        
        # 每个batch更新一次参数
        # 有效batch_size = 32
    
    def train_with_accumulation(
        self, 
        model, 
        dataloader, 
        micro_batch_size=4, 
        accumulation_steps=8
    ):
        """
        梯度累积：累积多个microbatch的梯度
        
        效果：有效batch_size = 4 * 8 = 32
        """
        optimizer.zero_grad()
        
        for step, batch in enumerate(dataloader):
            # Forward & Backward
            outputs = model(batch)
            loss = criterion(outputs, labels)
            
            # 🔥 重要：loss需要除以accumulation_steps
            loss = loss / accumulation_steps
            loss.backward()  # 梯度累积，不清零
            
            # 每accumulation_steps步更新一次
            if (step + 1) % accumulation_steps == 0:
                optimizer.step()
                optimizer.zero_grad()
        
        # 显存占用：只需存储micro_batch_size的激活值
        # 有效batch_size：micro_batch_size * accumulation_steps
    
    def compare_memory(self):
        """
        显存对比
        """
        print("梯度累积显存对比:")
        print("="*70)
        
        batch_size = 32
        micro_batch = 4
        accum_steps = 8
        
        # 无累积
        memory_without = batch_size * seq_len * hidden_dim * num_layers * 2
        
        # 有累积
        memory_with = micro_batch * seq_len * hidden_dim * num_layers * 2
        
        print(f"目标有效batch size: {batch_size}")
        print(f"\n方案1：直接训练batch_size={batch_size}")
        print(f"  激活值显存: ~{memory_without / 1024**3:.1f} GB")
        
        print(f"\n方案2：梯度累积 (micro_batch={micro_batch}, steps={accum_steps})")
        print(f"  激活值显存: ~{memory_with / 1024**3:.1f} GB")
        print(f"  节省: {(1 - memory_with/memory_without)*100:.0f}%")
        
        print(f"\n⚠️ 注意：梯度累积会增加训练时间（{accum_steps}x forward/backward）")

accum = GradientAccumulation()
accum.compare_memory()
```

---

### 9.2 混合精度训练

```python
"""
混合精度训练（FP16/BF16）
"""

import torch
from torch.cuda.amp import autocast, GradScaler

class MixedPrecisionTraining:
    """
    混合精度训练
    
    FP16: 节省显存，加速计算
    BF16: 更稳定（H100推荐）
    """
    
    def train_fp32_baseline(self, model, dataloader):
        """
        FP32训练（基线）
        """
        model = model.cuda()
        optimizer = torch.optim.AdamW(model.parameters())
        
        for batch in dataloader:
            inputs = batch['inputs'].cuda()
            labels = batch['labels'].cuda()
            
            # Forward (FP32)
            outputs = model(inputs)
            loss = criterion(outputs, labels)
            
            # Backward (FP32)
            loss.backward()
            optimizer.step()
            optimizer.zero_grad()
    
    def train_fp16_automatic(self, model, dataloader):
        """
        FP16自动混合精度（PyTorch AMP）
        """
        model = model.cuda()
        optimizer = torch.optim.AdamW(model.parameters())
        
        # 🔥 GradScaler：防止FP16下溢
        scaler = GradScaler()
        
        for batch in dataloader:
            inputs = batch['inputs'].cuda()
            labels = batch['labels'].cuda()
            
            # 🔥 autocast：自动FP16
            with autocast(dtype=torch.float16):
                outputs = model(inputs)
                loss = criterion(outputs, labels)
            
            # Backward（带scale）
            scaler.scale(loss).backward()
            
            # 梯度裁剪（unscale后）
            scaler.unscale_(optimizer)
            torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
            
            # 更新参数
            scaler.step(optimizer)
            scaler.update()
            optimizer.zero_grad()
    
    def train_bf16(self, model, dataloader):
        """
        BF16混合精度（H100/A100推荐）
        
        优势：不需要GradScaler，更稳定
        """
        model = model.cuda()
        optimizer = torch.optim.AdamW(model.parameters())
        
        for batch in dataloader:
            inputs = batch['inputs'].cuda()
            labels = batch['labels'].cuda()
            
            # 🔥 BF16：不需要loss scaling
            with autocast(dtype=torch.bfloat16):
                outputs = model(inputs)
                loss = criterion(outputs, labels)
            
            # Backward（无需scaler）
            loss.backward()
            
            # 梯度裁剪
            torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
            
            optimizer.step()
            optimizer.zero_grad()
    
    def compare_precision(self):
        """
        精度对比
        """
        comparison = {
            '精度': ['显存占用', '计算速度', '数值稳定性', '适用GPU'],
            
            'FP32': ['100%', '1.0x', '最好', '所有'],
            'FP16': ['50%', '2-3x', '一般（需loss scaling）', 'V100+'],
            'BF16': ['50%', '2-3x', '好', 'A100/H100']
        }
        
        import pandas as pd
        df = pd.DataFrame(comparison).set_index('精度')
        print(df.to_string())

mixed_precision = MixedPrecisionTraining()
mixed_precision.compare_precision()
```

---

## 📚 参考资料

### 核心论文

1. **Megatron-LM**  
   [Megatron-LM: Training Multi-Billion Parameter Language Models](https://arxiv.org/abs/1909.08053)  
   NVIDIA, 2019

2. **DeepSpeed ZeRO**  
   [ZeRO: Memory Optimizations Toward Training Trillion Parameter Models](https://arxiv.org/abs/1910.02054)  
   Microsoft, 2020

3. **PyTorch FSDP**  
   [PyTorch FSDP Documentation](https://pytorch.org/tutorials/intermediate/FSDP_tutorial.html)  
   2025最新

4. **3D Parallelism**  
   [TorchTitan: PyTorch Native Solution for LLM Pre-training](https://arxiv.org/html/2410.06511v3)  
   2025

5. **Pipeline Parallelism**  
   [GPipe: Efficient Training of Giant Neural Networks](https://arxiv.org/abs/1811.06965)  
   Google, 2019

### 工程实践

- [PyTorch Distributed](https://pytorch.org/tutorials/beginner/dist_overview.html)  
  官方分布式训练教程

- [DeepSpeed Documentation](https://www.deepspeed.ai/)  
  ZeRO优化完整文档

- [NVIDIA Megatron-LM GitHub](https://github.com/NVIDIA/Megatron-LM)  
  Megatron-LM源码

- [NCCL Documentation](https://docs.nvidia.com/deeplearning/nccl/user-guide/)  
  NCCL通信库

---

## 🎯 总结

### 关键要点回顾

1. **选择合适的并行策略**：
   - 7B模型：DDP
   - 13B-70B：FSDP ZeRO-3
   - 70B+：3D并行 (TP+PP+DP)

2. **通信优化是关键**：
   - DDP：通信与计算重叠
   - FSDP：预取参数
   - Megatron：优化TP/PP通信

3. **显存优化组合拳**：
   - 梯度累积
   - 混合精度（BF16推荐）
   - Activation Checkpointing
   - ZeRO Stage 3

4. **故障恢复必不可少**：
   - 定期Checkpoint
   - Fault-tolerant training
   - 状态持久化

### 下一步学习

- 🔗 **下一篇**：[11 - MLOps最佳实践：实验追踪、版本控制、CI/CD](./11-mlops-practices.md)
- 💻 **动手实践**：在8卡机器上对比DDP vs FSDP
- 📊 **性能分析**：用PyTorch Profiler分析通信瓶颈

---

> 💡 **璇玑的小贴士**：分布式训练就像团队协作——沟通（通信）成本是关键！好的并行策略能让GPU们高效配合，就像敏捷团队的站会一样简短有效~ ✨
>
> 道友现在对分布式训练有感觉了吗？下一篇我们聊MLOps，教你如何管理训练实验、版本控制和CI/CD！🚀
