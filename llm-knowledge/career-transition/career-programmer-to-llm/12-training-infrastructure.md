# 12 - 训练基础设施设计：GPU集群、存储、监控体系

> 🎯 **核心观点**：训练基础设施是LLM工程的基石。本文深入讲解GPU集群架构设计、Kubernetes + GPU Operator配置、SLURM作业调度、高性能存储（Lustre/GPFS）、RDMA/InfiniBand网络优化、成本优化策略（Spot实例/混合云）、Prometheus/Grafana监控体系，以及故障诊断与容量规划的实战经验。

---

## 📋 目录

1. [训练基础设施全景图](#infrastructure-overview)
2. [GPU集群架构设计](#gpu-cluster-architecture)
3. [Kubernetes + GPU Operator](#kubernetes-gpu)
4. [SLURM作业调度系统](#slurm-scheduling)
5. [高性能存储：Lustre vs GPFS](#high-performance-storage)
6. [网络优化：RDMA & InfiniBand](#network-optimization)
7. [资源调度策略](#resource-scheduling)
8. [成本优化：Spot实例与混合云](#cost-optimization)
9. [监控体系：DCGM + Prometheus + Grafana](#monitoring-system)
10. [故障诊断与恢复](#fault-diagnosis)
11. [容量规划与扩展](#capacity-planning)
12. [完整基础设施部署案例](#deployment-case)

---

<a name="infrastructure-overview"></a>
## 🏗️ 1. 训练基础设施全景图

### 1.1 完整架构图

```python
"""
LLM训练基础设施全景
"""

class TrainingInfrastructureArchitecture:
    """
    训练基础设施分层架构
    """
    
    def print_architecture(self):
        """
        打印基础设施架构图
        """
        print("""
╔════════════════════════════════════════════════════════════════════════╗
║                    LLM训练基础设施架构（2025）                           ║
╚════════════════════════════════════════════════════════════════════════╝

┌─────────────────────────────────────────────────────────────────────┐
│  Layer 5: 开发者界面                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐             │
│  │ Jupyter  │  │ VS Code  │  │   CLI    │  │  Web UI  │             │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘             │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│  Layer 4: 作业调度层                                                 │
│  ┌────────────────────┐        ┌────────────────────┐               │
│  │      SLURM         │   OR   │    Kubernetes      │               │
│  │  (HPC传统方案)     │        │  (云原生方案)      │               │
│  └────────────────────┘        └────────────────────┘               │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│  Layer 3: 编排与资源管理                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐             │
│  │ GPU      │  │ Network  │  │ Storage  │  │ Monitor  │             │
│  │ Operator │  │ Operator │  │  CSI     │  │  Agent   │             │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘             │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│  Layer 2: 网络与存储                                                 │
│  ┌────────────────────┐        ┌────────────────────┐               │
│  │  InfiniBand/RDMA   │        │   Lustre/GPFS      │               │
│  │  (400Gb/s)         │        │   (PB级存储)       │               │
│  └────────────────────┘        └────────────────────┘               │
└─────────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────────┐
│  Layer 1: 计算资源层                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐             │
│  │ Node 1   │  │ Node 2   │  │ Node 3   │  │ Node N   │             │
│  │ 8xH100   │  │ 8xH100   │  │ 8xH100   │  │ 8xH100   │             │
│  │ 640GB    │  │ 640GB    │  │ 640GB    │  │ 640GB    │             │
│  │ NVLink   │  │ NVLink   │  │ NVLink   │  │ NVLink   │             │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘             │
└─────────────────────────────────────────────────────────────────────┘
        """)

arch = TrainingInfrastructureArchitecture()
arch.print_architecture()
```

---

### 1.2 技术栈选型

```python
"""
2025年训练基础设施技术栈对比
"""

tech_stack_comparison = {
    '组件': ['作业调度', '容器编排', 'GPU管理', '网络', '存储', '监控'],
    
    'HPC方案 (传统)': [
        'SLURM',
        '无（直接SSH）',
        'CUDA手动安装',
        'InfiniBand',
        'Lustre',
        'Ganglia/Nagios'
    ],
    
    'K8s方案 (云原生)': [
        'Kueue/Volcano',
        'Kubernetes',
        'GPU Operator',
        'RDMA/Multus',
        'CSI (Lustre/EBS)',
        'Prometheus/Grafana'
    ],
    
    '混合方案 (推荐)': [
        'SLURM + K8s',
        'K8s (服务)',
        'GPU Operator',
        'InfiniBand + RDMA',
        'Lustre',
        'Prometheus + DCGM'
    ]
}

import pandas as pd
df = pd.DataFrame(tech_stack_comparison).set_index('组件')
print("技术栈对比:")
print(df.to_string())

print("\n🔥 2025推荐：K8s + SLURM混合架构")
print("  - K8s管理服务（推理、Web）")
print("  - SLURM管理训练作业（更成熟）")
```

---

<a name="gpu-cluster-architecture"></a>
## 🖥️ 2. GPU集群架构设计

### 2.1 GPU硬件选型（2025）

```python
"""
2025年GPU选型指南
"""

class GPUSelectionGuide:
    """
    GPU选型分析
    """
    
    def __init__(self):
        # 2025年主流训练GPU
        self.gpus = {
            'NVIDIA H100 SXM5': {
                'memory': '80GB HBM3',
                'bandwidth': '3.35 TB/s',
                'compute_fp16': '989 TFLOPS',
                'compute_fp8': '1979 TFLOPS',
                'nvlink': '900 GB/s',
                'price_per_hour': 5.00,
                'use_case': '🔥 大规模训练首选'
            },
            
            'NVIDIA H200': {
                'memory': '141GB HBM3e',
                'bandwidth': '4.8 TB/s',
                'compute_fp16': '989 TFLOPS',
                'compute_fp8': '1979 TFLOPS',
                'nvlink': '900 GB/s',
                'price_per_hour': 7.00,
                'use_case': '🔥 超大模型/长上下文'
            },
            
            'NVIDIA A100 80GB': {
                'memory': '80GB HBM2e',
                'bandwidth': '2.0 TB/s',
                'compute_fp16': '312 TFLOPS',
                'compute_tf32': '156 TFLOPS',
                'nvlink': '600 GB/s',
                'price_per_hour': 3.50,
                'use_case': '平衡性价比'
            },
            
            'NVIDIA L40S': {
                'memory': '48GB GDDR6',
                'bandwidth': '864 GB/s',
                'compute_fp16': '362 TFLOPS',
                'compute_fp8': '733 TFLOPS',
                'nvlink': '无',
                'price_per_hour': 2.00,
                'use_case': '推理+轻量训练'
            },
            
            'Consumer RTX 4090': {
                'memory': '24GB GDDR6X',
                'bandwidth': '1008 GB/s',
                'compute_fp16': '82.6 TFLOPS',
                'nvlink': '无',
                'price': '$1,599 (一次性)',
                'use_case': '个人研究/小模型'
            }
        }
    
    def recommend_gpu(self, model_size, budget, use_case):
        """
        根据需求推荐GPU
        """
        print(f"\n需求分析:")
        print(f"  模型规模: {model_size}")
        print(f"  预算: {budget}")
        print(f"  用途: {use_case}")
        print(f"\n推荐方案:")
        
        if model_size == '175B+':
            print(f"  🔥 NVIDIA H200 (141GB显存)")
            print(f"     理由: 超大模型需要超大显存")
        
        elif model_size == '70B':
            if budget == 'high':
                print(f"  🔥 NVIDIA H100 (80GB)")
                print(f"     理由: 最快训练速度")
            else:
                print(f"  🔥 NVIDIA A100 80GB")
                print(f"     理由: 性价比高")
        
        elif model_size == '7B-13B':
            if budget == 'low':
                print(f"  🔥 NVIDIA L40S 或 RTX 4090")
                print(f"     理由: 足够训练，成本最低")
            else:
                print(f"  🔥 NVIDIA A100 40GB")
                print(f"     理由: 训练效率高")
        
        else:  # < 7B
            print(f"  🔥 RTX 4090 或 A6000")
            print(f"     理由: 消费级卡够用")
    
    def calculate_cluster_config(self, target_model='70B', target_days=7):
        """
        计算集群配置
        """
        # 训练70B模型的估算
        # 参考DeepSeek-V3: 2.8M GPU hours for 671B model
        
        if target_model == '70B':
            # 估算：70B约需200K GPU-hours
            gpu_hours_needed = 200_000
        elif target_model == '7B':
            gpu_hours_needed = 1_000
        else:
            gpu_hours_needed = 500_000  # 175B+
        
        # 计算所需GPU数
        hours_in_period = target_days * 24
        num_gpus_needed = int(np.ceil(gpu_hours_needed / hours_in_period))
        
        # 8卡服务器
        num_nodes = int(np.ceil(num_gpus_needed / 8))
        
        print(f"\n集群配置计算 ({target_model}模型，{target_days}天训练):")
        print("="*70)
        print(f"  总GPU-hours: {gpu_hours_needed:,}")
        print(f"  训练时长: {target_days}天 = {hours_in_period}小时")
        print(f"  所需GPU数: {num_gpus_needed}")
        print(f"  所需节点数: {num_nodes} (8卡/节点)")
        print(f"  推荐GPU: H100 80GB")
        
        # 成本估算
        cost_per_gpu_hour = 5.00  # H100价格
        total_cost = gpu_hours_needed * cost_per_gpu_hour
        
        print(f"\n成本估算:")
        print(f"  GPU成本: ${total_cost:,}")
        print(f"  存储成本: ~${num_nodes * 100:,} (Lustre)")
        print(f"  网络成本: ~${num_nodes * 50:,} (InfiniBand)")
        print(f"  总计: ~${total_cost + num_nodes * 150:,}")

guide = GPUSelectionGuide()
guide.recommend_gpu(model_size='70B', budget='medium', use_case='训练')
guide.calculate_cluster_config(target_model='70B', target_days=7)
```

---

### 2.2 GPU服务器配置

```python
"""
标准GPU训练节点配置
"""

class GPUNodeConfiguration:
    """
    GPU节点配置规格
    """
    
    def dgx_h100_spec(self):
        """
        NVIDIA DGX H100配置（企业级）
        """
        return {
            'GPU': '8x H100 SXM5 80GB',
            'GPU互联': 'NVLink 4 (900 GB/s双向)',
            'CPU': '2x Intel Xeon Platinum 8480C (56核)',
            '内存': '2TB DDR5',
            '网络': '8x 200Gb/s InfiniBand',
            '存储': '30TB NVMe (8x 3.84TB)',
            '电源': '10.2 kW',
            '价格': '~$250,000',
            'TCO_3年': '~$450,000 (含电费、维护)'
        }
    
    def custom_8gpu_server(self):
        """
        自建8卡服务器配置（性价比）
        """
        return {
            'GPU': '8x RTX 4090 24GB',
            'GPU互联': 'PCIe 4.0 (64 GB/s，通过CPU)',
            'CPU': 'AMD Threadripper PRO 5995WX (64核)',
            '内存': '512GB DDR4',
            '网络': '2x 100Gb/s Ethernet (RoCE)',
            '存储': '8TB NVMe (2x 4TB)',
            '电源': '~3.5 kW',
            '价格': '~$40,000',
            'TCO_3年': '~$55,000',
            '性价比': '⭐⭐⭐⭐⭐'
        }
    
    def compare_configs(self):
        """
        对比企业级 vs 自建
        """
        dgx = self.dgx_h100_spec()
        custom = self.custom_8gpu_server()
        
        print("GPU服务器配置对比:")
        print("="*70)
        print(f"{'项目':<20} {'DGX H100 (企业级)':<25} {'自建8x4090':<25}")
        print("-"*70)
        
        keys = ['GPU', 'GPU互联', 'CPU', '内存', '网络', '价格', 'TCO_3年']
        for key in keys:
            print(f"{key:<20} {dgx.get(key, 'N/A'):<25} {custom.get(key, 'N/A'):<25}")
        
        print("\n选择建议:")
        print("  DGX H100: 大公司、关键任务、需要支持")
        print("  自建4090: 创业公司、研究、预算有限")

config = GPUNodeConfiguration()
config.compare_configs()
```

---

<a name="kubernetes-gpu"></a>
## ☸️ 3. Kubernetes + GPU Operator

### 3.1 K8s GPU集群搭建

```bash
# ============================================
# 1. 前置准备
# ============================================

# 1.1 安装containerd (所有节点)
sudo apt-get update
sudo apt-get install -y containerd

# 配置containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd

# 1.2 禁用swap (K8s要求)
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# 1.3 加载必要的内核模块
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# ============================================
# 2. 安装Kubernetes (所有节点)
# ============================================

# 2.1 添加K8s apt源
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

# 2.2 安装kubelet, kubeadm, kubectl
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# ============================================
# 3. 初始化Master节点
# ============================================

# 3.1 初始化控制平面 (仅Master节点)
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# 3.2 配置kubectl (Master节点)
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 3.3 安装网络插件 (Flannel)
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# ============================================
# 4. 加入Worker节点
# ============================================

# 4.1 在Worker节点执行 (Master节点会输出这个命令)
sudo kubeadm join 192.168.1.100:6443 --token <token> \
    --discovery-token-ca-cert-hash sha256:<hash>

# 4.2 验证节点加入 (Master节点)
kubectl get nodes

# 输出：
# NAME     STATUS   ROLES           AGE   VERSION
# master   Ready    control-plane   10m   v1.30.0
# worker1  Ready    <none>          5m    v1.30.0
# worker2  Ready    <none>          5m    v1.30.0

# ============================================
# 5. 安装NVIDIA GPU Operator
# ============================================

# 5.1 添加Helm repo
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update

# 5.2 安装GPU Operator
helm install --wait --generate-name \
    -n gpu-operator --create-namespace \
    nvidia/gpu-operator \
    --version=v25.10.1 \
    --set driver.enabled=true

# 5.3 验证GPU Operator安装
kubectl get pods -n gpu-operator

# 应该看到：
# - nvidia-driver-daemonset (每个GPU节点一个)
# - nvidia-device-plugin-daemonset
# - nvidia-dcgm-exporter (监控)
# - nvidia-operator-validator

# 5.4 验证GPU可用
kubectl get nodes -o json | jq '.items[].status.capacity'

# 输出应该包含：
# {
#   "nvidia.com/gpu": "8",  # ✅ 每个节点8块GPU
#   ...
# }
```

---

### 3.2 GPU Pod配置

```yaml
# gpu-training-job.yaml
# GPU训练作业配置

apiVersion: batch/v1
kind: Job
metadata:
  name: llama2-training
spec:
  template:
    spec:
      # 🔥 GPU资源请求
      containers:
      - name: trainer
        image: nvcr.io/nvidia/pytorch:24.01-py3
        
        # 资源请求
        resources:
          requests:
            nvidia.com/gpu: 8  # 请求8块GPU
            memory: "400Gi"
            cpu: "64"
          limits:
            nvidia.com/gpu: 8
            memory: "400Gi"
        
        # 环境变量
        env:
          - name: NCCL_DEBUG
            value: "INFO"
          - name: NCCL_IB_DISABLE
            value: "0"  # 启用InfiniBand
          - name: NCCL_SOCKET_IFNAME
            value: "ib0"  # InfiniBand网卡
        
        # 挂载存储
        volumeMounts:
          - name: training-data
            mountPath: /data
          - name: model-output
            mountPath: /output
          - name: shm
            mountPath: /dev/shm  # 共享内存（DataLoader需要）
        
        # 训练命令
        command:
          - "torchrun"
          - "--nproc_per_node=8"
          - "--nnodes=1"
          - "train.py"
          - "--config"
          - "/config/train_config.yaml"
      
      # 卷配置
      volumes:
        - name: training-data
          persistentVolumeClaim:
            claimName: lustre-pvc  # Lustre存储
        - name: model-output
          persistentVolumeClaim:
            claimName: model-output-pvc
        - name: shm
          emptyDir:
            medium: Memory
            sizeLimit: "64Gi"  # 共享内存大小
      
      # 节点选择
      nodeSelector:
        gpu-type: h100  # 只调度到H100节点
      
      # 容忍（Toleration）
      tolerations:
        - key: "nvidia.com/gpu"
          operator: "Exists"
          effect: "NoSchedule"
      
      restartPolicy: OnFailure
```

---

### 3.3 多节点训练Job配置

```yaml
# multi-node-training.yaml
# 多节点分布式训练配置

apiVersion: kubeflow.org/v1
kind: PyTorchJob
metadata:
  name: llama2-70b-distributed
spec:
  pytorchReplicaSpecs:
    # Master节点
    Master:
      replicas: 1
      restartPolicy: OnFailure
      template:
        spec:
          containers:
            - name: pytorch
              image: nvcr.io/nvidia/pytorch:24.01-py3
              resources:
                requests:
                  nvidia.com/gpu: 8
                  memory: "600Gi"
                limits:
                  nvidia.com/gpu: 8
              env:
                - name: NCCL_DEBUG
                  value: "INFO"
                - name: NCCL_IB_HCA
                  value: "mlx5"  # InfiniBand适配器
              command:
                - python
                - -m
                - torch.distributed.run
                - --nproc_per_node=8
                - --nnodes=8
                - --node_rank=0
                - --master_addr=$(MASTER_ADDR)
                - --master_port=29500
                - train.py
    
    # Worker节点
    Worker:
      replicas: 7  # 总共8个节点 (1 master + 7 workers)
      restartPolicy: OnFailure
      template:
        spec:
          containers:
            - name: pytorch
              image: nvcr.io/nvidia/pytorch:24.01-py3
              resources:
                requests:
                  nvidia.com/gpu: 8
                  memory: "600Gi"
                limits:
                  nvidia.com/gpu: 8
              env:
                - name: NCCL_DEBUG
                  value: "INFO"
                - name: NCCL_IB_HCA
                  value: "mlx5"
              command:
                - python
                - -m
                - torch.distributed.run
                - --nproc_per_node=8
                - --nnodes=8
                - --node_rank=$(RANK)
                - --master_addr=$(MASTER_ADDR)
                - --master_port=29500
                - train.py
```

---

<a name="slurm-scheduling"></a>
## 📅 4. SLURM作业调度系统

### 4.1 SLURM架构

```python
"""
SLURM架构组件
"""

class SLURMArchitecture:
    """
    SLURM (Simple Linux Utility for Resource Management)
    """
    
    def explain_components(self):
        """
        SLURM核心组件
        """
        print("""
SLURM架构:

┌─────────────────────────────────────────────────┐
│  用户提交作业                                    │
│  ┌──────┐  ┌──────┐  ┌──────┐                  │
│  │ sbatch│  │srun  │  │salloc│                  │
│  └───┬──┘  └───┬──┘  └───┬──┘                  │
│      └─────────┼─────────┘                       │
└────────────────┼─────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────┐
│  控制节点 (slurmctld)                           │
│  - 作业调度                                     │
│  - 资源分配                                     │
│  - 优先级管理                                   │
└────────────────┬────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────┐
│  计算节点 (slurmd)                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │ Node 1   │  │ Node 2   │  │ Node N   │      │
│  │ 8xH100   │  │ 8xH100   │  │ 8xH100   │      │
│  └──────────┘  └──────────┘  └──────────┘      │
└─────────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────────┐
│  数据库 (slurmdbd)                              │
│  - 作业历史                                     │
│  - 资源使用统计                                 │
│  - 用户配额                                     │
└─────────────────────────────────────────────────┘
        """)

arch = SLURMArchitecture()
arch.explain_components()
```

---

### 4.2 SLURM配置

```bash
# ============================================
# slurm.conf - SLURM主配置文件
# ============================================

# 集群名称
ClusterName=ml-training-cluster

# 控制节点
SlurmctldHost=master-node

# 调度器配置
SchedulerType=sched/backfill
SelectType=select/cons_tres  # 支持GRES (GPU)
SelectTypeParameters=CR_Core_Memory

# 资源配置
GresTypes=gpu

# 优先级
PriorityType=priority/multifactor
PriorityWeightAge=1000
PriorityWeightFairshare=10000
PriorityWeightJobSize=1000
PriorityWeightQOS=10000

# 作业时间限制
MaxJobCount=10000
MaxTaskCount=100000

# 账户配置
AccountingStorageType=accounting_storage/slurmdbd
AccountingStorageHost=db-node
AccountingStoragePort=6819

# ============================================
# 计算节点配置
# ============================================

# GPU节点 (H100)
NodeName=gpu-node[1-8] \
    Sockets=2 \
    CoresPerSocket=28 \
    ThreadsPerCore=2 \
    RealMemory=512000 \
    Gres=gpu:h100:8 \
    State=UNKNOWN

# 分区配置 (Partition)
PartitionName=gpu \
    Nodes=gpu-node[1-8] \
    Default=YES \
    MaxTime=7-00:00:00 \
    State=UP \
    OverSubscribe=NO \
    PriorityJobFactor=10

PartitionName=gpu-short \
    Nodes=gpu-node[1-2] \
    Default=NO \
    MaxTime=04:00:00 \
    State=UP \
    PriorityJobFactor=5

# ============================================
# gres.conf - GPU配置
# ============================================

# 自动检测GPU (SLURM 24.11+)
AutoDetect=nvml

# 或手动配置
# NodeName=gpu-node1 Name=gpu Type=h100 File=/dev/nvidia[0-7]
# NodeName=gpu-node2 Name=gpu Type=h100 File=/dev/nvidia[0-7]
```

---

### 4.3 SLURM作业提交

```bash
# ============================================
# 单节点8卡训练
# ============================================

#!/bin/bash
#SBATCH --job-name=llama2-7b-train
#SBATCH --partition=gpu
#SBATCH --nodes=1              # 单节点
#SBATCH --gres=gpu:8           # 🔥 请求8块GPU
#SBATCH --cpus-per-task=64     # CPU核心
#SBATCH --mem=400G             # 内存
#SBATCH --time=48:00:00        # 最长运行时间
#SBATCH --output=logs/train_%j.out
#SBATCH --error=logs/train_%j.err

# 加载模块
module load cuda/11.8
module load nccl/2.17

# 激活conda环境
conda activate llm-training

# 设置环境变量
export NCCL_DEBUG=INFO
export NCCL_IB_DISABLE=0

# 运行训练
torchrun --nproc_per_node=8 \
    train.py \
    --config configs/train_config.yaml

# ============================================
# 多节点64卡训练
# ============================================

#!/bin/bash
#SBATCH --job-name=llama2-70b-train
#SBATCH --partition=gpu
#SBATCH --nodes=8              # 8个节点
#SBATCH --gres=gpu:8           # 每节点8块GPU
#SBATCH --ntasks-per-node=1    # 每节点1个任务
#SBATCH --cpus-per-task=64
#SBATCH --mem=400G
#SBATCH --time=168:00:00       # 7天
#SBATCH --output=logs/train_%j.out
#SBATCH --error=logs/train_%j.err

# 获取节点列表
export MASTER_ADDR=$(scontrol show hostnames $SLURM_JOB_NODELIST | head -n 1)
export MASTER_PORT=29500
export WORLD_SIZE=$((SLURM_NNODES * 8))  # 总GPU数

# 在每个节点启动训练
srun torchrun \
    --nnodes=$SLURM_NNODES \
    --nproc_per_node=8 \
    --node_rank=$SLURM_NODEID \
    --master_addr=$MASTER_ADDR \
    --master_port=$MASTER_PORT \
    train.py \
    --config configs/train_config.yaml

# 提交作业
sbatch train_job.sh

# 查看作业状态
squeue -u $USER

# 输出：
#  JOBID PARTITION     NAME     USER ST       TIME  NODES
#  12345       gpu llama2-7  john  R      10:30      1

# 查看详细信息
scontrol show job 12345

# 取消作业
scancel 12345

# 查看GPU使用情况
squeue -o "%.18i %.9P %.8j %.8u %.2t %.10M %.6D %.20R %b"
```

---

<a name="high-performance-storage"></a>
## 💾 5. 高性能存储：Lustre vs GPFS

### 5.1 Lustre架构

根据 [AWS FSx for Lustre](https://aws.amazon.com/fsx/lustre/) 和 [Google Managed Lustre](https://cloud.google.com/managed-lustre/docs/overview)：

```python
"""
Lustre文件系统架构
"""

class LustreArchitecture:
    """
    Lustre分布式文件系统
    """
    
    def explain_architecture(self):
        """
        Lustre架构说明
        """
        print("""
Lustre文件系统架构:

┌─────────────────────────────────────────┐
│  客户端 (Compute Nodes)                 │
│  ┌──────┐  ┌──────┐  ┌──────┐           │
│  │ GPU  │  │ GPU  │  │ GPU  │           │
│  │ Node │  │ Node │  │ Node │           │
│  └──┬───┘  └──┬───┘  └──┬───┘           │
│     └─────────┼─────────┘               │
└───────────────┼─────────────────────────┘
                ↓ Lustre Client
┌───────────────────────────────────────────┐
│  元数据服务器 (MGS/MDS)                   │
│  - 文件名                                 │
│  - 目录结构                               │
│  - 权限信息                               │
└───────────────┬───────────────────────────┘
                ↓
┌───────────────────────────────────────────┐
│  对象存储服务器 (OSS/OST)                 │
│  ┌────────┐  ┌────────┐  ┌────────┐      │
│  │ OST 1  │  │ OST 2  │  │ OST N  │      │
│  │ 100TB  │  │ 100TB  │  │ 100TB  │      │
│  └────────┘  └────────┘  └────────┘      │
│  (数据分条带存储，并行读写)               │
└───────────────────────────────────────────┘
        """)
        
        print("\nLustre特点:")
        print("  ✅ PB级容量")
        print("  ✅ TB/s级吞吐")
        print("  ✅ 数百万IOPS")
        print("  ✅ 亚毫秒延迟")
        print("  ✅ POSIX兼容（像本地文件系统）")
    
    def performance_specs(self):
        """
        Lustre性能规格（2025）
        """
        specs = {
            'AWS FSx for Lustre': {
                '吞吐量': '最高 1200 GB/s per client',
                '容量': 'PB级',
                'IOPS': '数百万',
                '延迟': '亚毫秒',
                '价格': '$0.005/GB-月 起',
                '集成': 'SageMaker HyperPod'
            },
            
            'Google Managed Lustre': {
                '吞吐量': '最高 1 TB/s',
                '容量': 'PB级',
                'IOPS': '极高',
                '延迟': '低',
                '集成': 'GKE, Compute Engine'
            },
            
            '自建Lustre': {
                '吞吐量': '取决于OST数量',
                '容量': '可扩展到EB级',
                'IOPS': '可扩展',
                '延迟': '取决于硬件',
                '价格': '硬件成本',
                '灵活性': '⭐⭐⭐⭐⭐'
            }
        }
        
        import pandas as pd
        df = pd.DataFrame(specs).T
        print("\nLustre方案对比:")
        print(df.to_string())

arch = LustreArchitecture()
arch.explain_architecture()
arch.performance_specs()
```

---

### 5.2 Lustre集群搭建

```bash
# ============================================
# Lustre集群部署 (自建)
# ============================================

# === 服务器规划 ===
# MGS/MDS: 1台 (元数据服务器)
# OSS: 4台 (对象存储服务器，每台挂载2个OST)
# 客户端: N台GPU训练节点

# === 1. MGS/MDS节点配置 ===

# 1.1 安装Lustre软件
sudo apt-get install lustre-server-utils

# 1.2 创建MGS
sudo mkfs.lustre --mgs /dev/sdb  # sdb是MGS专用盘

# 1.3 挂载MGS
sudo mkdir -p /mnt/mgs
sudo mount -t lustre /dev/sdb /mnt/mgs

# 1.4 创建MDS
sudo mkfs.lustre \
    --mdt \
    --fsname=mlfs \  # 文件系统名称
    --mgsnode=192.168.1.10@tcp \  # MGS地址
    --index=0 \
    /dev/sdc  # sdc是MDS专用盘

# 1.5 挂载MDS
sudo mkdir -p /mnt/mdt
sudo mount -t lustre /dev/sdc /mnt/mdt

# === 2. OSS节点配置 (每个OSS节点) ===

# 2.1 安装Lustre
sudo apt-get install lustre-server-utils

# 2.2 创建OST (每个OSS创建2个OST)
# OSS节点1
sudo mkfs.lustre \
    --ost \
    --fsname=mlfs \
    --mgsnode=192.168.1.10@tcp \
    --index=0 \
    /dev/sdb  # OST 0

sudo mkfs.lustre \
    --ost \
    --fsname=mlfs \
    --mgsnode=192.168.1.10@tcp \
    --index=1 \
    /dev/sdc  # OST 1

# 2.3 挂载OST
sudo mkdir -p /mnt/ost0 /mnt/ost1
sudo mount -t lustre /dev/sdb /mnt/ost0
sudo mount -t lustre /dev/sdc /mnt/ost1

# === 3. 客户端配置 (GPU训练节点) ===

# 3.1 安装Lustre客户端
sudo apt-get install lustre-client-dkms

# 3.2 挂载Lustre文件系统
sudo mkdir -p /mnt/lustre
sudo mount -t lustre 192.168.1.10@tcp:/mlfs /mnt/lustre

# 3.3 验证
df -h /mnt/lustre
lfs df -h  # Lustre专用命令

# 输出：
# filesystem_summary:  400.0T  100.0T  300.0T   25% /mnt/lustre[OST:8]

# === 4. 性能调优 ===

# 4.1 设置条带化（Striping）
lfs setstripe -c 4 /mnt/lustre/training_data  # 4个OST并行

# 4.2 查看条带配置
lfs getstripe /mnt/lustre/training_data

# 4.3 优化大文件读写
lfs setstripe -c -1 /mnt/lustre/checkpoints  # 使用所有OST
```

---

### 5.3 Lustre性能测试

```python
"""
Lustre性能Benchmark
"""

import subprocess
import time
import os

class LustrePerformanceBenchmark:
    """
    Lustre性能测试
    """
    
    def test_sequential_write(self, file_size_gb=10, lustre_path='/mnt/lustre'):
        """
        测试顺序写性能
        """
        file_path = os.path.join(lustre_path, 'benchmark_write.dat')
        file_size_bytes = file_size_gb * 1024 ** 3
        
        print(f"测试顺序写 ({file_size_gb}GB)...")
        
        # 使用dd测试
        start = time.time()
        subprocess.run([
            'dd',
            'if=/dev/zero',
            f'of={file_path}',
            f'bs=1M',
            f'count={file_size_gb * 1024}',
            'conv=fsync'
        ], check=True, capture_output=True)
        duration = time.time() - start
        
        bandwidth_gbps = file_size_gb / duration
        print(f"  写带宽: {bandwidth_gbps:.2f} GB/s")
        
        return bandwidth_gbps
    
    def test_sequential_read(self, file_size_gb=10, lustre_path='/mnt/lustre'):
        """
        测试顺序读性能
        """
        file_path = os.path.join(lustre_path, 'benchmark_write.dat')
        
        print(f"测试顺序读 ({file_size_gb}GB)...")
        
        # 清除缓存
        subprocess.run(['sudo', 'sh', '-c', 'echo 3 > /proc/sys/vm/drop_caches'])
        
        # 使用dd测试
        start = time.time()
        subprocess.run([
            'dd',
            f'if={file_path}',
            'of=/dev/null',
            'bs=1M'
        ], check=True, capture_output=True)
        duration = time.time() - start
        
        bandwidth_gbps = file_size_gb / duration
        print(f"  读带宽: {bandwidth_gbps:.2f} GB/s")
        
        return bandwidth_gbps
    
    def test_iops(self, lustre_path='/mnt/lustre'):
        """
        测试IOPS (使用fio)
        """
        print("测试随机读写IOPS...")
        
        # 使用fio工具
        fio_config = f"""
[global]
directory={lustre_path}
size=10G
direct=1
ioengine=libaio
iodepth=32
numjobs=4

[random-read]
rw=randread
bs=4k

[random-write]
rw=randwrite
bs=4k
"""
        
        with open('/tmp/fio_config.ini', 'w') as f:
            f.write(fio_config)
        
        result = subprocess.run(
            ['fio', '/tmp/fio_config.ini'],
            capture_output=True,
            text=True
        )
        
        # 解析结果
        lines = result.stdout.split('\n')
        for line in lines:
            if 'IOPS=' in line:
                print(f"  {line.strip()}")

# 运行Benchmark
benchmark = LustrePerformanceBenchmark()
write_bw = benchmark.test_sequential_write(file_size_gb=10)
read_bw = benchmark.test_sequential_read(file_size_gb=10)
benchmark.test_iops()

# 典型结果：
# 顺序写: 8-15 GB/s (取决于OST数量)
# 顺序读: 10-20 GB/s
# 随机读IOPS: 100K-500K
# 随机写IOPS: 50K-200K
```

---

### 5.4 DataLoader with Lustre优化

```python
"""
PyTorch DataLoader + Lustre优化
"""

import torch
from torch.utils.data import DataLoader, Dataset
import os

class LustreOptimizedDataLoader:
    """
    针对Lustre优化的DataLoader配置
    """
    
    def create_dataloader(
        self,
        dataset,
        batch_size=32,
        num_workers=8,
        prefetch_factor=2
    ):
        """
        Lustre优化的DataLoader
        
        关键优化：
        1. num_workers适中（8-16，不是越多越好）
        2. prefetch_factor适中（2-4）
        3. pin_memory=True
        4. persistent_workers=True
        """
        dataloader = DataLoader(
            dataset,
            batch_size=batch_size,
            
            # 🔥 Lustre优化参数
            num_workers=8,              # 适中worker数
            prefetch_factor=2,           # 预取因子
            pin_memory=True,             # 固定内存（加速GPU传输）
            persistent_workers=True,     # 保持worker进程
            
            shuffle=True
        )
        
        return dataloader
    
    def optimize_file_layout(self, data_dir='/mnt/lustre/training_data'):
        """
        优化Lustre文件布局
        """
        print("Lustre文件布局优化:")
        
        # 1. 大文件：使用更多OST
        os.system(f"lfs setstripe -c -1 {data_dir}/large_files")  # 所有OST
        
        # 2. 小文件：使用少量OST（减少元数据压力）
        os.system(f"lfs setstripe -c 1 {data_dir}/small_files")
        
        # 3. Checkpoint目录：高条带数
        os.system(f"lfs setstripe -c 8 {data_dir}/checkpoints")
        
        print("✅ 文件布局已优化")
    
    def benchmark_data_loading(self, dataloader):
        """
        Benchmark数据加载速度
        """
        import time
        
        print("\nDataLoader性能测试:")
        
        # Warmup
        for _ in range(10):
            batch = next(iter(dataloader))
        
        # Benchmark
        start = time.time()
        num_batches = 100
        for i, batch in enumerate(dataloader):
            if i >= num_batches:
                break
        duration = time.time() - start
        
        batches_per_sec = num_batches / duration
        samples_per_sec = batches_per_sec * dataloader.batch_size
        
        print(f"  吞吐量: {batches_per_sec:.2f} batches/s")
        print(f"  样本/秒: {samples_per_sec:.0f}")
        print(f"  平均batch时间: {duration/num_batches*1000:.1f}ms")
        
        # 瓶颈诊断
        if batches_per_sec < 10:
            print("  ⚠️ 数据加载慢，可能瓶颈：")
            print("    - Lustre IO慢（检查lfs df）")
            print("    - num_workers太少（增加到16）")
            print("    - 数据预处理慢（优化transform）")

optimizer = LustreOptimizedDataLoader()
dataloader = optimizer.create_dataloader(dataset)
optimizer.benchmark_data_loading(dataloader)
```

---

<a name="network-optimization"></a>
## 🌐 6. 网络优化：RDMA & InfiniBand

### 6.1 RDMA原理

```python
"""
RDMA (Remote Direct Memory Access) 原理
"""

class RDMAExplainer:
    """
    RDMA vs 传统TCP/IP
    """
    
    def compare_network_stacks(self):
        """
        对比RDMA和TCP/IP
        """
        print("网络通信对比:")
        print("="*70)
        
        print("\n传统TCP/IP:")
        print("""
        GPU Memory → CPU Memory → TCP Stack → NIC → Network
                      ↑            ↑          ↑
                   CPU拷贝      CPU处理    中断/上下文切换
        
        问题：
        - CPU参与数据传输（占用CPU）
        - 多次内存拷贝（慢）
        - 内核协议栈开销（延迟高）
        """)
        
        print("\nRDMA (InfiniBand):")
        print("""
        GPU Memory → RDMA NIC → Network → RDMA NIC → GPU Memory
                      ↑                                  ↑
                直接访问（Zero-copy）            直接写入
        
        优势：
        ✅ 零拷贝（Zero-copy）
        ✅ 内核旁路（Kernel bypass）
        ✅ CPU卸载（CPU offload）
        ✅ 低延迟（<2微秒）
        ✅ 高带宽（400 Gb/s）
        """)
    
    def latency_comparison(self):
        """
        延迟对比
        """
        latencies = {
            '网络类型': ['延迟', '带宽', '适用场景'],
            
            'TCP/IP (Ethernet)': [
                '50-100 μs',
                '100 Gb/s',
                '单机/小规模'
            ],
            
            'RDMA over Converged Ethernet (RoCE)': [
                '5-10 μs',
                '200 Gb/s',
                '中等规模'
            ],
            
            'InfiniBand': [
                '1-2 μs',
                '400 Gb/s',
                '🔥 大规模训练'
            ]
        }
        
        import pandas as pd
        df = pd.DataFrame(latencies).set_index('网络类型')
        print("\n网络延迟对比:")
        print(df.to_string())
        
        print("\n🔥 结论：InfiniBand是大规模训练的标配")

explainer = RDMAExplainer()
explainer.compare_network_stacks()
explainer.latency_comparison()
```

---

### 6.2 InfiniBand集群配置

```bash
# ============================================
# InfiniBand网络配置
# ============================================

# === 1. 安装Mellanox OFED驱动 ===

# 1.1 下载MLNX_OFED (所有节点)
wget https://www.mellanox.com/downloads/ofed/MLNX_OFED-24.10-1.0.0.0/MLNX_OFED_LINUX-24.10-1.0.0.0-ubuntu22.04-x86_64.tgz

tar -xzvf MLNX_OFED_LINUX-24.10-1.0.0.0-ubuntu22.04-x86_64.tgz
cd MLNX_OFED_LINUX-24.10-1.0.0.0-ubuntu22.04-x86_64

# 1.2 安装
sudo ./mlnxofedinstall --force

# 1.3 重启OFED服务
sudo /etc/init.d/openibd restart

# 1.4 验证安装
ibstat

# 输出应该显示IB网卡信息：
# CA 'mlx5_0'
#     CA type: MT4123
#     Number of ports: 1
#     Firmware version: 28.42.1000
#     Port 1:
#         State: Active
#         Physical state: LinkUp
#         Rate: 400 Gb/s
#         Link layer: InfiniBand

# === 2. 配置IP over IB (IPoIB) ===

# 2.1 配置IB网卡IP
sudo ip addr add 10.0.0.1/24 dev ib0  # 根据节点编号修改
sudo ip link set ib0 up

# 2.2 测试IB网络连通性
ping -I ib0 10.0.0.2  # Ping其他节点

# === 3. 配置GPUDirect RDMA ===

# 3.1 加载nv_peer_mem模块
sudo modprobe nv_peer_mem

# 3.2 验证
lsmod | grep nv_peer_mem

# === 4. 性能测试 ===

# 4.1 IB带宽测试 (两节点之间)
# 节点1 (server):
ib_write_bw

# 节点2 (client):
ib_write_bw 10.0.0.1

# 输出：
#  #bytes     #iterations    BW peak[GB/sec]    BW average[GB/sec]
#  65536      5000           48.92              48.88

# 4.2 GPU-GPU RDMA测试
ib_write_bw -d mlx5_0 -a --use_cuda=0  # 使用GPU 0

# 输出应该显示GPUDirect RDMA生效
```

---

### 6.3 NCCL与InfiniBand集成

```python
"""
NCCL配置InfiniBand
"""

import os

class NCCLInfiniBandConfig:
    """
    NCCL + InfiniBand配置
    """
    
    def set_environment_variables(self):
        """
        设置NCCL环境变量
        """
        env_vars = {
            # === InfiniBand配置 ===
            'NCCL_IB_DISABLE': '0',              # 启用IB
            'NCCL_IB_HCA': 'mlx5',               # IB适配器前缀
            'NCCL_IB_GID_INDEX': '3',            # GID索引（RoCE用）
            
            # === GPUDirect RDMA ===
            'NCCL_IB_CUDA_SUPPORT': '1',         # 启用GPUDirect
            
            # === 性能调优 ===
            'NCCL_IB_TIMEOUT': '22',             # 超时（默认18）
            'NCCL_IB_QPS_PER_CONNECTION': '4',   # 每连接QP数
            'NCCL_IB_TC': '106',                 # Traffic Class
            
            # === 调试 ===
            'NCCL_DEBUG': 'INFO',                # 调试级别
            'NCCL_DEBUG_SUBSYS': 'INIT,NET',     # 子系统调试
            
            # === 网卡选择 ===
            'NCCL_SOCKET_IFNAME': 'ib0',         # 使用IB网卡
            'NCCL_NET_GDR_LEVEL': '5',           # GPUDirect等级
        }
        
        print("NCCL InfiniBand环境变量配置:")
        print("="*70)
        for key, value in env_vars.items():
            print(f"export {key}={value}")
            os.environ[key] = str(value)
        
        print("\n✅ NCCL已配置使用InfiniBand")
    
    def verify_nccl_ib(self):
        """
        验证NCCL使用InfiniBand
        """
        print("\n验证NCCL配置:")
        print("="*70)
        
        # 运行简单的NCCL测试
        test_script = """
import torch
import torch.distributed as dist

dist.init_process_group(backend='nccl')
rank = dist.get_rank()
world_size = dist.get_world_size()

tensor = torch.ones(1024, 1024).cuda(rank)
dist.all_reduce(tensor)

if rank == 0:
    print("✅ NCCL AllReduce成功")
"""
        
        print("运行NCCL测试...")
        print("查看日志中的网络类型：")
        print("  [INFO] NET/IB : Using [0]mlx5_0:1/IB")
        print("  ↑ 说明正在使用InfiniBand")

config = NCCLInfiniBandConfig()
config.set_environment_variables()
config.verify_nccl_ib()
```

---

<a name="cost-optimization"></a>
## 💰 7. 成本优化：Spot实例与混合云

### 7.1 Spot实例策略

根据 [AWS Spot Instances最佳实践](https://docs.aws.amazon.com/whitepapers/latest/cost-optimization-leveraging-ec2-spot-instances/spot-best-practices.html)：

```python
"""
Spot实例成本优化
"""

class SpotInstanceOptimization:
    """
    Spot实例优化策略
    
    节省：最高90%成本
    风险：可能被中断（2分钟通知）
    """
    
    def calculate_savings(self):
        """
        计算Spot实例节省
        """
        # 2025年AWS GPU实例价格
        pricing = {
            'p4d.24xlarge (8xA100 40GB)': {
                'on_demand': 32.77,  # $/hour
                'spot_avg': 9.83,    # $/hour (平均)
                'saving': 0.70
            },
            'p5.48xlarge (8xH100 80GB)': {
                'on_demand': 98.32,
                'spot_avg': 29.50,
                'saving': 0.70
            },
            'g5.48xlarge (8xA10G 24GB)': {
                'on_demand': 16.29,
                'spot_avg': 4.89,
                'saving': 0.70
            }
        }
        
        print("Spot实例价格对比（AWS，2025）:")
        print("="*70)
        for instance, prices in pricing.items():
            print(f"\n{instance}:")
            print(f"  On-Demand: ${prices['on_demand']:.2f}/hour")
            print(f"  Spot均价:  ${prices['spot_avg']:.2f}/hour")
            print(f"  节省:      {prices['saving']:.0%}")
            
            # 训练7天成本
            hours = 7 * 24
            on_demand_cost = prices['on_demand'] * hours
            spot_cost = prices['spot_avg'] * hours
            
            print(f"  7天训练成本:")
            print(f"    On-Demand: ${on_demand_cost:,.0f}")
            print(f"    Spot:      ${spot_cost:,.0f}")
            print(f"    节省:      ${on_demand_cost - spot_cost:,.0f}")
    
    def spot_interruption_handling(self):
        """
        Spot中断处理策略
        """
        print("\nSpot实例中断处理:")
        print("="*70)
        
        strategies = {
            '1. Checkpoint频繁保存': {
                'description': '每N步保存checkpoint',
                'code': '''
# 每100步保存checkpoint
if step % 100 == 0:
    save_checkpoint(model, optimizer, step)
''',
                'recovery_time': '损失<100步训练'
            },
            
            '2. 监听中断信号': {
                'description': 'AWS会在2分钟前发送中断通知',
                'code': '''
# 监听EC2元数据服务
import requests
import signal

def check_spot_interruption():
    try:
        r = requests.get(
            'http://169.254.169.254/latest/meta-data/spot/instance-action',
            timeout=1
        )
        if r.status_code == 200:
            print("⚠️ Spot中断通知！保存checkpoint...")
            save_checkpoint(model, optimizer, step)
            exit(0)
    except:
        pass

# 每30秒检查一次
while training:
    check_spot_interruption()
    time.sleep(30)
''',
                'recovery_time': '0损失（提前保存）'
            },
            
            '3. Checkpoint到持久化存储': {
                'description': '保存到S3/EBS而非本地',
                'code': '''
# 保存到S3
import boto3

s3 = boto3.client('s3')
torch.save(checkpoint, '/tmp/checkpoint.pth')
s3.upload_file(
    '/tmp/checkpoint.pth',
    'my-bucket',
    f'checkpoints/checkpoint_step_{step}.pth'
)
''',
                'recovery_time': '新实例可立即恢复'
            },
            
            '4. 多实例类型fallback': {
                'description': '配置多种Spot实例类型',
                'code': '''
# EC2 Auto Scaling配置
{
  "OverrideInstances": [
    {"InstanceType": "p4d.24xlarge"},
    {"InstanceType": "p3dn.24xlarge"},  # fallback
    {"InstanceType": "p3.16xlarge"}      # fallback 2
  ],
  "SpotAllocationStrategy": "price-capacity-optimized"
}
''',
                'recovery_time': '自动切换到其他类型'
            }
        }
        
        for strategy, details in strategies.items():
            print(f"\n{strategy}")
            print(f"  {details['description']}")
            print(f"  恢复时间: {details['recovery_time']}")
            if 'code' in details:
                print(f"  代码示例:")
                for line in details['code'].strip().split('\n'):
                    print(f"    {line}")

optimizer = SpotInstanceOptimization()
optimizer.calculate_savings()
optimizer.spot_interruption_handling()
```

---

### 7.2 混合云架构

```python
"""
混合云训练架构
"""

class HybridCloudArchitecture:
    """
    混合云（On-premise + Cloud）
    
    策略：
    - 日常训练：自建集群
    - 峰值需求：Cloud burst
    - 成本优化：Reserved + Spot混合
    """
    
    def design_hybrid_architecture(self):
        """
        混合云架构设计
        """
        print("混合云架构:")
        print("="*70)
        print("""
┌──────────────────────────────────────────────────┐
│  On-Premise集群 (自建)                           │
│  ┌─────────────────────────────────────────┐     │
│  │ 基础容量: 64x H100                      │     │
│  │ 利用率目标: 80%                         │     │
│  │ 用途: 日常训练、实验                     │     │
│  └─────────────────────────────────────────┘     │
└──────────────┬───────────────────────────────────┘
               │
               ↓ 峰值时burst到云端
┌──────────────────────────────────────────────────┐
│  Cloud (AWS/GCP/Azure)                           │
│  ┌─────────────────────────────────────────┐     │
│  │ Reserved实例: 32x H100 (3年，省50%)     │     │
│  │ Spot实例: 0-128x H100 (省70%)           │     │
│  │ 用途: 弹性扩展、紧急训练                 │     │
│  └─────────────────────────────────────────┘     │
└──────────────────────────────────────────────────┘
        """)
        
        # 成本分析
        self.calculate_hybrid_cost()
    
    def calculate_hybrid_cost(self):
        """
        混合云成本计算
        """
        print("\n成本对比（年度）:")
        print("="*70)
        
        # 场景：平均64 GPU使用，峰值128 GPU
        
        # 方案1: 全部On-Premise
        on_premise_capex = 250_000 * 16  # 16台DGX H100
        on_premise_opex = 100_000  # 电费+维护
        on_premise_total = on_premise_capex + on_premise_opex
        
        print("\n方案1：全部On-Premise (128 GPU)")
        print(f"  CAPEX (硬件): ${on_premise_capex:,}")
        print(f"  OPEX (电费/维护): ${on_premise_opex:,}/年")
        print(f"  3年TCO: ${(on_premise_total + on_premise_opex * 2):,}")
        
        # 方案2: 全部Cloud (On-Demand)
        hours_per_year = 365 * 24
        cloud_on_demand = 98.32 * 128 * hours_per_year  # 128x H100
        
        print("\n方案2：全部Cloud On-Demand (128 GPU)")
        print(f"  年度成本: ${cloud_on_demand:,}")
        print(f"  3年TCO: ${cloud_on_demand * 3:,}")
        
        # 方案3: 混合云
        # - On-Premise: 64 GPU (基础负载)
        # - Reserved: 32 GPU (70%时间使用)
        # - Spot: 32 GPU (峰值时使用，30%时间)
        
        hybrid_on_premise = 250_000 * 8 + 50_000  # 8台DGX + 运维
        hybrid_reserved = 98.32 * 32 * hours_per_year * 0.5  # Reserved省50%
        hybrid_spot = 98.32 * 32 * hours_per_year * 0.3 * 0.3  # Spot省70%，用30%时间
        hybrid_total_annual = hybrid_on_premise + hybrid_reserved + hybrid_spot
        
        print("\n方案3：混合云 (64 on-premise + 32 reserved + 32 spot)")
        print(f"  On-Premise: ${hybrid_on_premise:,}/年")
        print(f"  Reserved: ${hybrid_reserved:,}/年")
        print(f"  Spot: ${hybrid_spot:,}/年")
        print(f"  年度成本: ${hybrid_total_annual:,}")
        print(f"  3年TCO: ${hybrid_total_annual * 3:,}")
        
        print("\n📊 成本对比总结:")
        print(f"  全On-Premise 3年: ${(on_premise_total + on_premise_opex * 2):,}")
        print(f"  全Cloud 3年:      ${cloud_on_demand * 3:,}")
        print(f"  混合云 3年:       ${hybrid_total_annual * 3:,}")
        print(f"\n🔥 混合云节省: {(1 - hybrid_total_annual * 3 / (cloud_on_demand * 3)) * 100:.0f}%")

hybrid = HybridCloudArchitecture()
hybrid.design_hybrid_architecture()
```

---

<a name="monitoring-system"></a>
## 📊 8. 监控体系：DCGM + Prometheus + Grafana

### 8.1 DCGM (Data Center GPU Manager)

```bash
# ============================================
# DCGM安装与配置
# ============================================

# 1. 安装DCGM (所有GPU节点)
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt-get update
sudo apt-get install -y datacenter-gpu-manager

# 2. 启动DCGM服务
sudo systemctl start nvidia-dcgm
sudo systemctl enable nvidia-dcgm

# 3. 验证DCGM
dcgmi discovery -l

# 输出：
# 8 GPUs found.
# +--------+----------------------------------------------------------------------+
# | GPU ID | Device Information                                                   |
# +--------+----------------------------------------------------------------------+
# | 0      | Name: NVIDIA H100 80GB HBM3                                          |
# | 1      | Name: NVIDIA H100 80GB HBM3                                          |
# ...

# 4. 查看GPU指标
dcgmi dmon -e 100,101,102,103,104  # 温度、功率、SM利用率、显存

# ============================================
# DCGM Exporter for Prometheus
# ============================================

# 使用Helm安装DCGM Exporter
helm repo add gpu-helm-charts https://nvidia.github.io/dcgm-exporter/helm-charts
helm repo update

helm install dcgm-exporter gpu-helm-charts/dcgm-exporter \
    --namespace gpu-monitoring \
    --create-namespace

# 验证
kubectl get pods -n gpu-monitoring

# DCGM Exporter会暴露metrics在 :9400/metrics
```

---

### 8.2 完整监控栈部署

```yaml
# ============================================
# Prometheus配置（监控GPU）
# ============================================

# prometheus-values.yaml (Helm配置)
prometheus:
  prometheusSpec:
    # Scrape DCGM Exporter
    additionalScrapeConfigs:
      - job_name: 'dcgm-exporter'
        kubernetes_sd_configs:
          - role: pod
            namespaces:
              names:
                - gpu-monitoring
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_label_app]
            action: keep
            regex: dcgm-exporter
    
    # 存储配置
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: fast-ssd
          resources:
            requests:
              storage: 100Gi
    
    # 资源配置
    resources:
      requests:
        cpu: 2000m
        memory: 8Gi
      limits:
        cpu: 4000m
        memory: 16Gi

# 安装Prometheus Stack
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
    -f prometheus-values.yaml \
    -n monitoring \
    --create-namespace
```

---

### 8.3 Grafana Dashboard配置

```python
"""
Grafana GPU监控Dashboard
"""

def create_gpu_monitoring_dashboard():
    """
    创建GPU监控Dashboard
    """
    panels = [
        {
            'title': 'GPU利用率',
            'query': 'DCGM_FI_DEV_GPU_UTIL',
            'description': '目标：>80%',
            'alert_threshold': 50  # <50%告警（利用率低）
        },
        {
            'title': 'GPU显存使用',
            'query': 'DCGM_FI_DEV_FB_USED / DCGM_FI_DEV_FB_TOTAL * 100',
            'description': '显存使用百分比',
            'alert_threshold': 95  # >95%告警（接近OOM）
        },
        {
            'title': 'GPU温度',
            'query': 'DCGM_FI_DEV_GPU_TEMP',
            'description': 'GPU温度（℃）',
            'alert_threshold': 85  # >85℃告警
        },
        {
            'title': 'GPU功率',
            'query': 'DCGM_FI_DEV_POWER_USAGE',
            'description': '当前功耗（W）',
            'alert_threshold': 700  # H100最大700W
        },
        {
            'title': 'NVLink带宽',
            'query': 'DCGM_FI_PROF_NVLINK_TX_BYTES + DCGM_FI_PROF_NVLINK_RX_BYTES',
            'description': 'NVLink传输带宽（GB/s）',
            'alert_threshold': None
        },
        {
            'title': 'PCIe带宽',
            'query': 'DCGM_FI_PROF_PCIE_TX_BYTES + DCGM_FI_PROF_PCIE_RX_BYTES',
            'description': 'PCIe传输带宽（GB/s）',
            'alert_threshold': None
        },
        {
            'title': 'Tensor Core利用率',
            'query': 'DCGM_FI_PROF_PIPE_TENSOR_ACTIVE',
            'description': 'Tensor Core活跃百分比',
            'alert_threshold': 50
        },
        {
            'title': 'ECC错误',
            'query': 'DCGM_FI_DEV_ECC_DBE_VOL_TOTAL',
            'description': 'ECC双位错误（严重）',
            'alert_threshold': 0  # 任何错误都告警
        }
    ]
    
    print("GPU监控Dashboard面板:")
    print("="*70)
    for i, panel in enumerate(panels, 1):
        print(f"\n{i}. {panel['title']}")
        print(f"   PromQL: {panel['query']}")
        print(f"   说明: {panel['description']}")
        if panel['alert_threshold'] is not None:
            print(f"   告警阈值: {panel['alert_threshold']}")
    
    return panels

# 创建Dashboard
panels = create_gpu_monitoring_dashboard()

# Grafana自动生成的Dashboard ID
grafana_dashboards = {
    'NVIDIA DCGM Exporter Dashboard': 12219,
    'NVIDIA GPU Monitoring': 15117,
}

print("\n\n推荐Grafana Dashboard (可导入):")
for name, dashboard_id in grafana_dashboards.items():
    print(f"  {name}: https://grafana.com/grafana/dashboards/{dashboard_id}")
```

---

<a name="fault-diagnosis"></a>
## 🔧 9. 故障诊断与恢复

### 9.1 常见GPU集群故障

```python
"""
GPU集群故障诊断手册
"""

class GPUClusterTroubleshooting:
    """
    GPU集群故障诊断
    """
    
    def diagnose_gpu_not_detected(self):
        """
        故障1：GPU检测不到
        """
        print("❌ 故障：kubectl无法看到GPU资源")
        print("="*70)
        
        diagnostics = [
            {
                'check': '1. 检查NVIDIA驱动',
                'command': 'nvidia-smi',
                'expected': '显示GPU列表',
                'fix': 'sudo apt-get install nvidia-driver-535'
            },
            {
                'check': '2. 检查GPU Operator Pod',
                'command': 'kubectl get pods -n gpu-operator',
                'expected': '所有Pod Running',
                'fix': 'kubectl logs -n gpu-operator <pod-name>'
            },
            {
                'check': '3. 检查Device Plugin',
                'command': 'kubectl get ds -n gpu-operator nvidia-device-plugin-daemonset',
                'expected': 'DaemonSet Ready',
                'fix': '检查节点label和toleration'
            },
            {
                'check': '4. 检查节点GPU资源',
                'command': 'kubectl describe node <node-name> | grep nvidia.com/gpu',
                'expected': 'Capacity: nvidia.com/gpu: 8',
                'fix': '重启GPU Operator: helm upgrade ...'
            }
        ]
        
        for diag in diagnostics:
            print(f"\n{diag['check']}")
            print(f"  命令: {diag['command']}")
            print(f"  期望: {diag['expected']}")
            print(f"  修复: {diag['fix']}")
    
    def diagnose_nccl_hang(self):
        """
        故障2：NCCL通信挂起
        """
        print("\n\n❌ 故障：训练卡死（NCCL hang）")
        print("="*70)
        
        diagnostics = [
            {
                'symptom': '所有进程卡在AllReduce',
                'possible_causes': [
                    '网络配置错误',
                    'InfiniBand未启用',
                    '防火墙阻止通信',
                    '网卡故障'
                ],
                'diagnosis_steps': [
                    '1. 检查NCCL日志: export NCCL_DEBUG=INFO',
                    '2. 测试网络: nccl-tests/build/all_reduce_perf -b 8 -e 128M -f 2 -g 8',
                    '3. 检查IB状态: ibstat',
                    '4. 测试节点间通信: ib_write_bw <remote-ip>'
                ],
                'fixes': [
                    'export NCCL_IB_DISABLE=0  # 启用IB',
                    'export NCCL_SOCKET_IFNAME=ib0  # 指定IB网卡',
                    'export NCCL_IB_HCA=mlx5  # 指定IB适配器',
                    '检查/etc/hosts配置主机名'
                ]
            }
        ]
        
        for diag in diagnostics:
            print(f"\n症状: {diag['symptom']}")
            print(f"\n可能原因:")
            for cause in diag['possible_causes']:
                print(f"  - {cause}")
            print(f"\n诊断步骤:")
            for step in diag['diagnosis_steps']:
                print(f"  {step}")
            print(f"\n修复方法:")
            for fix in diag['fixes']:
                print(f"  {fix}")
    
    def diagnose_oom(self):
        """
        故障3：GPU OOM
        """
        print("\n\n❌ 故障：GPU Out of Memory")
        print("="*70)
        
        print("""
诊断步骤：

1. 检查显存占用
   nvidia-smi
   
   分析：
   - 模型参数占用
   - 梯度占用
   - 优化器状态占用
   - 激活值占用

2. 计算理论显存需求
   total_memory = model_params * (2 + 2 + 8 + activation_multiplier)
   - 2 bytes: FP16参数
   - 2 bytes: FP16梯度
   - 8 bytes: Adam优化器（2个moment + FP32 master）
   - activation_multiplier: 通常10-20倍参数量

3. 解决方案（按优先级）：
   ✅ 减小batch size
   ✅ 启用梯度累积
   ✅ 启用gradient checkpointing
   ✅ 使用混合精度（FP16/BF16）
   ✅ 使用FSDP/DeepSpeed ZeRO
   ✅ 减小序列长度
        """)

troubleshooter = GPUClusterTroubleshooting()
troubleshooter.diagnose_gpu_not_detected()
troubleshooter.diagnose_nccl_hang()
troubleshooter.diagnose_oom()
```

---

<a name="capacity-planning"></a>
## 📐 10. 容量规划与扩展

### 10.1 容量规划计算器

```python
"""
GPU集群容量规划
"""

class GPUCapacityPlanner:
    """
    GPU集群容量规划工具
    """
    
    def __init__(self):
        # GPU规格
        self.gpu_specs = {
            'H100': {'memory_gb': 80, 'price_per_hour': 5.00},
            'A100-80GB': {'memory_gb': 80, 'price_per_hour': 3.50},
            'A100-40GB': {'memory_gb': 40, 'price_per_hour': 2.50},
        }
    
    def estimate_training_time(
        self,
        model_params_b=70,
        dataset_tokens_b=1000,
        num_gpus=64,
        gpu_type='H100'
    ):
        """
        估算训练时间
        
        基于DeepSeek-V3经验：
        - 671B模型，14.8T tokens → 2788K GPU-hours (H800)
        """
        # 简化估算公式（经验）
        # GPU-hours ≈ model_params_B * dataset_tokens_B * 0.004
        
        gpu_hours = model_params_b * dataset_tokens_b * 0.004
        
        # 考虑效率损失（通信、IO等）
        efficiency = 0.85  # 85%效率
        actual_gpu_hours = gpu_hours / efficiency
        
        # 计算wallclock时间
        wallclock_hours = actual_gpu_hours / num_gpus
        wallclock_days = wallclock_hours / 24
        
        # 成本
        cost_per_hour = self.gpu_specs[gpu_type]['price_per_hour']
        total_cost = actual_gpu_hours * cost_per_hour
        
        print(f"训练时间估算:")
        print("="*70)
        print(f"模型规模: {model_params_b}B参数")
        print(f"数据规模: {dataset_tokens_b}B tokens")
        print(f"GPU配置: {num_gpus}x {gpu_type}")
        print(f"\n估算结果:")
        print(f"  GPU-hours: {actual_gpu_hours:,.0f}")
        print(f"  实际训练时间: {wallclock_days:.1f} 天")
        print(f"  总成本: ${total_cost:,.0f}")
        print(f"  每天成本: ${total_cost/wallclock_days:,.0f}")
        
        return wallclock_days, total_cost
    
    def plan_cluster_size(
        self,
        monthly_training_jobs=10,
        avg_model_size_b=13,
        avg_dataset_b=100,
        target_turnaround_days=3
    ):
        """
        规划集群规模
        """
        print(f"\n集群规模规划:")
        print("="*70)
        print(f"需求:")
        print(f"  每月训练作业: {monthly_training_jobs}")
        print(f"  平均模型规模: {avg_model_size_b}B")
        print(f"  平均数据规模: {avg_dataset_b}B tokens")
        print(f"  目标完成时间: {target_turnaround_days}天")
        
        # 单个作业GPU-hours
        gpu_hours_per_job = avg_model_size_b * avg_dataset_b * 0.004 / 0.85
        
        # 单个作业所需GPU数（在target_turnaround_days内完成）
        gpus_per_job = gpu_hours_per_job / (target_turnaround_days * 24)
        
        # 考虑并发（假设平均3个作业同时运行）
        concurrent_jobs = 3
        total_gpus_needed = gpus_per_job * concurrent_jobs
        
        # 向上取整到8的倍数（8卡服务器）
        num_nodes = int(np.ceil(total_gpus_needed / 8))
        total_gpus = num_nodes * 8
        
        print(f"\n推荐配置:")
        print(f"  单作业GPU-hours: {gpu_hours_per_job:,.0f}")
        print(f"  单作业所需GPU: {gpus_per_job:.0f}")
        print(f"  并发作业数: {concurrent_jobs}")
        print(f"  总GPU需求: {total_gpus_needed:.0f}")
        print(f"  推荐节点数: {num_nodes} (8卡/节点)")
        print(f"  总GPU数: {total_gpus}")
        
        # 成本估算
        gpu_type = 'H100'
        monthly_cost = (
            total_gpus * 
            self.gpu_specs[gpu_type]['price_per_hour'] * 
            24 * 30
        )
        
        print(f"\n成本估算 ({gpu_type}):")
        print(f"  月度成本: ${monthly_cost:,.0f}")
        print(f"  年度成本: ${monthly_cost * 12:,.0f}")

planner = GPUCapacityPlanner()

# 示例1：70B模型训练
planner.estimate_training_time(
    model_params_b=70,
    dataset_tokens_b=1000,
    num_gpus=64,
    gpu_type='H100'
)

# 示例2：集群规模规划
planner.plan_cluster_size(
    monthly_training_jobs=10,
    avg_model_size_b=13,
    avg_dataset_b=100,
    target_turnaround_days=3
)
```

---

## 📚 参考资料

### GPU管理

1. **NVIDIA GPU Operator**  
   [GPU Operator Documentation](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/)  
   K8s GPU管理（2025 v25.10.1）

2. **NVIDIA DCGM**  
   [DCGM Exporter](https://github.com/NVIDIA/dcgm-exporter)  
   GPU监控工具

### 作业调度

3. **SLURM**  
   [SLURM Documentation](https://slurm.schedmd.com/documentation.html)  
   作业调度系统（v25.11）

4. **Kubeflow PyTorchJob**  
   [PyTorchJob Guide](https://www.kubeflow.org/docs/components/training/pytorch/)  
   K8s分布式训练

### 高性能存储

5. **Lustre**  
   - [AWS FSx for Lustre](https://aws.amazon.com/fsx/lustre/)  
   - [Google Managed Lustre](https://cloud.google.com/managed-lustre/docs/overview)  
   PB级高性能存储

6. **Lustre vs GPFS**  
   [Lustre vs GPFS Comparison](https://www.baculasystems.com/blog/lustre-vs-gpfs/)  
   两大HPC文件系统对比

### 网络优化

7. **RDMA & InfiniBand**  
   [NVIDIA InfiniBand Solutions](https://docs.nvidia.com/networking/)  
   高速网络配置

8. **GPUDirect RDMA**  
   [GPUDirect RDMA Benchmark](https://docs.oracle.com/en/learn/gpudirect-rdma-ib-write-bw/)  
   GPU直接RDMA通信

### 成本优化

9. **AWS Spot Instances**  
   [Spot Best Practices](https://docs.aws.amazon.com/whitepapers/latest/cost-optimization-leveraging-ec2-spot-instances/spot-best-practices.html)  
   Spot实例最佳实践

### 监控告警

10. **Prometheus & Grafana**  
    [Grafana ML Monitoring](https://grafana.com/blog/monitoring-machine-learning-models-in-production-with-grafana-and-clearml/)  
    ML监控实践

---

## 🎯 总结

### 关键要点回顾

1. **GPU选型**：H100首选大规模训练，A100性价比高，RTX 4090适合研究
2. **编排选择**：K8s（云原生）+ SLURM（HPC成熟）混合使用
3. **存储系统**：Lustre是PB级训练数据的标准方案
4. **网络优化**：InfiniBand + RDMA是大规模训练必备
5. **成本优化**：混合云 + Spot实例可节省50-70%
6. **监控体系**：DCGM + Prometheus + Grafana三件套

### 基础设施规模参考

| 模型规模 | GPU数量 | 网络 | 存储 | 成本/7天训练 |
|---------|---------|------|------|-------------|
| 7B | 8 | Ethernet | NVMe | ~$3K |
| 13B-30B | 32 | InfiniBand | Lustre | ~$30K |
| 70B | 64 | InfiniBand | Lustre | ~$150K |
| 175B+ | 512+ | InfiniBand | Lustre | ~$1.5M |

### 传统程序员的优势

- ✅ **系统架构能力**：分布式系统、网络、存储是已有技能
- ✅ **运维经验**：监控、告警、故障诊断直接迁移
- ✅ **成本意识**：资源优化、容量规划是工程师基本功

### 下一步学习

- 🔗 **下一篇**：[13 - 生产化部署指南：从训练到服务的完整流程](./13-production-deployment.md)
- 💻 **动手实践**：搭建本地K8s + GPU Operator环境
- 📊 **实战项目**：配置DCGM + Grafana监控自己的GPU

---

> 💡 **璇玑的小贴士**：训练基础设施就像搭建数据中心——GPU是CPU，InfiniBand是网络，Lustre是分布式存储。传统程序员的系统架构能力在这里是核心优势！硬件和软件的协同优化，才能发挥GPU的最大价值~ ✨
>
> 道友现在对训练基础设施有感觉了吗？下一篇我们聊生产化部署，教你如何把训练好的模型稳定高效地serving起来！🚀