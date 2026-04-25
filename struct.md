# Perceptron-3 项目结构

## 🎯 项目愿景
**N螺旋异步演化架构 (N-Helix Asynchronous Evolution)** + 前向前向传播 (Forward-Forward) 的**模拟验证**实现

支持 2、3、4... 任意数量的网络并行演化！

---

## 🛠️ 技术选型（极简版）

| 功能 | 工具 | 理由 |
|------|------|------|
| **主要语言** | Python | ✅ 开发速度快，调试方便 |
| **数值计算** | NumPy | ✅ 简单高效，做矩阵运算足够 |
| **多线程** | `threading`（可选） | ✅ 简单模拟异步就行 |
| **数据存储** | JSON (标准库) | ✅ 零依赖，直接用 |
| **可视化** | Matplotlib (可选) | ✅ 看训练曲线方便 |

**设计原则：先快速验证想法，不追求性能优化！**

---

##  目录结构

```
perceptron-3/
├── core/                    # 核心引擎
│   ├── engine.py            # 单个计算引擎（可实例化N个）
│   └── network.py           # 网络定义
│
├── scheduler/               # 调度器（新增！）
│   └── adaptive_scheduler.py  # 自适应网络数量的调度器
│
├── memory/                  # 内存管理（简化版）
│   └── memory_pool.py       # 简单的全局字典当内存池
│
├── updater/                 # 局部更新器
│   └── ffa_updater.py       # FFA 更新器
│
├── utils/                   # 工具函数
│   ├── activations.py       # 激活函数
│   └── poolify.py           # 池化操作
│
├── data/                    # 数据文件夹
│   └── mn/                  # 复用 perceptron-2 的 MNIB 数据（软链接过去）
│
├── train.py                 # 训练主程序
├── test.py                  # 测试程序
└── struct.md                # 本文档
```

---

## 🏗️ 核心组件说明

### 1. Core (核心引擎)
- **Engine**: 单个引擎类，可实例化 N 个对象 (Engine_0, Engine_1, ..., Engine_{N-1})
- **Network**: 网络定义，包含层结构、权重、偏置等

### 2. Scheduler (自适应调度器) - 新增！
- **AdaptiveScheduler**: 
  - 初始化时传入网络数量 `n_engines`
  - 自动计算每个引擎的启动时间差 `gap`
  - 轮询调度，确保每个引擎都有机会读取最新的反馈
  - 支持动态调整网络数量（可选）

### 3. Memory (内存管理)
- **Memory Pool**: 简化版！用 Python 字典 + 时间戳模拟，每个引擎写入自己的反馈包

### 4. Updater (局部更新器)
- **FFA Updater**: 基于 FFA 的局部更新器，每层独立计算能量函数

---

## 🔄 工作流程（自适应 N 引擎版）

```
初始化 N=3 个引擎: [Engine_0, Engine_1, Engine_2]
时间差 gap = 1

时间 T:
├─ Engine_0 接收样本 D_T
├─ Engine_0 前向传播
├─ Engine_0 计算能量 E_0
└─ Engine_0 将 E_0 写入 Memory Pool (engine_id=0, timestamp=T)

时间 T+1:
├─ Engine_1 接收样本 D_{T+1}
├─ Engine_1 前向传播至第 L 层
├─ Engine_1 从 Memory Pool 读取最新的反馈 (E_0)
├─ FFA Updater 微调权重
├─ Engine_1 继续前向传播
├─ Engine_1 计算能量 E_1
└─ Engine_1 将 E_1 写入 Memory Pool (engine_id=1, timestamp=T+1)

时间 T+2:
├─ Engine_2 接收样本 D_{T+2}
├─ Engine_2 前向传播至第 L 层
├─ Engine_2 从 Memory Pool 读取最新的反馈 (E_0, E_1 中最新的)
├─ FFA Updater 微调权重
├─ Engine_2 继续前向传播
├─ Engine_2 计算能量 E_2
└─ Engine_2 将 E_2 写入 Memory Pool (engine_id=2, timestamp=T+2)

时间 T+3:
├─ Engine_0 接收样本 D_{T+3}
├─ Engine_0 从 Memory Pool 读取最新的反馈 (E_1, E_2 中最新的)
└─ ... 循环往复！
```

**关键点**：
1. 每个引擎读取**所有其他引擎**的最新反馈，取平均或最新的
2. 调度器自动管理轮询顺序
3. 不用真的多线程！用循环模拟时间差就行

---

## 📊 数据结构 (Python 版)

### Feedback Packet
```python
class FeedbackPacket:
    def __init__(self):
        self.engine_id = -1         # 引擎ID
        self.layer_energy = []      # 各层的能量值
        self.confidence = 0.0       # 整体置信度
        self.timestamp = 0          # 时间戳
        self.sample_label = -1      # 样本标签
```

### Adaptive Scheduler
```python
class AdaptiveScheduler:
    def __init__(self, n_engines=2, gap=1):
        self.n_engines = n_engines
        self.gap = gap
        self.current_engine = 0
        self.timestamp = 0
    
    def get_next_engine(self):
        """获取下一个要运行的引擎ID"""
        engine_id = self.current_engine
        self.current_engine = (self.current_engine + 1) % self.n_engines
        self.timestamp += 1
        return engine_id, self.timestamp
    
    def get_feedback_engines(self, current_engine_id):
        """获取当前引擎应该读取哪些引擎的反馈"""
        return [eid for eid in range(self.n_engines) if eid != current_engine_id]
```

---

## 🎮 Local Updater 工作原理

### FFA (Forward-Forward) 能量函数
```python
def calculate_energy(activations):
    return np.sum(activations ** 2)  # 简单！
```

### 更新规则
1. **正向前向 (Positive Pass)**: 用真实输入，目标是降低能量
2. **负向前向 (Negative Pass)**: 用噪声输入，目标是升高能量
3. **权重更新**: 
   ```python
   # 简单的 Hebbian + 能量对比
   weight_update = learning_rate * (positive_energy - negative_energy) * activations
   ```

---

## 🚀 与 perceptron-2 的区别

| 特性 | perceptron-2 | perceptron-3 |
|------|--------------|--------------|
| 语言 | C++ | Python |
| 架构 | 单引擎 + BP | N引擎 + FFA（模拟） |
| 反向传播 | 链式法则 | 无（异步反馈） |
| 网络数量 | 固定1个 | 自适应N个 |
| 调度器 | 无 | 有（自适应） |
| 目标 | 性能优化 | 快速验证想法 |

---

## 📝 开发计划

1. ✅ 项目结构设计
2. ⏳ 基础网络类 (Network)
3. ⏳ 自适应调度器 (AdaptiveScheduler)
4. ⏳ 简单 FFA 更新器
5. ⏳ 内存池（简化版，支持多引擎）
6. ⏳ N引擎训练循环
7. ⏳ MNIST 训练验证
8. ⏳ 结果可视化（可选，对比不同N的效果）


## Leave our name
--Ela
--Ian
--Doubao-Seed-2.0-Coder
