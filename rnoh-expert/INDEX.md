# RNOH Expert Skill - 索引目录

本文档包含 RNOH (React Native for OpenHarmony) 的完整知识体系。

## 文档结构

### 1. skill.md (3,544 行) - 主技能文档
**主技能文档，包含完整的 RNOH 知识体系**

**章节目录**：
- 版本信息和激活条件
- RNOH 架构概览
- 核心实现详解
  - MountingManagerCAPI
  - ArkUINode
  - TurboModuleFactory
  - ComponentInstance
  - RNInstanceCAPI
- 组件实现细节
  - ScrollView
  - TextInput
  - Image
- Fabric vs Paper 对比
- .harmony.js 补丁分析
- 常见问题深度分析
  - zIndex 失效
  - 布局问题
  - 手势异常
  - 性能问题
- 性能优化建议
- 开发调试技巧
- 迁移指南 (RN 0.51/0.59 → RNOH)
- 补充模块和功能
- 高级功能模块
- 更多功能模块
- 终极补充

### 2. QUICK_REFERENCE.md (192 行) - 快速参考指南
**快速查找常见问题和解决方案**

**内容概览**：
- 基本信息
- 项目结构
- 核心组件位置
- 组件支持状态表
- 常用第三方库
- 常见问题快速解决
- 调试命令
- 平台检测
- 性能优化建议
- 迁移检查清单

### 3. RESEARCH_FINDINGS.md (207 行) - 研究发现总结
**四轮研究的完整发现和来源链接**

**研究总结**：
- 研究概述
- 官方文档链接
- 架构亮点
- 关键实现细节
- 重要文件和用途
- 关键特性和能力

### 4. DETAILED_RESEARCH_SUPPLEMENT.md (878 行) - 详细研究补充
**深入实现分析和代码示例**

**详细分析**：
- MountingManagerCAPI 详细分析
- ArkUINode 详细分析
- ComponentInstance 详细分析
- ScrollViewComponentInstance 详细分析
- TextInputComponentInstance 详细分析
- ImageComponentInstance 详细分析
- TurboModuleFactory 详细分析
- .harmony.js 补丁分析
- 常见问题深度分析
- 性能优化建议
- 调试技巧

## 使用指南

### 快速查找
1. 先查看 `QUICK_REFERENCE.md` 找到问题类别
2. 根据需要跳转到 `skill.md` 相应章节
3. 查阅 `DETAILED_RESEARCH_SUPPLEMENT.md` 获取代码示例

### 深入学习
1. 阅读 `skill.md` 完整了解架构
2. 查看 `RESEARCH_FINDINGS.md` 了解研究来源
3. 研究 `DETAILED_RESEARCH_SUPPLEMENT.md` 理解实现细节

### 问题解决
1. 在 `QUICK_REFERENCE.md` 查找问题
2. 在 `skill.md` 常见问题章节深入了解
3. 查阅源码位置进行进一步分析

## 知识覆盖范围

### ✅ 已完全覆盖

**架构**：
- ✅ Fabric + TurboModule 架构
- ✅ 四线程模型 (MAIN, JS, BACKGROUND, WORKER)
- ✅ C-API vs ArkTS 组件
- ✅ ContentSlot vs XComponent

**组件**：
- ✅ View, Text, Image
- ✅ ScrollView, FlatList, SectionList
- ✅ TextInput, Switch, Slider
- ✅ Modal, ActivityIndicator
- ✅ StatusBar, SafeAreaView

**模块**：
- ✅ TurboModule 系统
- ✅ TouchEventDispatcher
- ✅ NativeAnimatedTurboModule
- ✅ AsyncStorage
- ✅ Permissions
- ✅ Clipboard

**功能**：
- ✅ Accessibility 无障碍
- ✅ 手势处理
- ✅ 键盘处理
- ✅ 深度链接
- ✅ 下拉刷新
- ✅ 日期选择器
- ✅ 底部导航

**工具**：
- ✅ Metro 配置
- ✅ 远程调试
- ✅ React DevTools
- ✅ MCP 调试工具

### ❌ 不支持

**iOS 专用组件**：
- ❌ TabBarIOS → 使用 @react-navigation/bottom-tabs
- ❌ SegmentedControlIOS → 使用 react-native-segmented-control
- ❌ SliderIOS → 使用标准 Slider

## 研究统计

| 项目 | 数值 |
|------|------|
| 研究轮次 | 4 轮 |
| 文档总行数 | 4,821 行 |
| 参考来源 | 50+ 篇 |
| 源码文件分析 | 20+ 文件 |
| 第三方库覆盖 | 20+ 库 |
| 组件实现分析 | 15+ 组件 |

## 更新历史

- **2026-01-12**: 第四轮研究完成
  - 添加不支持的组件说明
  - 添加下拉刷新功能
  - 添加日期时间选择器
  - 添加底部导航和分段控制器
  - 创建快速参考指南
  - 创建索引文档

- **2026-01-12**: 第三轮研究完成
  - 添加基础组件实现 (Switch, Modal, ActivityIndicator)
  - 添加存储系统 (AsyncStorage, 文件系统)
  - 添加深度链接
  - 添加剪贴板和分享
  - 添加权限管理补充

- **2026-01-12**: 第二轮研究完成
  - 添加动画系统 (NativeAnimatedTurboModule)
  - 添加网络模块
  - 添加平台模块和 API
  - 添加调试工具
  - 添加第三方库生态

- **2026-01-12**: 第一轮研究完成
  - 核心架构分析
  - 关键实现详解
  - 组件实现分析
  - 常见问题分析
  - 性能优化建议

---

**最后更新**: 2026年1月12日
**状态**: 研究完成，文档齐全
**维护**: 持续更新以反映 RNOH 最新变化
