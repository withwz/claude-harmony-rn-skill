# RNOH (React Native for OpenHarmony) 快速参考指南

## 基本信息

| 项目 | 值 |
|------|-----|
| **版本** | 0.72.102 |
| **基于** | React Native 0.72.5 |
| **架构** | Fabric + TurboModule |
| **SDK** | HarmonyOS 5.0.0.61+ |
| **源码路径** | `/Users/wuzhao/Desktop/ty/rnoh/ohos_react_native` |

## 项目结构

```
ohos_react_native/
├── react-native-harmony/          # NPM 包 (0.72.102)
├── react-native-harmony-cli/      # CLI 工具
├── react-native-harmony-hvigor-plugin/  # Hvigor 插件
├── react-native-harmony-sample-package/  # 示例
├── tester/                         # 测试应用
│   └── harmony/react_native_openharmony/
│       ├── src/main/cpp/          # C++ 实现 (~1073 文件)
│       └── src/main/ets/          # ArkTS 实现 (~51 文件)
└── docs/                          # 文档
```

## 核心组件位置

| 组件 | 文件位置 |
|------|----------|
| MountingManagerCAPI | `tester/.../cpp/RNOH/MountingManagerCAPI.cpp` |
| ArkUINode | `tester/.../cpp/RNOH/arkui/ArkUINode.cpp` |
| TurboModuleFactory | `tester/.../cpp/RNOH/TurboModuleFactory.cpp` |
| NativeAnimatedTurboModule | `tester/.../cpp/RNOHCorePackage/TurboModules/Animated/` |
| TouchEventDispatcher | `tester/.../cpp/RNOH/arkui/TouchEventDispatcher.h` |

## 组件支持状态

| 组件 | 状态 | 备注 |
|------|------|------|
| View, Text, Image | ✅ 支持 | C-API 实现 |
| ScrollView, FlatList | ✅ 支持 | 虚拟化列表 |
| TextInput | ✅ 支持 | 完整功能 |
| Switch | ✅ 支持 | 基于 ToggleNode |
| Modal | ✅ 支持 | 折叠屏适配 |
| ActivityIndicator | ✅ 支持 | LoadingProgressNode |
| TabBarIOS | ❌ 不支持 | 使用 @react-navigation/bottom-tabs |
| SegmentedControlIOS | ❌ 不支持 | 使用 react-native-segmented-control |

## 常用第三方库

```bash
# 导航
npm install @react-navigation/bottom-tabs @react-navigation/native

# 手势
npm install @react-native-oh-tpl/react-native-gesture-handler

# 图片
npm install @react-native-ohos/react-native-fast-image

# 存储
npm install @react-native-oh-tpl/react-native-async-storage

# 权限
npm install @react-native-ohos/react-native-permissions

# 剪贴板
npm install @react-native-oh-tpl/clipboard
```

## 常见问题快速解决

### zIndex 失效
```css
/* 确保同时设置 position */
.child {
  position: 'absolute';  /* 或 'relative' */
  zIndex: 10;
}
```

### 键盘遮挡
```javascript
// 方案1: 使用 ArkUI 原生行为
// RNApp 配置 expandSafeArea()

// 方案2: KeyboardAvoidingView
<KeyboardAvoidingView behavior="padding" style={{ flex: 1 }}>
  {/* 内容 */}
</KeyboardAvoidingView>
```

### setNativeProps 废弃
```javascript
// 旧代码 (不支持)
this.refs.myComponent.setNativeProps({ style: { opacity: 0.5 } });

// 新代码
const [opacity, setOpacity] = useState(1);
<View style={{ opacity }} />
```

### TurboModule 获取
```javascript
// 正确
import { TurboModuleRegistry } from 'react-native';
const MyModule = TurboModuleRegistry.getEnforcing('MyModule');

// 错误 (已废弃)
const MyModule = NativeModules.MyModule;
```

## 调试命令

```bash
# 端口转发
hdc rport tcp:8081 tcp:8081

# 查看窗口
# (使用 MCP 工具)
mcp__harmonyos-ui__list_windows

# 获取 UI 树
mcp__harmonyos-ui__get_ui_tree

# 截图
mcp__harmonyos-ui__screenshot /tmp/screenshot.jpeg
```

## 平台检测

```javascript
import { Platform } from 'react-native';

if (Platform.OS === 'harmony') {
  // OpenHarmony 特定代码
}

// 文件扩展名
// .harmony.js / .harmony.tsx / .harmony.ts
```

## 性能优化建议

1. **优先使用 C-API 组件** - 性能最优
2. **TurboModule 线程选择**
   - 纯逻辑 → cxxTurboModule (JS 线程)
   - 系统 API → ArkTSTurboModule (MAIN/WORKER)
3. **列表优化** - 使用 getItemLayout
4. **图片优化** - 使用 FastImage
5. **减少重渲染** - React.memo, useMemo, useCallback

## 迁移检查清单 (RN 0.51/0.59 → RNOH)

- [ ] 使用 `TurboModuleRegistry.getEnforcing()` 代替 `NativeModules`
- [ ] 移除 `removeEventListener`，使用 `subscription.remove()`
- [ ] 使用 `Platform.OS === 'harmony'` 检测平台
- [ ] 日期格式改为 ISO 8601
- [ ] 移除 `setNativeProps`，使用 React state
- [ ] 检查 zIndex，确保设置 position
- [ ] 验证 flexbox 布局
- [ ] 测试手势和动画

## 官方资源

- **源码**: [gitee.com/openharmony-sig/ohos_react_native](https://gitee.com/openharmony-sig/ohos_react_native)
- **文档**: [react-native-oh-library.github.io](https://react-native-oh-library.github.io/usage-docs/)
- **GitHub**: [github.com/react-native-oh-library](https://github.com/react-native-oh-library) (288+ 仓库)

## 线程模型

| 线程 | 用途 |
|------|------|
| MAIN | UI 操作、ArkTS TurboModules |
| JS | JavaScript 运行时、ShadowTree |
| BACKGROUND | 实验性 (不推荐) |
| WORKER | TurboModule 执行、防阻塞 |

## TurboModule 类型

| 类型 | 实现 | 用途 |
|------|------|------|
| cxxTurboModule | 纯 C++ | 平台无关逻辑 |
| ArkTSTurboModule | ArkTS | HarmonyOS 系统 API |

---

**更新日期**: 2026年1月12日
**研究轮次**: 4 轮完整研究
**文档总行数**: 3544+ 行
