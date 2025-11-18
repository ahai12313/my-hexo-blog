---
title: 06-PyTorch模型保存和加载的两种方式
date: 2025-11-18 22:24:32
tags:
categories: 深度学习
---
# PyTorch模型保存和加载的两种方式

## 方式1：保存/加载整个模型
```python
# 保存
torch.save(model, 'model.pth')

# 加载（无需重新定义模型类）
model = torch.load('model.pth')
model.eval()
```

## 方式2：保存/加载模型参数（推荐）
```python
# 保存
torch.save(model.state_dict(), 'model_weights.pth')

# 加载（需要先定义模型结构）
model = NeuralNetwork()  # 必须先定义模型类
model.load_state_dict(torch.load('model_weights.pth'))
model.eval()
```

## 对比总结
- **方式1**：简单但文件大，兼容性差
- **方式2**：灵活且文件小，兼容性好（推荐用于生产环境）

**最佳实践**：使用方式2，可以保存更多训练信息：
```python
# 保存检查点
torch.save({
    'epoch': epoch,
    'model_state_dict': model.state_dict(),
    'optimizer_state_dict': optimizer.state_dict(),
    'loss': loss,
}, 'checkpoint.pth')
```