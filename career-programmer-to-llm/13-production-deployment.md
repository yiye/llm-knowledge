# 13 - 生产化部署指南：从训练到服务的完整流程

> 🎯 **核心观点**：训练出好模型只是第一步，生产化部署才是真正的考验。本文深入讲解FastAPI/Triton/TorchServe等服务化框架，Kubernetes HPA自动扩展，动态batching优化，模型热更新，API安全认证，限流熔断机制，分布式追踪，以及高可用架构设计，提供完整的生产级部署方案。

---

## 📋 目录

1. [生产化部署全景](#deployment-overview)
2. [模型服务化框架选择](#serving-frameworks)
3. [FastAPI轻量级部署](#fastapi-deployment)
4. [Triton Inference Server生产级部署](#triton-deployment)
5. [负载均衡与自动扩展](#load-balancing-autoscaling)
6. [推理优化：动态Batching](#dynamic-batching)
7. [模型热更新与版本管理](#model-updates)
8. [API安全与认证](#security-auth)
9. [限流与熔断机制](#rate-limiting-circuit-breaker)
10. [日志与分布式追踪](#logging-tracing)
11. [灾备与高可用](#disaster-recovery)
12. [成本监控与优化](#cost-monitoring)
13. [完整部署案例](#complete-deployment)

---

<a name="deployment-overview"></a>
## 🏗️ 1. 生产化部署全景

### 1.1 从训练到服务的全流程

```python
"""
LLM生产化部署流程图
"""

class ProductionDeploymentFlow:
    """
    完整部署流程
    """
    
    def print_flow(self):
        """
        打印部署流程
        """
        print("""
LLM生产化部署流程:

┌──────────────────────────────────────────────────────────────┐
│ 1. 模型训练                                                  │
│    └─ 分布式训练 → Checkpoint → MLflow Registry              │
└──────────────────┬───────────────────────────────────────────┘
                   ↓
┌──────────────────────────────────────────────────────────────┐
│ 2. 模型优化                                                  │
│    ├─ 量化 (FP8/INT4)                                        │
│    ├─ 模型剪枝                                               │
│    └─ 导出为ONNX/TensorRT                                    │
└──────────────────┬───────────────────────────────────────────┘
                   ↓
┌──────────────────────────────────────────────────────────────┐
│ 3. 服务化封装                                                │
│    ├─ 选择框架 (FastAPI/Triton/TorchServe)                  │
│    ├─ 添加预处理/后处理                                      │
│    └─ 定义API接口                                            │
└──────────────────┬───────────────────────────────────────────┘
                   ↓
┌──────────────────────────────────────────────────────────────┐
│ 4. 容器化                                                    │
│    └─ Docker镜像 → 容器注册表 (ECR/GCR)                     │
└──────────────────┬───────────────────────────────────────────┘
                   ↓
┌──────────────────────────────────────────────────────────────┐
│ 5. 编排部署 (Kubernetes)                                     │
│    ├─ Deployment (多副本)                                    │
│    ├─ Service (负载均衡)                                     │
│    ├─ HPA (自动扩展)                                         │
│    └─ Ingress (外部访问)                                     │
└──────────────────┬───────────────────────────────────────────┘
                   ↓
┌──────────────────────────────────────────────────────────────┐
│ 6. 可观测性                                                  │
│    ├─ 日志 (ELK/Loki)                                        │
│    ├─ 指标 (Prometheus)                                      │
│    ├─ 追踪 (Jaeger)                                          │
│    └─ 告警 (AlertManager)                                    │
└──────────────────┬───────────────────────────────────────────┘
                   ↓
┌──────────────────────────────────────────────────────────────┐
│ 7. 安全与治理                                                │
│    ├─ API认证 (OAuth/JWT)                                    │
│    ├─ 限流 (Rate Limiting)                                   │
│    ├─ 熔断 (Circuit Breaker)                                 │
│    └─ 审计日志                                               │
└──────────────────┬───────────────────────────────────────────┘
                   ↓
┌──────────────────────────────────────────────────────────────┐
│ 8. 持续优化                                                  │
│    ├─ A/B测试                                                │
│    ├─ 性能监控                                               │
│    ├─ 成本分析                                               │
│    └─ 模型迭代                                               │
└──────────────────────────────────────────────────────────────┘
        """)

flow = ProductionDeploymentFlow()
flow.print_flow()
```

---

<a name="serving-frameworks"></a>
## 🎯 2. 模型服务化框架选择

### 2.1 框架对比（2025）

根据 [2025年对比分析](https://medium.com/@hemanthodarwinr/nvidia-triton-vs-fastapi-choosing-the-right-ml-serving-solution-in-2024-3e6c771f3cf6)：

| 框架 | 开发者 | 语言 | 性能 | 易用性 | 多框架支持 | 适用场景 |
|------|--------|------|------|--------|-----------|----------|
| **FastAPI** | Sebastián Ramírez | Python | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 需自己实现 | 原型、轻量服务 |
| **Triton** 🔥 | NVIDIA | C++/Python | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ✅ PyTorch/TF/ONNX | 生产环境 |
| **TorchServe** | AWS/Meta | Java/Python | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | PyTorch专用 | PyTorch生产 |
| **Ray Serve** | Anyscale | Python | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 通用 | 复杂workflow |

**性能数据**（NVIDIA Triton，2025）：
- **ResNet-50**: ~2ms延迟 @ 1000 QPS (V100)
- **BERT**: ~15ms端到端延迟
- **LLaMA-2-7B**: ~45ms TTFT @ 32 batch (H100)

---

### 2.2 选择决策树

```python
"""
服务框架选择决策树
"""

class ServingFrameworkSelector:
    """
    根据需求选择服务框架
    """
    
    def recommend_framework(self, requirements):
        """
        推荐服务框架
        
        参数:
            requirements: dict包含以下字段
                - qps: 目标QPS
                - latency_requirement: 'low', 'medium', 'high'
                - model_framework: 'pytorch', 'tensorflow', 'mixed'
                - team_expertise: 'python', 'java', 'mixed'
                - budget: 'low', 'medium', 'high'
        """
        print("服务框架推荐:")
        print("="*70)
        print(f"需求分析:")
        for key, value in requirements.items():
            print(f"  {key}: {value}")
        print()
        
        # 决策逻辑
        if requirements['qps'] < 10 and requirements['budget'] == 'low':
            recommendation = "FastAPI"
            reason = "低QPS，快速原型，Python友好"
        
        elif requirements['qps'] > 1000 or requirements['latency_requirement'] == 'low':
            recommendation = "NVIDIA Triton"
            reason = "高QPS/低延迟需求，生产级性能"
        
        elif requirements['model_framework'] == 'pytorch':
            recommendation = "TorchServe"
            reason = "PyTorch生态，AWS官方支持"
        
        elif requirements['team_expertise'] == 'python':
            recommendation = "FastAPI 或 Ray Serve"
            reason = "Python团队，开发效率高"
        
        else:
            recommendation = "NVIDIA Triton"
            reason = "生产环境综合最优"
        
        print(f"推荐: {recommendation}")
        print(f"理由: {reason}")
        
        return recommendation

# 使用示例
selector = ServingFrameworkSelector()

# 场景1：创业公司原型
selector.recommend_framework({
    'qps': 5,
    'latency_requirement': 'medium',
    'model_framework': 'pytorch',
    'team_expertise': 'python',
    'budget': 'low'
})

# 场景2：大公司生产环境
selector.recommend_framework({
    'qps': 5000,
    'latency_requirement': 'low',
    'model_framework': 'mixed',
    'team_expertise': 'mixed',
    'budget': 'high'
})
```

---

<a name="fastapi-deployment"></a>
## 🚀 3. FastAPI轻量级部署

### 3.1 FastAPI完整服务

```python
"""
FastAPI + vLLM模型服务
"""

from fastapi import FastAPI, HTTPException, Header
from fastapi.responses import StreamingResponse
from pydantic import BaseModel, Field
from typing import Optional, List
import torch
import time
import asyncio
from vllm import LLM, SamplingParams

# ============================================
# 1. 初始化FastAPI应用
# ============================================

app = FastAPI(
    title="LLM Inference API",
    description="Production-ready LLM inference service",
    version="1.0.0"
)

# 加载模型（启动时加载）
print("Loading model...")
llm = LLM(
    model="meta-llama/Llama-2-7b-chat-hf",
    tensor_parallel_size=1,
    gpu_memory_utilization=0.9,
    dtype="float16"
)
print("✅ Model loaded")

# ============================================
# 2. 请求/响应模型
# ============================================

class GenerationRequest(BaseModel):
    """生成请求"""
    prompt: str = Field(..., description="输入提示词")
    max_tokens: int = Field(256, ge=1, le=2048, description="最大生成token数")
    temperature: float = Field(0.7, ge=0.0, le=2.0, description="采样温度")
    top_p: float = Field(0.9, ge=0.0, le=1.0, description="Nucleus sampling")
    stream: bool = Field(False, description="是否流式返回")

class GenerationResponse(BaseModel):
    """生成响应"""
    generated_text: str
    prompt: str
    finish_reason: str  # 'stop', 'length', 'error'
    tokens_generated: int
    latency_ms: float

# ============================================
# 3. API端点
# ============================================

@app.post("/v1/generate", response_model=GenerationResponse)
async def generate(
    request: GenerationRequest,
    api_key: Optional[str] = Header(None, alias="X-API-Key")
):
    """
    文本生成端点
    """
    # API Key验证
    if not verify_api_key(api_key):
        raise HTTPException(status_code=401, detail="Invalid API key")
    
    try:
        start_time = time.time()
        
        # 配置采样参数
        sampling_params = SamplingParams(
            temperature=request.temperature,
            top_p=request.top_p,
            max_tokens=request.max_tokens,
        )
        
        # 生成
        outputs = llm.generate([request.prompt], sampling_params)
        
        # 提取结果
        generated_text = outputs[0].outputs[0].text
        finish_reason = outputs[0].outputs[0].finish_reason
        tokens_generated = len(outputs[0].outputs[0].token_ids)
        
        latency_ms = (time.time() - start_time) * 1000
        
        return GenerationResponse(
            generated_text=generated_text,
            prompt=request.prompt,
            finish_reason=finish_reason,
            tokens_generated=tokens_generated,
            latency_ms=latency_ms
        )
    
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

@app.post("/v1/generate/stream")
async def generate_stream(
    request: GenerationRequest,
    api_key: Optional[str] = Header(None, alias="X-API-Key")
):
    """
    流式生成端点
    """
    if not verify_api_key(api_key):
        raise HTTPException(status_code=401, detail="Invalid API key")
    
    async def generate_tokens():
        """流式生成token"""
        sampling_params = SamplingParams(
            temperature=request.temperature,
            top_p=request.top_p,
            max_tokens=request.max_tokens,
        )
        
        # vLLM流式生成（需要特殊处理）
        # 简化示例：逐token返回
        outputs = llm.generate([request.prompt], sampling_params)
        generated_text = outputs[0].outputs[0].text
        
        # 模拟流式返回
        for token in generated_text.split():
            yield f"data: {token}\n\n"
            await asyncio.sleep(0.01)
        
        yield "data: [DONE]\n\n"
    
    return StreamingResponse(generate_tokens(), media_type="text/event-stream")

# ============================================
# 4. 健康检查
# ============================================

@app.get("/health")
async def health():
    """健康检查端点"""
    try:
        # 测试模型推理
        test_output = llm.generate(["test"], SamplingParams(max_tokens=1))
        
        return {
            "status": "healthy",
            "model": "llama-2-7b-chat",
            "gpu_available": torch.cuda.is_available(),
            "gpu_count": torch.cuda.device_count()
        }
    except Exception as e:
        raise HTTPException(status_code=503, detail=f"Service unhealthy: {str(e)}")

@app.get("/metrics")
async def metrics():
    """Prometheus metrics"""
    from prometheus_client import generate_latest
    return generate_latest()

# ============================================
# 5. API Key验证
# ============================================

API_KEYS = {
    "sk-test-key-1": {"user": "alice", "rate_limit": 1000},
    "sk-test-key-2": {"user": "bob", "rate_limit": 100}
}

def verify_api_key(api_key: str) -> bool:
    """验证API Key"""
    return api_key in API_KEYS

# ============================================
# 6. 启动服务
# ============================================

if __name__ == "__main__":
    import uvicorn
    
    uvicorn.run(
        app,
        host="0.0.0.0",
        port=8000,
        workers=1,  # FastAPI + vLLM只用1个worker
        log_level="info"
    )
```

---

### 3.2 Docker化部署

```dockerfile
# Dockerfile for FastAPI + vLLM

FROM nvidia/cuda:12.1.0-runtime-ubuntu22.04

# 安装Python
RUN apt-get update && apt-get install -y \
    python3.10 \
    python3-pip \
    && rm -rf /var/lib/apt/lists/*

# 安装依赖
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# requirements.txt内容：
# fastapi==0.109.0
# uvicorn[standard]==0.27.0
# vllm==0.3.0
# prometheus-client==0.19.0
# pydantic==2.5.0

# 复制代码
WORKDIR /app
COPY app.py .
COPY models/ ./models/

# 暴露端口
EXPOSE 8000

# 健康检查
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# 启动命令
CMD ["python3", "app.py"]
```

```bash
# 构建与运行

# 1. 构建Docker镜像
docker build -t llm-service:v1.0.0 .

# 2. 运行容器
docker run -d \
    --name llm-service \
    --gpus all \
    -p 8000:8000 \
    -e MODEL_NAME=llama-2-7b-chat \
    -v $(pwd)/models:/app/models \
    llm-service:v1.0.0

# 3. 测试
curl -X POST http://localhost:8000/v1/generate \
    -H "Content-Type: application/json" \
    -H "X-API-Key: sk-test-key-1" \
    -d '{
        "prompt": "Explain quantum computing:",
        "max_tokens": 100,
        "temperature": 0.7
    }'

# 4. 查看日志
docker logs -f llm-service

# 5. 查看GPU使用
docker exec llm-service nvidia-smi
```

---

<a name="triton-deployment"></a>
## 🏭 4. Triton Inference Server生产级部署

### 4.1 Triton Model Repository

```python
"""
Triton模型仓库结构
"""

# 目录结构
"""
model_repository/
├── llama2-7b-chat/
│   ├── config.pbtxt          # 模型配置
│   └── 1/                     # 版本1
│       ├── model.plan         # TensorRT引擎
│       └── config.json        # 模型元数据
├── preprocessing/
│   ├── config.pbtxt
│   └── 1/
│       └── model.py           # Python预处理逻辑
└── ensemble_model/
    └── config.pbtxt           # 组合模型配置
"""

# config.pbtxt for LLaMA model
llama_config = """
name: "llama2-7b-chat"
backend: "tensorrtllm"
max_batch_size: 256

# 输入
input [
  {
    name: "input_ids"
    data_type: TYPE_INT32
    dims: [-1]
  },
  {
    name: "input_lengths"
    data_type: TYPE_INT32
    dims: [1]
    reshape: { shape: [] }
  }
]

# 输出
output [
  {
    name: "output_ids"
    data_type: TYPE_INT32
    dims: [-1, -1]
  },
  {
    name: "sequence_length"
    data_type: TYPE_INT32
    dims: [-1]
  }
]

# 实例组（多GPU）
instance_group [
  {
    count: 1
    kind: KIND_GPU
    gpus: [0]
  }
]

# 🔥 动态Batching配置
dynamic_batching {
  preferred_batch_size: [16, 32, 64, 128]
  max_queue_delay_microseconds: 5000  # 最长等待5ms
  
  # 优先级队列
  priority_levels: 2
  default_priority_level: 1
  
  # Batching策略
  preserve_ordering: false  # 不保证顺序（提高吞吐）
}

# 模型热身
model_warmup [
  {
    name: "warmup_sample_1"
    batch_size: 1
    inputs: {
      key: "input_ids"
      value: {
        data_type: TYPE_INT32
        dims: [10]
        zero_data: true
      }
    }
  }
]

# 优化配置
optimization {
  cuda {
    graphs: true              # 启用CUDA Graph
    busy_wait_events: true
  }
  
  graph_optimization_level: 1
}
"""

with open('model_repository/llama2-7b-chat/config.pbtxt', 'w') as f:
    f.write(llama_config)

print("✅ Triton模型配置已生成")
```

---

### 4.2 Triton Server部署

```bash
# ============================================
# Triton Inference Server部署
# ============================================

# 1. 拉取Triton Docker镜像
docker pull nvcr.io/nvidia/tritonserver:25.06-trtllm-python-py3

# 2. 启动Triton服务器
docker run -d \
    --gpus all \
    --shm-size=8g \  # 共享内存（重要！）
    -p 8000:8000 \   # HTTP
    -p 8001:8001 \   # gRPC
    -p 8002:8002 \   # Metrics
    -v $(pwd)/model_repository:/models \
    nvcr.io/nvidia/tritonserver:25.06-trtllm-python-py3 \
    tritonserver \
        --model-repository=/models \
        --strict-model-config=false \
        --log-verbose=1

# 3. 查看服务状态
curl http://localhost:8000/v2/health/ready

# 输出：
# {
#   "ready": true
# }

# 4. 查看已加载模型
curl http://localhost:8000/v2/models

# 5. 获取模型配置
curl http://localhost:8000/v2/models/llama2-7b-chat/config

# ============================================
# Triton客户端调用
# ============================================

# Python客户端
import tritonclient.http as httpclient
import numpy as np

# 创建客户端
client = httpclient.InferenceServerClient(url="localhost:8000")

# 准备输入
prompt = "Explain quantum computing:"
input_ids = tokenizer.encode(prompt)

inputs = [
    httpclient.InferInput("input_ids", [1, len(input_ids)], "INT32"),
    httpclient.InferInput("input_lengths", [1], "INT32")
]

inputs[0].set_data_from_numpy(np.array([input_ids], dtype=np.int32))
inputs[1].set_data_from_numpy(np.array([len(input_ids)], dtype=np.int32))

# 推理
outputs = [
    httpclient.InferRequestedOutput("output_ids"),
    httpclient.InferRequestedOutput("sequence_length")
]

response = client.infer(
    model_name="llama2-7b-chat",
    inputs=inputs,
    outputs=outputs
)

# 解析输出
output_ids = response.as_numpy("output_ids")
generated_text = tokenizer.decode(output_ids[0])

print(f"Generated: {generated_text}")
```

---

### 4.3 Triton Ensemble Model

```python
"""
Triton Ensemble: 组合预处理+推理+后处理
"""

# ensemble_config.pbtxt
ensemble_config = """
name: "ensemble_llama2"
platform: "ensemble"
max_batch_size: 256

input [
  {
    name: "text_input"
    data_type: TYPE_STRING
    dims: [1]
  }
]

output [
  {
    name: "text_output"
    data_type: TYPE_STRING
    dims: [1]
  }
]

# 🔥 Ensemble步骤
ensemble_scheduling {
  step [
    {
      model_name: "preprocessing"
      model_version: -1
      input_map {
        key: "text_input"
        value: "text_input"
      }
      output_map {
        key: "input_ids"
        value: "tokenized_input"
      }
    },
    {
      model_name: "llama2-7b-chat"
      model_version: -1
      input_map {
        key: "input_ids"
        value: "tokenized_input"
      }
      output_map {
        key: "output_ids"
        value: "model_output"
      }
    },
    {
      model_name: "postprocessing"
      model_version: -1
      input_map {
        key: "output_ids"
        value: "model_output"
      }
      output_map {
        key: "text_output"
        value: "text_output"
      }
    }
  ]
}
"""

# 优势：
# ✅ 用户只需发送原始文本
# ✅ Triton内部处理预处理+推理+后处理
# ✅ 减少网络往返
# ✅ 统一的性能优化
```

---

<a name="load-balancing-autoscaling"></a>
## ⚖️ 5. 负载均衡与自动扩展

### 5.1 Kubernetes Deployment + HPA

```yaml
# ============================================
# llm-deployment.yaml
# Kubernetes部署配置
# ============================================

apiVersion: apps/v1
kind: Deployment
metadata:
  name: llm-inference
  labels:
    app: llm-service
    version: v1.0.0
spec:
  replicas: 3  # 初始副本数
  
  selector:
    matchLabels:
      app: llm-service
  
  template:
    metadata:
      labels:
        app: llm-service
        version: v1.0.0
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8000"
        prometheus.io/path: "/metrics"
    
    spec:
      # 节点选择
      nodeSelector:
        gpu-type: "a100"
      
      # 容忍
      tolerations:
        - key: "nvidia.com/gpu"
          operator: "Exists"
          effect: "NoSchedule"
      
      containers:
        - name: llm-container
          image: gcr.io/my-project/llm-service:v1.0.0
          imagePullPolicy: IfNotPresent
          
          # 资源请求
          resources:
            requests:
              nvidia.com/gpu: 1
              cpu: "8"
              memory: "32Gi"
            limits:
              nvidia.com/gpu: 1
              cpu: "16"
              memory: "48Gi"
          
          # 环境变量
          env:
            - name: MODEL_NAME
              value: "llama-2-7b-chat"
            - name: CUDA_VISIBLE_DEVICES
              value: "0"
            - name: NCCL_DEBUG
              value: "WARN"
          
          # 端口
          ports:
            - containerPort: 8000
              name: http
              protocol: TCP
          
          # 探针
          # 存活探针（Liveness Probe）
          livenessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 60
            periodSeconds: 30
            timeoutSeconds: 10
            failureThreshold: 3
          
          # 就绪探针（Readiness Probe）
          readinessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            successThreshold: 1
            failureThreshold: 3
          
          # 启动探针（Startup Probe）
          startupProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 0
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 30  # 最多等待300秒启动
          
          # 卷挂载
          volumeMounts:
            - name: model-cache
              mountPath: /root/.cache/huggingface
            - name: shm
              mountPath: /dev/shm
      
      # 卷
      volumes:
        - name: model-cache
          emptyDir: {}
        - name: shm
          emptyDir:
            medium: Memory
            sizeLimit: "8Gi"

---
# ============================================
# Service配置
# ============================================

apiVersion: v1
kind: Service
metadata:
  name: llm-service
  labels:
    app: llm-service
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 8000
      protocol: TCP
      name: http
  selector:
    app: llm-service
  
  # 🔥 会话亲和性（同一客户端路由到同一Pod）
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600

---
# ============================================
# HPA (Horizontal Pod Autoscaler)配置
# ============================================

apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: llm-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: llm-inference
  
  minReplicas: 2   # 最小副本
  maxReplicas: 10  # 最大副本
  
  # 🔥 基于多个指标扩展
  metrics:
    # 1. 基于请求队列长度（推荐）
    - type: Pods
      pods:
        metric:
          name: inference_queue_length
        target:
          type: AverageValue
          averageValue: "10"  # 队列长度>10时扩展
    
    # 2. 基于GPU利用率
    - type: Pods
      pods:
        metric:
          name: gpu_utilization_percent
        target:
          type: AverageValue
          averageValue: "80"  # GPU利用率>80%时扩展
    
    # 3. 基于请求延迟
    - type: Pods
      pods:
        metric:
          name: inference_latency_p95_seconds
        target:
          type: AverageValue
          averageValue: "2"  # P95延迟>2s时扩展
  
  # 扩展行为
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # 缩容稳定期5分钟
      policies:
        - type: Percent
          value: 50  # 每次最多缩容50%
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0  # 立即扩容
      policies:
        - type: Percent
          value: 100  # 每次最多扩容100%（翻倍）
          periodSeconds: 30
        - type: Pods
          value: 2  # 每次最少扩容2个Pod
          periodSeconds: 30
      selectPolicy: Max  # 使用最激进的策略
```

---

### 5.2 自定义Metrics Server

```python
"""
自定义HPA Metrics（队列长度）
"""

from prometheus_client import Gauge
import queue

# 定义metric
inference_queue_length = Gauge(
    'inference_queue_length',
    'Number of requests in the inference queue'
)

class InferenceQueue:
    """
    推理请求队列（用于batching和HPA）
    """
    
    def __init__(self, max_size=1000):
        self.queue = queue.Queue(maxsize=max_size)
    
    def add_request(self, request):
        """添加请求到队列"""
        self.queue.put(request)
        
        # 更新metric
        inference_queue_length.set(self.queue.qsize())
    
    def get_batch(self, max_batch_size=32, timeout=0.05):
        """
        从队列获取batch
        
        策略：
        - 等待timeout秒收集请求
        - 或达到max_batch_size立即返回
        """
        batch = []
        deadline = time.time() + timeout
        
        while len(batch) < max_batch_size and time.time() < deadline:
            try:
                request = self.queue.get(timeout=0.01)
                batch.append(request)
            except queue.Empty:
                if batch:  # 有部分请求，直接返回
                    break
        
        # 更新metric
        inference_queue_length.set(self.queue.qsize())
        
        return batch

# HPA会根据inference_queue_length自动扩缩容
```

---

<a name="dynamic-batching"></a>
## 📦 6. 推理优化：动态Batching

### 6.1 动态Batching实现

```python
"""
动态Batching推理服务
"""

import asyncio
from typing import List, Dict
import torch

class DynamicBatchingInferenceServer:
    """
    动态Batching推理服务
    
    核心思想：
    1. 收集多个请求
    2. 组成batch一次推理
    3. 提高GPU利用率
    """
    
    def __init__(self, model, max_batch_size=32, max_wait_ms=50):
        self.model = model
        self.max_batch_size = max_batch_size
        self.max_wait_ms = max_wait_ms
        
        # 请求队列
        self.request_queue = asyncio.Queue()
        
        # 启动batch处理任务
        asyncio.create_task(self.batch_processor())
    
    async def batch_processor(self):
        """
        后台任务：持续从队列获取请求并批处理
        """
        while True:
            # 收集一个batch的请求
            batch_requests = []
            deadline = asyncio.get_event_loop().time() + self.max_wait_ms / 1000
            
            while len(batch_requests) < self.max_batch_size:
                timeout = max(0, deadline - asyncio.get_event_loop().time())
                
                try:
                    request = await asyncio.wait_for(
                        self.request_queue.get(),
                        timeout=timeout
                    )
                    batch_requests.append(request)
                except asyncio.TimeoutError:
                    break
            
            if not batch_requests:
                await asyncio.sleep(0.01)
                continue
            
            # 批处理推理
            await self.process_batch(batch_requests)
    
    async def process_batch(self, requests: List[Dict]):
        """
        批处理推理
        """
        # 提取所有prompt
        prompts = [req['prompt'] for req in requests]
        
        # 批量推理
        start_time = time.time()
        
        outputs = self.model.generate(
            prompts,
            sampling_params=SamplingParams(
                temperature=0.7,
                max_tokens=256
            )
        )
        
        latency = time.time() - start_time
        
        # 将结果返回给各个请求
        for i, req in enumerate(requests):
            result = {
                'generated_text': outputs[i].outputs[0].text,
                'latency_ms': latency * 1000,
                'batch_size': len(requests)
            }
            
            # 通过Future返回结果
            req['future'].set_result(result)
    
    async def generate(self, prompt: str) -> Dict:
        """
        异步生成接口
        """
        # 创建Future
        future = asyncio.Future()
        
        # 将请求加入队列
        await self.request_queue.put({
            'prompt': prompt,
            'future': future
        })
        
        # 等待结果
        result = await future
        return result

# ============================================
# FastAPI集成动态Batching
# ============================================

from fastapi import FastAPI

app = FastAPI()

# 初始化推理服务器
inference_server = DynamicBatchingInferenceServer(
    model=llm,
    max_batch_size=32,
    max_wait_ms=50
)

@app.post("/generate")
async def generate_endpoint(request: GenerationRequest):
    """
    使用动态batching的生成端点
    """
    result = await inference_server.generate(request.prompt)
    return result

# 效果对比：
"""
无Batching:
  - 每个请求单独推理
  - GPU利用率: 20-30%
  - 吞吐量: 5 QPS
  - 延迟: 100ms

有动态Batching (batch=32):
  - 请求组batch推理
  - GPU利用率: 80-90%
  - 吞吐量: 60 QPS (12x提升！)
  - 延迟: 150ms (轻微增加，可接受)
"""
```

---

<a name="security-auth"></a>
## 🔒 7. API安全与认证

### 7.1 多层安全架构

```python
"""
生产级API安全实现
"""

from fastapi import FastAPI, Depends, HTTPException, Security
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from jose import JWTError, jwt
from passlib.context import CryptContext
from datetime import datetime, timedelta
import secrets

app = FastAPI()

# ============================================
# 1. JWT认证
# ============================================

# JWT配置
SECRET_KEY = secrets.token_urlsafe(32)  # 生产环境用环境变量
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 60

# 密码加密
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# 用户数据库（生产环境用真实数据库）
users_db = {
    "alice@example.com": {
        "username": "alice",
        "email": "alice@example.com",
        "hashed_password": pwd_context.hash("secret123"),
        "disabled": False,
        "role": "admin",
        "rate_limit": 10000  # 每小时请求限制
    }
}

# JWT Bearer scheme
security = HTTPBearer()

def create_access_token(data: dict, expires_delta: timedelta = None):
    """
    创建JWT token
    """
    to_encode = data.copy()
    
    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(minutes=15)
    
    to_encode.update({"exp": expire})
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    
    return encoded_jwt

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Security(security)
):
    """
    从JWT获取当前用户
    """
    token = credentials.credentials
    
    try:
        # 解码JWT
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username: str = payload.get("sub")
        
        if username is None:
            raise HTTPException(status_code=401, detail="Invalid token")
        
    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid token")
    
    # 查询用户
    user = users_db.get(username)
    if user is None:
        raise HTTPException(status_code=401, detail="User not found")
    
    return user

# ============================================
# 2. 登录端点
# ============================================

from pydantic import BaseModel

class LoginRequest(BaseModel):
    username: str
    password: str

@app.post("/auth/login")
async def login(request: LoginRequest):
    """
    用户登录获取token
    """
    # 验证用户
    user = users_db.get(request.username)
    
    if not user or not pwd_context.verify(request.password, user['hashed_password']):
        raise HTTPException(status_code=401, detail="Incorrect username or password")
    
    if user['disabled']:
        raise HTTPException(status_code=401, detail="User disabled")
    
    # 创建token
    access_token_expires = timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    access_token = create_access_token(
        data={"sub": request.username, "role": user['role']},
        expires_delta=access_token_expires
    )
    
    return {
        "access_token": access_token,
        "token_type": "bearer",
        "expires_in": ACCESS_TOKEN_EXPIRE_MINUTES * 60
    }

# ============================================
# 3. 受保护的API端点
# ============================================

@app.post("/v1/generate")
async def generate(
    request: GenerationRequest,
    current_user: dict = Depends(get_current_user)
):
    """
    受JWT保护的生成端点
    """
    # 检查用户权限
    if current_user['role'] not in ['admin', 'user']:
        raise HTTPException(status_code=403, detail="Insufficient permissions")
    
    # 检查速率限制（后续实现）
    check_rate_limit(current_user['username'], current_user['rate_limit'])
    
    # 推理...
    result = inference_server.generate(request.prompt)
    
    # 记录审计日志
    log_api_usage(
        user=current_user['username'],
        endpoint='/v1/generate',
        tokens_used=result['tokens_generated']
    )
    
    return result

# ============================================
# 4. API Key方式（更简单）
# ============================================

from fastapi.security.api_key import APIKeyHeader

API_KEY_NAME = "X-API-Key"
api_key_header = APIKeyHeader(name=API_KEY_NAME, auto_error=False)

# API Keys数据库
api_keys_db = {
    "sk-proj-abc123xyz": {
        "user": "project-a",
        "rate_limit": 1000,
        "enabled": True
    }
}

async def get_api_key(api_key: str = Security(api_key_header)):
    """
    验证API Key
    """
    if api_key not in api_keys_db:
        raise HTTPException(
            status_code=401,
            detail="Invalid API Key"
        )
    
    key_info = api_keys_db[api_key]
    
    if not key_info['enabled']:
        raise HTTPException(
            status_code=401,
            detail="API Key disabled"
        )
    
    return key_info

@app.post("/v1/generate-simple")
async def generate_simple(
    request: GenerationRequest,
    key_info: dict = Depends(get_api_key)
):
    """
    基于API Key的简单认证
    """
    # 检查速率限制
    check_rate_limit(key_info['user'], key_info['rate_limit'])
    
    # 推理...
    return result
```

---

<a name="rate-limiting-circuit-breaker"></a>
## 🚦 8. 限流与熔断机制

### 8.1 Token Bucket限流

```python
"""
Token Bucket限流算法
"""

import time
from threading import Lock

class TokenBucketRateLimiter:
    """
    Token Bucket限流器
    
    原理：
    - 令牌以固定速率生成
    - 请求消耗令牌
    - 没有令牌则拒绝请求
    """
    
    def __init__(self, rate_per_second=10, burst=20):
        """
        参数:
            rate_per_second: 每秒生成的令牌数
            burst: 桶容量（允许的突发请求数）
        """
        self.rate = rate_per_second
        self.capacity = burst
        self.tokens = burst  # 初始满桶
        self.last_update = time.time()
        self.lock = Lock()
    
    def allow_request(self, tokens_needed=1):
        """
        检查是否允许请求
        
        返回：(allowed, wait_time)
        """
        with self.lock:
            now = time.time()
            elapsed = now - self.last_update
            
            # 生成新令牌
            self.tokens = min(
                self.capacity,
                self.tokens + elapsed * self.rate
            )
            self.last_update = now
            
            if self.tokens >= tokens_needed:
                # 消耗令牌
                self.tokens -= tokens_needed
                return True, 0
            else:
                # 计算需要等待的时间
                needed = tokens_needed - self.tokens
                wait_time = needed / self.rate
                return False, wait_time
    
    def wait_for_token(self, tokens_needed=1, timeout=None):
        """
        等待令牌（阻塞）
        """
        start = time.time()
        
        while True:
            allowed, wait_time = self.allow_request(tokens_needed)
            
            if allowed:
                return True
            
            if timeout and (time.time() - start) > timeout:
                return False
            
            time.sleep(min(wait_time, 0.1))

# ============================================
# FastAPI集成限流
# ============================================

from fastapi import FastAPI, Request, HTTPException
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

# 初始化限流器
limiter = Limiter(key_func=get_remote_address)
app = FastAPI()
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

# 按IP限流
@app.post("/v1/generate")
@limiter.limit("10/minute")  # 每分钟10次
async def generate(request: Request, gen_request: GenerationRequest):
    """限流的生成端点"""
    result = llm.generate(gen_request.prompt)
    return result

# 按API Key限流（更精细）
user_limiters = {}

def get_user_limiter(user_id: str, rate_limit: int):
    """
    获取用户专属限流器
    """
    if user_id not in user_limiters:
        user_limiters[user_id] = TokenBucketRateLimiter(
            rate_per_second=rate_limit / 3600,  # 转换为每秒
            burst=rate_limit // 10  # 允许10%突发
        )
    return user_limiters[user_id]

@app.post("/v1/generate-with-key")
async def generate_with_key(
    request: GenerationRequest,
    user_info: dict = Depends(get_api_key)
):
    """
    按用户限流
    """
    user_id = user_info['user']
    rate_limit = user_info['rate_limit']
    
    # 获取限流器
    limiter = get_user_limiter(user_id, rate_limit)
    
    # 检查是否允许
    allowed, wait_time = limiter.allow_request()
    
    if not allowed:
        raise HTTPException(
            status_code=429,
            detail=f"Rate limit exceeded. Retry after {wait_time:.2f}s",
            headers={"Retry-After": str(int(wait_time))}
        )
    
    # 推理
    result = llm.generate(request.prompt)
    return result
```

---

### 8.2 Circuit Breaker实现

```python
"""
熔断器实现
"""

from enum import Enum
import time
from collections import deque

class CircuitState(Enum):
    """熔断器状态"""
    CLOSED = "closed"      # 正常
    OPEN = "open"          # 熔断（拒绝请求）
    HALF_OPEN = "half_open"  # 半开（试探恢复）

class CircuitBreaker:
    """
    熔断器
    
    防止雪崩：当下游服务故障时，快速失败而非等待超时
    """
    
    def __init__(
        self,
        failure_threshold=5,      # 失败5次触发熔断
        success_threshold=2,      # 成功2次恢复
        timeout_seconds=60,       # 熔断60秒后进入半开状态
        window_size=100           # 滑动窗口大小
    ):
        self.failure_threshold = failure_threshold
        self.success_threshold = success_threshold
        self.timeout_seconds = timeout_seconds
        self.window_size = window_size
        
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.success_count = 0
        self.last_failure_time = None
        
        # 滑动窗口记录最近的请求结果
        self.recent_results = deque(maxlen=window_size)
    
    def call(self, func, *args, **kwargs):
        """
        通过熔断器调用函数
        """
        # 检查熔断状态
        if self.state == CircuitState.OPEN:
            # 检查是否超时（可以尝试恢复）
            if time.time() - self.last_failure_time > self.timeout_seconds:
                self.state = CircuitState.HALF_OPEN
                self.success_count = 0
                print("🔄 熔断器进入半开状态，尝试恢复")
            else:
                raise HTTPException(
                    status_code=503,
                    detail="Service circuit breaker is OPEN"
                )
        
        # 调用函数
        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        
        except Exception as e:
            self._on_failure()
            raise e
    
    def _on_success(self):
        """成功回调"""
        self.recent_results.append(True)
        
        if self.state == CircuitState.HALF_OPEN:
            self.success_count += 1
            
            # 连续成功，恢复到CLOSED
            if self.success_count >= self.success_threshold:
                self.state = CircuitState.CLOSED
                self.failure_count = 0
                print("✅ 熔断器已关闭，服务恢复正常")
    
    def _on_failure(self):
        """失败回调"""
        self.recent_results.append(False)
        self.failure_count += 1
        self.last_failure_time = time.time()
        
        # 计算失败率
        if len(self.recent_results) >= 10:
            recent_failures = sum(1 for r in list(self.recent_results)[-10:] if not r)
            failure_rate = recent_failures / 10
            
            # 触发熔断
            if failure_rate > 0.5 or self.failure_count >= self.failure_threshold:
                self.state = CircuitState.OPEN
                print(f"❌ 熔断器已打开（失败率: {failure_rate:.0%}）")
        
        # 半开状态下失败，立即回到OPEN
        if self.state == CircuitState.HALF_OPEN:
            self.state = CircuitState.OPEN
            print("❌ 半开状态失败，熔断器重新打开")

# ============================================
# FastAPI集成熔断器
# ============================================

# 创建下游服务的熔断器
model_circuit_breaker = CircuitBreaker(
    failure_threshold=5,
    timeout_seconds=30
)

@app.post("/v1/generate-protected")
async def generate_with_circuit_breaker(request: GenerationRequest):
    """
    带熔断保护的生成端点
    """
    try:
        # 通过熔断器调用推理
        result = model_circuit_breaker.call(
            llm.generate,
            request.prompt,
            sampling_params=SamplingParams(max_tokens=request.max_tokens)
        )
        
        return {"generated_text": result[0].outputs[0].text}
    
    except HTTPException:
        # 熔断器打开，返回降级响应
        return {
            "generated_text": "Service temporarily unavailable. Please try again later.",
            "fallback": True
        }

# 查看熔断器状态
@app.get("/circuit-breaker/status")
async def circuit_breaker_status():
    """
    熔断器状态查询
    """
    return {
        "state": model_circuit_breaker.state.value,
        "failure_count": model_circuit_breaker.failure_count,
        "success_count": model_circuit_breaker.success_count,
        "recent_failure_rate": (
            sum(1 for r in model_circuit_breaker.recent_results if not r) /
            len(model_circuit_breaker.recent_results)
            if model_circuit_breaker.recent_results else 0
        )
    }
```

---

<a name="logging-tracing"></a>
## 📝 9. 日志与分布式追踪

### 9.1 结构化日志

```python
"""
结构化日志实现
"""

import logging
import json
from datetime import datetime
import uuid

class StructuredLogger:
    """
    结构化日志
    
    格式：JSON (便于ELK/Loki解析)
    """
    
    def __init__(self, service_name="llm-inference"):
        self.service_name = service_name
        self.logger = logging.getLogger(service_name)
        self.logger.setLevel(logging.INFO)
        
        # JSON格式handler
        handler = logging.StreamHandler()
        handler.setFormatter(self.JSONFormatter())
        self.logger.addHandler(handler)
    
    class JSONFormatter(logging.Formatter):
        """JSON格式化器"""
        def format(self, record):
            log_data = {
                "timestamp": datetime.utcnow().isoformat(),
                "level": record.levelname,
                "logger": record.name,
                "message": record.getMessage(),
                "module": record.module,
                "function": record.funcName,
                "line": record.lineno,
            }
            
            # 添加额外字段
            if hasattr(record, 'request_id'):
                log_data['request_id'] = record.request_id
            if hasattr(record, 'user_id'):
                log_data['user_id'] = record.user_id
            if hasattr(record, 'latency_ms'):
                log_data['latency_ms'] = record.latency_ms
            
            return json.dumps(log_data)
    
    def log_request(
        self,
        request_id: str,
        user_id: str,
        endpoint: str,
        prompt_length: int
    ):
        """记录请求"""
        self.logger.info(
            f"Request received: {endpoint}",
            extra={
                'request_id': request_id,
                'user_id': user_id,
                'prompt_length': prompt_length
            }
        )
    
    def log_response(
        self,
        request_id: str,
        status_code: int,
        latency_ms: float,
        tokens_generated: int
    ):
        """记录响应"""
        self.logger.info(
            f"Response sent: status={status_code}",
            extra={
                'request_id': request_id,
                'latency_ms': latency_ms,
                'tokens_generated': tokens_generated
            }
        )

# FastAPI中间件集成
from fastapi import Request
from starlette.middleware.base import BaseHTTPMiddleware

logger = StructuredLogger()

class LoggingMiddleware(BaseHTTPMiddleware):
    """
    日志中间件
    """
    
    async def dispatch(self, request: Request, call_next):
        # 生成request_id
        request_id = str(uuid.uuid4())
        request.state.request_id = request_id
        
        # 记录请求
        logger.log_request(
            request_id=request_id,
            user_id=request.headers.get('X-User-ID', 'unknown'),
            endpoint=request.url.path,
            prompt_length=len(await request.body())
        )
        
        # 处理请求
        start_time = time.time()
        response = await call_next(request)
        latency_ms = (time.time() - start_time) * 1000
        
        # 记录响应
        logger.log_response(
            request_id=request_id,
            status_code=response.status_code,
            latency_ms=latency_ms,
            tokens_generated=0  # 从response提取
        )
        
        # 添加request_id到响应头
        response.headers['X-Request-ID'] = request_id
        
        return response

app.add_middleware(LoggingMiddleware)
```

---

### 9.2 分布式追踪（Jaeger）

根据 [Jaeger v2文档](https://www.jaegertracing.io/docs/latest/)：

```python
"""
OpenTelemetry + Jaeger分布式追踪
"""

from opentelemetry import trace
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor

# ============================================
# 1. 初始化OpenTelemetry
# ============================================

# 配置Jaeger exporter
jaeger_exporter = JaegerExporter(
    agent_host_name="jaeger-agent",  # Jaeger Agent地址
    agent_port=6831,
)

# 配置Trace Provider
trace.set_tracer_provider(TracerProvider())
tracer = trace.get_tracer(__name__)

# 添加Span Processor
span_processor = BatchSpanProcessor(jaeger_exporter)
trace.get_tracer_provider().add_span_processor(span_processor)

# ============================================
# 2. 自动instrumentation FastAPI
# ============================================

FastAPIInstrumentor.instrument_app(app)
RequestsInstrumentor().instrument()

# ============================================
# 3. 手动添加Span
# ============================================

@app.post("/v1/generate-traced")
async def generate_with_tracing(request: GenerationRequest):
    """
    带分布式追踪的生成端点
    """
    with tracer.start_as_current_span("generate_request") as span:
        # 添加span属性
        span.set_attribute("user.id", "alice")
        span.set_attribute("prompt.length", len(request.prompt))
        span.set_attribute("max_tokens", request.max_tokens)
        
        # 子span: 预处理
        with tracer.start_as_current_span("preprocess"):
            input_ids = tokenizer.encode(request.prompt)
            span.set_attribute("input_ids.length", len(input_ids))
        
        # 子span: 模型推理
        with tracer.start_as_current_span("model_inference"):
            start = time.time()
            outputs = llm.generate([request.prompt])
            inference_time = time.time() - start
            
            span.set_attribute("inference.latency_ms", inference_time * 1000)
            span.set_attribute("inference.tokens_generated", 
                              len(outputs[0].outputs[0].token_ids))
        
        # 子span: 后处理
        with tracer.start_as_current_span("postprocess"):
            generated_text = outputs[0].outputs[0].text
        
        # 记录总体指标
        span.set_attribute("tokens.generated", len(outputs[0].outputs[0].token_ids))
        
        return {
            "generated_text": generated_text,
            "trace_id": span.get_span_context().trace_id
        }

# Jaeger UI查看：
# http://localhost:16686
# 可以看到完整的请求trace：
# generate_request (100ms)
#   ├─ preprocess (5ms)
#   ├─ model_inference (90ms)
#   └─ postprocess (5ms)
```

---

<a name="disaster-recovery"></a>
## 🛡️ 10. 灾备与高可用

### 10.1 高可用架构

```python
"""
高可用架构设计
"""

class HighAvailabilityArchitecture:
    """
    高可用部署架构
    """
    
    def design_ha_architecture(self):
        """
        设计HA架构
        """
        print("""
高可用架构 (Multi-Region):

┌────────────────────────────────────────────────────────────┐
│  Global Load Balancer (CloudFlare/AWS Route53)             │
│  - Geo-routing (就近访问)                                   │
│  - Health check (故障自动切换)                              │
└─────────────┬────────────────┬─────────────────────────────┘
              ↓                ↓
    ┌─────────────────┐  ┌─────────────────┐
    │  Region 1 (主)  │  │  Region 2 (备)  │
    │  us-west-2      │  │  us-east-1      │
    └─────┬───────────┘  └─────┬───────────┘
          ↓                    ↓
┌──────────────────┐    ┌──────────────────┐
│  K8s Cluster 1   │    │  K8s Cluster 2   │
│  ┌────────────┐  │    │  ┌────────────┐  │
│  │Deployment  │  │    │  │Deployment  │  │
│  │replicas: 5 │  │    │  │replicas: 3 │  │
│  └────────────┘  │    │  └────────────┘  │
│  ┌────────────┐  │    │  ┌────────────┐  │
│  │   Ingress  │  │    │  │   Ingress  │  │
│  └────────────┘  │    │  └────────────┘  │
└──────────────────┘    └──────────────────┘
          ↓                    ↓
┌──────────────────┐    ┌──────────────────┐
│  Model Storage   │←───│  Model Storage   │
│  (S3主区域)      │同步│  (S3副本)        │
└──────────────────┘    └──────────────────┘
        """)
        
        print("\n高可用目标 (SLA):")
        print("  可用性: 99.95% (每月停机≤21分钟)")
        print("  RPO (恢复点目标): <5分钟")
        print("  RTO (恢复时间目标): <30分钟")
    
    def calculate_availability(self):
        """
        计算可用性
        """
        # 单实例可用性
        single_instance_availability = 0.99  # 99%
        
        # N个独立实例的可用性
        # Availability = 1 - (1 - single)^N
        
        replicas = [1, 2, 3, 5, 10]
        
        print("\n副本数与可用性:")
        print("="*70)
        for n in replicas:
            availability = 1 - (1 - single_instance_availability) ** n
            downtime_minutes = (1 - availability) * 365 * 24 * 60
            
            print(f"  {n}副本: {availability:.5f} ({100*availability:.3f}%) "
                  f"→ 年停机 {downtime_minutes:.1f} 分钟")

ha = HighAvailabilityArchitecture()
ha.design_ha_architecture()
ha.calculate_availability()
```

---

### 10.2 故障转移实现

```yaml
# ============================================
# Kubernetes多区域部署
# ============================================

# region1-deployment.yaml (主区域)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: llm-inference-primary
  namespace: production
spec:
  replicas: 5
  selector:
    matchLabels:
      app: llm-service
      region: us-west-2
  template:
    spec:
      affinity:
        # Pod反亲和性（分散到不同节点）
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: llm-service
              topologyKey: kubernetes.io/hostname
      
      containers:
        - name: llm
          image: gcr.io/my-project/llm-service:v1.0.0
          resources:
            requests:
              nvidia.com/gpu: 1

---
# region2-deployment.yaml (备用区域)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: llm-inference-secondary
  namespace: production
spec:
  replicas: 3  # 备用区域副本少一些
  # ... 其他配置同主区域

---
# ============================================
# Ingress with Failover
# ============================================

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: llm-ingress
  annotations:
    # Nginx Ingress配置
    nginx.ingress.kubernetes.io/upstream-max-fails: "3"
    nginx.ingress.kubernetes.io/upstream-fail-timeout: "30s"
    
    # 跨区域故障转移
    nginx.ingress.kubernetes.io/server-snippet: |
      location / {
        proxy_next_upstream error timeout http_503;
        proxy_next_upstream_tries: 3;
      }
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: llm-service
                port:
                  number: 80
```

---

<a name="cost-monitoring"></a>
## 💰 11. 成本监控与优化

### 11.1 推理成本追踪

```python
"""
推理成本实时监控
"""

from prometheus_client import Counter, Histogram
import datetime

# Prometheus指标
tokens_processed_total = Counter(
    'llm_tokens_processed_total',
    'Total tokens processed',
    ['user_id', 'model']
)

inference_cost_dollars = Counter(
    'llm_inference_cost_dollars_total',
    'Total inference cost in dollars',
    ['user_id', 'model']
)

class CostTracker:
    """
    成本追踪器
    """
    
    def __init__(self):
        # 定价（每百万token，美元）
        self.pricing = {
            'llama-2-7b': {
                'input': 0.20,
                'output': 0.30
            },
            'llama-2-70b': {
                'input': 1.00,
                'output': 1.50
            }
        }
        
        # GPU成本（每小时）
        self.gpu_cost_per_hour = {
            'A100-40GB': 2.50,
            'H100-80GB': 5.00
        }
    
    def track_inference_cost(
        self,
        user_id: str,
        model_name: str,
        input_tokens: int,
        output_tokens: int
    ):
        """
        追踪推理成本
        """
        # 计算token成本
        input_cost = (input_tokens / 1_000_000) * self.pricing[model_name]['input']
        output_cost = (output_tokens / 1_000_000) * self.pricing[model_name]['output']
        total_cost = input_cost + output_cost
        
        # 更新Prometheus指标
        tokens_processed_total.labels(
            user_id=user_id,
            model=model_name
        ).inc(input_tokens + output_tokens)
        
        inference_cost_dollars.labels(
            user_id=user_id,
            model=model_name
        ).inc(total_cost)
        
        # 存储到数据库（用于账单）
        self.save_to_db({
            'timestamp': datetime.datetime.utcnow(),
            'user_id': user_id,
            'model': model_name,
            'input_tokens': input_tokens,
            'output_tokens': output_tokens,
            'cost_usd': total_cost
        })
        
        return total_cost
    
    def generate_user_bill(self, user_id: str, month: str):
        """
        生成用户账单
        """
        # 从数据库查询该用户该月的所有请求
        records = self.query_db(user_id=user_id, month=month)
        
        # 汇总
        total_input_tokens = sum(r['input_tokens'] for r in records)
        total_output_tokens = sum(r['output_tokens'] for r in records)
        total_cost = sum(r['cost_usd'] for r in records)
        
        bill = {
            'user_id': user_id,
            'month': month,
            'total_requests': len(records),
            'total_input_tokens': total_input_tokens,
            'total_output_tokens': total_output_tokens,
            'total_cost_usd': total_cost,
            'breakdown_by_model': self.breakdown_by_model(records)
        }
        
        return bill

# FastAPI集成
cost_tracker = CostTracker()

@app.post("/v1/generate-with-billing")
async def generate_with_billing(
    request: GenerationRequest,
    user_info: dict = Depends(get_current_user)
):
    """
    带成本追踪的生成端点
    """
    # 推理
    result = llm.generate(request.prompt)
    
    # 计算token数
    input_tokens = len(tokenizer.encode(request.prompt))
    output_tokens = result.tokens_generated
    
    # 追踪成本
    cost = cost_tracker.track_inference_cost(
        user_id=user_info['username'],
        model_name='llama-2-7b',
        input_tokens=input_tokens,
        output_tokens=output_tokens
    )
    
    return {
        **result,
        'usage': {
            'input_tokens': input_tokens,
            'output_tokens': output_tokens,
            'total_tokens': input_tokens + output_tokens,
            'cost_usd': cost
        }
    }
```

---

<a name="complete-deployment"></a>
## 🎯 12. 完整部署案例

### 12.1 生产级部署清单

```python
"""
生产部署清单
"""

production_deployment_checklist = {
    '✅ 基础设施': [
        'K8s集群搭建（多节点）',
        'GPU Operator安装',
        '高性能存储（Lustre/NFS）',
        'Ingress Controller（Nginx/Traefik）',
        'DNS配置'
    ],
    
    '✅ 服务化': [
        '选择服务框架（FastAPI/Triton）',
        'Docker镜像构建',
        'Model Registry集成',
        '健康检查端点',
        'Metrics暴露'
    ],
    
    '✅ 可靠性': [
        'HPA自动扩展',
        '多副本部署（≥3）',
        'PodDisruptionBudget配置',
        'Pod反亲和性',
        '多区域部署'
    ],
    
    '✅ 性能': [
        '动态Batching',
        '模型量化（FP8/INT4）',
        'KV Cache优化',
        'GPU利用率>80%',
        'P95延迟<2s'
    ],
    
    '✅ 安全': [
        'JWT/API Key认证',
        'HTTPS (TLS证书)',
        '限流（Rate Limiting）',
        '熔断（Circuit Breaker）',
        '输入验证'
    ],
    
    '✅ 可观测性': [
        'Prometheus监控',
        'Grafana Dashboard',
        'Jaeger分布式追踪',
        'ELK/Loki日志聚合',
        '告警规则配置'
    ],
    
    '✅ 成本': [
        'Token使用量追踪',
        'GPU利用率监控',
        '成本告警',
        '用户计费',
        'Reserved实例优化'
    ],
    
    '✅ 流程': [
        'CI/CD Pipeline',
        'Staging环境',
        'A/B测试',
        '金丝雀发布',
        '回滚机制'
    ]
}

for category, items in production_deployment_checklist.items():
    print(f"\n{category}")
    for item in items:
        print(f"  □ {item}")

print("\n\n🔥 生产部署最小标准：")
print("  - 至少3个副本")
print("  - HPA自动扩展")
print("  - 健康检查+自动重启")
print("  - Prometheus监控")
print("  - API认证")
print("  - 限流机制")
```

---

## 📚 参考资料

### 服务框架

1. **NVIDIA Triton**  
   [Triton Inference Server](https://docs.nvidia.com/deeplearning/triton-inference-server/)  
   生产级推理服务器（v25.06）

2. **FastAPI**  
   [FastAPI Documentation](https://fastapi.tiangolo.com/)  
   现代Python Web框架

3. **TorchServe**  
   [TorchServe GitHub](https://github.com/pytorch/serve)  
   PyTorch官方服务框架

### Kubernetes

4. **K8s HPA Autoscaling**  
   [GKE GPU Inference Autoscaling](https://cloud.google.com/kubernetes-engine/docs/how-to/machine-learning/inference/autoscaling)  
   GPU推理自动扩展（2025）

5. **GPU Operator**  
   [NVIDIA GPU Operator](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/)  
   K8s GPU管理（v25.10.1）

### 可观测性

6. **Prometheus**  
   [Prometheus Documentation](https://prometheus.io/docs/)  
   监控系统

7. **Jaeger**  
   [Jaeger Tracing](https://www.jaegertracing.io/docs/latest/)  
   分布式追踪（v2，2024）

8. **OpenTelemetry**  
   [OpenTelemetry Python](https://opentelemetry.io/docs/instrumentation/python/)  
   可观测性标准

### 安全与治理

9. **API Rate Limiting**  
   [Rate Limiting Best Practices](https://medium.com/@gynanrudr0/protecting-apis-with-rate-limiting-throttling-and-circuit-breakers-f6570c065c1c)  
   限流与熔断

10. **Circuit Breaker Pattern**  
    [API Circuit Breaker](https://www.unkey.com/glossary/api-circuit-breaker)  
    熔断器模式

---

## 🎯 总结

### 关键要点回顾

1. **框架选择**：FastAPI（原型）、Triton（生产）、TorchServe（PyTorch）
2. **性能优化**：动态Batching提升GPU利用率至80%+
3. **高可用**：≥3副本 + HPA + 多区域
4. **安全**：JWT认证 + 限流 + 熔断
5. **可观测性**：Prometheus + Jaeger + 结构化日志
6. **成本**：Token级别追踪与计费

### 生产部署性能目标

| 指标 | 目标值 | 说明 |
|------|--------|------|
| **可用性** | 99.95% | 每月≤21分钟停机 |
| **P95延迟** | <2s | 95%请求<2秒 |
| **GPU利用率** | >80% | 充分利用资源 |
| **错误率** | <0.1% | 高质量服务 |
| **成本/百万token** | <$1 | 成本优化 |

### 传统程序员的优势

- ✅ **微服务经验**：API设计、负载均衡、熔断限流是已有技能
- ✅ **DevOps能力**：K8s、Docker、CI/CD直接迁移
- ✅ **监控思维**：Prometheus、Grafana、日志分析是传统技能
- ✅ **成本意识**：资源优化、容量规划是工程师基本功

### 下一步学习

- 🔗 **下一篇**：[14 - 职业分工详解：各细分方向的能力要求与发展路径](./14-career-paths.md)
- 💻 **动手实践**：部署一个FastAPI服务到K8s
- 📊 **实战项目**：配置完整的监控告警系统

---

> 💡 **璇玑的小贴士**：生产化部署就像开餐厅——不仅要菜做得好（模型训练），还要服务快（推理优化）、不出错（高可用）、账算得清（成本监控）。传统程序员的全栈能力在这里是巨大优势！✨
>
> 道友现在对生产部署有感觉了吗？工程篇到这里就结束啦！接下来是职业篇，我们聊聊LLM训练的各个职业方向和发展路径~ 🚀