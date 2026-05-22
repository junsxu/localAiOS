# 阶段 6：性能层 · 分层路由

> **核心目标**：发挥多设备架构的独有优势
>
> 简单问题 → 小模型（快速响应），复杂问题 → 大模型（高质量输出）。成本与性能最优化。

---

## 知识体系

| 主题 | 详细文档 |
|------|---------|
| 模型路由器实现（规则/LLM/自适应） | [router.md](./router.md) |
| 性能基准测试与监控 | [benchmarks.md](./benchmarks.md) |

---

## 核心思想：模型路由器

```
用户请求
    │
    ▼
┌─────────────────────┐
│  Router（路由器）    │
│  · 分析请求复杂度    │
│  · 选择最优模型      │
└──────────┬──────────┘
           │
     ┌─────┴───────┐
     │             │
  简单问题      复杂问题
     │             │
     ▼             ▼
 小模型         大模型
 3B/7B          14B/32B
 Windows核显    WSL独显
 < 1s响应       高质量输出
```

---

## 硬件资源映射

| 设备 | 资源 | 承载模型 | 典型延迟 |
|------|------|----------|----------|
| Windows（核显）| Intel iGPU | 3B / 7B Embedding | 1~3s |
| WSL（独显）| NVIDIA GPU | 14B / 32B 推理 | 3~10s |
| NAS | 存储 | 向量库冷数据 | N/A |

> Mac Studio 256GB 架构下可同时常驻 7B/14B/32B，详见 [benchmarks.md](./benchmarks.md)。

---

## 路由策略概览

| 策略 | 延迟开销 | 准确度 | 适用场景 |
|------|----------|--------|----------|
| 基于规则 | ~0ms | 中 | 明确场景（翻译、总结） |
| 基于 LLM 分类 | ~200ms | 高 | 通用场景（推荐） |
| 自适应（负载感知）| ~0ms | 中 | 高并发场景 |

> 三种策略的完整实现见 [router.md](./router.md)。

---

## 实操任务

### 任务 1：部署路由服务

```bash
cd services/router
pip install fastapi uvicorn requests
uvicorn main:app --host 0.0.0.0 --port 8000
```

- [ ] `POST http://localhost:8000/chat` 可以根据问题复杂度自动选择模型

### 任务 2：验证路由决策

```bash
# 简单问题 → 应该路由到小模型
curl -X POST http://localhost:8000/chat \
  -H "Content-Type: application/json" \
  -d '{"prompt": "翻译：Hello World"}'

# 复杂问题 → 应该路由到大模型
curl -X POST http://localhost:8000/chat \
  -H "Content-Type: application/json" \
  -d '{"prompt": "请详细分析 Transformer 架构中 Attention 机制的数学原理，并比较 MHA 和 GQA 的差异"}'
```

- [ ] 返回结果中 `complexity` 字段正确反映路由决策

### 任务 3：性能基准测试

```bash
# 运行基准测试脚本
python benchmarks/run_benchmark.py --model qwen2.5:7b --prompts 10
python benchmarks/run_benchmark.py --model qwen2.5:14b --prompts 10
```

> 完整基准测试方法见 [benchmarks.md](./benchmarks.md)。

- [ ] 7B 模型平均响应 < 3s，14B 模型平均响应 < 10s

### 任务 4：高负载下的自动降级

```bash
# 模拟高并发（10 个并发请求）
python benchmarks/load_test.py --concurrency 10 --requests 50
```

- [ ] 高负载时路由自动降级到小模型，不超时

---

## 验收标准

- [ ] ✅ 简单问题 < 1s 响应（小模型）
- [ ] ✅ 复杂问题路由到大模型，响应质量更高
- [ ] ✅ 路由分类本身消耗 < 200ms
- [ ] ✅ 高负载时自动降级，不超时
- [ ] ✅ `/stats` 端点可监控两侧延迟

---

> ⬅️ [上一阶段：系统编排](../05-orchestration/README.md) | ➡️ [终态：AI OS](../07-aios/README.md)
