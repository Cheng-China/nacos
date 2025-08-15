# Nacos 流量控制文档

## 文档概述

本目录包含 Nacos 流量控制（TPS Control）系统的完整文档，帮助开发者深入理解和学习 Nacos 的流量治理实现。

## 文档结构

### 📖 [nacos-tps-control-guide.md](./nacos-tps-control-guide.md)
**Nacos 流量控制实现原理与学习指南**

这是核心学习文档，包含：
- 🏗️ 整体架构设计和关键组件介绍
- 📁 详细的目录结构说明
- 🗺️ 分阶段的学习路径指导
- 🔧 关键类和接口的详细说明
- ⚙️ 配置方法和最佳实践
- ❓ 常见问题解答

**适合人群**: 想要了解 Nacos TPS 控制原理的开发者

### 💻 [nacos-tps-control-examples.md](./nacos-tps-control-examples.md)
**Nacos TPS 控制实战示例**

这是实践操作文档，包含：
- 🚀 基础使用示例和配置方法
- 🔍 核心代码的深度解析
- 🛠️ 自定义扩展实现示例
- 📊 监控告警配置方法
- 🧪 完整的测试示例
- 🏭 生产环境部署建议

**适合人群**: 需要在项目中实际使用 TPS 控制功能的开发者

## 快速开始

### 第一步：理解基础概念
从 [学习指南](./nacos-tps-control-guide.md) 开始，了解：
- TPS 控制的基本原理
- Nacos 中的架构设计
- 关键组件的作用

### 第二步：动手实践
参考 [实战示例](./nacos-tps-control-examples.md)，学习：
- 如何在代码中使用 `@TpsControl` 注解
- 如何配置限流规则
- 如何扩展自定义功能

### 第三步：深入源码
按照学习指南中的路径，阅读源码：

```
1. core/src/main/java/com/alibaba/nacos/core/control/TpsControl.java
2. plugin/control/src/main/java/com/alibaba/nacos/plugin/control/ControlManagerCenter.java
3. plugin/control/src/main/java/com/alibaba/nacos/plugin/control/tps/TpsControlManager.java
4. core/src/main/java/com/alibaba/nacos/core/control/http/NacosHttpTpsFilter.java
5. plugin/control/src/main/java/com/alibaba/nacos/plugin/control/tps/barrier/
```

## 学习建议

### 🎯 明确学习目标
- **快速应用**: 只需阅读实战示例的基础使用部分
- **深入理解**: 完整阅读学习指南，了解设计原理
- **扩展开发**: 重点学习插件化架构和自定义扩展示例

### 📚 学习顺序
1. 先读概念，后看代码
2. 先理解架构，后学习细节
3. 先跑通示例，后尝试扩展

### 🛠️ 实践建议
- 在本地环境搭建 Nacos 服务
- 运行文档中的示例代码
- 尝试修改限流参数观察效果
- 查看相关日志了解执行流程

## 相关资源

- **Nacos 官网**: https://nacos.io/
- **GitHub 仓库**: https://github.com/alibaba/nacos
- **社区文档**: https://nacos.io/docs/latest/
- **问题反馈**: https://github.com/alibaba/nacos/issues

## 贡献指南

如果您发现文档中的错误或有改进建议，欢迎：
- 提交 Issue 反馈问题
- 提交 Pull Request 完善文档
- 分享您的使用经验和最佳实践

---

*文档最后更新时间: 2024年*