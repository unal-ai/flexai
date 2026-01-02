# Flex:AI 技术验真报告

**审计日期**: 2026-01-02  
**代码库**: unal-ai/flexai  
**审计方法**: 静态代码分析 + 架构对照验证

---

## 执行摘要

| 维度 | 得分 (0-10) | 评价 |
|------|-------------|------|
| 1. 虚拟化颗粒度与实现机制 | **6/10** | 用户态API拦截 + PID控制器，非内核级 |
| 2. 拉远虚拟化与通信损耗 | **1/10** | 代码中几乎没有实现 |
| 3. 异构兼容性与"南向"生态 | **3/10** | 仅NVIDIA GPU和华为Ascend两种 |
| 4. 2012实验室"黑科技"含金量 | **4/10** | 时间片调度存在，但无保序流图/Checkpoint |
| 5. 隔离性与QoS保障 | **5/10** | 基于PID控制器的简单限流，无HBM带宽监控 |

### 🏆 **总分: 19/50**

### 📌 **核心结论**

> **"务实的工程开源，但与发布会宣传存在明显差距"**

Flex:AI 是一个**真实可用的 GPU/NPU 算力切分工具**，但实现方式为**用户态 API Hook（LD_PRELOAD机制）**，而非演讲所暗示的"内核级虚拟化"或"硬件级切分"。代码库中**几乎不存在跨节点拉远虚拟化**的实现，**RDMA 通信栈完全缺失**。

---

## 详细分析

### 1. 虚拟化颗粒度与实现机制 (6/10)

#### ✅ 存在的实现

**GPU 侧 (NVIDIA CUDA)**:
- **文件**: `GPU-Virtual-Service/xpu-pool-service/direct/cuda/src/hooks/cuda_hooks.cpp`
- **技术**: 使用 `dlsym(RTLD_NEXT, ...)` 拦截 CUDA Driver API
- **拦截的关键函数**:
  ```cpp
  // 内存分配拦截
  FUNC_HOOK_BEGIN(cuMemAlloc_v2, CUdeviceptr *dptr, size_t bytesize)
    auto memGuard = CudaResourceLimiter::Instance().GuardedMemoryCheck(bytesize);
    if (!memGuard.enough) {
      return CUDA_ERROR_OUT_OF_MEMORY;
    }
    return original(dptr, bytesize);
  FUNC_HOOK_END

  // Kernel Launch 拦截（算力限制）
  FUNC_HOOK_BEGIN(cuLaunchKernel, ...)
    CudaResourceLimiter::Instance().ComputingPowerLimiter();
    return original(...);
  FUNC_HOOK_END
  ```

- **算力限制器**: `gpu_core_limiter.cpp` 使用 **PID 控制器**动态调整延迟
  ```cpp
  long GpuCoreLimiter::PidController::CalculateDelay(int diff) {
    // 增量式 PID 控制
    long delay = lround(kp * (diff - prevDiff1) + ki * diff + kd * (diff - coeffDouble * prevDiff1 + prevDiff2));
    prevDiff2 = prevDiff1;
    prevDiff1 = diff;
    return delay;
  }
  ```

**NPU 侧 (华为 Ascend)**:
- **文件**: `GPU-Virtual-Service/xpu-pool-service/direct/acl/src/hooks/runtime_hooks.cpp`
- **拦截的函数**: `rtKernelLaunch`, `rtMalloc`, `rtSetDevice` 等 30+ 个 ACL Runtime API

#### ❌ 缺失的部分

1. **无内核模块**: 没有 `*.ko` 或 Kernel Module 代码，全部在用户态实现
2. **非 MIG 软件切分**: 确实不是 MIG，但切分粒度依赖 NVML 采样频率（约 167ms）
3. **无 ioctl hooking**: 没有真正的 Driver 层拦截

#### 📊 证据

| 文件 | 行号 | 关键代码 |
|------|------|----------|
| `cuda_hooks.cpp` | L39-50 | `cuLaunchKernel` 等 Kernel Launch 函数拦截 |
| `gpu_core_limiter.cpp` | L21-30 | PID 控制器延迟注入 |
| `hook_helper.h` | L10-16 | `dlsym(RTLD_NEXT, ...)` 宏定义 |

---

### 2. 拉远虚拟化与通信损耗 (1/10)

#### ⚠️ 严重缺失

演讲中高调宣传的 **"vXP over Fabric"**、**"RDMA 传输"**、**"损耗 5% 以内"** 在代码库中**几乎没有实现**。

#### 搜索结果

```bash
$ grep -ri "rdma\|ibverbs\|infiniband" .
README.md:      # 提到 RDMA，但仅作为功能描述
npu_manager.h:  # 无实际 RDMA 代码
npu_manager.cpp:# 无实际 RDMA 代码
```

- **无 `libibverbs` 依赖**
- **无 `rdma_cm` 连接管理**
- **无 Command Buffer 序列化/反序列化代码**
- **无远程 GPU/NPU 指令转发逻辑**

#### 唯一的网络代码

