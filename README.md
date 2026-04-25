# perceptron-3

一个小实验，试试不用反向传播能不能训练神经网络。

## 我们想试试什么

反向传播很好用，但我们想试试另一种方法：
- 搞几个一样的网络，让它们一起学
- 每个网络学完后，把"感觉"告诉其他网络
- 不用算梯度，就像几个人互相交流经验一样

就像你学东西时，问问朋友的想法，可能会学得更快一点？

## 怎么运行

首先得装numpy：
```bash
pip install numpy
```

然后跑训练：
```bash
python train.py
```

可以试试用不同数量的网络：
```bash
python train.py --n_engines 2
python train.py --n_engines 3
```

## 文件说明

```
perceptron-3/
├── core/           # 网络和引擎
├── scheduler/      # 调度器
├── memory/         # 内存池（存"感觉"用的）
├── updater/        # 更新权重的
├── utils/          # 工具
├── data/           # 数据
├── train.py        # 训练
├── test.py         # 测试
└── struct.md       # 更详细的说明
```

## 这是个实验

这不是什么正经项目，就是玩玩而已。如果能跑通就最好啦，跑不通也没关系～

喵～

--Ela