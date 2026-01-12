---
name: 鸿蒙-RN
description: 在鸿蒙上运行RN的指导文档,包含RNOH官网,gitcode代码仓库地址,华为文档官网地址。
---
    
鸿蒙相关问题不知道调用websearch去找文档,看官网学习,不要瞎猜瞎写
以下是几个关键地址:
- 基础能力地址:https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/ability-kit   
- RNOH git地址:https://gitcode.com/openharmony-sig/ohos_react_native
- hidumper:华为官网地址https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/hidumper


如果看到比如让你看看屏幕的信息,注意去调用harmonyos-ui这个mcp,这是我开发的一个mcp,能读取鸿蒙设备ui以及其他一些信息,代码也是你生成的,我也不知道还有其他什么功能,后边需要一起完善


鸿蒙的RN版本是从RN0.72.5开始做的,也就是新的Fabric架构版本的,我现在做的工作大部分都是老版本的比如0.51或者0.59的RN在鸿蒙上运行,如果出现问题,优先看看是不是断层更新的问题,目前已经发现了已下几个问题
- zIndex 问题,Fabric版本的RN更符合规范
- 手势相关问题,Fabric版本的RN更符合规范
- Animated相关问题,更卡顿


## 常见问题
- UI 不显示: 检查 list_windows 确认前台应用
- 布局错乱: 用 get_ui_tree 看 drawRegion
- 组件未渲染: 检查 list_abilities 确认 Ability 状态