代码库中的网络通信使用的是 **gRPC over TCP**，仅用于：
- Device Plugin 与 Kubelet 通信
- Exporter 指标采集

```go
// GPU-device-plugin/pkg/plugin/plugin.go
conn, err := grpc.Dial(unixSocketPath, grpc.WithInsecure(), ...)
```

#### 📌 结论

> **"拉远虚拟化"功能目前为空壳，代码库中不存在任何跨节点算力聚合的实现。**

---

### 3. 异构兼容性与"南向"生态 (3/10)

#### ✅ 支持的硬件

| 厂商 | 硬件 | 实现状态 | 代码位置 |
|------|------|----------|----------|
| NVIDIA | CUDA GPU | ✅ 完整 | `direct/cuda/` |
| 华为 | Ascend NPU | ✅ 完整 | `direct/acl/` |
| AMD | ROCm | ❌ 不存在 | - |
| 寒武纪 | MLU | ❌ 不存在 | - |
| Intel | Gaudi | ❌ 不存在 | - |

#### 架构分析

```cpp
// xpu_manager.h - 抽象基类
class XpuManager {
public:
    virtual int InitXpu() = 0;
    virtual int DeviceCount() = 0;
    virtual int CurrentDevice() = 0;
    virtual int MemoryUsed(size_t &used) = 0;
    virtual std::string_view ConfigPath() = 0;
};
```

存在一个简单的 HAL 抽象层，但**只有两个实现类**:
- `GpuManager` (NVIDIA)
- `NpuManager` (Ascend)

#### ❌ 缺失的"异构兼容"

演讲中提到的 **"南向接口标准"** 和 **"兼容多厂商硬件"** 在代码中体现为：
- 仅有 Interface 定义
- **没有** AMD ROCm 适配代码
- **没有** 寒武纪/Intel 相关代码
- **没有** 厂商无关的通用 HAL 实现

---

### 4. 2012实验室"黑科技"含金量 (4/10)

#### ✅ 存在的功能

**1. 时间片轮转调度器 (Ascend NPU)**

```cpp
// npu_timeslice_scheduler.h
class NpuTimesliceScheduler {
    constexpr static clock::duration TIME_UNIT = std::chrono::milliseconds(1);  // 1ms 时间片
    constexpr static int PERIOD_UNIT_NUMBER = 100;  // 100 个时间单位一个周期
    
    void SchedulerThread(bool &terminating);
    void ExecuteTimeslice(clock::time_point begin);
    void ExecuteIdleTime();
};
```

这是一个**基于共享内存的多进程时间片调度器**，使用 CAS 原子操作实现进程间同步。

**2. 死锁恢复（部分实现）**

```cpp
// npu_timeslice_scheduler.cpp L96-122
void NpuTimesliceScheduler::SelectNewCurrent() {
    // 检测 current 节点是否超时（死亡）
    if (now - curTimestamp > ERR_CHECK_TIMEOUT) {
        return;
    }
    // 尝试 CAS 选举新的 current
    if (context_->current.compare_exchange_strong(cur, best)) {
        log_warn("SelectNewCurrent result {} from node {} to {}", best, idx_, cur);
    }
}
```

#### ❌ 缺失的功能

| 演讲承诺 | 代码状态 | 评价 |
|----------|----------|------|
| 保序流图 (Order-preserving flow graph) | ❌ 不存在 | `graph.go` 只是简单的拓扑邻接矩阵 |
| 状态迁移 (Checkpointing/Migration) | ❌ 不存在 | 无 Checkpoint/Restore 相关代码 |
| Safe-point 机制 | ⚠️ 部分存在 | 有超时检测，但无完整 Safe-point |
| 指令流保序 | ❌ 不存在 | 无 Stream/Command 重排序逻辑 |

#### 📊 代码质量

- **总代码行数**: ~11,334 行 (Go + C++)
- **测试代码行数**: ~561 行 (Go)
- **测试覆盖率**: 约 5%（极低）
- **C++ 无单元测试**: 仅有 `#ifdef UNIT_TEST` 条件编译标记

---

### 5. 隔离性与QoS保障 (5/10)

#### ✅ 实现的隔离机制

**1. 显存隔离**

```cpp
// cuda_hooks.cpp - cuMemAlloc_v2 Hook
CUresult FUNC_HOOK_BEGIN(cuMemAlloc_v2, CUdeviceptr *dptr, size_t bytesize)
  auto memGuard = CudaResourceLimiter::Instance().GuardedMemoryCheck(bytesize);
  if (memGuard.Error()) {
    return CUDA_ERROR_UNKNOWN;
  }
  if (!memGuard.enough) {
    return CUDA_ERROR_OUT_OF_MEMORY;
  }
  return original(dptr, bytesize);
FUNC_HOOK_END
```

**2. 算力限制 (PID 控制)**

```cpp
// gpu_core_limiter.cpp
void GpuCoreLimiter::ComputingPowerLimiter() {
    int delay = GetDelay(gpu_.CurrentDevice());
    if (delay != 0) {
        std::this_thread::sleep_for(std::chrono::microseconds(delay));
    }
}
```

#### ❌ 缺失的 QoS 功能

| 功能 | 状态 | 说明 |
|------|------|------|
| HBM 带宽监控 | ❌ 缺失 | 无带宽采样代码 |
| 算子级不干扰 | ⚠️ 粗粒度 | 只有 Kernel Launch 级别 |
| 性能计数器采集 | ⚠️ 有限 | 仅 `smUtil` 利用率 |
| 动态优先级调整 | ❌ 缺失 | 无优先级概念 |
| 实时反馈控制回路 | ⚠️ 简单 | PID 控制器 167ms 采样 |

#### "吵闹邻居"问题

当前实现**无法防止显存带宽竞争**：
- 只限制了显存总量，不限制访问频率
- PID 控制器只控制 Kernel Launch 频率，不控制单个 Kernel 内部的显存访问

---

## 风险提示

### 🚨 开发者使用此代码库的最大坑

1. **强绑定华为生态**
   - 调度组件 (`vc-scheduler`, `vc-controller-manager`) 以预编译二进制提供
   - 源码不完整，无法独立构建完整系统
   - NPU 虚拟化强依赖 `libascendcl.so` 等华为私有库

2. **性能虚标风险**
   - 用户态 Hook 引入的延迟未在文档中说明
   - PID 控制器参数硬编码，不同负载下效果差异大
   - 167ms 采样周期可能导致短暂超限

3. **跨节点功能完全缺失**
   - 如果基于 README 的描述进行容量规划，将严重失误
   - "RDMA 拉远"目前纯属路线图

4. **测试覆盖不足**
   - C++ 核心代码无单元测试
   - Go 测试仅覆盖边缘模块
   - 无集成测试/端到端测试

5. **文档与代码不符**
   - README 描述的功能远超实际代码
   - "多级智能调度" 实际上只有简单的 Binpack

---

## 技术架构图

```
┌─────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                        │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────────────┐  ┌──────────────────┐                 │
│  │   vc-scheduler   │  │vc-controller-mgr │   Volcano调度   │
│  │  (预编译二进制)   │  │  (预编译二进制)   │                 │
│  └────────┬─────────┘  └────────┬─────────┘                 │
│           │                     │                            │
│  ┌────────▼─────────────────────▼─────────┐                 │
│  │         gpu-device-plugin (Go)          │   设备插件      │
│  │  - 资源注册 (huawei.com/vgpu-*)        │                 │
│  │  - 配置写入 (/etc/xpu/vgpu.config)     │                 │
│  └─────────────────┬───────────────────────┘                 │
│                    │                                         │
│  ┌─────────────────▼───────────────────────┐                │
│  │         Container Runtime                │                │
│  │  ┌─────────────────────────────────┐    │                │
│  │  │  LD_PRELOAD=libcuda_direct.so   │    │   用户态Hook   │
│  │  │  ┌───────────────────────────┐  │    │                │
│  │  │  │    cuda_hooks.cpp         │  │    │                │
│  │  │  │  - cuMemAlloc 拦截        │  │    │                │
│  │  │  │  - cuLaunchKernel 拦截    │  │    │                │
│  │  │  └───────────────────────────┘  │    │                │
│  │  │  ┌───────────────────────────┐  │    │                │
│  │  │  │  gpu_core_limiter.cpp     │  │    │   PID控制器    │
│  │  │  │  - 监测 SM 利用率         │  │    │                │
│  │  │  │  - 动态延迟注入           │  │    │                │
│  │  │  └───────────────────────────┘  │    │                │
│  │  └─────────────────────────────────┘    │                │
│  └─────────────────────────────────────────┘                │
│                    │                                         │
│  ┌─────────────────▼───────────────────────┐                │
│  │         NVIDIA Driver / NVML            │   厂商驱动     │
│  └─────────────────────────────────────────┘                │
└─────────────────────────────────────────────────────────────┘

❌ 缺失模块:
- 跨节点 RDMA 通信层
- AMD ROCm / 寒武纪适配
- Checkpoint/Migration 功能
- 保序流图调度器
```

---

## 附录：关键文件索引

| 功能 | 文件路径 | 行数 |
|------|----------|------|
| CUDA Hook 入口 | `direct/cuda/src/hooks/cuda_hooks.cpp` | 359 |
| GPU 算力限制器 | `direct/cuda/src/gpu_core_limiter.cpp` | 132 |
| GPU 管理器 | `direct/cuda/src/gpu_manager.cpp` | 155 |
| ACL Hook 入口 | `direct/acl/src/hooks/runtime_hooks.cpp` | 256 |
| NPU 时间片调度 | `direct/acl/src/npu_timeslice_scheduler.cpp` | 206 |
| NPU 核心限制器 | `direct/acl/src/npu_core_limiter.cpp` | 143 |
| 设备插件主逻辑 | `GPU-device-plugin/pkg/plugin/plugin.go` | 419 |
| XPU 抽象基类 | `direct/common/include/xpu_manager.h` | 30 |

---

**报告生成方法**: 基于静态代码分析与架构对照验证  
**审计原则**: "代码不撒谎"
