---
name: rnoh-expert
description: RNOH (React Native for OpenHarmony) v0.72.x 专家知识。仅在以下场景使用：分析 RNOH 源码、调试 RN 0.51/0.59 迁移到 RNOH 的问题、调查鸿蒙特有的 React Native 实现细节、或使用 ContentSlot/TurboModule 架构时。
allowed-tools: Read, Grep, Glob, Bash, WebSearch, Task
---

# RNOH (React Native for OpenHarmony) 专家指南

## 版本信息

| 项目 | 版本 |
|------|------|
| RNOH 当前版本 | 0.72.58 (NPM 包) |
| RNOH 发布版本 | v5.0.0.813 (2024-12-26) |
| React Native | 0.72.5 (Fabric 新架构) |
| HarmonyOS SDK | 5.0.0.61+ (NEXT) |
| 架构 | Fabric + TurboModule |
| 推荐分支 | 0.72.5-ohos-5.0-release |
| 代码库路径 | `/Users/wuzhao/Desktop/ty/rnoh/react-native-openharmony` |

### 版本说明
- **NPM 版本**: `@rnoh/react-native-harmony@0.72.58`
- **发布版本**: 采用四段式版本号，如 `v5.0.0.813`
- **分支说明**:
  - `0.72.5-ohos-5.0-release`: 稳定发布分支，推荐用于生产环境
  - `master`: 主分支，不保证质量
  - `dev`/`partner-dev`: 开发分支，不保证质量

---

## 激活条件

此 Skill 在以下场景激活：

### 核心场景
- 分析或修改 RNOH 源码
- 调试 RN 老版本 (0.51/0.59) 迁移到 RNOH 的问题
- 询问鸿蒙特有的 React Native 实现细节
- 使用 ContentSlot、TurboModule 或 C-API 组件架构
- 调查 RNOH 中的 zIndex、手势或动画问题

### 典型问题示例

#### 架构与源码相关
- "为什么我的组件在鸿蒙上布局错乱？"
- "zIndex 在鸿蒙上不生效怎么办？"
- "如何理解 RNOH 的三阶段渲染流程？"
- "ArkUINode 和 ArkTS 组件有什么区别？"

#### TurboModule 相关
- "如何创建一个自定义的 TurboModule？"
- "TurboModule 在 ArkTS 和 C++ 中有什么区别？"
- "StatusBarTurboModule 是如何实现状态栏控制的？"

#### 迁移相关
- "从 RN 0.59 升级需要改哪些代码？"
- "RN 0.51/0.59 的老代码在鸿蒙上不兼容怎么办？"

#### 鸿蒙特有 API
- "ContentSlot 如何接入使用？"
- "@ohos.window API 如何正确使用？"
- "如何使用鸿蒙的 avoidArea 获取状态栏高度？"

#### 测试与调试
- "如何运行 tester 目录下的测试？"
- "测试计划中的 iteration 是什么意思？"

---

## RNOH 架构概览

### 整体架构层次

```
┌─────────────────────────────────────────────────────────┐
│                    React Native JS 层                    │
│  (Components, APIs, Application Code)                   │
│  - Libraries/*.harmony.js (36个鸿蒙专用组件补丁)          │
└─────────────────────────────────────────────────────────┘
                         ↓ JSI
┌─────────────────────────────────────────────────────────┐
│                   React Common C++ 层                    │
│  (ShadowTree, Scheduler, Yoga Layout)                    │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│              RNOH 适配层 (核心实现区域)                   │
│  ┌────────────────────────────────────────────────┐     │
│  │  C++ 层 (~1073 个文件)                          │     │
│  │  - ComponentInstance (组件实例)                 │     │
│  │  - TurboModule (模块系统)                       │     │
│  │  - MountingManager (挂载管理器)                 │     │
│  │  - ArkUINode (C-API 封装)                       │     │
│  │  - ArkJS (JSI 封装)                             │     │
│  │  - ArkTSBridge (ArkTS 桥接)                     │     │
│  └────────────────────────────────────────────────┘     │
│                         ↓ N-API                         │
│  ┌────────────────────────────────────────────────┐     │
│  │  ArkTS 层 (~51 个文件)                          │     │
│  │  - RNInstance (实例管理)                        │     │
│  │  - RNSurface (页面容器)                         │     │
│  │  - TurboModule (ArkTS 实现)                    │     │
│  │  - ComponentManager (组件管理器)                │     │
│  │  - RNAbility (应用入口)                         │     │
│  └────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│                OpenHarmony OS 层 (ArkUI)                 │
│  - Native C-API                                         │
│  - ArkUI Components                                     │
│  - System APIs                                          │
└─────────────────────────────────────────────────────────┘
```

### 线程模型

```cpp
enum TaskThread {
  MAIN       = 0,  // 主线程/UI线程
                      // - ArkUI 组件生命周期管理
                      // - 组件树管理
                      // - TurboModule 执行 (ArkTS)
                      // - 交互事件处理

  JS,                // JS 运行时线程
                      // - 执行 React/JS 代码
                      // - 创建 ShadowTree
                      // - Yoga 布局计算
                      // - 生成 Mutations

  BACKGROUND,         // 后台任务线程 (实验性，不推荐)
                      // - 部分布局任务
                      // - ShadowTree 比较
                      // - 注意：稳定性风险，不推荐使用

  WORKER,             // Worker 线程
                      // - TurboModule 执行
                      // - 模块间通信
                      // - 减轻主线程负担
};
```

### 项目目录结构

**根目录**: `/Users/wuzhao/Desktop/ty/rnoh/react-native-openharmony`

```
react-native-openharmony/
├── react-native-harmony/           # 核心库 (@rnoh/react-native-harmony)
│   ├── Libraries/                  # React Native 组件库
│   │   ├── Components/            # 组件实现
│   │   ├── *.harmony.js           # 鸿蒙专用补丁 (~25个文件)
│   │   └── *.harmony.ts           # 鸿蒙专用 TypeScript 补丁
│   ├── types/                     # TypeScript 类型定义
│   └── package.json               # 版本: 0.72.58
│
├── react-native-harmony-cli/      # CLI 工具
│   └── package.json               # 版本: 0.0.28
│
├── react-native-harmony-sample-package/  # 示例包
│   ├── src/main/ets/              # ArkTS 源码
│   │   ├── SampleTurboModule.ets  # 示例 TurboModule (ArkTS)
│   │   ├── SampleWorkerTurboModule.ets  # Worker TurboModule
│   │   └── SamplePackage.ets      # Package 定义
│   └── src/main/cpp/              # C++ 源码
│       ├── SampleTurboModuleSpec.h
│       └── SamplePackage.cpp
│
├── tester/                         # 测试应用
│   ├── harmony/                   # 鸿蒙测试项目
│   │   └── react_native_openharmony/src/main/ets/
│   │       ├── RNOHCorePackage/   # 核心 Package 实现
│   │       │   ├── turboModules/  # TurboModule 集合 (~34个)
│   │       │   │   ├── StatusBarTurboModule.ts
│   │       │   │   ├── NetworkingTurboModule.ts
│   │       │   │   ├── ToastAndroidTurboModule.ts
│   │       │   │   └── ...
│   │       │   ├── components/    # 组件实现
│   │       │   └── componentManagers/
│   │       ├── RNSurface.ets      # React Native 表面
│   │       └── RNOH/              # RNOH 核心框架
│   ├── plan/                      # 测试计划文档
│   │   ├── iteration_*.md         # 迭代测试记录
│   │   └── SUMMARY_*.md           # 测试总结
│   ├── tests/                     # 测试用例
│   └── package.json
│
├── docs/                          # 文档
│   ├── zh-cn/                     # 中文文档
│   └── en/                        # 英文文档
│
└── README_zh.md                   # 中文 README
```

### 鸿蒙专用组件补丁

RNOH 通过 `.harmony.js` 和 `.harmony.ts` 文件提供鸿蒙专用实现：

| 组件 | 补丁文件 | 主要修改内容 |
|------|----------|-------------|
| View | View.harmony.js | pointerEvents 展平控制 |
| Image | Image.harmony.js | 鸿蒙图片加载适配 |
| TextInput | TextInput.harmony.js | 鸿蒙输入法适配 |
| ScrollView | ScrollView.harmony.js | flashScrollIndicators、键盘模式 |
| StatusBar | StatusBar.harmony.js | 状态栏 API 适配 |
| UIManager | UIManager.harmony.js | 鸿蒙特有 UI 管理器 |

#### 补丁文件示例

**View.harmony.js - pointerEvents 控制**
```javascript
// RNOH patch: 禁用特定 pointerEvents 的 View 展平
// 原因: HarmonyOS 的触摸处理需要完整的组件层级
collapsable={["box-only", "none"].includes(newPointerEvents) ? false : collapsable}
```

**ScrollView.harmony.js - JS 端实现 flashScrollIndicators**
```javascript
// RNOH patch - resolving flashScrollIndicators command on the JS side
// as it is currently not feasible on the native side
flashScrollIndicators: () => void = () => {
  this.state.showScrollIndicator = true;
  setTimeout(() => {
    this.state.showScrollIndicator = false;
    this.forceUpdate();
  }, 500);
  this.forceUpdate();
};
```

**ScrollView.harmony.js - 平台检测**
```javascript
// RNOH: patch - add support for OpenHarmony
if ((Platform.OS === "android" || Platform.OS === "harmony") &&
    this.props.keyboardDismissMode === "on-drag") {
  dismissKeyboard();
}
```

**ScrollView.harmony.js - RefreshControl 样式拆分**
```javascript
// RNOH: patch - use Android implementation
const { outer, inner } = splitLayoutProps(flattenStyle(props.style));
return React.cloneElement(
  refreshControl,
  { style: StyleSheet.compose(baseStyle, outer) },
  <NativeDirectionalScrollView {...props} style={StyleSheet.compose(baseStyle, inner)}>
    {contentContainer}
  </NativeDirectionalScrollView>
);
```

### TurboModule 实现方式

RNOH 支持两种 TurboModule 实现方式：

1. **ArkTS TurboModule** (推荐，性能更好)
   - 位置: `tester/harmony/.../turboModules/*.ts`
   - 继承: `UITurboModule`
   - 示例: `StatusBarTurboModule.ts`

2. **C++ TurboModule**
   - 位置: `.../cpp/*.cpp`, `.../*.h`
   - 使用 Codegen 生成规范
   - 示例: `SampleTurboModuleSpec.h`

### 鸿蒙特有 API 说明

RNOH 中常用的 HarmonyOS 系统 API：

#### @ohos.window - 窗口管理
```typescript
// StatusBarTurboModule.ts 中的使用示例
import window from '@ohos.window';

// 获取窗口实例
const windowInstance = await window.getLastWindow(this.ctx.uiAbilityContext);

// 获取状态栏高度
const statusBarHeight = windowInstance.getWindowAvoidArea(window.AvoidAreaType.TYPE_SYSTEM).topRect.height;

// 设置系统栏属性
windowInstance.setWindowSystemBarProperties({
  statusBarColor: '#FF0000',
  statusBarContentColor: '#FFFFFF'
});

// 监听避让区域变化
windowInstance.on('avoidAreaChange', (data) => {
  if(data.type === window.AvoidAreaType.TYPE_SYSTEM){
    // 更新状态栏高度
  }
});

// 设置状态栏显示/隐藏
await windowInstance.setSpecificSystemBarEnabled('status', false);
```

#### 颜色插值动画（HarmonyOS API 12 之前解决方案）

由于 HarmonyOS API 12 之前不支持 `enableStatusBarAnimation`，RNOH 实现了手动颜色插值：

```typescript
// 颜色插值函数 - 生成渐变色数组
function interpolateColor(color1: string, color2: string, steps: number): string[] {
  let colorStartRGB = hexToRGB(color1);
  let colorStopRGB = hexToRGB(color2);

  let rStep = (colorStopRGB[0] - colorStartRGB[0]) / (steps - 1);
  let gStep = (colorStopRGB[1] - colorStartRGB[1]) / (steps - 1);
  let bStep = (colorStopRGB[2] - colorStartRGB[2]) / (steps - 1);
  let aStep = (colorStopRGB[3] - colorStartRGB[3]) / (steps - 1);

  let colorArray: string[] = [];
  for (let i = 0; i < steps; i++) {
    let r = Math.min(255, Math.max(0, colorStartRGB[0] + (rStep * i)));
    let g = Math.min(255, Math.max(0, colorStartRGB[1] + (gStep * i)));
    let b = Math.min(255, Math.max(0, colorStartRGB[2] + (bStep * i)));
    let a = Math.min(255, Math.max(0, colorStartRGB[3] + (aStep * i)));
    colorArray.push(rgbToHex(Math.round(r), Math.round(g), Math.round(b), Math.round(a)));
  }
  return colorArray;
}

// 应用颜色动画 - 20帧渐变，每帧16ms (~60fps)
async setColor(color: number, animated: boolean) {
  const colorString = convertColorValueToHex(color);

  if (!animated) {
    windowInstance.setWindowSystemBarProperties({ statusBarColor: colorString });
  } else {
    // 手动创建颜色渐变动画
    let colors = interpolateColor(this._currentColor, colorString, 20);
    let i = 0;
    let intervalId = setInterval(() => {
      windowInstance.setWindowSystemBarProperties({ statusBarColor: colors[i] });
      i++;
      if (i >= colors.length) {
        clearInterval(intervalId);
      }
    }, 16); // ~60fps
  }
  this._currentColor = colorString;
}
```

**关键技术点**：
- 20 帧渐变，每帧 16ms，约 60fps
- RGB/A 分量独立插值
- 保存当前颜色状态，支持连续动画

#### px2vp - 像素转换
```typescript
declare function px2vp(px: number): number;
// 将物理像素转换为虚拟像素
const scaledHeight = px2vp(statusBarHeight);
```

#### 其他常用 @ohos API
| API 模块 | 用途 | 示例场景 |
|----------|------|----------|
| @ohos.file.fs | 文件系统操作 | 图片缓存、本地存储 |
| @ohos.net.http | 网络请求 | NetworkingTurboModule |
| @ohos.multimedia.image | 图片处理 | Image 组件解码 |
| @ohos.sensor | 传感器访问 | 加速度计、陀螺仪 |
| @ohos.geolocation | 地理位置 | Geolocation 模块 |

### 核心概念：JSI (JavaScript Interface)

**JSI** 是 React Native 新架构的通信基础，替代了旧版本的 Bridge：

**关键特性**：
- **同步调用**：JavaScript 可以直接调用 C++ 函数，无需异步序列化
- **零拷贝**：数据直接在内存中传递，无需 JSON 序列化/反序列化
- **类型安全**：通过 TypeScript/Flow 定义规范，Codegen 生成类型安全代码

**JSI 在 RNOH 中的应用**：
```cpp
// ArkJS - JSI 的 RNOH 封装
class ArkJS {
  // JSI Value ↔ NAPI Value 转换
  napi_value getNapiValue(jsi::Value const&);
  jsi::Value getJSIValue(napi_value);

  // JSI Value ↔ Dynamic 转换
  folly::dynamic getDynamic(napi_value);
  jsi::Value valueFromDynamic(folly::dynamic const&);
};
```

---

## ArkUI C-API 封装实现

### ArkUINode 基类架构

**位置**: `tester/harmony/react_native_openharmony/src/main/cpp/RNOH/arkui/ArkUINode.h`

**核心设计**：
```cpp
class ArkUINode {
protected:
  ArkUI_NodeHandle m_nodeHandle;           // ArkUI 原生节点句柄
  std::shared_ptr<NodeApi> m_nodeApi;      // Node API 指针
  ArkUINodeDelegate* m_arkUINodeDelegate;  // 事件委托

public:
  // 构造函数：从 NodeHandle 或 NodeType 创建
  ArkUINode(ArkUI_NodeHandle nodeHandle);
  ArkUINode(const ArkUINode::Context::Shared context, ArkUI_NodeType nodeType);

  // 核心属性设置方法
  virtual ArkUINode& setPosition(facebook::react::Point const& position);
  virtual ArkUINode& setSize(facebook::react::Size const& size);
  virtual ArkUINode& setLayoutRect(Point, Size, Float pointScaleFactor);
  virtual ArkUINode& setBackgroundColor(uint32_t color);
  virtual ArkUINode& setTransform(facebook::react::Transform const&);
  virtual ArkUINode& setOpacity(float opacity);
  virtual ArkUINode& setBorderRadius(float borderRadius);

  // 事件处理
  virtual void onNodeEvent(ArkUI_NodeEventType eventType, EventArgs& eventArgs);
  virtual void onTouchIntercept(const ArkUI_UIInputEvent* event);

  // 节点操作
  ArkUI_NodeHandle getArkUINodeHandle() const { return m_nodeHandle; }
};
```

**事件接收器注册机制**：
```cpp
// 全局静态事件接收器
static void receiveEvent(ArkUI_NodeEvent* event) {
  auto eventType = OH_ArkUI_NodeEvent_GetEventType(event);
  auto nodeHandle = OH_ArkUI_NodeEvent_GetNodeHandle(event);

  // 从 userData 获取 ArkUINode 指针
  ArkUINode* target = static_cast<ArkUINode*>(
      NativeNodeApi::getInstance()->getUserData(nodeHandle)
  );

  // 调用实例方法
  target->onNodeEvent(eventType, componentEvent->data);
}
```

**智能属性缓存机制**：
```cpp
// TextNode 的属性缓存示例
class TextNode : public ArkUINode {
private:
  enum Flag {
    FLAG_FONTSIZE, FLAG_FONTCOLOR, FLAG_LINEHEIGHT, ...
  };
  std::bitset<32> m_initFlag;  // 跟踪哪些属性已设置

public:
  TextNode& setFontSize(float fontSize) {
    // 避免重复设置相同值
    if (!m_initFlag[FLAG_FONTSIZE] || fontSize != m_fontSize) {
      ArkUI_NumberValue value[] = {{.f32 = fontSize}};
      setAttribute(NODE_TEXT_FONT_SIZE, {value, 1});
      m_initFlag[FLAG_FONTSIZE] = true;
      m_fontSize = fontSize;
    }
    return *this;
  }
};
```

### 核心 C-API 组件

| 组件类 | ArkUI 节点类型 | 功能描述 |
|--------|---------------|----------|
| `StackNode` | `ARKUI_NODE_STACK` | View 容器，支持子节点插入/删除 |
| `TextNode` | `ARKUI_NODE_TEXT` | 文本显示，支持丰富样式和格式化 |
| `ImageNode` | `ARKUI_NODE_IMAGE` | 图片显示，支持网络和本地资源 |
| `TextInputNode` | `ARKUI_NODE_TEXT_INPUT` | 文本输入，支持键盘和事件 |
| `ScrollViewNode` | `ARKUI_NODE_SCROLL` | 滚动容器，支持水平和垂直滚动 |
| `ColumnNode` | `ARKUI_NODE_COLUMN` | 列布局容器 |
| `RefreshNode` | `ARKUI_NODE_REFRESH` | 下拉刷新组件 |
| `LoadingProgressNode` | `ARKUI_NODE_LOADING_PROGRESS` | 加载指示器 |

### StackNode 容器节点

**位置**: `tester/harmony/react_native_openharmony/src/main/cpp/RNOH/arkui/StackNode.h`

```cpp
class StackNode : public ArkUINode {
protected:
  StackNodeDelegate* m_stackNodeDelegate;

public:
  explicit StackNode(const ArkUINode::Context::Shared& context);

  // 子节点管理
  void insertChild(ArkUINode& child, std::size_t index);
  void addChild(ArkUINode& child);
  void removeChild(ArkUINode& child);

  // 委托事件
  void onClick();
  void setStackNodeDelegate(StackNodeDelegate* delegate);
};

// 实现
void StackNode::insertChild(ArkUINode& child, std::size_t index) {
  m_nodeApi->insertChildAt(
      m_nodeHandle,
      child.getArkUINodeHandle(),
      static_cast<int32_t>(index)
  );
}

void StackNode::onNodeEvent(ArkUI_NodeEventType eventType, EventArgs& eventArgs) {
  ArkUINode::onNodeEvent(eventType, eventArgs);

  if (eventType == NODE_ON_CLICK) {
    // 过滤非触屏/鼠标事件
    if (eventArgs[3].i32 != UI_INPUT_EVENT_SOURCE_TYPE_TOUCH_SCREEN &&
        eventArgs[3].i32 != UI_INPUT_EVENT_SOURCE_TYPE_MOUSE) {
      onClick();
    }
  }

  if (eventType == NODE_ON_HOVER && m_stackNodeDelegate) {
    if (eventArgs[0].i32) {
      m_stackNodeDelegate->onHoverIn();
    } else {
      m_stackNodeDelegate->onHoverOut();
    }
  }
}
```

### TextNode 文本节点

**位置**: `tester/harmony/react_native_openharmony/src/main/cpp/RNOH/arkui/TextNode.h`

**丰富文本支持**：
```cpp
class TextNode : public ArkUINode {
public:
  // 基础属性
  TextNode& setTextContent(const std::string& text);
  TextNode& setFontColor(uint32_t fontColor);
  TextNode& setFontSize(float fontSize);
  TextNode& setFontFamily(const std::string& fontFamily);

  // 样式属性
  TextNode& setTextLineHeight(float textLineHeight);
  TextNode& setTextDecoration(int32_t decorationStyle, uint32_t decorationColor);
  TextNode& setTextCase(int32_t textCase);
  TextNode& setTextLetterSpacing(float textLetterSpacing);

  // 段落控制
  TextNode& setTextMaxLines(int32_t textMaxLines);
  TextNode& setTextAlign(int32_t align);
  TextNode& setTextOverflow(int32_t textOverflow);

  // 高级特性
  TextNode& setTextContentWithStyledString(std::shared_ptr<TextMeasureInfo> info);
  TextNode& setSelectedBackgroundColor(uint32_t color);
  TextNode& setTextDataDetectorType(int32_t enable, const ArkUI_NumberValue types[], int size);
};
```

### ArkUISurface 管理机制

**位置**: `tester/harmony/react_native_openharmony/src/main/cpp/RNOH/arkui/ArkUISurface.h`

```cpp
class ArkUISurface : public Surface {
public:
  // Surface 生命周期管理
  void start(..., folly::dynamic const& initialProps, ...);
  void stop(std::function<void()> onStop);
  void setProps(folly::dynamic const& props);

  // 约束和测量
  facebook::react::Size measure(...);
  void updateConstraints(...);

  // ContentSlot 集成
  void attachToNodeContent(NodeContentHandle nodeContentHandle);
  void detachFromNodeContent();

private:
  facebook::react::SurfaceHandler m_surfaceHandler;
  ComponentInstance::Shared m_rootView;
  std::optional<NodeContentHandle> m_nodeContentHandle;
};
```

---

## ContentSlot 接入详解

### 从 XComponent 迁移到 ContentSlot

**迁移原因**：
1. **性能更优**：ContentSlot 在内存和渲染性能上优于 XComponent
2. **API 演进**：XComponent 的 NODE 类型不再演进
3. **跨模块支持**：XComponent 的 libraryname 不支持跨 module 使用

### ContentSlot 接入流程

**调用链**：
```
[ArkTS] RNSurface.aboutToAppear()
    ↓
[ArkTS] surfaceHandle.attachRootView(id, tag, nodeContent)
    ↓
[ArkTS] → [C++] RNInstanceCAPI::attachRootView(NodeContentHandle, surfaceId)
    ↓
[C++] ArkUISurface::attachToNodeContent(nodeContentHandle)
    ↓
[C++] NodeContentHandle::addNode(m_rootView->getLocalRootArkUINode())
    ↓
[C++] OH_ArkUI_NodeContent_AddNode(contentHandle, arkUINodeHandle)
    ↓
[ArkUI] ContentSlot 显示 C++ 创建的 ArkUI 节点树
```

**C++ 侧实现**：
```cpp
// 1. createSurface 时创建 ArkUISurface
void RNInstanceCAPI::createSurface(SurfaceId surfaceId, ...) {
    m_surfaceById.emplace(surfaceId, ArkUISurface(...));
    m_rootView = componentInstanceFactory->create(...);
}

// 2. startSurface 时连接到 NodeContent
void ArkUISurface::attachToNodeContent(NodeContentHandle nodeContentHandle) {
    m_threadGuard.assertThread();

    // 防止重复附加
    if (m_nodeContentHandle.has_value()) {
        throw RNOHError("Surface already attached");
    }

    m_nodeContentHandle = nodeContentHandle;

    // 关键：将 RootView 的 ArkUI 节点添加到 NodeContent
    m_nodeContentHandle.value().addNode(m_rootView->getLocalRootArkUINode());

    HarmonyReactMarker::logMarker(
        HarmonyReactMarkerId::CONTENT_APPEARED,
        m_rootView->getTag()
    );
}
```

**NodeContentHandle 封装**：
```cpp
class NodeContentHandle final {
public:
  // 从 NAPI Value 创建
  static NodeContentHandle fromNapiValue(napi_env env, napi_value value);

  // 节点操作
  NodeContentHandle& addNode(ArkUINode& node);
  NodeContentHandle& insertNode(ArkUINode& node, int32_t index);
  NodeContentHandle& removeNode(ArkUINode& node);

private:
  ArkUI_NodeContentHandle m_contentHandle;

  static void maybeThrow(int32_t status);
};

// 实现
NodeContentHandle& NodeContentHandle::addNode(ArkUINode& node) {
    maybeThrow(
        OH_ArkUI_NodeContent_AddNode(m_contentHandle, node.getArkUINodeHandle())
    );
    return *this;
}

NodeContentHandle NodeContentHandle::fromNapiValue(napi_env env, napi_value value) {
    ArkUI_NodeContentHandle content;
    maybeThrow(OH_ArkUI_GetNodeContentFromNapiValue(env, value, &content));
    return {content};
}
```

**ArkTS 侧实现**：
```typescript
@Component
struct ContentAdaptiveSurface {
  private surfaceHandle!: SurfaceHandle;
  private rootViewNodeContent: NodeContent = new NodeContent();

  aboutToAppear() {
    // 1. 创建或获取 SurfaceHandle
    this.surfaceHandle = this.ctx.rnInstance.createSurface(appKey);

    // 2. 设置 DisplayMode
    this.surfaceHandle.setDisplayMode(DisplayMode.Visible);

    // 3. 附加到 NodeContent（关键步骤）
    this.surfaceHandle.attachRootView(
        this.ctx.rnInstance.getId(),
        this.surfaceHandle.getTag(),
        this.rootViewNodeContent  // C++ 侧会通过此 NodeContent 添加节点
    );
  }

  build() {
    Stack() {
      // 根据架构选择渲染方式
      if (this.ctx.rnInstance.getArchitecture() === "C_API") {
        ContentSlot(this.rootViewNodeContent)  // ContentSlot 接入点
      } else {
        // ArkTS 架构的渲染路径
        RNView({ ctx: this.ctx, tag: this.surfaceHandle.getTag() })
      }
    }
  }
}
```

---

## ComponentInstance 核心架构

### ComponentInstance 基类核心职责

**位置**: `tester/harmony/react_native_openharmony/src/main/cpp/RNOH/ComponentInstance.h`

ComponentInstance 是所有组件实例的**抽象基类**，核心职责包括：

#### 核心职责 1: 组件生命周期管理
```cpp
virtual void onCreate() {}  // 构造后回调
virtual ~ComponentInstance() = default;
```

#### 核心职责 2: 父子关系管理
```cpp
void insertChild(ComponentInstance::Shared childComponentInstance, std::size_t index);
void removeChild(ComponentInstance::Shared childComponentInstance);
virtual void onChildInserted(ComponentInstance::Shared const& childComponentInstance, std::size_t index);
virtual void onChildRemoved(ComponentInstance::Shared const& childComponentInstance);
```

#### 核心职责 3: Props/State/Layout 更新接口
```cpp
// 由 MountingManagerCAPI 调用的更新方法
virtual void setProps(facebook::react::Props::Shared props);
virtual void setState(facebook::react::State::Shared state);
virtual void setLayout(facebook::react::LayoutMetrics layoutMetrics);
virtual void setEventEmitter(facebook::react::SharedEventEmitter eventEmitter);
```

#### 核心职责 4: TouchTarget 接口实现
```cpp
class ComponentInstance : public TouchTarget,
                         public std::enable_shared_from_this<ComponentInstance> {
  // TouchTarget 实现
  Tag getTouchTargetTag() const override;
  facebook::react::SharedTouchEventEmitter getTouchEventEmitter() const override;
  virtual std::vector<TouchTarget::Shared> getTouchTargetChildren() override;
};
```

#### 核心职责 5: ArkUINode 暴露
```cpp
// 必须由子类实现，返回组件的本地根节点
virtual ArkUINode& getLocalRootArkUINode() = 0;
```

### Props 处理流程

**从 JS Props 到 C++ 属性的完整流程**：

```
JS Props (folly::dynamic)
    ↓
React Native ShadowNode
    ↓ PropsParserContext
ConcreteProps (类型安全的 C++ 对象)
    ↓ MountingManagerCAPI::updateComponentWithShadowView
ComponentInstance::setProps()
    ↓ CppComponentInstance::onPropsChanged()
ArkUINode 属性设置方法
    ↓ NativeNodeApi
HarmonyOS Native Node C-API
```

**CppComponentInstance Props 处理示例**：

```cpp
template <typename ShadowNodeT>
class CppComponentInstance : public ComponentInstance {
  using ConcreteProps = typename ShadowNodeT::ConcreteProps;
  SharedConcreteProps m_props;

public:
  void setProps(facebook::react::Props::Shared props) final {
    auto newProps = std::dynamic_pointer_cast<const ConcreteProps>(props);
    this->onPropsChanged(newProps);  // 虚方法，子类可重写
    m_props = newProps;
  }

protected:
  virtual void onPropsChanged(SharedConcreteProps const& concreteProps);
};
```

**ViewProps 处理详细实现**：

```cpp
virtual void onPropsChanged(SharedConcreteProps const& concreteProps) {
  auto props = std::static_pointer_cast<const facebook::react::ViewProps>(concreteProps);
  auto old = std::static_pointer_cast<const facebook::react::ViewProps>(m_props);

  // 1. 背景色处理（增量更新）
  if (old && *(props->backgroundColor) != *(old->backgroundColor)) {
    this->getLocalRootArkUINode().setBackgroundColor(props->backgroundColor);
  }

  // 2. 边框处理
  facebook::react::BorderMetrics borderMetrics =
      props->resolveBorderMetrics(this->m_layoutMetrics);

  if (old && borderMetrics.borderWidths != m_oldBorderMetrics.borderWidths) {
    this->getLocalRootArkUINode().setBorderWidth(borderMetrics.borderWidths);
  }

  // 3. 变换处理（考虑 Animated 管理）
  auto isTransformManagedByAnimated = getIgnoredPropKeys().count("transform") > 0;
  if (!isTransformManagedByAnimated) {
    if (!old) {
      if (props->transform != defaultTransform) {
        this->getLocalRootArkUINode().setTransform(
            props->transform, m_layoutMetrics.pointScaleFactor);
        markBoundingBoxAsDirty();
      }
    }
  }

  // 4. 透明度处理
  this->setOpacity(props);

  // 5. 无障碍属性处理
  this->getLocalRootArkUINode().setAccessibilityState(props->accessibilityState);

  // 6. 指针事件处理
  if (!old || props->pointerEvents != old->pointerEvents) {
    this->getLocalRootArkUINode().setHitTestMode(props->pointerEvents);
    this->getLocalRootArkUINode().setEnabled(
        props->pointerEvents != facebook::react::PointerEventsMode::None);
  }

  // 保存旧值用于增量更新
  m_oldBorderMetrics = borderMetrics;
}
```

### LayoutMetrics 传递到 ArkUI

**LayoutMetrics 结构**：

```cpp
namespace facebook::react {
struct LayoutMetrics {
  Rect frame;                    // {Point origin, Size size}
  Float pointScaleFactor;        // 设备像素比
  LayoutDirection layoutDirection;  // LTR/RTL
};
}
```

**设置流程**：

```cpp
void CppComponentInstance::onLayoutChanged(LayoutMetrics const& layoutMetrics) {
  // 调用 ArkUINode 的 setLayoutRect
  this->getLocalRootArkUINode().setLayoutRect(
      layoutMetrics.frame.origin,
      layoutMetrics.frame.size,
      layoutMetrics.pointScaleFactor);

  // 处理布局方向变化
  if (layoutMetrics.layoutDirection != m_layoutMetrics.layoutDirection) {
    ArkUI_Direction direction = convertLayoutDirection(layoutMetrics.layoutDirection);
    this->getLocalRootArkUINode().setDirection(direction);
  }

  // 标记边界框需要重新计算
  markBoundingBoxAsDirty();
}
```

**setLayoutRect 详细实现**：

```cpp
ArkUINode& ArkUINode::setLayoutRect(
    facebook::react::Point const& position,
    facebook::react::Size const& size,
    facebook::react::Float pointScaleFactor) {

  // 1. 单位转换：dp -> px
  ArkUI_NumberValue value[] = {
    {.i32 = static_cast<int32_t>(position.x * pointScaleFactor + 0.5)},
    {.i32 = static_cast<int32_t>(position.y * pointScaleFactor + 0.5)},
    {.i32 = static_cast<int32_t>(size.width * pointScaleFactor + 0.5)},
    {.i32 = static_cast<int32_t>(size.height * pointScaleFactor + 0.5)}
  };

  // 2. 保存尺寸供后续查询
  saveSize(value[2].i32, value[3].i32);

  // 3. 调用 HarmonyOS C-API
  ArkUI_AttributeItem item = {value, sizeof(value) / sizeof(ArkUI_NumberValue)};
  maybeThrow(NativeNodeApi::getInstance()->setAttribute(
      m_nodeHandle, NODE_LAYOUT_RECT, &item));

  return *this;
}
```

### 关键设计模式

#### RAII (Resource Acquisition Is Initialization)

```cpp
class ArkUINode {
  ArkUI_NodeHandle m_nodeHandle;  // 资源在构造时获取

public:
  ArkUINode(ArkUI_NodeHandle nodeHandle) : m_nodeHandle(nodeHandle) {
    RNOH_ASSERT(nodeHandle != nullptr);
    NODE_BY_HANDLE.emplace(m_nodeHandle, this);  // 注册到全局映射
  }

  virtual ~ArkUINode() noexcept {
    NODE_BY_HANDLE.erase(m_nodeHandle);
    NativeNodeApi::getInstance()->disposeNode(m_nodeHandle);  // 析构时释放
  }
};
```

#### 委托模式

```cpp
// StackNode 委托给 StackNodeDelegate
class StackNodeDelegate {
  virtual void onClick() {}
  virtual void onHoverIn() {}
  virtual void onHoverOut() {}
};

class StackNode : public ArkUINode {
  StackNodeDelegate* m_stackNodeDelegate;

  void onNodeEvent(ArkUI_NodeEventType eventType, EventArgs& eventArgs) override {
    if (eventType == NODE_ON_CLICK) {
      if (m_stackNodeDelegate) {
        m_stackNodeDelegate->onClick();  // 委托处理
      }
    }
  }
};

// ViewComponentInstance 实现 StackNodeDelegate
class ViewComponentInstance : public StackNodeDelegate {
  void onClick() override {
    if (m_eventEmitter) {
      m_eventEmitter->dispatchEvent("click", ...);
    }
  }
};
```

#### 增量更新优化

```cpp
// 只在属性真正变化时才调用 C-API
if (old && *(props->backgroundColor) != *(old->backgroundColor)) {
  // 只有背景色变化时才调用 C-API
  this->getLocalRootArkUINode().setBackgroundColor(props->backgroundColor);
} else if (!old && props->backgroundColor != ARKUI_DEFAULT_BACKGROUND_COLOR) {
  this->getLocalRootArkUINode().setBackgroundColor(props->backgroundColor);
} else {
  // Do nothing here.
}
```

---

## Fabric 渲染流程详解

### 渲染三阶段

**React Native 新架构的渲染流水线分为三个主要阶段**：

```
1. Render (渲染)
   └─ React 创建组件树
   └─ Fiber 调度和协调
   └─ 生成 React Element Tree

2. Commit (提交)
   ├─ ShadowTree 创建
   ├─ ShadowNode 创建
   ├─ Yoga 布局计算 (在 JS 线程)
   └─ ShadowTree 比较

3. Mount (挂载)
   ├─ MountingCoordinator 生成 Mutations
   ├─ MountingManager 处理 Mutations
   ├─ ComponentInstance 创建/更新
   └─ ArkUI 节点操作
```

### Mutation 类型

**MountingManagerCAPI 处理的五种 Mutation**：

| Mutation | 描述 | 操作 |
|----------|------|------|
| CREATE | 创建新组件 | 创建 ComponentInstance 和 ArkUINode |
| UPDATE | 更新组件属性 | 调用 setProps, setLayout |
| INSERT | 插入子组件 | 调用 insertChild |
| REMOVE | 移除子组件 | 调用 removeChild |
| DELETE | 删除组件 | 从注册表中删除 |

### MountingManagerCAPI 实现

**位置**: `tester/harmony/react_native_openharmony/src/main/cpp/RNOH/MountingManagerCAPI.h`

```cpp
class MountingManagerCAPI : public MountingManager {
public:
  // Fabric 生命周期钩子
  void willMount(MutationList const& mutations) override;
  void doMount(MutationList const& mutations) override;
  void didMount(MutationList const& mutations) override;

  // 命令和状态更新
  void dispatchCommand(
      const facebook::react::ShadowView& shadowView,
      const std::string& commandName,
      folly::dynamic const& args) override;

  void updateView(
      facebook::react::Tag tag,
      folly::dynamic props,
      facebook::react::ComponentDescriptor const& componentDescriptor) override;

private:
  // 判断是否为 C-API 组件
  bool isCAPIComponent(facebook::react::ShadowView const& shadowView);

  // 组件名缓存（优化性能）
  std::unordered_set<std::string> m_cApiComponentNames;
  std::unordered_set<std::string> m_arkTsComponentNames;
};
```

**Mutation 处理流程**：
```cpp
void MountingManagerCAPI::didMount(MutationList const& mutations) {
    // 1. 分离 ArkTS 和 C-API mutations
    auto validMutations = getValidMutations(mutations);

    // 2. 先处理 ArkTS mutations
    m_arkTsMountingManager->didMount(validMutations);

    // 3. 清理预分配队列
    m_componentInstanceProvider->clearPreallocationRequestQueue();

    // 4. 逐个处理 C-API mutations
    for (uint64_t i = 0; i < mutations.size(); ++i) {
        auto const& mutation = mutations[i];
        try {
            this->handleMutation(mutation);
        } catch (std::exception const& e) {
            LOG(ERROR) << "Mutation " << getMutationNameFromType(mutation.type)
                       << " failed: " << e.what();
        }
    }
}

void MountingManagerCAPI::handleMutation(Mutation const& mutation) {
    switch (mutation.type) {
        case facebook::react::ShadowViewMutation::Create: {
            auto componentInstance = m_componentInstanceProvider->getComponentInstance(
                mutation.newChildShadowView.tag,
                mutation.newChildShadowView.componentHandle,
                mutation.newChildShadowView.componentName
            );

            if (componentInstance == nullptr) {
                componentInstance = m_componentInstanceProvider->createArkTSComponent(...);
            }

            m_componentInstanceRegistry->insert(componentInstance);
            updateComponentWithShadowView(componentInstance, mutation.newChildShadowView);
            break;
        }

        case facebook::react::ShadowViewMutation::Insert: {
            auto parentComponentInstance = m_componentInstanceRegistry->findByTag(
                mutation.parentShadowView.tag
            );
            auto newChildComponentInstance = m_componentInstanceRegistry->findByTag(
                mutation.newChildShadowView.tag
            );

            if (parentComponentInstance != nullptr && newChildComponentInstance != nullptr) {
                parentComponentInstance->insertChild(
                    newChildComponentInstance,
                    mutation.index
                );
            }
            break;
        }

        case facebook::react::ShadowViewMutation::Update: {
            auto componentInstance = m_componentInstanceRegistry->findByTag(
                mutation.newChildShadowView.tag
            );
            if (componentInstance != nullptr) {
                updateComponentWithShadowView(componentInstance, mutation.newChildShadowView);
            }
            break;
        }

        case facebook::react::ShadowViewMutation::Remove: {
            auto parentComponentInstance = m_componentInstanceRegistry->findByTag(
                mutation.parentShadowView.tag
            );
            if (parentComponentInstance) {
                auto childComponentInstance = m_componentInstanceRegistry->findByTag(
                    mutation.oldChildShadowView.tag
                );
                parentComponentInstance->removeChild(childComponentInstance);
            }
            break;
        }

        case facebook::react::ShadowViewMutation::Delete: {
            m_componentInstanceRegistry->deleteByTag(mutation.oldChildShadowView.tag);
            break;
        }
    }
}
```

**组件更新**：
```cpp
void MountingManagerCAPI::updateComponentWithShadowView(
    ComponentInstance::Shared const& componentInstance,
    facebook::react::ShadowView const& shadowView) {

    // 1. 更新 Tag->Id 映射
    m_componentInstanceRegistry->updateTagById(
        shadowView.tag,
        shadowView.props->nativeId,
        componentInstance->getId()
    );

    // 2. 批量更新所有属性
    componentInstance->setShadowView(shadowView);
    componentInstance->setLayout(shadowView.layoutMetrics);
    componentInstance->setEventEmitter(shadowView.eventEmitter);
    componentInstance->setState(shadowView.state);
    componentInstance->setProps(shadowView.props);
}
```

### ComponentInstance 生命周期

**位置**: `tester/harmony/react_native_openharmony/src/main/cpp/RNOH/ComponentInstance.h`

```cpp
class ComponentInstance {
public:
    struct Context {
        Tag tag;
        ComponentHandle componentHandle;
        std::string componentName;
        Dependencies::Shared dependencies;
        ArkUINode::Context::Shared arkUINodeContext;
    };

    // 1. 创建阶段
    ComponentInstance(Context ctx);
    virtual void onCreate() {}

    // 2. 属性更新阶段
    virtual void setProps(facebook::react::Props::Shared props) {}
    virtual void setState(facebook::react::State::Shared state) {}
    virtual void setLayout(facebook::react::LayoutMetrics layoutMetrics) {}
    virtual void setEventEmitter(facebook::react::SharedEventEmitter eventEmitter) {}

    // 3. 命令处理阶段
    virtual void handleCommand(std::string const& commandName, folly::dynamic const& args) {
        this->onCommandReceived(commandName, args);
    }

    // 4. 子节点管理
    void insertChild(ComponentInstance::Shared childComponentInstance, std::size_t index);
    void removeChild(ComponentInstance::Shared childComponentInstance);

    // 5. 最终化阶段
    virtual void finalizeUpdates() {
        this->onFinalizeUpdates();
    }

    // 6. 销毁阶段
    virtual ~ComponentInstance() = default;

protected:
    // 子类可重写的钩子
    virtual void onChildInserted(ComponentInstance::Shared const& childComponentInstance, std::size_t index) {}
    virtual void onChildRemoved(ComponentInstance::Shared const& childComponentInstance) {}
    virtual void onFinalizeUpdates() {}
    virtual void onCommandReceived(std::string const& commandName, folly::dynamic const& args) {}
};

// 父子关系管理实现
void ComponentInstance::insertChild(
    ComponentInstance::Shared childComponentInstance,
    std::size_t index) {

    // 1. 设置父子关系
    childComponentInstance->setParent(shared_from_this());
    childComponentInstance->setIndex(index);

    // 2. 插入到子节点列表
    m_children.insert(m_children.begin() + index, childComponentInstance);

    // 3. 通知子类
    this->onChildInserted(childComponentInstance, index);

    // 4. 更新 ArkUI 节点树
    auto& childArkUINode = childComponentInstance->getLocalRootArkUINode();
    this->getLocalRootArkUINode().insertChild(childArkUINode, index);
}
```

### ComponentJSIBinder 绑定机制

**位置**: `tester/harmony/react_native_openharmony/src/main/cpp/RNOH/BaseComponentJSIBinder.h`

```cpp
class BaseComponentJSIBinder : public ComponentJSIBinder {
public:
    facebook::jsi::Object createBindings(facebook::jsi::Runtime& rt) override {
        facebook::jsi::Object baseManagerConfig(rt);

        // 1. NativeProps 定义
        baseManagerConfig.setProperty(rt, "NativeProps", this->createNativeProps(rt));

        // 2. Constants 定义
        baseManagerConfig.setProperty(rt, "Constants", this->createConstants(rt));

        // 3. Commands 定义
        baseManagerConfig.setProperty(rt, "Commands", this->createCommands(rt));

        // 4. 事件类型定义
        baseManagerConfig.setProperty(
            rt, "bubblingEventTypes", this->createBubblingEventTypes(rt)
        );
        baseManagerConfig.setProperty(
            rt, "directEventTypes", this->createDirectEventTypes(rt)
        );

        return baseManagerConfig;
    }

protected:
    virtual facebook::jsi::Object createNativeProps(facebook::jsi::Runtime& rt) {
        facebook::jsi::Object nativeProps(rt);

        // 基础属性
        nativeProps.setProperty(rt, "nativeID", true);
        nativeProps.setProperty(rt, "opacity", "number");
        nativeProps.setProperty(rt, "backgroundColor", "Color");

        // 布局属性
        nativeProps.setProperty(rt, "width", true);
        nativeProps.setProperty(rt, "height", true);

        // 可访问性属性
        nativeProps.setProperty(rt, "accessibilityLabel", "string");
        nativeProps.setProperty(rt, "accessibilityRole", true);
        nativeProps.setProperty(rt, "accessibilityState", true);

        return nativeProps;
    }

    virtual facebook::jsi::Object createBubblingEventTypes(facebook::jsi::Runtime& rt) {
        facebook::jsi::Object events(rt);

        // 触摸事件（冒泡 + 捕获）
        events.setProperty(
            rt, "topTouchStart",
            createBubblingCapturedEvent(rt, "onTouchStart")
        );
        events.setProperty(
            rt, "topTouchMove",
            createBubblingCapturedEvent(rt, "onTouchMove")
        );

        return events;
    }

    virtual facebook::jsi::Object createDirectEventTypes(facebook::jsi::Runtime& rt) {
        facebook::jsi::Object events(rt);

        // 直接事件（不冒泡）
        events.setProperty(rt, "topLayout", createDirectEvent(rt, "onLayout"));
        events.setProperty(rt, "topClick", createDirectEvent(rt, "onClick"));

        return events;
    }
};
```

---

## TurboModule 架构详解

### TurboModule 类型对比

| 特性 | cxxTurboModule | ArkTSTurboModule |
|------|----------------|------------------|
| **实现语言** | 纯 C++ | ArkTS |
| **系统 API 依赖** | 无 | 依赖 HarmonyOS API |
| **性能** | 最优（无跨语言调用） | 较优（N-API 桥接） |
| **适用场景** | 动画、计算、数据处理 | 网络、文件、系统能力 |
| **线程模型** | JS 线程 | MAIN/WORKER 线程 |

### TurboModuleFactory 注册机制

**位置**: `tester/harmony/react_native_openharmony/src/main/cpp/RNOH/TurboModuleFactory.h`

```cpp
class TurboModuleFactory {
public:
    struct ArkTSTurboModuleEnvironment {
        napi_env napiEnv;
        NapiRef arkTSTurboModuleProviderRef;
    };

    using SharedTurboModule = std::shared_ptr<facebook::react::TurboModule>;

    SharedTurboModule create(
        std::shared_ptr<facebook::react::CallInvoker> jsInvoker,
        const std::string& name,
        std::shared_ptr<EventDispatcher> eventDispatcher,
        std::shared_ptr<MessageQueueThread> jsQueue,
        std::shared_ptr<facebook::react::Scheduler> scheduler,
        RNInstance::SafeWeak instance
    ) const;

private:
    std::vector<std::shared_ptr<TurboModuleFactoryDelegate>> m_delegates;
    std::unordered_map<TaskThread, ArkTSTurboModuleEnvironment>
        m_arkTSTurboModuleEnvironmentByTaskThread;
};

// 创建流程
SharedTurboModule TurboModuleFactory::create(
    std::shared_ptr<facebook::react::CallInvoker> jsInvoker,
    const std::string& name,
    ...
) const {
    // 1. 确定 TurboModule 运行线程
    auto arkTSTurboModuleThread =
        this->findArkTSTurboModuleThread(name).value_or(TaskThread::JS);

    // 2. 构建上下文
    Context ctx{
        .jsInvoker = jsInvoker,
        .instance = instance.lock(),
        .safeInstance = instance,
        .arkTSMessageHub = m_arkTSMessageHub,
        .env = arkTSTurboModuleEnvironment.napiEnv,
        .arkTSTurboModuleInstanceRef =
            arkTSTurboModuleThread == TaskThread::JS
            ? NapiRef{}
            : this->maybeGetArkTsTurboModuleInstanceRef(name, ...),
        .turboModuleThread = arkTSTurboModuleThread,
        .taskExecutor = m_taskExecutor,
        .eventDispatcher = eventDispatcher,
        .jsQueue = jsQueue,
        .scheduler = scheduler
    };

    // 3. UIManager 特殊处理
    if (name == "UIManager") {
        return std::make_shared<UIManagerModule>(
            ctx, name, std::move(m_componentBinderByString)
        );
    }

    // 4. 委托给其他 delegates
    auto result = this->delegateCreatingTurboModule(ctx, name);

    // 5. 验证 ArkTS 和 C++ 双侧实现
    auto arkTSTurboModule = std::dynamic_pointer_cast<const ArkTSTurboModule>(result);
    if (arkTSTurboModule != nullptr && !ctx.arkTSTurboModuleInstanceRef) {
        throw FatalRNOHError(
            "Couldn't find Turbo Module '" + name + "' on the ArkTS side.",
            {"Have you linked a package that provides this turbo module?"}
        );
    }

    return result;
}
```

### ArkTSTurboModule 实现机制

**位置**: `tester/harmony/react_native_openharmony/src/main/cpp/RNOH/ArkTSTurboModule.h`

```cpp
class ArkTSTurboModule : public TurboModule {
public:
    struct Context : public TurboModule::Context {
        napi_env env;
        NapiRef arkTSTurboModuleInstanceRef;
        TaskThread turboModuleThread;
        TaskExecutor::Shared taskExecutor;
        // ...
    };

    // 同步调用（阻塞等待结果）
    folly::dynamic callSync(
        const std::string& methodName,
        std::vector<ArkJS::IntermediaryArg> args);

    // 异步调用（忽略结果）
    void scheduleCall(
        facebook::jsi::Runtime& runtime,
        const std::string& methodName,
        const facebook::jsi::Value* args,
        size_t argsCount);

    // Promise 调用
    facebook::jsi::Value callAsync(
        facebook::jsi::Runtime& runtime,
        const std::string& methodName,
        const facebook::jsi::Value* args,
        size_t argsCount);
};

// 同步调用实现
folly::dynamic ArkTSTurboModule::callSync(
    const std::string& methodName,
    std::vector<IntermediaryArg> args) {

    folly::dynamic result;

    // 在目标线程同步执行
    m_ctx.taskExecutor->runSyncTask(
        m_ctx.turboModuleThread,
        [ctx = m_ctx, &methodName, &args, &result]() {
            ArkJS arkJS(ctx.env);

            // 1. 转换参数
            auto napiArgs = arkJS.convertIntermediaryValuesToNapiValues(std::move(args));

            // 2. 获取 ArkTS TurboModule 实例
            auto napiTurboModuleObject = arkJS.getObject(ctx.arkTSTurboModuleInstanceRef);

            // 3. 调用 ArkTS 方法
            auto napiResult = napiTurboModuleObject.call(methodName, napiArgs);

            // 4. 转换结果
            result = arkJS.getDynamic(napiResult);
        }
    );

    return result;
}

// 异步 Promise 调用
jsi::Value ArkTSTurboModule::callAsync(
    jsi::Runtime& runtime,
    const std::string& methodName,
    const jsi::Value* jsiArgs,
    size_t argsCount) {

    return react::createPromiseAsJSIValue(
        runtime,
        [&, args = convertJSIValuesToIntermediaryValues(...)](
            jsi::Runtime& runtime2,
            std::shared_ptr<react::Promise> jsiPromise) mutable {

            react::LongLivedObjectCollection::get(runtime2).add(jsiPromise);

            m_ctx.taskExecutor->runTask(
                m_ctx.turboModuleThread,
                [name = this->name_, methodName, args = std::move(args),
                 env = m_ctx.env, weakJsiPromise = std::weak_ptr(jsiPromise)]() mutable {

                    ArkJS arkJS(env);

                    // 调用 ArkTS 返回 Promise
                    auto n_promisedResult = arkJS.getObject(arkTSTurboModuleInstanceRef)
                        .call(methodName, arkJS.convertIntermediaryValuesToNapiValues(std::move(args)));

                    // 处理 Promise 结果
                    Promise(env, n_promisedResult).then([&runtime2, weakJsiPromise](auto args) {
                        auto jsiPromise = weakJsiPromise.lock();
                        if (jsiPromise) {
                            jsiPromise->resolve(preparePromiseResolverResult(runtime2, args));
                            jsiPromise->allowRelease();
                        }
                    }).catch([&runtime2, weakJsiPromise](auto error) {
                        // 错误处理
                    });
                }
            );
        }
    );
}
```

### TurboModule 调用链

#### 完整调用流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                        JavaScript Layer                          │
│  NativeSampleTurboModule.pushStringToHarmony("hello", eventId)   │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│                       JSI Runtime                                │
│  __turboModuleProxy("SampleTurboModule")                        │
│    → TurboModule.get("pushStringToHarmony")                     │
│    → HostFunction invoker                                        │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│                    TurboModule C++                               │
│  ArkTSTurboModule::call(runtime, "pushStringToHarmony", args)   │
│    → convertJSIValuesToIntermediaryValues()                      │
│    → callSync("pushStringToHarmony", intermediaryArgs)           │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│                   TaskExecutor                                   │
│  runSyncTask(TaskThread::MAIN, lambda)                          │
│    → NapiTaskRunner::runSyncTask()                              │
│    → 在主线程上执行 lambda                                        │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│                      ArkJS Layer                                 │
│  ArkJS(env).convertIntermediaryValuesToNapiValues(args)         │
│    → folly::dynamic → napi_value 转换                            │
│    → 处理 IntermediaryCallback                                    │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│                   N-API Layer                                    │
│  napi_call_function(                                            │
│    turboModuleObject,                                           │
│    "pushStringToHarmony",                                        │
│    napiArgs                                                     │
│  )                                                              │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│                  ArkTS TurboModule                               │
│  SampleTurboModule.pushStringToHarmony(arg, id)                 │
│    → emitter.emit({eventId: id}, {data})                        │
│    → HarmonyOS Event Emitter                                    │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         ↓
┌─────────────────────────────────────────────────────────────────┐
│              HarmonyOS Native API                                │
│  @ohos.events.emitter.emit()                                    │
└─────────────────────────────────────────────────────────────────┘
```

#### 同步调用详解

```cpp
// 1. JSI 入口点 - call 方法
jsi::Value ArkTSTurboModule::call(
    jsi::Runtime& runtime,
    const std::string& methodName,
    const jsi::Value* jsiArgs,
    size_t argsCount) {
  // 将 JSI Value 转换为 IntermediaryArg
  auto args = convertJSIValuesToIntermediaryValues(
      runtime, m_ctx.jsInvoker, jsiArgs, argsCount);
  // 调用同步方法
  return jsi::valueFromDynamic(runtime, callSync(methodName, std::move(args)));
}

// 2. 同步调用实现 - callSync
folly::dynamic ArkTSTurboModule::callSync(
    const std::string& methodName,
    std::vector<IntermediaryArg> args) {
  folly::dynamic result;
  m_ctx.taskExecutor->runSyncTask(
      m_ctx.turboModuleThread, [ctx = m_ctx, &methodName, &args, &result]() {
        ArkJS arkJS(ctx.env);
        // 转换 IntermediaryArg 到 napi_value
        auto napiArgs = arkJS.convertIntermediaryValuesToNapiValues(std::move(args));
        // 获取 ArkTS TurboModule 对象
        auto napiTurboModuleObject = arkJS.getObject(ctx.arkTSTurboModuleInstanceRef);
        // 调用 ArkTS 方法
        auto napiResult = napiTurboModuleObject.call(methodName, napiArgs);
        // 转换结果为 folly::dynamic
        result = arkJS.getDynamic(napiResult);
      });
  return result;
}
```

#### 异步调用详解

```cpp
jsi::Value ArkTSTurboModule::callAsync(
    jsi::Runtime& runtime,
    const std::string& methodName,
    const jsi::Value* jsiArgs,
    size_t argsCount) {
  auto args = convertJSIValuesToIntermediaryValues(
      runtime, m_ctx.jsInvoker, jsiArgs, argsCount);

  // 创建 Promise
  return react::createPromiseAsJSIValue(
      runtime,
      [&, args = std::move(args)](
          jsi::Runtime& runtime2,
          std::shared_ptr<react::Promise> jsiPromise) mutable {
        react::LongLivedObjectCollection::get(runtime2).add(jsiPromise);

        // 在 TurboModuleThread 上异步执行
        m_ctx.taskExecutor->runTask(
            m_ctx.turboModuleThread,
            [name = this->name_,
             methodName,
             args = std::move(args),
             env = m_ctx.env,
             arkTSTurboModuleInstanceRef = m_ctx.arkTSTurboModuleInstanceRef,
             jsInvoker = m_ctx.jsInvoker,
             &runtime2,
             weakJsiPromise = std::weak_ptr<react::Promise>(jsiPromise)]() mutable {
              ArkJS arkJS(env);
              try {
                // 调用 ArkTS 方法返回 Promise
                auto n_promisedResult =
                    arkJS.getObject(arkTSTurboModuleInstanceRef)
                        .call(methodName, arkJS.convertIntermediaryValuesToNapiValues(std::move(args)));

                // 处理 Promise 结果
                Promise(env, n_promisedResult)
                    .then([&runtime2, weakJsiPromise, env, jsInvoker](auto args) {
                      jsInvoker->invokeAsync([&runtime2, weakJsiPromise, args = std::move(args)]() {
                        auto jsiPromise = weakJsiPromise.lock();
                        if (!jsiPromise) return;
                        jsiPromise->resolve(preparePromiseResolverResult(runtime2, args));
                        jsiPromise->allowRelease();
                      });
                    })
                    .catch_([&runtime2, weakJsiPromise, env, jsInvoker](auto args) {
                      jsInvoker->invokeAsync([&runtime2, weakJsiPromise, args = std::move(args)]() {
                        auto jsiPromise = weakJsiPromise.lock();
                        if (!jsiPromise) return;
                        jsiPromise->reject(preparePromiseRejectionResult(args));
                        jsiPromise->allowRelease();
                      });
                    });
              } catch (const std::exception& e) {
                jsInvoker->invokeAsync([message = std::string(e.what()), weakJsiPromise] {
                  auto jsiPromise = weakJsiPromise.lock();
                  if (!jsiPromise) return;
                  jsiPromise->reject(message);
                  jsiPromise->allowRelease();
                });
              }
            });
      });
}
```

#### 调用方式对比

| 特性 | 同步调用 | 异步调用 (Promise) | Fire-and-Forget |
|------|------------------------|-------------------|-----------------|
| **方法** | `call()` / `callSync()` | `callAsync()` | `scheduleCall()` |
| **返回类型** | 直接返回结果 | 返回 Promise | undefined |
| **线程模型** | 阻塞等待结果 | 非阻塞，Promise 回调 | 非阻塞，无回调 |
| **TaskExecutor** | `runSyncTask()` | `runTask()` | `runTask()` |
| **用途** | 简单 getter | 耗时操作、网络请求 | 不需要结果的调用 |
| **性能** | 较慢（阻塞） | 快（非阻塞） | 最快（无回调处理） |

---

## 关键文件路径

### 项目结构

```
/Users/wuzhao/Desktop/ty/rnoh/ohos_react_native/
│
├── react-native-harmony/              # 核心 SDK 包 (NPM)
│   ├── index.js                       # 主入口
│   ├── Libraries/                     # JS 层核心库
│   │   ├── Components/                # UI 组件 (36个 .harmony.js)
│   │   │   ├── View/View.harmony.js
│   │   │   ├── Text/Text.harmony.js
│   │   │   ├── Image/Image.harmony.js
│   │   │   ├── ScrollView/
│   │   │   ├── TextInput/
│   │   │   ├── SafeAreaView/
│   │   │   └── Animated/
│   │   ├── Core/                      # 核心功能
│   │   ├── Renderer/                  # 渲染器
│   │   ├── StyleSheet/                # 样式系统
│   │   ├── Utilities/                 # 工具函数
│   │   ├── Alert/                     # Alert API
│   │   ├── Settings/                  # Settings API
│   │   ├── Vibration/                 # Vibration API
│   │   └── Share/                     # Share API
│   ├── types/                         # TypeScript 类型定义
│   ├── harmony/                       # HarmonyOS 相关
│   │   └── rnoh-hvigor-plugin-*.tgz
│   └── metro.config.js                # Metro 配置
│
├── react-native-harmony-cli/          # CLI 工具
│   ├── src/
│   │   ├── commands/                  # 命令实现
│   │   │   ├── init/                  # 项目初始化
│   │   │   ├── run-harmony/           # 运行应用
│   │   │   └── bundle/                # 打包命令
│   │   ├── codegen/                   # Codegen 代码生成器
│   │   ├── autolinking/               # 自动链接功能
│   │   ├── core/                      # 核心逻辑
│   │   └── simulator/                 # 模拟器管理
│   └── package.json                   # 版本 0.0.40
│
├── react-native-harmony-hvigor-plugin/ # Hvigor 构建插件
│
├── react-native-harmony-sample-package/ # 示例原生模块包
│
├── tester/                            # 测试应用
│   ├── harmony/                       # HarmonyOS 测试项目
│   │   ├── entry/                     # 主模块
│   │   │   ├── src/main/
│   │   │   │   ├── cpp/               # C++ 原生代码
│   │   │   │   ├── ets/               # ArkTS 代码
│   │   │   │   └── resources/         # 资源文件
│   │   ├── react_native_openharmony/  # RNOH 核心库
│   │   │   └── src/main/
│   │   │       ├── cpp/               # C++ 实现 (~1073 文件)
│   │   │       │   ├── RNOH/          # 核心框架
│   │   │       │   ├── RNOHCorePackage/ # 核心包实现
│   │   │       │   └── patches/       # React Native Core 补丁
│   │   │       └── ets/               # ArkTS 实现 (~51 文件)
│   │   │           ├── RNOH/          # 核心框架
│   │   │           └── RNOHCorePackage/ # 核心包
│   │   ├── sample_package/            # 示例原生模块包
│   │   ├── multi_surface/             # 多 Surface 测试
│   │   └── performance_measurement/   # 性能测试
│   ├── components/                    # 测试组件
│   ├── examples/                      # 示例代码
│   ├── jests/                         # Jest 测试
│   └── benchmarks/                    # 性能基准测试
│
├── template/                          # 项目模板
│
└── docs/                              # 文档
    ├── zh-cn/                         # 中文文档
    │   ├── 环境搭建.md
    │   ├── 架构介绍.md
    │   ├── 渲染三阶段.md
    │   ├── 常见开发场景.md
    │   ├── RN升级需要开发者适配整理.md
    │   └── faqs/
    └── en/                             # 英文文档
```

### 核心 C++ 文件

```
tester/harmony/react_native_openharmony/src/main/cpp/
│
├── RNOH/                              # 核心框架
│   ├── arkui/                         # ArkUI C-API 封装
│   │   ├── ArkUINode.cpp/h            # ArkUI 节点基类
│   │   ├── StackNode.cpp/h            # Stack 节点
│   │   ├── ImageNode.cpp/h            # Image 节点
│   │   ├── TextNode.cpp/h             # Text 节点
│   │   ├── TextInputNode.cpp/h        # TextInput 节点
│   │   ├── ScrollViewNode.cpp/h       # ScrollView 节点
│   │   ├── RefreshNode.cpp/h          # Refresh 节点
│   │   ├── ColumnNode.cpp/h           # Column 节点
│   │   ├── CustomNode.cpp/h           # 自定义节点
│   │   ├── ArkUISurface.cpp/h         # Surface 管理
│   │   ├── ArkUINodeRegistry.cpp/h    # 节点注册表
│   │   └── NodeContentHandle.cpp/h    # NodeContent 句柄
│   ├── TaskExecutor/                  # 任务执行器
│   │   └── uv/                        # libuv 事件循环
│   ├── react-native-jsvm/             # JS VM 集成
│   ├── executor/                      # 执行器
│   ├── Performance/                   # 性能监控
│   ├── ArkJS.cpp/h                    # ArkJS JSI 封装
│   ├── ArkTSBridge.cpp/h              # ArkTS 桥接
│   ├── ArkTSTurboModule.cpp/h         # TurboModule 基类
│   ├── RNInstanceCAPI.cpp/h           # C-API 实例实现
│   ├── RNInstanceArkTS.cpp/h          # ArkTS 实例实现 (已废弃)
│   ├── MountingManagerCAPI.cpp/h      # C-API 挂载管理器
│   ├── ComponentInstance.cpp/h        # 组件实例基类
│   ├── CppComponentInstance.h         # C++ 组件实例
│   ├── TurboModuleFactory.cpp/h       # TurboModule 工厂
│   ├── Package.cpp/h                  # 包定义
│   └── TextMeasurer.cpp/h             # 文本测量
│
├── RNOHCorePackage/                   # 核心包实现
│   ├── ComponentBinders/              # 组件绑定器 (19个)
│   │   ├── ViewComponentJSIBinder.h
│   │   ├── TextComponentJSIBinder.h
│   │   ├── ImageComponentJSIBinder.h
│   │   ├── ScrollViewComponentJSIBinder.h
│   │   └── ...
│   ├── ComponentInstances/            # 组件实例实现
│   │   ├── ActivityIndicatorComponentInstance.cpp/h
│   │   ├── ImageComponentInstance.cpp/h
│   │   ├── ScrollViewComponentInstance.cpp/h
│   │   ├── TextComponentInstance.cpp/h
│   │   └── ...
│   ├── TurboModules/                  # TurboModule 实现
│   │   ├── Animated/                  # 动画模块
│   │   ├── Networking/                # 网络模块
│   │   ├── Blob/                      # Blob 模块
│   │   └── ...
│   ├── EventEmitRequestHandlers/      # 事件发送处理器
│   └── GlobalBinders/                 # 全局绑定器
│
└── patches/                           # React Native Core 补丁
    └── react_native_core/
        └── react/renderer/
            ├── components/            # 组件补丁
            │   ├── textinput/
            │   ├── text/
            │   └── rncore/
            ├── core/                  # 渲染核心
            ├── mounting/              # 挂载系统
            └── uimanager/             # UI 管理器
```

### 核心 ArkTS 文件

```
tester/harmony/react_native_openharmony/src/main/ets/
│
├── RNOH/                              # 核心框架
│   ├── RNAbility.ets                  # RN Ability 基类
│   ├── RNOHPackage.ets                # RNOH 包定义
│   ├── RNInstancesCoordinator.ets     # 多实例协调器
│   ├── RNInstanceRegistry.ets         # 实例注册表
│   ├── RNInstance.ets                 # RN 实例定义
│   ├── RNDevLoadingView.ets           # 开发加载视图
│   ├── DevMenu.ets                    # 开发者菜单
│   ├── RNSurface.ets                  # RN Surface 容器
│   ├── RNOHApp.ets                    # RNOH 应用
│   ├── RNComponentDataSource.ets      # 组件数据源
│   ├── EtsRNOHContext.ets             # ArkTS RNOH 上下文
│   ├── TaskRunner.ets                 # 任务运行器
│   ├── UITaskRunner.ets               # UI 任务运行器
│   ├── WorkerTaskRunner.ets           # Worker 任务运行器
│   ├── EtsUITurboModule.ets           # ArkTS UI TurboModule 基类
│   ├── EtsWorkerTurboModule.ets       # ArkTS Worker TurboModule 基类
│   ├── TouchDispatcher.ets            # 触摸事件分发器
│   └── CustomRNComponentFrameNodeFactory.ets  # 自定义组件工厂
│
└── RNOHCorePackage/                   # 核心包
    ├── componentManagers/             # 组件管理器
    ├── turboModules/                  # TurboModule
    │   ├── Networking/
    │   │   ├── Http.ets
    │   │   └── XMLHttpRequest.ets
    │   └── Blob/
    │       └── Blob.ets
    └── components/                    # 组件实现
        ├── RNViewBase/
        ├── RNScrollView/
        ├── RNText/
        └── ...
```

---

## React Native 版本迁移

### 从 RN 0.51/0.59 迁移到 RNOH 的关键变更

#### 1. TurboModule 替代 NativeModules

**旧版本 (RN 0.51/0.59)**：
```javascript
import { NativeModules } from 'react-native';
const { SampleModule } = NativeModules;
const result = SampleModule.method();
```

**新版本 (RNOH 0.72)**：
```javascript
import { TurboModuleRegistry } from 'react-native';
const SampleModule = TurboModuleRegistry.getEnforcing('SampleModule');
const result = SampleModule.method();
```

#### 2. 事件监听器移除方法变更

**旧版本**：
```javascript
DeviceEventEmitter.removeEventListener('event', handler);
```

**新版本**：
```javascript
const subscription = DeviceEventEmitter.addListener('event', handler);
// 需要移除时
subscription.remove();
```

#### 3. setNativeProps 废弃

**旧版本**：
```javascript
component.setNativeProps({ style: { opacity: 0.5 } });
```

**新版本**：
```javascript
// 使用 state 管理
const [opacity, setOpacity] = useState(1);
<View style={{ opacity }} />
```

**注意**：在 Fabric 架构下，`setNativeProps` 只生效一次，不应该再使用。

#### 4. 日期格式变更

**旧版本**：
```javascript
// 可能报错
new Date('2023/08/08 00:00:00').getTime();
```

**新版本**：
```javascript
// 使用 ISO 8601 格式
new Date('2023-08-08T00:00:00').getTime();
```

#### 5. OpenHarmony 平台判断

**新 API**：
```javascript
import { PlatformUtils } from 'react-native';

if (PlatformUtils.isHarmony()) {
  // OpenHarmony 特定代码
}
```

#### 6. 文件后缀

- iOS: `.ios.tsx`
- Android: `.android.tsx`
- **OpenHarmony: `.harmony.tsx`**

#### 7. 组件导入路径

```javascript
// 使用鸿蒙专用组件
import { View } from 'react-native';
// 自动使用 View.harmony.js (如果存在)
```

### RNOH 5.0.0.500 破坏性变更

| 变更项 | 旧版本 | 新版本 (5.0.0.500+) |
|--------|--------|---------------------|
| 字体配置 | `fontOptions` 数组 | `fontResourceByFontFamily` map |
| 字体目录 | `src/main/resources/rawfile/assets/assets/fonts/` | `src/main/resources/rawfile/fonts/` |
| 字体优先级 | `fontFamily` 属性 > 主题字体 | 主题字体 > `fontFamily` 属性 |
| Half Leading | 可选 | 设置 `lineHeight` 时强制启用 |

**Half Leading 影响**：
- 改变文本垂直居中行为
- 多行文本的垂直对齐可能发生变化
- 需要重新调整文本相关的布局

### Fabric 架构差异

| 方面 | 旧架构 | Fabric 架构 |
|------|--------|-------------|
| 渲染系统 | UIManager | Fabric UIManager |
| 组件注册 | NativeViewHierarchyManager | ComponentDescriptor |
| 布局计算 | 单线程 | 支持并行 |
| CSS 规范 | 相对宽松 | 更严格 |
| 性能 | 较慢 | 更快 |

**CSS 规范变更影响**：
- zIndex 需要配合正确的定位使用
- flexbox 规则更严格
- 百分比高度要求父容器有明确高度

---

## 已知迁移问题与解决方案

### 布局和渲染问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| zIndex 失效 | Fabric 版本规范更严格 | 检查定位属性是否正确设置，使用 `get_ui_tree` 查看 `drawRegion` |
| 布局错乱 | Yoga 集成差异 | 检查 flexbox 属性，验证父容器高度 |
| 手势异常 | 事件分发机制变更 | 检查事件监听器注册，使用新的事件 API |
| Animated 卡顿 | 架构差异 | 使用 `NativeAnimatedTurboModule` (cxx 实现) |
| 组件不显示 | ContentSlot 接入问题 | 检查 `attachToNodeContent` 调用 |

### 常见错误

1. **"Unbound variable" 错误**
   - 检查 TurboModule 是否正确注册
   - 验证 Codegen 生成的代码

2. **事件不触发**
   - 使用新的 `addListener` + `subscription.remove()` 模式
   - 检查事件名称是否正确

3. **样式不生效**
   - 检查是否使用了废弃的 `setNativeProps`
   - 验证样式属性名称符合规范

---

## 开发与调试

### 环境要求

| 工具 | 版本要求 |
|------|----------|
| DevEco Studio | 5.0.3.706+ |
| HarmonyOS SDK | 5.0.0.61 (SP1) |
| Node.js | 16.x+ |
| 华为开发者账号 | 用于签名 |

### 构建流程

#### 1. 本地安装 CLI

```bash
cd /Users/wuzhao/Desktop/ty/rnoh/ohos_react_native/react-native-harmony-cli
npm install
npm pack

cd ../react-native-harmony-sample-package
npm install
npm pack

cd ../tester
npm run i
```

#### 2. 启动 Metro

```bash
npm start

# 在另一个终端设置端口转发
hdc rport tcp:8081 tcp:8081
```

#### 3. 在 DevEco Studio 中构建

1. 打开 `tester/harmony` 项目
2. 完成签名配置
3. 运行到设备/模拟器

### 调试工具

#### Metro 热更新
- 通过 `MetroJSBundleProvider` 实现
- 支持开发时的快速迭代

#### React DevTools
- 已集成到 RNOH
- 可以检查组件树和 props

#### 性能追踪
- 内置 tracing 功能
- 可以分析性能瓶颈

#### 错误对话框
- 调试模式下可用
- 显示详细的错误信息

#### Bundle 加载
- 支持 Metro（开发）
- 支持文件（生产）
- 支持资源包

### 鸿蒙 UI 调试 (MCP 工具)

使用 `harmonyos-ui` MCP 服务器调试 UI 问题：

```bash
# 检查前台应用
list_windows

# 获取 UI 树结构
get_ui_tree [pid]

# 列出 Ability 状态
list_abilities

# 截图
screenshot [outputPath]
```

**使用示例**：

1. **组件不显示问题**：
```bash
# 1. 确认前台应用
list_windows

# 2. 查看 UI 树
get_ui_tree

# 3. 检查组件是否在树中
# 4. 检查 drawRegion 是否正确
```

2. **布局问题**：
```bash
# 1. 获取 UI 树
get_ui_tree <pid>

# 2. 查看 drawRegion 和 layout 属性
# 3. 验证 Yoga 布局计算结果
```

---

## 事件处理与手势

### 事件分发流程

```
1. 注册阶段
   └─ 组件通过 props 注册事件监听器
       ↓

2. 接收阶段
   └─ ArkUI 事件通过回调到达 C++ 层
       ↓

3. 处理阶段
   ├─ 触摸事件 → TouchDispatcher
   └─ 组件事件 → EventEmitter
       ↓

4. 传播阶段
   └─ 事件通过 event emitters 发送到 JS
```

### 触摸事件类型

- `DOWN` - 手指按下
- `MOVE` - 手指移动
- `UP` - 手指抬起
- `CANCEL` - 触摸取消

### 组件事件类型

- `Click` - 点击事件
- `Hover` - 悬停事件
- `Focus` - 焦点事件
- `Change` - 值变化事件

**注意**：触摸事件在鸿蒙触摸屏设备上可能被禁用以避免与系统手势冲突。

### 手势系统

RNOH 的手势系统遵循新的 React Native 模式：

```javascript
// 使用新的手势系统
import { GestureDetector, Gesture } from 'react-native';

const tap = Gesture.Tap()
  .onStart(() => {
    console.log('Tap!');
  });

<GestureDetector gesture={tap}>
  <View />
</GestureDetector>
```

---

## 性能优化

### C-API 优势

1. **最小化 C 端**
   - 减少跨语言调用
   - 直接使用原生 API

2. **无数据转换**
   - 直接使用原始数据类型
   - string、enum 不需转换

3. **属性 Diff**
   - 避免重复设置相同属性
   - 减少不必要的 UI 更新

4. **组件预创建**
   - 子线程提前创建组件
   - 减少主线程阻塞

### 并行化特性

#### ShadowTree 并行化 (实验性)

**配置**：
```cpp
#define PARALLELIZATION_ENABLE 1
```

**效果**：
- JS 层并行化
- C++ 层并行化
- 需要充分测试稳定性

#### Worker 线程

**用途**：
- TurboModule 独立线程执行
- 避免阻塞主线程 UI 绘制
- 提升应用响应性

**适用场景**：
- 网络请求
- 文件操作
- 复杂计算

### LazyForEach 渲染优化

**位置**: `RNOHCorePackage/components/RNListComponent.ets`

**核心实现**：
```typescript
LazyForEach(this.ctx.createComponentDataSource({ tag: this.tag }),
  (descriptorWrapper: DescriptorWrapper) => {
    (this.ctx as RNComponentContext).wrappedRNComponentBuilder.builder(
      (this.ctx as RNComponentContext),
      descriptorWrapper.tag
    )
  },
  (descriptorWrapper: DescriptorWrapper) =>
    descriptorWrapper.tag.toString() + "@" + descriptorWrapper.renderKey
)
```

**优势**：
- **仅渲染可见区域组件**：大幅减少内存占用
- **基于渲染键差量更新**：避免不必要的重新渲染
- **自动回收不可见组件**：内存管理自动化

**关键点**：
- `createComponentDataSource` 创建懒加载数据源
- `tag + "@" + renderKey` 作为唯一标识，确保正确复用
- 适用于长列表场景，如 FlatList、SectionList

### 事件计数机制

**位置**: `RNOHCorePackage/components/RNTextInput/`

**问题**：JS 端和 Native 端状态可能不一致

**解决方案**：
```typescript
// 事件计数检查
canUpdateWithEventCount(eventCount: number | undefined): boolean {
  return eventCount !== undefined && eventCount >= this.nativeEventCount;
}

// 设置值时检查事件计数
maybeSetValue(newValue: string, mostRecentEventCount?: number): void {
  if (this.canUpdateWithEventCount(mostRecentEventCount)) {
    this.value = newValue;
  }
}

// 粘贴处理
onPaste(): void {
  this.textWasPastedOrCut = true;
}

// 按键事件分发
dispatchKeyEvent(currentSelection: Selection): void {
  if (this.textWasPastedOrCut) {
    this.textWasPastedOrCut = false;
  } else if (this.valueChanged) {
    const noPreviousSelection = this.selection.start === this.selection.end;
    const cursorDidNotMove = currentSelection.start === this.selection.start;
    const cursorMovedBackwardsOrAtBeginning =
      (currentSelection.start < this.selection.start) || currentSelection.start <= 0;

    if (!cursorMovedBackwardsOrAtBeginning && (noPreviousSelection || !cursorDidNotMove)) {
      const key = this.value.charAt(currentSelection.start - 1);
      this.ctx.rnInstance.emitComponentEvent(this.descriptor.tag, "onKeyPress",
        this.createTextInputKeyEvent(key));
    }
  }
}
```

**关键点**：
- 使用 `eventCount` 避免竞态条件
- 区分粘贴、删除和普通输入
- 只在真正需要时触发 onKeyPress 事件

### 属性更新优化

**位置**: `RNOH/AttributeModifier.ts`

**避免无效更新**：
```typescript
protected maybeAssignAttribute<TPropertyValue>(
  extractPropertyValue: (propertyHolder: TPropertyHolder) => TPropertyValue,
  assign: (propertyValue: TPropertyValue) => void,
  defaultValue: TPropertyValue,
): void {
  const newPropertyValue = extractPropertyValue(this.propertyHolder)
  let isNewValueEqualToDefault = newPropertyValue === defaultValue || Number.isNaN(newPropertyValue)

  // 数组比较
  if (Array.isArray(defaultValue) && Array.isArray(newPropertyValue)) {
    isNewValueEqualToDefault = areFirstLevelElementsEqual(newPropertyValue, defaultValue)
  }
  // 对象比较
  else if (typeof defaultValue === "object" && typeof newPropertyValue === "object") {
    isNewValueEqualToDefault = areFirstLevelPropsEqual(newPropertyValue, defaultValue)
  }

  // 只在值真正改变时调用 ArkUI API
  if (!isNewValueEqualToDefault) {
    assign(newPropertyValue)
  }
}
```

**关键点**：
- 数组和对象的浅比较
- 只在值真正改变时调用 API
- 避免无效的 UI 更新

### 性能建议

1. **使用 C-API 组件**
   - 优先使用内置的 C-API 组件
   - 避免不必要的 ArkTS 组件

2. **TurboModule 选择**
   - 纯计算逻辑使用 cxxTurboModule
   - 系统能力使用 ArkTSTurboModule

3. **减少跨语言调用**
   - 批量操作优于多次单次操作
   - 使用 JSI 直接调用而非桥接

4. **优化布局计算**
   - 避免深层嵌套
   - 使用合理的 flexbox 属性

---

## 开发原则

### 鸿蒙相关问题处理原则

1. **优先查阅官方文档**
   - 不要猜测或假设
   - 参考华为开发者文档

2. **参考源码实现**
   - RNOH 有 36 个 `.harmony.js` 补丁
   - 这些补丁展示了最佳实践

3. **使用 MCP 工具调试**
   - 使用 `harmonyos-ui` MCP 调试 UI 问题
   - 使用 `get_ui_tree` 检查组件树

4. **检查版本兼容性**
   - RN 0.72.5 / RNOH 0.72.102
   - HarmonyOS SDK 5.0.0.61+

5. **参考示例项目**
   - `docs/Samples/` 目录有多个示例
   - `tester/` 包含测试用例

### 官方文档链接

| 资源 | 链接 |
|------|------|
| HarmonyOS Ability | https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/ability-kit |
| RNOH 仓库 | https://gitcode.com/openharmony-sig/ohos_react_native |
| HiDumper | https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/hidumper |
| ContentSlot 文档 | https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/arkts-rendering-control-contentslot |
| RNOH 三方库文档 | https://react-native-oh-library.github.io/usage-docs/ |

### 网络资源

根据搜索结果，以下资源对理解 RNOH 架构有帮助：

1. [鸿蒙版React Native架构浅析](https://openharmonycrossplatform.csdn.net/690f13f90e4c466a32e5f12f.html) - CSDN
2. [React Native跨平台鸿蒙开发高级应用原理](https://ai6s.net/69265768791c233193d04253.html)
3. [React Native 新架构 - 渲染流水线](https://reactnative.cn/architecture/render-pipeline)
4. [ReactNative新架构Fabric2024年实践踩坑总结](https://my.oschina.net/emacs_9733956/blog/19177279)
5. [React Native 通信机制详解 - 新架构](https://juejin.cn/post/7576210031873343529)
6. [鸿蒙中RNOH架构介绍](https://juejin.cn/post/7473777534543740937)
7. [OpenHarmony使用ContentSlot占位组件管理Native API创建的组件](https://blog.csdn.net/m0_68635815/article/details/155262925)

---

## 常见问题排查

### UI 不显示

**排查步骤**：

1. 运行 `list_windows` 确认前台应用
2. 检查 `get_ui_tree` 查看组件树
3. 用 `list_abilities` 验证 Ability 状态
4. 检查 ContentSlot 是否正确连接

### 布局问题

**排查步骤**：

1. 检查 UI 树输出中的 `drawRegion`
2. 验证 JS 线程上的 Yoga 布局计算
3. 检查 MAIN 线程的组件 mutations
4. 验证 flexbox 属性是否符合规范

### 手势/触摸问题

**排查步骤**：

1. 验证事件监听器注册
2. 检查 native code 中的触摸事件分发器
3. 查阅 `.harmony.js` 补丁的手势处理
4. 确认是否与系统手势冲突

### 性能问题

**排查步骤**：

1. 检查工作是否可以移到 WORKER 线程
2. 验证重操作是否使用 TurboModule
3. 使用内置性能追踪分析瓶颈
4. 检查是否有不必要的重渲染

### TurboModule 问题

**排查步骤**：

1. 验证 Codegen 生成的代码
2. 检查 Module 是否正确注册
3. 确认线程类型 (UI/Worker)
4. 检查 N-API 绑定是否正确

### 常见问题与解决方案

#### RNOH_C_API_ARCH 未设置
**现象**: 启动后闪退，提示没有设置 `RNOH_C_API_ARCH`

**解决方案**:
1. 在环境变量中设置 `RNOH_C_API_ARCH=1`
2. 重启 DevEco Studio
3. 运行 `Build > Clean Project`，重新编译
4. 或在 `CMakeLists.txt` 中设置：`set(RNOH_C_API_ARCH, 1)`

#### libRNOHApp 绑定失败
**错误信息**: `Couldn't create bindings between ETS and CPP. libRNOHApp is undefined`

**原因**: `librnoh_app.so` 不存在或依赖的 `libhermes.so` 未打包

**解决方案**:
1. 在 `entry/build-profile.json5` 中添加 `externalNativeOptions`
2. 确认 `libhermes.so` 打包到 hap 包中
3. 在 har 模块的 `build-profile.json5` 中添加：
   ```json5
   "nativeLib": {
     "excludeFromHar": false
   }
   ```

#### 路径过长导致编译失败
**现象**: 找不到 `TextLayoutManager.cpp`

**原因**: 工程路径太长

**解决**: 缩短工程路径

#### Codegen 生成文件找不到
**现象**: 找不到 `react_native_openharmony/generated/ts` 文件

**解决方案**:
1. 执行 Codegen
2. 通过 `--cpp-output-path` 和 `--rnoh-module-path` 参数调整输出位置
3. 或手动复制 generated 文件夹到正确位置

#### Linking.canOpenURL() 一直返回 false
**问题**: 未在 `module.json5` 中配置 scheme

**解决**: 在 `module.json5` 的 `querySchemes` 字段配置：
```json5
"querySchemes": ["http", "https", "yourapp"]
```

#### 键盘事件未响应
**现象**: Keyboard 下的监听事件未响应

**原因**: RNOH 只监听最上层子窗口的键盘事件

**解决方案**:
1. 调整 RN 显示的窗口，让其显示在 `lastWindow`
2. 修改 RNOH 源码，监听对应窗口（如 `MainWindow`）的键盘事件

#### 字体变小问题（5.0.0.500 版本）
**现象**:
1. 同一个 bundle 在自定义 UIAbility 场景下字体明显变小
2. 使用 RNAbility 时，Metro 加载的 RN 页面字体变小

**原因**: CPP 侧拿到 `fontScale` 为 0（正常为 1）

**解决方案**:
- 现象1：在 `Ability.onWindowStageCreate()` 中初始化 `RNInstancesCoordinator`
- 现象2：延迟预加载 Metro bundle 的代码

#### setNativeProps 只能生效一次
**问题**: 启用 Fabric 后，`setNativeProps` 只在应用打开后生效一次

**原因**: RN 框架社区已知问题，`setNativeProps` 在 0.72.5 版本已废弃

**解决**: 使用 `setState` 进行状态设置

### 性能优化最佳实践

#### 使用 Release 版本
**问题**: Debug 包性能有瓶颈（滚动缓慢、动画卡顿）

**解决方案**: 使用 release 版本的 RN 包编译 release 版本

#### React 18 Automatic Batching
**特性**: 默认启用自动批量变更状态（`concurrentRoot: true`）

**优势**: 每 6~7 次 `setState` 才进行一次页面重新渲染

**建议**: 默认开启，不要修改 `disableConcurrentRoot` 配置

#### PureComponent 和 React.memo
**PureComponent**: 浅比较 `props` 和 `state`，避免不必要的渲染

**React.memo**: props 未改变时跳过重新渲染

**注意**: 深层次数据变化不触发渲染

#### 列表优化
```typescript
<FlatList
  data={largeData}
  renderItem={renderItem}
  keyExtractor={keyExtractor}
  removeClippedSubviews={true}  // 开启子视图剪裁
  // 长列表不设置 initialNumToRender（序号 >= initialNumToRender 的 Item 渲染速度较慢）
/>
```

#### setState 优化
```typescript
// 合并 setState
setState((prevState) => ({
  ...prevState,
  field1: value1,
  field2: value2
}));

// 使用 Promise.all 合并多个异步 setState
await Promise.all([
  setStateAsync(value1),
  setStateAsync(value2)
]);
```

#### 预加载策略
1. **预加载 RN 页面**：在应用启动时提前加载资源
2. **预创建 RN 实例**：在应用启动阶段预创建 React Native 实例
3. **FrameNode 预加载**：通过 FrameNode 提前进行资源加载和渲染

### 内存管理最佳实践

#### 总是清理定时器
```typescript
useEffect(() => {
  const timeoutId = setTimeout(() => {
    // 操作
  }, 1000);
  return () => clearTimeout(timeoutId);
}, []);
```

#### 总是停止动画
```typescript
useEffect(() => {
  animation.start();
  return () => animation.stop();
}, []);
```

#### 总是移除监听器
```typescript
useEffect(() => {
  const subscription = emitter.addListener('event', handler);
  return () => subscription.remove();
}, []);
```

### 类型安全最佳实践

#### 避免使用 any
```typescript
// ❌ 避免
function foo(data: any) { ... }

// ✅ 推荐
function foo(data: DataType) { ... }
```

#### 避免使用 @ts-ignore
```typescript
// ❌ 避免
// @ts-ignore
const value = obj.property;

// ✅ 推荐
// eslint-disable-next-line @typescript-eslint/no-explicit-any
const value = obj.property as any;
```

#### 避免滥用非空断言
```typescript
// ❌ 避免
const value = context!.property;

// ✅ 推荐
if (!context) {
  throw new Error('Context not available');
}
const value = context.property;
```

### 开发工作流程

```bash
# 1. 安装依赖
npm run i

# 2. 代码生成
npm run codegen

# 3. 开发构建
npm run dev

# 4. 类型检查
npm run typecheck

# 5. 代码检查
npm run lint:js

# 6. 格式化
npm run format

# 7. 运行测试
npm run test

# 8. 完整验证
npm run verify
```

### 调试技巧

#### 性能分析
```typescript
import { Benchmarker } from './benchmarks/Benchmarker';

<Benchmarker
  samplesCount={100}
  renderContent={(key) => <MyComponent key={key} />}
/>
```

#### 日志系统
```typescript
// 创建带上下文的logger
this.logger = this.ctx.logger.clone("ComponentName");

// 使用
this.logger.debug('Message');
this.logger.info('Message');
this.logger.warn('Message');
this.logger.error('Message');
```

#### 错误处理
```typescript
import { RNOHError } from './RNOHError';

RNOHError.throw(
  RNOHErrorDomain.RNOH,
  'INVALID_COMPONENT_TAG',
  `Component with tag ${tag} does not exist`,
  {tag}
);
```

---

## 代码示例

### 创建自定义 C-API 组件

```cpp
// MyCustomComponentInstance.h
class MyCustomComponentInstance : public CppComponentInstance {
protected:
  void createArkUINode() override {
    m_arkUINode = std::make_shared<StackNode>();
  }
};

// MyCustomComponentJSIBinder.h
class MyCustomComponentJSIBinder : public ComponentJSIBinder {
  // 实现 JSI 绑定
};
```

### 创建 TurboModule

**TypeScript 规范**：
```typescript
// MyModule.ts
import type { TurboModule } from 'react-native';
export interface Spec extends TurboModule {
  doSomething(value: string): Promise<number>;
}
```

**C++ 实现**：
```cpp
// MyModule.cpp
class MyModule : public NativeMyModuleSpec {
  int doSomething(std::string value) override {
    // 实现逻辑
    return 42;
  }
};
```

**注册**：
```cpp
std::shared_ptr<TurboModule> createTurboModule(...) {
  if (moduleName == "MyModule") {
    return std::make_shared<MyModule>(jsInvoker);
  }
  // ...
}
```

---

## 总结

### RNOH 项目规模

- **C++ 文件**: ~1073 个
- **ArkTS 文件**: ~51 个
- **核心组件**: 20+
- **TurboModule**: 15+
- **代码行数**: 估计 10万+ 行

### 架构特点

1. **新架构**: 基于 React Native 0.72 新架构
2. **分层设计**: JS → C++ → ArkTS → Native
3. **双模式**: 支持 C-API 和 ArkTS 组件
4. **高性能**: C-API 直接操作 ArkUI
5. **向后兼容**: 提供迁移指南

### 技术栈

- **JS 引擎**: Hermes (默认) 或 JSVM
- **布局引擎**: Yoga
- **渲染**: Fabric UIManager
- **模块系统**: TurboModule (JSI)
- **原生接口**: N-API (ArkTS ↔ C++)
- **UI 框架**: ArkUI (C-API + 声明式)

### 适用场景

- 需要跨平台开发的 HarmonyOS 应用
- 从 React Native 迁移到 HarmonyOS
- 需要高性能渲染的场景
- 需要复用现有 React Native 代码库

---

## 补充模块和功能 (2026年1月更新)

### NativeAnimated TurboModule 动画系统

**位置**: `tester/harmony/react_native_openharmony/src/main/cpp/RNOHCorePackage/TurboModules/Animated/`

**核心组件**：
- `NativeAnimatedTurboModule` - 动画 TurboModule 主入口
- `AnimatedNodesManager` - 动画节点管理器
- `AnimatedNode` 及其子类 - 各类动画节点
  - `ValueAnimatedNode` - 数值节点
  - `TransformAnimatedNode` - 变换节点
  - `StyleAnimatedNode` - 样式节点
  - `InterpolationAnimatedNode` - 插值节点
  - `TrackingAnimatedNode` - 跟踪节点
  - `DiffClampAnimatedNode` - 差值钳制节点
  - `PropsAnimatedNode` - 属性节点
  - `ModulusAnimatedNode` - 模数节点

**关键方法**：
```cpp
// 创建动画节点
void createAnimatedNode(facebook::react::Tag tag, folly::dynamic const& config);

// 连接动画节点
void connectAnimatedNodes(facebook::react::Tag parentTag, facebook::react::Tag childTag);

// 连接到视图
void connectAnimatedNodeToView(facebook::react::Tag nodeTag, facebook::react::Tag viewTag);

// 开始动画
void startAnimatingNode(
    facebook::react::Tag animationId,
    facebook::react::Tag nodeTag,
    folly::dynamic const& config,
    EndCallback&& endCallback);

// 运行更新
void runUpdates(long long frameTimeNanos);

// 设置 Native Props
void setNativeProps(facebook::react::Tag tag, folly::dynamic const& props);
```

**VSync 集成**：
```cpp
class NativeAnimatedTurboModule {
    std::shared_ptr<VSyncListener> m_vsyncListener;
    std::unique_ptr<OH_DisplaySoloist, void (*)(OH_DisplaySoloist*)> m_nativeDisplaySoloist;

    void requestAnimationFrame();
    void startDisplaySoloist();
    void stopDisplaySoloist();
};
```

**动画节点类型**：

| 节点类型 | 功能 | 适用场景 |
|---------|------|----------|
| ValueAnimatedNode | 存储和更新数值 | 基础值存储 |
| TransformAnimatedNode | 处理变换矩阵 | 旋转、缩放、平移 |
| StyleAnimatedNode | 处理样式属性 | 样式动画 |
| InterpolationAnimatedNode | 值映射插值 | 非线性动画曲线 |
| TrackingAnimatedNode | 跟踪其他节点值 | 跟随动画 |
| DiffClampAnimatedNode | 差值钳制 | 限制动画范围 |
| PropsAnimatedNode | 直接属性更新 | 属性动画 |

### Networking TurboModule 网络模块

**位置**: `tester/harmony/react_native_openharmony/src/main/cpp/RNOHCorePackage/TurboModules/NetworkingTurboModule.h`

**实现方式**：
```cpp
class NetworkingTurboModule : public ArkTSTurboModule {
public:
    NetworkingTurboModule(
        const ArkTSTurboModule::Context ctx,
        const std::string name);
};
```

**网络功能通过 ArkTS 实现**：
- Http.ets - HTTP 请求实现
- XMLHttpRequest.ets - XHR API 实现

**相关包**：
- `@ohmi/react-native-webview` - WebView 组件
- `react-native-webview-bridge` - WebView 桥接库

### 调试和开发工具

#### Metro 配置

**位置**: `react-native-harmony/metro.config.js`

**鸿蒙专用配置**：
```javascript
const { createHarmonyMetroConfig } = require('@react-native-oh/metro-config');

module.exports = createHarmonyMetroConfig({
    // 鸿蒙特定配置
});
```

#### 远程 JS 调试

**启用步骤**：
1. 运行 React Native 应用
2. 打开开发者菜单（摇晃设备或调用 API）
3. 选择 "Debug JS Remotely"
4. 在浏览器中访问调试地址

**DevTools 支持**：
- React DevTools 集成
- 元素检查
- Props 查看和编辑
- 性能分析

#### 端口转发

```bash
hdc rport tcp:8081 tcp:8081
```

### 平台模块和 API

#### Platform OS 检测

**RNOH 支持 `Platform.OS === 'harmony'`**：

```javascript
import { Platform } from 'react-native';

if (Platform.OS === 'harmony') {
    // OpenHarmony 特定代码
}
```

#### Dimensions API

**支持的标准 API**：
```javascript
import { Dimensions } from 'react-native';

const { width, height } = Dimensions.get('window');
const { width: screenWidth, height: screenHeight } = Dimensions.get('screen');
```

**OpenHarmony 像素单位**：
- `vp` (virtual pixels) - 逻辑像素单位
- 默认 `designWidth` 为 720
- `fontSizeScale` - 字体缩放比例（范围 0-3.2，默认 1）

#### Vibration 和 HapticFeedback

**可用包**：
- `@react-native-ohos/react-native-haptic-feedback` - 触觉反馈
- 标准 `Vibration` API - 震动

**使用示例**：
```javascript
import ReactNativeHapticFeedback from '@react-native-ohos/react-native-haptic-feedback';

ReactNativeHapticFeedback.trigger('impactLight');
```

### 第三方库生态

#### 官方资源

1. **[RNOH 三方库文档](https://react-native-oh-library.github.io/usage-docs/)** - 官方文档
2. **[react-native-oh-library](https://github.com/react-native-oh-library)** - GitHub 组织
3. **[react-native-openharmony-third-party-library](https://github.com/react-native-oh-library)** - 288 个仓库

#### 支持的库

| 库名 | OpenHarmony 适配 |
|------|-----------------|
| react-navigation | ✅ 支持 |
| react-native-gesture-handler | ✅ 支持 |
| react-native-webview | ✅ 支持 (@ohmi/react-native-webview) |
| react-native-safe-area-context | ✅ 支持 |
| react-native-haptic-feedback | ✅ 支持 (@react-native-ohos/react-native-haptic-feedback) |
| react-native-device-info | ✅ 支持 |
| react-native-maps | ✅ 支持 (@tuya-oh/react-native-maps) |

#### 库链接流程

**自动链接**：
```bash
npm install @react-native-oh-tpl/xxx-library
```

**手动链接**（如需要）：
1. 将 `.har` 文件放入 `libs/` 目录
2. 配置 `CMakeLists.txt`
3. 在 `PackageProvider.ets` 中注册

### 导航集成

#### react-navigation 支持

**基础用法**：
```javascript
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';

const Stack = createNativeStackNavigator();

function App() {
  return (
    <NavigationContainer>
      <Stack.Navigator>
        <Stack.Screen name="Home" component={HomeScreen} />
        <Stack.Screen name="Details" component={DetailsScreen} />
      </Stack.Navigator>
    </NavigationContainer>
  );
}
```

**导航栏问题**：
- HarmonyOS 原生 Navigation 组件可能被状态栏遮挡
- 使用 SafeAreaView 解决

### WebView 和桥接

#### react-native-webview

**可用版本**：
- `@ohmi/react-native-webview` - 13.10.2-rc.0.3.11

**功能**：
- 加载远程 URL
- JavaScript 注入
- 消息通信
- WebView 桥接

**桥接通信**：
```javascript
import { WebView } from '@ohmi/react-native-webview';

<WebView
  source={{ uri: 'https://example.com' }}
  onMessage={(event) => console.log(event.nativeEvent.data)}
/>
```

### Alert、Modal、Toast

#### Alert API

**标准 React Native Alert**：
```javascript
import { Alert } from 'react-native';

Alert.alert(
  '标题',
  '消息内容',
  [
    { text: '取消', style: 'cancel' },
    { text: '确定', onPress: () => console.log('确定') }
  ]
);
```

#### Toast

**跨平台 Toast 实现**：
- 基于 TurboModule 架构设计
- 参数系统构建
- UI 渲染机制
- 线程安全管理

**可用库**：
- `react-native-toast-notifications` - 支持 Android、iOS、Web

#### Modal

**标准 Modal 组件**：
```javascript
import { Modal } from 'react-native';

<Modal
  visible={isVisible}
  animationType="slide"
  onRequestClose={() => setIsVisible(false)}
>
  {/* 内容 */}
</Modal>
```

### 资源和 Assets 管理

#### 图片资源

**资源路径前缀**：
```cpp
const std::string RAWFILE_PREFIX = "resource://RAWFILE/assets/";
const std::string RESFILE_PREFIX = "file:///data/storage/el1/bundle/";
const std::string RESFILE_PATH = "/resources/resfile/assets/";
```

**Bundle 路径**：
```cpp
std::string ImageComponentInstance::getBundlePath() {
    auto internalInstance = std::dynamic_pointer_cast<RNInstanceInternal>(
        m_deps->rnInstance.lock()
    );
    return internalInstance->getBundlePath();
}
```

#### 字体资源

**RNOH 5.0.0.500+ 字体变更**：
| 变更项 | 旧版本 | 新版本 (5.0.0.500+) |
|--------|--------|---------------------|
| 字体目录 | `rawfile/assets/assets/fonts/` | `rawfile/fonts/` |
| 字体配置 | `fontOptions` 数组 | `fontResourceByFontFamily` map |
| 字体优先级 | `fontFamily` > 主题 | 主题 > `fontFamily` |

---

## 高级功能模块 (2026年1月更新 - 第二轮)

### 列表组件 (VirtualizedList / FlatList)

**RNOH 支持 React Native 标准列表组件**：

| 组件 | 功能 | 使用场景 |
|------|------|----------|
| VirtualizedList | 基础虚拟化列表 | 需要自定义渲染逻辑 |
| FlatList | 简化版虚拟列表 | 大多数列表场景 |
| SectionList | 分组列表 | 分组数据展示 |

**性能优化特性**：
- 窗口化渲染 - 只渲染可见区域
- 懒加载 - 按需创建子组件
- 内存优化 - 回收不可见组件

**关键实现**：
```javascript
import { FlatList } from 'react-native';

<FlatList
  data={data}
  renderItem={({ item }) => <Item title={item.title} />}
  keyExtractor={item => item.id}
  // 性能优化
  getItemLayout={(data, index) => ({
    length: ITEM_HEIGHT,
    offset: ITEM_HEIGHT * index,
    index,
  })}
  initialNumToRender={10}
  maxToRenderPerBatch={10}
  windowSize={5}
/>
```

### Accessibility 无障碍功能

**AccessibilityInfo API**：

```javascript
import { AccessibilityInfo } from 'react-native';

// 检查屏幕阅读器是否启用
AccessibilityInfo.isScreenReaderEnabled().then(enabled => {
  console.log('Screen reader:', enabled);
});

// 监听屏幕阅读器状态变化
AccessibilityInfo.addEventListener(
  'screenReaderChanged',
  (enabled) => {
    console.log('Screen reader changed:', enabled);
  }
);

// 检查减动画模式是否启用
AccessibilityInfo.isReduceMotionEnabled().then(enabled => {
  console.log('Reduce motion:', enabled);
});

// 推荐的无障碍属性
<View
  accessibilityLabel="登录按钮"
  accessibilityHint="点击进入登录页面"
  accessibilityRole="button"
  accessible={true}
/>
```

**无障碍角色映射** (ArkUINode)：
```cpp
const std::unordered_map<std::string, ArkUI_NodeType> NODE_TYPE_BY_ROLE_NAME = {
    {"button", ARKUI_NODE_BUTTON},
    {"togglebutton", ARKUI_NODE_TOGGLE},
    {"search", ARKUI_NODE_TEXT_INPUT},
    {"image", ARKUI_NODE_IMAGE},
    {"text", ARKUI_NODE_TEXT},
    {"adjustable", ARKUI_NODE_SLIDER},
    // ... 更多映射
};
```

### 触摸事件系统

**TouchEventDispatcher 架构**：

**位置**: `tester/harmony/react_native_openharmony/src/main/cpp/RNOH/arkui/TouchEventDispatcher.h`

**核心功能**：
```cpp
class TouchEventDispatcher {
public:
    // 分发触摸事件到 JS
    void dispatchTouchEvent(
        ArkUI_UIInputEvent* event,
        TouchTarget::Shared const& rootTarget);

    // 取消活动触摸（用于滚动时避免误触）
    void cancelActiveTouches();

private:
    // 为触摸注册目标
    TouchTarget::Shared registerTargetForTouch(
        TouchPoint touchPoint,
        TouchTarget::Shared const& rootTarget);

    // 目标查找和事件发送
    void findTargetAndSendTouchEvent(
        TouchTarget::Shared const& rootTarget,
        const TouchEvent& touchEvent);
};
```

**TouchTarget 接口**：

**位置**: `tester/harmony/react_native_openharmony/src/main/cpp/RNOH/TouchTarget.h`

```cpp
class TouchTarget {
public:
    // 计算子节点触摸点
    virtual Point computeChildPoint(
        Point const& point,
        TouchTarget::Shared const& child) const;

    // 是否包含触摸点
    virtual bool containsPoint(Point const& point) const = 0;

    // 是否可以处理触摸
    virtual bool canHandleTouch() const = 0;

    // 子节点是否可以处理触摸
    virtual bool canChildrenHandleTouch() const = 0;

    // 是否正在处理触摸（如滚动中）
    virtual bool isHandlingTouches() const {
        return false;
    }

    // 是否是 JS 响应者
    virtual bool isJSResponder() const {
        return false;
    }
};
```

**触摸事件流程**：
```
1. ArkUI 触摸事件 → NODE_TOUCH_EVENT
    ↓
2. TouchEventDispatcher::dispatchTouchEvent()
    ↓
3. registerTargetForTouch() - 查找目标组件
    ↓
4. findTargetAndSendTouchEvent() - 发送到 JS
    ↓
5. TouchEventEmitter 发送事件到 JavaScript
```

**取消触摸机制**：
```cpp
// 当 ScrollView 滚动时，可以取消活动触摸
// 防止滑动被误识别为点击
void cancelActiveTouches() {
    // 发送 CANCEL 事件到所有活动触摸
    // 清除触摸目标映射
}
```

### 手势处理

**react-native-gesture-handler 支持**：

**可用包**：
- `@react-native-oh-tpl/react-native-gesture-handler` - 官方适配版本 (v2.14.8)
- `@ohmi/react-native-gesture-handler` - 社区版本

**基础手势类型**：

| 手势 | 说明 | 使用场景 |
|------|------|----------|
| TapGesture | 点击手势 | 简单点击交互 |
| PanGesture | 拖动手势 | 滑动、拖拽 |
| PinchGesture | 缩放手势 | 图片缩放 |
| RotationGesture | 旋转手势 | 旋转操作 |
| LongPressGesture | 长按手势 | 长按菜单 |
| NativeViewGesture | 原生视图手势 | 与原生组件集成 |

**使用示例**：
```javascript
import { Gesture, GestureDetector } from 'react-native-gesture-handler';

const tap = Gesture.Tap()
  .numberOfTaps(2)
  .onStart(() => {
    console.log('Double tap!');
  });

const pan = Gesture.Pan()
  .onUpdate((event) => {
    setTranslateX(event.translationX);
    setTranslateY(event.translationY);
  })
  .onEnd(() => {
    // 处理结束
  });

<GestureDetector gesture={Gesture.Race(tap, pan)}>
  <Animated.View style={{ transform: [{ translateX }, { translateY }] }} />
</GestureDetector>
```

**兼容性说明**：
- HarmonyOS Next 可能存在兼容性问题
- 建议使用最新的 RNOH 适配版本
- 某些高级手势可能需要额外配置

### KeyboardAvoidingView 键盘处理

**RNOH 键盘避免特性**：

**双高度调整问题**：
- TextInput 聚焦时，自动调整高度避免键盘
- 使用 KeyboardAvoidingView 包裹时，页面布局也会调整
- 可能导致双重高度调整

**解决方案**：
```javascript
// 方案1: 使用 ArkUI 原生行为
// 在 RNApp 中配置 expandSafeArea()

// 方案2: 手动控制 KeyboardAvoidingView
import { KeyboardAvoidingView, Platform } from 'react-native';

<KeyboardAvoidingView
  behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
  enabled={true}
  style={{ flex: 1 }}
>
  {/* 内容 */}
</KeyboardAvoidingView>
```

**常见问题**：
| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 键盘遮挡内容 | KeyboardAvoidingView 不生效 | 检查父容器是否有明确高度 |
| 双重调整 | TextInput + KeyboardAvoidingView | 移除 KeyboardAvoidingView 或禁用 TextInput 自动调整 |
| 动画不流畅 | 频繁的布局计算 | 使用 behavior="padding" 或手动控制 |

### StatusBar 状态栏

**标准 StatusBar API**：

```javascript
import { StatusBar } from 'react-native';

// 设置状态栏样式
<StatusBar
  barStyle="dark-content"  // 或 "light-content"
  backgroundColor="#ffffff"
  hidden={false}
  translucent={false}
  networkActivityIndicatorVisible={true}
/>

// 动态控制
StatusBar.setBarStyle('dark-content');
StatusBar.setBackgroundColor('#ffffff');
StatusBar.setHidden(false);
```

**多 StatusBar 实例**：
```javascript
// 不同页面使用不同状态栏
function Screen1() {
  return (
    <>
      <StatusBar barStyle="dark-content" backgroundColor="white" />
      {/* 内容 */}
    </>
  );
}

function Screen2() {
  return (
    <>
      <StatusBar barStyle="light-content" backgroundColor="blue" />
      {/* 内容 */}
    </>
  );
}
```

**沉浸式状态栏**：
```javascript
// 与 SafeAreaView 配合
import { StatusBar, SafeAreaView } from 'react-native';

function App() {
  return (
    <SafeAreaView style={{ flex: 1, backgroundColor: 'white' }}>
      <StatusBar barStyle="dark-content" backgroundColor="white" />
      {/* 内容 */}
    </SafeAreaView>
  );
}
```

### Permissions 权限管理

**react-native-permissions for OpenHarmony**：

**包**: `@react-native-ohos/react-native-permissions`

**权限检查和请求**：
```javascript
import { Permissions, PERMISSIONS } from 'react-native-permissions';

// 检查权限
const checkPermission = async () => {
  const status = await Permissions.check(
    PERMISSIONS.HARMONY.CAMERA
  );
  console.log('Camera permission:', status);
};

// 请求权限
const requestPermission = async () => {
  const status = await Permissions.request(
    PERMISSIONS.HARMONY.CAMERA
  );
  console.log('Camera permission after request:', status);
};

// 检查并请求
const checkAndRequest = async () => {
  const status = await Permissions.check(
    PERMISSIONS.HARMONY.LOCATION
  );

  if (status !== 'granted') {
    const result = await Permissions.request(
      PERMISSIONS.HARMONY.LOCATION
    );
    return result === 'granted';
  }

  return true;
};
```

**OpenHarmony 权限常量**：
```javascript
PERMISSIONS.HARMONY.CAMERA
PERMISSIONS.HARMONY.LOCATION
PERMISSIONS.HARMONY.LOCATION_WHEN_IN_USE
PERMISSIONS.HARMONY.LOCATION_ALWAYS
PERMISSIONS.HARMONY.MICROPHONE
PERMISSIONS.HARMONY.READ_EXTERNAL_STORAGE
PERMISSIONS.HARMONY.WRITE_EXTERNAL_STORAGE
PERMISSIONS.HARMONY.READ_CONTACTS
PERMISSIONS.HARMONY.WRITE_CONTACTS
// ... 更多权限
```

**权限状态**：
- `granted` - 已授权
- `denied` - 已拒绝
- `limited` - 有限授权
- `undetermined` - 未确定

### 图片加载和缓存

**react-native-fast-image for OpenHarmony**：

**包**: `@react-native-ohos/react-native-fast-image` (v8.6.4-rc.1)

**高级图片功能**：
```javascript
import FastImage from 'react-native-fast-image';

// 基础用法
<FastImage
  source={{ uri: 'https://example.com/image.jpg' }}
  style={{ width: 200, height: 200 }}
/>

// 缓存控制
<FastImage
  source={{
    uri: 'https://example.com/image.jpg',
    priority: FastImage.priority.high,
  }}
  resizeMode={FastImage.resizeMode.contain}
/>

// 图片预加载
FastImage.preload([
  {
    uri: 'https://example.com/image1.jpg',
  },
  {
    uri: 'https://example.com/image2.jpg',
  },
]);
```

**优先级**：
```javascript
FastImage.priority.low
FastImage.priority.normal
FastImage.priority.high
```

**缓存模式**：
```javascript
FastImage.cacheMode.ignoreCache  // 忽略缓存
FastImage.cacheMode.web          // 使用 Web 缓存
FastImage.cacheMode.cacheOnly    // 仅使用缓存
```

**图片资源路径处理**：
```cpp
// RNOH 图片路径常量
const std::string RAWFILE_PREFIX = "resource://RAWFILE/assets/";
const std::string RESFILE_PREFIX = "file:///data/storage/el1/bundle/";
const std::string BASE_64_PREFIX = "data:";
const std::string BASE_64_MARK = ";base64,";

// 支持的图片类型
const std::unordered_set<std::string> validImageTypes = {
    "png", "jpeg", "jpg", "gif", "bmp", "webp"
};
```

### 性能优化和内存管理

**内存泄漏预防**：

**RNOH 特定问题**：
- 预分配视图 (preallocated views) 可能导致内存泄漏
- 已在最新版本中修复

**常见内存泄漏场景**：
```javascript
// 错误：未清理定时器
useEffect(() => {
  const timer = setInterval(() => {
    console.log('tick');
  }, 1000);
  // 缺少清理函数
}, []);

// 正确
useEffect(() => {
  const timer = setInterval(() => {
    console.log('tick');
  }, 1000);

  return () => clearInterval(timer);
}, []);

// 错误：未清理事件监听
useEffect(() => {
  Dimensions.addEventListener('change', handler);
  // 缺少清理
}, []);

// 正确
useEffect(() => {
  const subscription = Dimensions.addEventListener('change', handler);
  return () => subscription.remove();
}, []);
```

**性能优化建议**：

1. **列表优化**：
```javascript
<FlatList
  // 使用 getItemLayout 优化滚动
  getItemLayout={(data, index) => ({
    length: ITEM_HEIGHT,
    offset: ITEM_HEIGHT * index,
    index,
  })}
  // 控制渲染数量
  initialNumToRender={10}
  maxToRenderPerBatch={10}
  windowSize={5}
  // 移除内联函数
  renderItem={renderItem}
  keyExtractor={keyExtractor}
/>
```

2. **图片优化**：
```javascript
// 使用 FastImage 而非 Image
// 实现图片懒加载
// 使用适当的图片尺寸
// 启用缓存
```

3. **减少重渲染**：
```javascript
// 使用 React.memo
const Item = React.memo(({ data }) => {
  return <Text>{data.title}</Text>;
});

// 使用 useMemo 和 useCallback
const processedData = useMemo(() => {
  return data.map(item => process(item));
}, [data]);

const handlePress = useCallback(() => {
  // 处理点击
}, [dependency]);
```

4. **优化状态更新**：
```javascript
// 避免频繁状态更新
const [data, setData] = useState([]);

// 批量更新
setData(prevData => [...prevData, ...newItems]);

// 使用 useReducer 复杂状态
const [state, dispatch] = useReducer(reducer, initialState);
```

**性能分析**：
```javascript
// 使用 React DevTools Profiler
// 启用 Performance 标记
import { Performance } from 'react-native';

Performance.mark('start');
// 操作
Performance.measure('operation', 'start');
```

### 开发工具和调试

**Chrome DevTools**：
```bash
# 启用远程调试
# 1. 运行应用
# 2. 打开开发者菜单
# 3. 选择 "Debug JS Remotely"
# 4. 在 Chrome 中访问调试地址
```

**React DevTools**：
```javascript
// 集成到应用
import { devtools } from 'react-native-devtools';

if (__DEV__) {
  devtools();
}
```

**性能监控**：
```javascript
// Systrace 集成
import { Systrace } from 'react-native';

Systrace.beginEvent('CustomOperation');
// 操作
Systrace.endEvent();
```

**日志调试**：
```javascript
// 使用 console
console.log('Info');
console.warn('Warning');
console.error('Error');

// 使用 YellowBox/LogBox (开发模式)
import { LogBox } from 'react-native';

if (__DEV__) {
  LogBox.ignoreLogs(['Warning: ...']);
}
```

### 常见问题解决方案

#### 列表性能问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 滚动卡顿 | 渲染过多组件 | 使用 getItemLayout，减少 initialNumToRender |
| 内存泄漏 | 组件未正确卸载 | 检查 useEffect 清理函数 |
| 图片加载慢 | 未使用缓存 | 使用 FastImage |
| 按钮无响应 | 事件被拦截 | 检查父组件 pointerEvents |

#### 键盘问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 内容被遮挡 | KeyboardAvoidingView 不生效 | 确保父容器有明确高度 |
| 双重调整 | TextInput + KeyboardAvoidingView | 移除 KeyboardAvoidingView |
| 动画卡顿 | 频繁布局计算 | 使用 behavior="padding" |

#### 权限问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 权限未授予 | 用户拒绝 | 引导用户到设置页面 |
| 权限检查失败 | 权限名称错误 | 使用 PERMISSIONS.HARMONY.* |
| 权限弹窗不显示 | module.json 未配置 | 添加权限到配置文件 |

---

## 更多功能模块 (2026年1月更新 - 第三轮)

### 基础组件实现

#### Switch 开关组件

**位置**: `tester/harmony/react_native_openharmony/src/main/cpp/RNOHCorePackage/ComponentInstances/SwitchComponentInstance.h`

**实现方式**：
```cpp
class SwitchComponentInstance
    : public CppComponentInstance<SwitchShadowNode>,
      public ToggleNodeDelegate {
private:
  ToggleNode m_toggleNode;  // 使用 ArkUI Toggle 节点

public:
  void onPropsChanged(SharedConcreteProps const& props) override;
  void onValueChange(int32_t& value) override;
};
```

**使用示例**：
```javascript
import { Switch } from 'react-native';

<Switch
  value={isEnabled}
  onValueChange={(newValue) => setIsEnabled(newValue)}
  trackColor={{ false: '#767577', true: '#81b0ff' }}
  thumbColor={isEnabled ? '#f5dd4b' : '#f4f3f4'}
/>
```

**实现特点**：
- 基于 ArkUI `Toggle` 节点
- 支持自定义轨道和滑块颜色
- 响应式值变化通知

#### Modal 模态框

**位置**: `tester/harmony/react_native_openharmony/src/main/cpp/RNOHCorePackage/ComponentInstances/ModalHostViewComponentInstance.h`

**实现方式**：
```cpp
class ModalHostViewComponentInstance
    : public CppComponentInstance<ModalHostViewShadowNode>,
      public ArkUIDialogDelegate,
      public ArkTSMessageHub::Observer {
private:
  enum FoldStatus {
    FOLD_STATUS_UNKNOWN,
    FOLD_STATUS_EXPANDED,
    FOLD_STATUS_FOLDED,
    FOLD_STATUS_HALF_FOLDED
  };

  CustomNode m_virtualNode;
  StackNode m_rootStackNode;
  ArkUIDialogHandler m_dialogHandler;

public:
  void showDialog();
  void onShow() override;
  void onRequestClose() override;
};
```

**使用示例**：
```javascript
import { Modal, View, Text, TouchableOpacity } from 'react-native';

<Modal
  visible={isVisible}
  animationType="slide"
  transparent={true}
  onRequestClose={() => setIsVisible(false)}
>
  <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
    <View style={{ backgroundColor: 'white', padding: 20, borderRadius: 10 }}>
      <Text>Modal Content</Text>
      <TouchableOpacity onPress={() => setIsVisible(false)}>
        <Text>Close</Text>
      </TouchableOpacity>
    </View>
  </View>
</Modal>
```

**支持功能**：
- 多种动画类型 (`slide`, `fade`, `none`)
- 透明背景支持
- 折叠屏适配
- 对话框状态管理

**已知问题**：
- ActivityIndicator 作为 Modal 子组件时可能不工作
- 建议避免在 Modal 中直接嵌套 ActivityIndicator

#### ActivityIndicator 加载指示器

**位置**: `tester/harmony/react_native_openharmony/src/main/cpp/RNOHCorePackage/ComponentInstances/ActivityIndicatorComponentInstance.h`

**实现方式**：
```cpp
class ActivityIndicatorComponentInstance
    : public CppComponentInstance<ActivityIndicatorViewShadowNode> {
private:
  LoadingProgressNode m_loadingProgressNode;  // ArkUI 加载进度节点

public:
  void onPropsChanged(SharedConcreteProps const& props) override;
};
```

**使用示例**：
```javascript
import { ActivityIndicator, View } from 'react-native';

<View style={{ flex: 1, justifyContent: 'center' }}>
  <ActivityIndicator
    size="large"  // 或 "small"
    color="#0000ff"
    animating={isLoading}
  />
</View>
```

**支持的属性**：
- `size` - 大小 (`small`, `large`)
- `color` - 颜色
- `animating` - 是否动画

### 存储系统

#### AsyncStorage

**包**: `@react-native-oh-tpl/react-native-async-storage`

**安装和配置**：
```bash
# 安装
npm install @react-native-oh-tpl/react-native-async-storage

# 手动链接（如需要）
# 1. 将 .har 文件放入 libs/
# 2. 配置 CMakeLists.txt
# 3. 在 PackageProvider.ets 中注册
```

**使用示例**：
```javascript
import AsyncStorage from '@react-native-oh-tpl/react-native-async-storage';

// 存储数据
const storeData = async () => {
  try {
    await AsyncStorage.setItem('key', 'value');
  } catch (e) {
    console.error('Failed to save', e);
  }
};

// 读取数据
const readData = async () => {
  try {
    const value = await AsyncStorage.getItem('key');
    if (value !== null) {
      console.log('Retrieved:', value);
    }
  } catch (e) {
    console.error('Failed to read', e);
  }
};

// 删除数据
const removeData = async () => {
  try {
    await AsyncStorage.removeItem('key');
  } catch (e) {
    console.error('Failed to remove', e);
  }
};

// 清空所有
const clearAll = async () => {
  try {
    await AsyncStorage.clear();
  } catch (e) {
    console.error('Failed to clear', e);
  }
};
```

**适用场景**：
- 用户设置
- 会话数据
- 离线缓存
- 轻量级数据持久化

#### 文件系统 (react-native-fs)

**文档**: https://gitee.com/hu-jinyun/usage-docs/blob/master/zh-cn/react-native-fs.md

**配置要求**：
```cmake
# CMakeLists.txt
target_link_libraries(entry PUBLIC
  libace_napi.z.so
  libnative_drawing.so
  # 添加 fs 相关库
)
```

**使用示例**：
```javascript
import RNFS from 'react-native-fs';

// 读取文件
const readFile = async () => {
  const content = await RNFS.readFile('/path/to/file.txt', 'utf8');
  console.log(content);
};

// 写入文件
const writeFile = async () => {
  await RNFS.writeFile('/path/to/file.txt', 'Hello World', 'utf8');
};

// 检查文件是否存在
const exists = await RNFS.exists('/path/to/file.txt');

// 删除文件
await RNFS.unlink('/path/to/file.txt');

// 获取文件信息
const stat = await RNFS.stat('/path/to/file.txt');
console.log('Size:', stat.size);
console.log('Is file:', stat.isFile());
```

**HarmonyOS 原生 API**：
```javascript
// 使用 @ohos.file.fs
import fs from '@ohos.file.fs';

// 读取文件
const file = fs.openSync('/path/to/file.txt', fs.OpenMode.READ_ONLY);
const buffer = new ArrayBuffer(1024);
fs.readSync(file.fd, buffer);
fs.closeSync(file);
```

### 深度链接 (Deep Linking)

#### Linking API

**基础用法**：
```javascript
import { Linking } from 'react-native';

// 打开 URL
const openURL = async () => {
  try {
    const supported = await Linking.canOpenURL('https://example.com');
    if (supported) {
      await Linking.openURL('https://example.com');
    } else {
      Alert.alert('Cannot open this URL');
    }
  } catch (e) {
    console.error('Error opening URL', e);
  }
};

// 打开电话
await Linking.openURL('tel:1234567890');

// 打开短信
await Linking.openURL('smsto:1234567890');

// 发送邮件
await Linking.openURL('mailto:test@example.com');
```

#### Deep Link 配置

**module.json 配置**：
```json
{
  "module": {
    "abilities": [
      {
        "skills": [
          {
            "actions": [
              {
                "action": "ohos.want.action.viewData",
                "entities": [
                  "entity.system.browsable"
                ],
                "uris": [
                  {
                    "scheme": "myapp",
                    "host": "example",
                    "path": "details"
                  }
                ]
              }
            ]
          }
        ]
      }
    ]
  }
}
```

**处理 Deep Link**：
```javascript
import { Linking } from 'react-native';
import { useEffect } from 'react';

function App() {
  useEffect(() => {
    // 处理初始 URL
    Linking.getInitialURL().then((url) => {
      if (url) {
        handleOpenURL({ url });
      }
    });

    // 监听 URL 变化
    const subscription = Linking.addEventListener('url', ({ url }) => {
      handleOpenURL({ url });
    });

    return () => {
      subscription.remove();
    };
  }, []);

  const handleOpenURL = ({ url }) => {
    const path = url.split('/').pop();
    // 根据路径导航
    if (path === 'details') {
      navigation.navigate('Details');
    }
  };

  return <YourApp />;
}
```

**与 React Navigation 集成**：
```javascript
import { NavigationContainer } from '@react-navigation/native';
import { Linking } from 'react-native';

const linking = {
  prefixes: ['myapp://'],
  config: {
    screens: {
      Home: 'home',
      Details: 'details/:id',
    },
  },
};

<NavigationContainer linking={linking}>
  {/* ... */}
</NavigationContainer>
```

### 剪贴板和分享

#### Clipboard

**包**: `@react-native-oh-tpl/clipboard`

**使用示例**：
```javascript
import Clipboard from '@react-native-oh-tpl/clipboard';

// 复制文本
const copyToClipboard = async () => {
  await Clipboard.setString('Hello World');
  Alert.alert('Copied to clipboard');
};

// 粘贴文本
const pasteFromClipboard = async () => {
  const text = await Clipboard.getString();
  console.log('Clipboard content:', text);
};
```

#### Share API

**基础分享**：
```javascript
import { Share } from 'react-native';

const shareContent = async () => {
  try {
    const result = await Share.share({
      message: 'Check out this content!',
      url: 'https://example.com',
      title: 'Share Example',
    });

    if (result.action === Share.sharedAction) {
      if (result.activityType) {
        console.log('Shared with activity type:', result.activityType);
      }
    }
  } catch (e) {
    console.error('Error sharing', e);
  }
};
```

### 权限管理 (补充)

#### HarmonyOS 权限系统

**module.json 权限声明**：
```json
{
  "module": {
    "requestPermissions": [
      {
        "name": "ohos.permission.CAMERA",
        "reason": "$string:camera_reason",
        "usedScene": {
          "abilities": ["EntryAbility"],
          "when": "inuse"
        }
      },
      {
        "name": "ohos.permission.LOCATION",
        "reason": "$string:location_reason",
        "usedScene": {
          "abilities": ["EntryAbility"],
          "when": "always"
        }
      }
    ]
  }
}
```

**用户授权流程**：
```javascript
import { Permissions, PERMISSIONS } from '@react-native-ohos/react-native-permissions';
import { Alert, Linking } from 'react-native';

const requestCameraPermission = async () => {
  const status = await Permissions.check(PERMISSIONS.HARMONY.CAMERA);

  if (status === 'granted') {
    return true;
  }

  if (status === 'denied') {
    // 引导用户到设置页面
    Alert.alert(
      'Permission Required',
      'Camera permission is needed. Please enable it in settings.',
      [
        { text: 'Cancel', style: 'cancel' },
        { text: 'Settings', onPress: () => {
          Linking.openSettings();
        }}
      ]
    );
    return false;
  }

  // 请求权限
  const result = await Permissions.request(PERMISSIONS.HARMONY.CAMERA);
  return result === 'granted';
};
```

### 其他注意事项

#### 手动链接流程

由于 HarmonyOS 不支持 AutoLink，需要手动配置：

1. **下载 .har 文件**
   - 从 Releases 下载兼容的 tgz 包
   - 解压获取 .har 文件

2. **配置 libs/**
   - 将 .har 文件放入 `entry/libs/` 目录

3. **配置 CMakeLists.txt**
   ```cmake
   # 添加 HAR 文件引用
   oh_hap_har("libs/library.har")
   ```

4. **ArkTS 侧配置**
   - 在 `PackageProvider.ets` 中注册模块

#### 版本兼容性

| 组件 | RNOH 版本 | HarmonyOS SDK | DevEco Studio |
|------|-----------|---------------|---------------|
| react-native-harmony | 0.72.31+ | 5.0.2.126+ | 5.0.7.210+ |
| react-native-fast-image | 8.6.4-rc.1 | 5.0.0.61+ | 5.0.3.706+ |
| react-native-gesture-handler | 2.14.8 | 5.0.0.61+ | 5.0.3.706+ |
| react-native-permissions | Latest | 5.0.0.61+ | 5.0.3.706+ |

#### 常见链接问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 找不到模块 | .har 文件未正确放置 | 检查 libs/ 目录 |
| 编译错误 | CMakeLists.txt 配置错误 | 验证 target_link_libraries |
| 运行时崩溃 | ArkTS 侧未注册 | 检查 PackageProvider.ets |
| 类型错误 | TypeScript 定义缺失 | 添加 .d.ts 文件 |

---

## 终极补充 (2026年1月更新 - 第四轮)

### 不支持的组件

#### TabBarIOS
- **状态**: ❌ 不支持
- **原因**: iOS 专用遗留组件，未在 OpenHarmony 上实现
- **替代方案**: 使用 `@react-navigation/bottom-tabs`

#### SegmentedControlIOS
- **状态**: ❌ 不支持
- **原因**: 已从 React Native 核心移除
- **替代方案**: 使用 `react-native-segmented-control`

#### SliderIOS
- **状态**: ❌ 不支持
- **原因**: iOS 专用组件
- **替代方案**: 使用标准 `Slider` 组件

### 下拉刷新

#### react-native-pull
**包**: `@react-native-oh-tpl/react-native-pull`

**特性**：
- PullView & PullList 组件
- 支持下拉刷新
- 兼容 Android & iOS

**使用示例**：
```javascript
import { PullList } from '@react-native-oh-tpl/react-native-pull';

<PullList
  onPullRelease={() => {
    // 刷新数据
    fetchData().then(() => {
      pullList && pullList.endRefresh();
    });
  }}
  data={data}
  renderItem={({ item }) => <Item data={item} />}
/>
```

#### RefreshControl 标准组件
RNOH 支持标准的 RefreshControl 组件：

```javascript
import { ScrollView, RefreshControl } from 'react-native';

<ScrollView
  refreshControl={
    <RefreshControl
      refreshing={isRefreshing}
      onRefresh={onRefresh}
      colors={['#ff0000', '#00ff00', '#0000ff']}
      tintColor="#fff"
    />
  }
>
  {/* 内容 */}
</ScrollView>
```

### 日期时间选择器

#### @ohmi/datetimepicker
**包**: `@ohmi/datetimepicker` (v7.6.2-0.1.4)

**特性**：
- 日期选择器
- 时间选择器
- 日期时间选择器
- OpenHarmony 平台适配

**使用示例**：
```javascript
import DateTimePicker from '@ohmi/datetimepicker';

<DateTimePicker
  value={date}
  mode="date"  // 'date', 'time', 'datetime'
  display="default"
  onChange={(event, selectedDate) => {
    setCurrentDate(selectedDate);
  }}
/>
```

#### oh-date-picker
**特性**：
- OpenHarmony & HarmonyOS 平台专用
- 支持多种格式：年-月、年-月-日、年-月-日-时-分

### 进度条和滑块

#### Progress 进度条
HarmonyOS 原生 Progress 组件：
- 线性进度条
- 环形进度条
- 自定义样式

#### Slider 滑块
RNOH 支持标准 Slider 组件：
```javascript
import { Slider } from 'react-native';

<Slider
  value={sliderValue}
  onValueChange={(value) => setSliderValue(value)}
  minimumValue={0}
  maximumValue={100}
  step={1}
/>
```

### 底部导航

#### @react-navigation/bottom-tabs
**推荐方案**：使用 React Navigation 的底部标签导航

**安装**：
```bash
npm install @react-navigation/bottom-tabs @react-navigation/native
npm install @react-native-oh-tpl/react-native-safe-area-context
```

**使用示例**：
```javascript
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs';

const Tab = createBottomTabNavigator();

function MyTabs() {
  return (
    <Tab.Navigator>
      <Tab.Screen name="Home" component={HomeScreen} />
      <Tab.Screen name="Settings" component={SettingsScreen} />
    </Tab.Navigator>
  );
}
```

**测试环境**：
- RNOH: 0.72.11+
- SDK: OpenHarmony (API 11) 4.1.0.53+
- IDE: DevEco Studio 4.1.3.412+

#### HarmonyOS 原生 Tabs
使用 ArkUI 的 Tabs 组件：
```javascript
import { Tabs } from '@ohos.arkui.advanced.TabBar';

<Tabs barPosition={BarPosition.End}>
  {/* 标签页内容 */}
</Tabs>
```

### 分段控制器

#### react-native-segmented-control
**包**: `react-native-segmented-control`

**使用示例**：
```javascript
import SegmentedControl from 'react-native-segmented-control';

<SegmentedControl
  values={['One', 'Two']}
  selectedIndex={0}
  onChange={(event) => {
    setSelectedIndex(event.nativeEvent.selectedSegmentIndex);
  }}
/>
```

### 完整的 RNOH 知识体系总结

经过四轮深入研究，RNOH (React Native for OpenHarmony) 的完整知识体系包括：

#### 核心架构
- **版本**: 0.72.102 (基于 RN 0.72.5)
- **架构**: Fabric + TurboModule
- **线程模型**: MAIN, JS, BACKGROUND, WORKER
- **组件类型**: C-API 组件 + ArkTS 组件

#### 关键实现
1. **MountingManagerCAPI** - Mutation 调度
2. **ArkUINode** - ArkUI 节点封装
3. **TurboModule** - cxxTurboModule + ArkTSTurboModule
4. **ComponentInstance** - 组件生命周期
5. **TouchEventDispatcher** - 触摸事件处理

#### 支持的组件
| 分类 | 组件 | 状态 |
|------|------|------|
| 基础 | View, Text, Image | ✅ 完全支持 |
| 输入 | TextInput, Switch, Slider | ✅ 完全支持 |
| 滚动 | ScrollView, FlatList, SectionList | ✅ 完全支持 |
| 模态 | Modal, ActivityIndicator | ✅ 支持 (有限) |
| 导航 | StatusBar, SafeAreaView | ✅ 支持 |
| iOS专用 | TabBarIOS, SegmentedControlIOS, SliderIOS | ❌ 不支持 |

#### 第三方库生态
- **官方文档**: https://react-native-oh-library.github.io/usage-docs/
- **GitHub**: https://github.com/react-native-oh-library (288+ 仓库)
- **常用库**:
  - @react-navigation/bottom-tabs ✅
  - @react-native-oh-tpl/react-native-gesture-handler ✅
  - @react-native-ohos/react-native-fast-image ✅
  - @react-native-oh-tpl/react-native-async-storage ✅
  - @react-native-ohos/react-native-permissions ✅
  - @react-native-oh-tpl/clipboard ✅

#### 开发工具
- Metro 配置 (`createHarmonyMetroConfig`)
- 远程 JS 调试
- React DevTools
- Chrome DevTools
- MCP 调试工具 (harmonyos-ui)

#### 性能优化
- C-API 组件优先
- TurboModule 线程选择
- 列表虚拟化
- 图片缓存
- 减少重渲染

#### 常见问题
- zIndex 失效 → 使用 position: absolute
- KeyboardAvoidingView 双重调整 → 移除或配置
- setNativeProps 废弃 → 使用 React state
- Modal + ActivityIndicator 不兼容 → 避免嵌套

#### 迁移指南 (RN 0.51/0.59 → RNOH)
- 使用 TurboModuleRegistry.getEnforcing()
- 移除 removeEventListener，使用 subscription.remove()
- Platform.OS === 'harmony' 检测
- 日期格式 ISO 8601
- .harmony.tsx 文件扩展名

---

## 组件注册和绑定机制深度解析 (2026年1月更新 - 第四轮)

### ComponentJSIBinder 工作原理

**核心职责**：连接 JavaScript 和 C++ 层，定义组件的属性、事件、命令。

**架构层次**：

```cpp
// 基础绑定器 - tester/harmony/react_native_openharmony/src/main/cpp/RNOH/BaseComponentJSIBinder.h

class BaseComponentJSIBinder : public ComponentJSIBinder {
public:
  facebook::jsi::Object createBindings(facebook::jsi::Runtime& rt) override {
    facebook::jsi::Object baseManagerConfig(rt);
    baseManagerConfig.setProperty(rt, "NativeProps", this->createNativeProps(rt));
    baseManagerConfig.setProperty(rt, "Constants", this->createConstants(rt));
    baseManagerConfig.setProperty(rt, "Commands", this->createCommands(rt));
    baseManagerConfig.setProperty(rt, "bubblingEventTypes", this->createBubblingEventTypes(rt));
    baseManagerConfig.setProperty(rt, "directEventTypes", this->createDirectEventTypes(rt));
    return baseManagerConfig;
  }
};
```

**专用绑定器示例**：

```cpp
// ViewComponentJSIBinder
class ViewComponentJSIBinder : public BaseComponentJSIBinder {
protected:
  facebook::jsi::Object createNativeProps(facebook::jsi::Runtime& rt) override {
    auto nativeProps = BaseComponentJSIBinder::createNativeProps(rt);
    // 视图特有属性
    nativeProps.setProperty(rt, "removeClippedSubviews", "boolean");
    nativeProps.setProperty(rt, "collapsable", "boolean");
    nativeProps.setProperty(rt, "pointerEvents", "string");
    return nativeProps;
  }
};

// TextComponentJSIBinder
class TextComponentJSIBinder : public ViewComponentJSIBinder {
protected:
  facebook::jsi::Object createNativeProps(facebook::jsi::Runtime& rt) override {
    auto nativeProps = ViewComponentJSIBinder::createNativeProps(rt);
    // 文本特有属性
    nativeProps.setProperty(rt, "selectionColor", "Color");
    nativeProps.setProperty(rt, "textDecorationStyle", "TextDecorationStyle");
    nativeProps.setProperty(rt, "textAlignment", "string");
    return nativeProps;
  }
};

// ImageComponentJSIBinder
class ImageComponentJSIBinder : public ViewComponentJSIBinder {
protected:
  facebook::jsi::Object createDirectEventTypes(facebook::jsi::Runtime& rt) override {
    facebook::jsi::Object events = ViewComponentJSIBinder::createDirectEventTypes(rt);
    events.setProperty(rt, "topLoadStart", createDirectEvent(rt, "onLoadStart"));
    events.setProperty(rt, "topLoad", createDirectEvent(rt, "onLoad"));
    events.setProperty(rt, "topError", createDirectEvent(rt, "onError"));
    events.setProperty(rt, "topLoadEnd", createDirectEvent(rt, "onLoadEnd"));
    return events;
  }
};
```

### ComponentDescriptor 和 ShadowNode 关系

**ComponentDescriptor** - 组件描述符：
```cpp
// 定义组件的创建和属性转换
class SampleViewComponentDescriptor final
    : public ConcreteComponentDescriptor<SampleViewShadowNode> {
public:
  SampleViewComponentDescriptor(ComponentDescriptorParameters const& parameters)
      : ConcreteComponentDescriptor(parameters) {}
};
```

**ShadowNode** - 组件内存表示：
```cpp
// 包含组件状态和属性
class SampleViewShadowNode = ConcreteViewShadowNode<
    SampleViewComponentName,
    SampleViewProps,
    SampleViewEventEmitter>;
```

### 组件注册表机制

**组件描述符提供者**：
```cpp
// RNOHCorePackage.h
std::vector<facebook::react::ComponentDescriptorProvider>
createComponentDescriptorProviders() override {
  return {
      facebook::react::concreteComponentDescriptorProvider<
          facebook::react::ViewComponentDescriptor>(),
      facebook::react::concreteComponentDescriptorProvider<
          facebook::react::ImageComponentDescriptor>(),
      facebook::react::concreteComponentDescriptorProvider<
          facebook::react::TextComponentDescriptor>(),
      // ... 更多组件
  };
}
```

**组件 JSI 绑定器注册**：
```cpp
ComponentJSIBinderByString createComponentJSIBinderByName() override {
  return {
      {"RCTView", std::make_shared<ViewComponentJSIBinder>()},
      {"RCTImageView", std::make_shared<ImageComponentJSIBinder>()},
      {"RCTVirtualText", std::make_shared<ViewComponentJSIBinder>()},
      {"Paragraph", std::make_shared<TextComponentJSIBinder>()},
      {"ScrollView", std::make_shared<ScrollViewComponentJSIBinder>()},
      // ... 更多组件映射
  };
}
```

**组件实例创建**：
```cpp
ComponentInstance::Shared createComponentInstance(
    const ComponentInstance::Context& ctx) override {
  if (ctx.componentName == "RootView") {
    return std::make_shared<ViewComponentInstance>(std::move(ctx));
  }
  if (ctx.componentName == "View") {
    return std::make_shared<CustomNodeComponentInstance>(std::move(ctx));
  }
  if (ctx.componentName == "Paragraph") {
    return std::make_shared<TextComponentInstance>(std::move(ctx));
  }
  if (ctx.componentName == "Image") {
    return std::make_shared<ImageComponentInstance>(std::move(ctx));
  }
  // ... 更多组件映射
  return nullptr;
}
```

### TurboModule 注册流程

**cxxTurboModule 注册**：
```cpp
class RNOHCoreTurboModuleFactoryDelegate : public TurboModuleFactoryDelegate {
public:
  SharedTurboModule createTurboModule(Context ctx, const std::string& name)
      const override {
    if (name == "AccessibilityInfo") {
      return std::make_shared<AccessibilityInfoTurboModule>(ctx, name);
    } else if (name == "AlertManager") {
      return std::make_shared<AlertManagerTurboModule>(ctx, name);
    } else if (name == "Animated") {
      return std::make_shared<NativeAnimatedTurboModule>(ctx, name);
    } else if (name == "Appearance") {
      return std::make_shared<AppearanceTurboModule>(ctx, name);
    } else if (name == "DeviceInfo") {
      return std::make_shared<DeviceInfoTurboModule>(ctx, name);
    } else if (name == "DevSettings") {
      return std::make_shared<DevSettingsTurboModule>(ctx, name);
    } else if (name == "Dimensions") {
      return std::make_shared<DimensionsTurboModule>(ctx, name);
    } else if (name == "KeyboardObserver") {
      return std::make_shared<KeyboardObserverTurboModule>(ctx, name);
    } else if (name == "Linking") {
      return std::make_shared<LinkingTurboModule>(ctx, name);
    } else if (name == "Networking") {
      return std::make_shared<NetworkingTurboModule>(ctx, name);
    } else if (name == "Platform") {
      return std::make_shared<PlatformTurboModule>(ctx, name);
    } else if (name == "StatusBar") {
      return std::make_shared<StatusBarTurboModule>(ctx, name);
    } else if (name == "Timing") {
      return std::make_shared<TimingTurboModule>(ctx, name);
    } else if (name == "UIManager") {
      return std::make_shared<UIManagerModule>(
          ctx, name, std::move(m_componentBinderByString)
      );
    }
    return nullptr;
  }
};
```

**ArkTSTurboModule 线程分配**：
```cpp
TaskThread findArkTSTurboModuleThread(const std::string& name) const override {
  if (name == "Networking") {
    return TaskThread::WORKER;
  }
  // 默认 JS 线程
  return TaskThread::JS;
}
```

### 组件创建完整流程

```
1. JS 层调用 React.createElement('View')
    ↓
2. Fabric UIManager 接收创建请求
    ↓
3. ComponentRegistry 查找 ComponentDescriptor
    ↓
4. ComponentDescriptor 创建 ShadowNode
    ↓
5. ComponentInstanceFactory 创建 ComponentInstance
    ↓
6. ComponentInstance 创建 ArkUINode
    ↓
7. MountingManager 处理 Mutation
    ↓
8. ArkUI 渲染组件
```

### 事件注册和处理流程

**事件类型定义**：
```cpp
// 冒泡事件 (Bubbling Events) - 支持冒泡和捕获
facebook::jsi::Object createBubblingEventTypes(facebook::jsi::Runtime& rt) override {
  facebook::jsi::Object events(rt);
  events.setProperty(
      rt, "topTouchStart",
      createBubblingCapturedEvent(rt, "onTouchStart")
  );
  events.setProperty(
      rt, "topTouchMove",
      createBubblingCapturedEvent(rt, "onTouchMove")
  );
  events.setProperty(
      rt, "topTouchEnd",
      createBubblingCapturedEvent(rt, "onTouchEnd")
  );
  return events;
}

// 直接事件 (Direct Events) - 不冒泡
facebook::jsi::Object createDirectEventTypes(facebook::jsi::Runtime& rt) override {
  facebook::jsi::Object events(rt);
  events.setProperty(rt, "topLayout", createDirectEvent(rt, "onLayout"));
  events.setProperty(rt, "topClick", createDirectEvent(rt, "onClick"));
  return events;
}
```

**事件处理流程**：
```
1. 用户交互触发 ArkUI 事件
    ↓
2. ArkUINode::onNodeEvent 接收事件
    ↓
3. ComponentInstance 处理事件
    ↓
4. EventEmitter 转换为 React 事件
    ↓
5. 通过 JSI 发送到 JS 层
    ↓
6. JS 事件处理函数执行
```

---

## 文本测量和渲染机制深度解析 (2026年1月更新)

### TextMeasurer 工作原理

**位置**: `tester/harmony/react_native_openharmony/src/main/cpp/RNOH/TextMeasurer.h`

**核心接口**：
```cpp
class TextMeasurer : public facebook::react::TextLayoutManagerDelegate {
public:
  // 测量文本尺寸
  TextMeasurement measure(
      AttributedString attributedString,
      ParagraphAttributes paragraphAttributes,
      LayoutConstraints layoutConstraints) override;

  // 创建排版对象
  ArkUITypographyBuilder measureTypography(
      AttributedString const& attributedString,
      ParagraphAttributes const& paragraphAttributes,
      LayoutConstraints const& layoutConstraints);

  // 设置测量参数
  void setTextMeasureParams(float fontScale, float scale, int densityDpi, bool halfLeading);
};
```

**测量流程**：

1. **文本预处理** - `dealTextCase`：
```cpp
void dealTextCase(AttributedString& attributedString, ParagraphAttributes const& paragraphAttributes) {
    float fontMultiplier = 1.0;
    if (paragraphAttributes.allowFontScaling) {
        fontMultiplier = m_fontScale;
        if (!isnan(paragraphAttributes.maxFontSizeMultiplier) &&
            paragraphAttributes.maxFontSizeMultiplier >= 1) {
            fontMultiplier = std::min(m_fontScale, paragraphAttributes.maxFontSizeMultiplier);
        }
    }
    for (auto& fragment : attributedString.getFragments()) {
        fragment.textAttributes.fontSize *= fontMultiplier;
    }
}
```

2. **缓存检查** - 通过 `TextMeasureRegistry` 检查是否已测量过相同内容

3. **自适应字体** - 如果启用 `adjustsFontSizeToFit`，使用二分查找计算合适的字体大小

4. **排版构建** - 使用 `ArkUITypographyBuilder` 构建排版对象

### Half Leading 机制

**Half Leading 配置**：
```cpp
// 在 RNInstanceCAPI.cpp 中
auto halfLeading = ArkTSBridge::getInstance()->getMetadata("half_leading") == "true";
textMeasurer->setTextMeasureParams(fontScale, scale, m_densityDpi, halfLeading);
```

**Half Leading 应用**：
```cpp
void addTextFragment(const Fragment& fragment) {
    // 设置 Half Leading
    OH_Drawing_SetTextStyleHalfLeading(textStyle.get(),
        m_halfleading || !isnan(fragment.textAttributes.lineHeight));

    // 行高处理
    if (!isnan(fragment.textAttributes.lineHeight) &&
        fragment.textAttributes.lineHeight > 0) {
        double fontHeight = fragment.textAttributes.lineHeight / fontSize;
        OH_Drawing_SetTextStyleFontHeight(textStyle.get(), fontHeight);
    }
}
```

**Half Leading 变更 (RNOH 5.0.0.500+)**：
- 设置 `lineHeight` 属性时，Half Leading 会被强制开启
- 确保文本在容器中垂直居中位置正确

### 字体注册机制

**字体注册**：
```cpp
void TextMeasurer::registerFont(...) {
    auto fontData = fontFilePath[0] == '/'
        ? readSandboxFile(fontFilePath)
        : readRawFile(resourceManager.get(), fontFilePath);

    m_fontFileContentByFontFamily.emplace(name, std::move(fontData));
    m_fontCollection.reset(); // 重新创建字体集合
    TextMeasureRegistry::getTextMeasureRegistry().clear(); // 清空缓存
}
```

**字体优先级 (RNOH 5.0.0.500+)**：
- 主题字体 > `fontFamily` 属性
- 字体目录变更：`rawfile/fonts/`（旧：`rawfile/assets/assets/fonts/`）

### 文本属性处理

**字体大小**：
```cpp
void addTextFragment(const Fragment& fragment) {
    auto fontSize = fragment.textAttributes.fontSize;
    if (fontSize <= 0) {
        fontSize = 14; // 默认字体大小
    }
    OH_Drawing_SetTextStyleFontSize(textStyle.get(), fontSize * m_scale);
}
```

**字体族处理**：
```cpp
std::vector<const char*> fontFamilies;
if (!m_defaultFontFamilyName.empty()) {
    fontFamilies.emplace_back(m_defaultFontFamilyName.c_str());
}
if (!fragment.textAttributes.fontFamily.empty()) {
    fontFamilies.emplace_back(fragment.textAttributes.fontFamily.c_str());
}
if (fontFamilies.size() > 0) {
    OH_Drawing_SetTextStyleFontFamilies(textStyle.get(),
        fontFamilies.size(), fontFamilies.data());
}
```

**字重映射**：
```cpp
if (fragment.textAttributes.fontWeight.has_value()) {
    OH_Drawing_SetTextStyleFontWeight(
        textStyle.get(),
        mapValueToFontWeight(int(fragment.textAttributes.fontWeight.value()))
    );
}
```

### 文本样式缓存

**TextMeasureRegistry 缓存结构**：
```cpp
struct TextMeasureInfo {
    ArkUITypographyBuilder builder;
    ArkUITypography typography;
};

class TextMeasureRegistry {
    std::unordered_map<std::string, std::shared_ptr<TextMeasureInfo>> m_keyToMeasureInfo;
    folly::EvictingCacheMap<TextMeasureCacheKey, std::shared_ptr<TextMeasureInfo>>
        m_textMeasureInfoCache{facebook::react::kSimpleThreadSafeCacheSizeCap};
};
```

**缓存键生成**：
```cpp
std::string key = std::to_string(m_rnInstanceId) + "_" +
    std::to_string(attributedString.getFragments()[0].parentShadowView.tag) + "_" +
    std::to_string(attributedString.getFragments()[0].parentShadowView.surfaceId);
```

### 字体自适应算法

**二分查找实现**：
```cpp
std::pair<ArkUITypographyBuilder, ArkUITypography> findFitFontSize(
    int maxFontSize, AttributedString& attributedString,
    ParagraphAttributes& paragraphAttributes, LayoutConstraints& layoutConstraints) {

    int minFontSize = 1;
    if (!isnan(paragraphAttributes.minimumFontScale)) {
        minFontSize = ceil(paragraphAttributes.minimumFontScale * maxFontSize);
    }

    int finalFontSize = minFontSize;
    int initFontSize = maxFontSize;

    while(minFontSize <= maxFontSize) {
        int curFontSize = ceil((minFontSize + maxFontSize) * 1.0 / 2);
        float ratio = 1.0 * curFontSize / initFontSize;

        // 应用新的字体大小
        for (int i = 0; i < attributedString.getFragments().size(); ++i) {
            int newFontSize = ceil(
                backupAttributedString.getFragments()[i].textAttributes.fontSize * ratio);
            attributedString.getFragments()[i].textAttributes.fontSize = newFontSize;
        }

        // 检查是否满足约束条件
        auto typography = measureTypography(attributedString, paragraphAttributes, layoutConstraints);
        if (typography.getHeight() <= layoutConstraints.maximumSize.height &&
            (paragraphAttributes.maximumNumberOfLines == 0 ||
             !typography.getExceedMaxLines())) {
            finalFontSize = curFontSize;
            minFontSize = curFontSize + 1;
        } else {
            maxFontSize = curFontSize - 1;
        }
    }

    return std::make_pair(finalTypographyBuilder, finalTypography);
}
```

### AttributedString 富文本支持

**Fragment 结构**：
```cpp
class AttributedString::Fragment {
    std::string string;                        // 文本内容
    TextAttributes textAttributes;            // 文本属性
    ShadowView parentShadowView;              // 父节点信息

    bool isAttachment() const;                // 是否为附件（图片等）
    bool isContentEqual(const Fragment &rhs) const;
};
```

**富文本处理**：
```cpp
void addFragment(const Fragment& fragment) {
    if (!fragment.isAttachment()) {
        addTextFragment(fragment);            // 添加文本片段
    } else {
        addAttachment(fragment);              // 添加附件（图片等）
    }
}
```

### 文本测量与 Yoga 布局集成

**基线对齐支持**：
```cpp
// Yoga 布局中处理基线对齐
if (YGNodeAlignItem(node, child) == YGAlignBaseline) {
    const float ascent = YGBaseline(child, layoutContext) +
        child->getLeadingMargin(YGFlexDirectionColumn, availableInnerWidth).unwrap();
    const float descent = child->getLayout().measuredDimensions[YGDimensionHeight] +
        child->getMarginForAxis(YGFlexDirectionColumn, availableInnerWidth).unwrap() -
        ascent;
    maxAscentForCurrentLine = YGFloatMax(maxAscentForCurrentLine, ascent);
    maxDescentForCurrentLine = YGFloatMax(maxDescentForCurrentLine, descent);
}
```

**测量结果返回**：
```cpp
TextMeasurement TextMeasurer::measure(...) {
    const auto& typography = measureInfo.value()->typography;
    auto height = typography.getHeight();
    auto longestLineWidth = typography.getLongestLineWidth();
    auto attachments = typography.getAttachments();
    return {{.width = longestLineWidth + 0.5, .height = height}, attachments};
}
```

### 行度量获取

**LineMetrics 获取**：
```cpp
void getLineMetrics(std::vector<OH_Drawing_LineMetrics>& data) const {
    auto count = OH_Drawing_TypographyGetLineCount(m_typography.get());
    for (int i = 0; i < count; i++) {
        OH_Drawing_LineMetrics metrics;
        OH_Drawing_TypographyGetLineMetricsAt(m_typography.get(), i, &metrics);
        data.push_back(metrics);
    }
}
```

---

## SafeArea 和安全区域处理深度解析 (2026年1月更新)

### SafeAreaView 实现

**位置**: `react-native-harmony/Libraries/Components/SafeAreaView/SafeAreaView.harmony.tsx`

**核心类型定义**：
```typescript
type SafeAreaInsets = {
  top: number;
  left: number;
  right: number;
  bottom: number;
};

interface SafeAreaTurboModuleProtocol {
  getInitialInsets(): SafeAreaInsets;
}
```

**动态边距更新**：
```typescript
useEffect(function subscribeToSafeAreaChanges() {
  const subscription = (RCTDeviceEventEmitter as any).addListener(
    "SAFE_AREA_INSETS_CHANGE",
    (insets: SafeAreaInsets) => {
      setTopInset(insets.top);
      setBottomInset(insets.bottom);
      setLeftInset(insets.left);
      setRightInset(insets.right);
    }
  );
  return () => subscription.remove();
}, []);
```

### SafeAreaInsetsProvider 实现

**位置**: `tester/harmony/react_native_openharmony/src/main/ets/RNOH/SafeAreaInsetsProvider.ts`

**创建安全区域插入值**：
```typescript
async createSafeAreaInsets(): Promise<SafeAreaInsets> {
  const win = await WindowUtils.getLastWindow(this.uiAbilityContext)

  // 并行获取多种避免区域
  const [displayCutoutInfo, systemAvoidArea, cutoutAvoidArea, navigationAvoidArea] =
    await Promise.all([
      display.getDefaultDisplaySync().getCutoutInfo(),
      win.getWindowAvoidArea(WindowUtils.AvoidAreaType.TYPE_SYSTEM),
      win.getWindowAvoidArea(WindowUtils.AvoidAreaType.TYPE_CUTOUT),
      win.getWindowAvoidArea(WindowUtils.AvoidAreaType.TYPE_NAVIGATION_INDICATOR),
    ]);

  // 构建瀑布屏避免区域
  const waterfallAvoidArea: WindowUtils.AvoidArea = {
    visible: true,
    leftRect: displayCutoutInfo.waterfallDisplayAreaRects.left,
    rightRect: displayCutoutInfo.waterfallDisplayAreaRects.right,
    topRect: displayCutoutInfo.waterfallDisplayAreaRects.top,
    bottomRect: displayCutoutInfo.waterfallDisplayAreaRects.bottom
  };

  // 合并所有避免区域
  const avoidAreas = [cutoutAvoidArea, waterfallAvoidArea, systemAvoidArea, navigationAvoidArea];
  const insets = getSafeAreaInsetsFromAvoidAreas(avoidAreas, win.getWindowProperties().windowRect);

  // 像素转换为 VP 单位
  return mapProps(insets, (val) => px2vp(val));
}
```

**从避免区域计算安全区域**：
```typescript
function getSafeAreaInsetsFromAvoidAreas(avoidAreas: WindowUtils.AvoidArea[], windowSize: {
  width: number,
  height: number
}): SafeAreaInsets {
  return avoidAreas.reduce((currentInsets, avoidArea) => {
    return {
      top: Math.max(currentInsets.top, avoidArea.topRect.height + avoidArea.topRect.top),
      left: Math.max(currentInsets.left, avoidArea.leftRect.width + avoidArea.leftRect.left),
      right: Math.max(currentInsets.right,
        avoidArea.rightRect.left > 0 ? windowSize.width - avoidArea.rightRect.left : 0),
      bottom: Math.max(currentInsets.bottom,
        avoidArea.bottomRect.top > 0 ? windowSize.height - avoidArea.bottomRect.top : 0),
    }
  }, { top: 0, left: 0, right: 0, bottom: 0 });
}
```

### StatusBarTurboModule 实现

**位置**: `tester/harmony/react_native_openharmony/src/main/ets/RNOHCorePackage/turboModules/StatusBarTurboModule.ts`

**状态栏高度获取**：
```typescript
// 获取系统避免区域的高度
const statusBarHeight = windowInstance.getWindowAvoidArea(window.AvoidAreaType.TYPE_SYSTEM).topRect.height;
const scaledStatusBarHeight = px2vp(statusBarHeight);
```

**状态栏样式设置**：
```typescript
async setStyle(style: string, animated: boolean) {
  let systemBarProperties = {
    statusBarContentColor: '#E5FFFFFF',
    enableStatusBarAnimation: animated,
  };
  if (style === 'dark-content') {
    systemBarProperties.statusBarContentColor = '#000000';
  }
  await windowInstance.setWindowSystemBarProperties(systemBarProperties);
}
```

**状态栏显示/隐藏**：
```typescript
async setHidden(hidden: boolean, withAnimation?: string) {
  await windowInstance.setSpecificSystemBarEnabled('status', !hidden);
  this._isStatusBarHidden = hidden;
  this.eventEmitter.emit("SYSTEM_BAR_VISIBILITY_CHANGE", { hidden });
}
```

### 避免区域类型

**HarmonyOS 支持的避免区域类型**：

| 类型 | 说明 | 使用场景 |
|------|------|----------|
| `TYPE_SYSTEM` | 系统栏（状态栏、导航栏） | 常规安全区域 |
| `TYPE_CUTOUT` | 刘海屏/挖孔区域 | 异形屏适配 |
| `TYPE_NAVIGATION_INDICATOR` | 导航指示器 | 底部手势区 |
| `TYPE_WATERFALL` | 瀑布屏区域 | 曲面屏适配 |

### 复杂边距计算逻辑

**顶部边距计算**：
```typescript
const getPaddingTop = (inset: number, pageY: number) => {
  return Math.max(0, inset - (pageY < 0 ? pageY * -1 : pageY));
}
```

**底部边距计算**（处理多种场景）：
```typescript
const getPaddingBottom = (insetBottom: number, insetTop: number, paddingTop: number,
                         height: number, windowHeight: number, pageY: number, positionY: number) => {
  // 处理不可见或视口外的情况
  if (height === 0 || (pageY === 0 && height < windowHeight)) {
    return Math.max(0, insetBottom - (Math.round(windowHeight) - Math.round(height)));
  }

  // 处理全高度组件
  if (Math.round(height) >= Math.round(windowHeight) && pageY === 0) {
    return positionY === 0 ? insetBottom : 0;
  }

  // 默认情况
  return Math.max(0, insetBottom - (windowHeight - height + paddingTop));
};
```

### 窗口避免区域监听

**注册监听**：
```typescript
constructor(private uiAbilityContext: common.UIAbilityContext) {
  this.createSafeAreaInsets().then((insets) => this.updateInsets(insets));
  WindowUtils.getLastWindow(uiAbilityContext).then((window) => {
    window.on("avoidAreaChange", this.onSafeAreaChange.bind(this))
  });
}

// 处理避免区域变化
public onSafeAreaChange() {
  this.createSafeAreaInsets().then((insets) => this.updateInsets(insets))
}
```

### SafeAreaTurboModule 架构

```typescript
export class SafeAreaTurboModule extends UITurboModule {
  public static readonly NAME = 'SafeAreaTurboModule';

  constructor(ctx: UITurboContext, statusBarTurboModule: StatusBarTurboModule) {
    super(ctx);
    this.initialInsets = ctx.safeAreaInsetsProvider.safeAreaInsets;

    // 监听安全区域变化
    this.cleanUpCallbacks.push(ctx.safeAreaInsetsProvider.eventEmitter.subscribe(
      "SAFE_AREA_INSETS_CHANGE",
      this.onSafeAreaChange.bind(this)
    ));

    // 状态栏变化时更新安全区域
    this.cleanUpCallbacks.push(statusBarTurboModule.subscribe(
      "SYSTEM_BAR_VISIBILITY_CHANGE",
      () => ctx.safeAreaInsetsProvider.onSafeAreaChange()
    ));
  }

  public getInitialInsets(): SafeAreaInsets {
    return this.initialInsets;
  }
}
```

### 建议的 SafeAreaContext 实现

```typescript
// SafeAreaContext.tsx
import React, { createContext, useContext, useEffect, useState } from 'react';

// 安全区域上下文
const SafeAreaContext = createContext<{
  insets: SafeAreaInsets;
  windowInsets: SafeAreaInsets;
}>({
  insets: { top: 0, left: 0, right: 0, bottom: 0 },
  windowInsets: { top: 0, left: 0, right: 0, bottom: 0 },
});

// 安全区域提供者组件
export const SafeAreaProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [insets, setInsets] = useState<SafeAreaInsets>({ top: 0, left: 0, right: 0, bottom: 0 });

  useEffect(() => {
    // 获取初始插入值
    const initialInsets = safeAreaTurboModule.getInitialInsets();
    setInsets(initialInsets);

    // 监听变化
    const subscription = RCTDeviceEventEmitter.addListener(
      "SAFE_AREA_INSETS_CHANGE",
      setInsets
    );

    return () => subscription.remove();
  }, []);

  return (
    <SafeAreaContext.Provider value={{ insets, windowInsets: insets }}>
      {children}
    </SafeAreaContext.Provider>
  );
};

// 使用 Hook
export const useSafeArea = () => {
  const context = useContext(SafeAreaContext);
  if (!context) {
    throw new Error('useSafeArea must be used within SafeAreaProvider');
  }
  return context.insets;
};
```

### 使用示例

**基本使用**：
```typescript
import { SafeAreaView, StatusBar, View } from 'react-native';

export function App() {
  const [isStatusBarHidden, setIsStatusBarHidden] = useState(false);

  return (
    <>
      <StatusBar hidden={isStatusBarHidden} />
      <SafeAreaView style={{ flex: 1, backgroundColor: '#f0f0f0' }}>
        <View style={{ flex: 1, alignItems: 'center', justifyContent: 'center' }}>
          {/* 应用内容 */}
        </View>
      </SafeAreaView>
    </>
  );
}
```

**嵌套 SafeAreaView**：
```typescript
export function NestedExample() {
  return (
    <SafeAreaView style={{ flex: 1, backgroundColor: 'black' }}>
      <View style={{ flex: 1, padding: 20 }}>
        <SafeAreaView style={{ backgroundColor: 'blue' }}>
          <View style={{ padding: 16, backgroundColor: 'yellow' }}>
            <Text>内层 SafeAreaView 不会添加额外的边距</Text>
          </View>
        </SafeAreaView>
      </View>
    </SafeAreaView>
  );
}
```

---

## 第三方库适配和兼容性指南 (2026年1月更新)

### 补丁化移植策略

**核心原则**：
- 基于上游社区稳定版本进行适配
- 使用 `.harmony.js` 补丁文件处理鸿蒙特有差异
- 保持 RN 层 API 一致性

**库版本兼容性警告**（2025年5月）：
> 最新的是0.3.0版本，但是千万不要直接输入命令安装，它会默认安装最新版本的三方库导致和RN的openharmony版本不兼容！

### 第三方库列表

**官方资源**：
- **文档**: https://react-native-oh-library.github.io/usage-docs/
- **GitHub**: https://github.com/react-native-oh-library (288+ 仓库)

**常用库适配状态**：

| 库名 | OpenHarmony 适配 | 包名 |
|------|-----------------|------|
| react-navigation | ✅ 支持 | @react-navigation/* |
| react-native-gesture-handler | ✅ 支持 | @react-native-oh-tpl/react-native-gesture-handler |
| react-native-reanimated | ✅ 支持 | @react-native-oh-tpl/react-native-reanimated |
| react-native-webview | ✅ 支持 | @ohmi/react-native-webview |
| react-native-safe-area-context | ✅ 支持 | react-native-safe-area-context |
| react-native-screens | ✅ 支持 | react-native-screens |
| react-native-pager-view | ✅ 支持 | @react-native-oh-tpl/react-native-pager-view |
| react-native-tab-view | ✅ 支持 | react-native-tab-view |
| react-native-haptic-feedback | ✅ 支持 | @react-native-ohos/react-native-haptic-feedback |
| react-native-device-info | ✅ 支持 | react-native-device-info |
| react-native-maps | ✅ 支持 | @tuya-oh/react-native-maps |
| @react-native-oh-tpl/react-native-fast-image | ✅ 支持 | @react-native-ohos/react-native-fast-image (v8.6.4-rc.1) |
| @react-native-oh-tpl/react-native-async-storage | ✅ 支持 | @react-native-oh-tpl/react-native-async-storage |
| @react-native-ohos/react-native-permissions | ✅ 支持 | @react-native-ohos/react-native-permissions |
| @react-native-oh-tpl/clipboard | ✅ 支持 | @react-native-oh-tpl/clipboard |
| jpush-react-native | ✅ 支持 | jpush-react-native (极光推送) |

### 库链接流程

**自动链接**：
```bash
npm install @react-native-oh-tpl/xxx-library
```

**手动链接**（如需要）：
```bash
# 1. 将 .har 文件放入 libs/ 目录
# 2. 配置 CMakeLists.txt
# 3. 在 PackageProvider.ets 中注册
```

**CMakeLists.txt 配置**：
```cmake
target_link_libraries(entry PUBLIC
  libace_napi.z.so
  libnative_drawing.so
  # 添加第三方库
)
```

**PackageProvider.ets 注册**：
```typescript
import { PackageProvider } from './PackageProvider';
import { ThirdPartyLibraryPackage } from '@react-native-oh-tpl/xxx-library';

export default new PackageProvider({
  packages: [new ThirdPartyLibraryPackage()]
});
```

### react-navigation 集成

**基础用法**：
```javascript
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';

const Stack = createNativeStackNavigator();

function App() {
  return (
    <NavigationContainer>
      <Stack.Navigator>
        <Stack.Screen name="Home" component={HomeScreen} />
        <Stack.Screen name="Details" component={DetailsScreen} />
      </Stack.Navigator>
    </NavigationContainer>
  );
}
```

**底部导航栏**：
```javascript
import { createBottomTabNavigator } from '@react-navigation/bottom-tabs';

const Tab = createBottomTabNavigator();

function TabNavigator() {
  return (
    <Tab.Navigator>
      <Tab.Screen name="Home" component={HomeScreen} />
      <Tab.Screen name="Profile" component={ProfileScreen} />
    </Tab.Navigator>
  );
}
```

**注意事项**：
- HarmonyOS 原生 Navigation 组件可能被状态栏遮挡
- 使用 SafeAreaView 解决

### WebView 集成

**包**: `@ohmi/react-native-webview` (v13.10.2-rc.0.3.11)

**使用示例**：
```javascript
import { WebView } from '@ohmi/react-native-webview';

<WebView
  source={{ uri: 'https://example.com' }}
  onMessage={(event) => console.log(event.nativeEvent.data)}
  javaScriptEnabled={true}
  domStorageEnabled={true}
/>
```

**桥接通信**：
```javascript
// RN → WebView
webViewRef.current.postMessage('Hello from RN');

// WebView → RN
window.ReactNativeWebView.postMessage('Hello from WebView');
```

### 推送通知集成

**极光推送**：
- **文档**: https://github.com/react-native-oh-library/usage-docs/blob/master/zh-cn/jpush-react-native.md
- **包**: `jpush-react-native`

**Push Kit** (华为官方):
- **文档**: https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/push-send-alert

### 图片加载优化

**Fast Image**：
```javascript
import FastImage from '@react-native-ohos/react-native-fast-image';

// 基础用法
<FastImage
  source={{ uri: 'https://example.com/image.jpg' }}
  style={{ width: 200, height: 200 }}
/>

// 缓存控制
<FastImage
  source={{
    uri: 'https://example.com/image.jpg',
    priority: FastImage.priority.high,
  }}
  resizeMode={FastImage.resizeMode.contain}
/>

// 图片预加载
FastImage.preload([
  { uri: 'https://example.com/image1.jpg' },
  { uri: 'https://example.com/image2.jpg' },
]);
```

**优先级**：
- `FastImage.priority.low`
- `FastImage.priority.normal`
- `FastImage.priority.high`

**缓存模式**：
- `FastImage.cacheMode.ignoreCache`
- `FastImage.cacheMode.web`
- `FastImage.cacheMode.cacheOnly`

---

## 网络资源和参考链接汇总 (2026年1月更新)

### 官方文档和资源

| 资源 | 链接 |
|------|------|
| RNOH 官方仓库 | https://gitcode.com/openharmony-sig/ohos_react_native |
| RNOH 三方库文档 | https://react-native-oh-library.github.io/usage-docs/ |
| react-native-oh-library | https://github.com/react-native-oh-library |
| HarmonyOS Ability 文档 | https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/ability-kit |
| HiDumper 文档 | https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/hidumper |
| ContentSlot 文档 | https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/arkts-rendering-control-contentslot |
| OpenHarmony 文档 | https://gitee.com/openharmony/docs |
| HarmonyOS 沉浸式界面开发 | https://gitee.com/openharmony/docs/blob/master/zh-cn/third-party-cases/immersion-mode.md |

### 技术文章和教程

**架构分析**：
- [鸿蒙版React Native架构浅析](https://openharmonycrossplatform.csdn.net/690f13f90e4c466a32e5f12f.html) - CSDN
- [React Native跨平台鸿蒙开发高级应用原理](https://ai6s.net/69265768791c233193d04253.html)
- [React Native 新架构 - 渲染流水线](https://reactnative.cn/architecture/render-pipeline)
- [ReactNative新架构Fabric2024年实践踩坑总结](https://my.oschina.net/emacs_9733956/blog/19177279)
- [React Native 通信机制详解 - 新架构](https://juejin.cn/post/7576210031873343529)
- [鸿蒙中RNOH架构介绍](https://juejin.cn/post/7473777534543740937)

**组件开发**：
- [OpenHarmony使用ContentSlot占位组件管理Native API创建的组件](https://blog.csdn.net/m0_68635815/article/details/155262925)
- [React Native for OpenHarmony：跨平台应用开发实战与架构解析](https://openharmonycrossplatform.csdn.net/6960a99a6554f1331aa0d32a.html)

**迁移和适配**：
- [React Native 老项目适配Harmony](https://juejin.cn/post/7458326985078587407)
- [已有React Native工程如何适配HarmonyOS](https://www.woshipm.com/share/6310666.html)
- [鸿蒙版React Native：跨平台应用开发实战与架构解析](https://openharmonycrossplatform.csdn.net/690f13f90e4c466a32e5f12f.html)

**性能优化**：
- [小红书APP的全新鸿蒙NEXT端性能优化技术实践](http://www.52im.net/thread-4821-1-1.html)
- [HarmonyOS NEXT鸿蒙（开发进阶）基于RN框架实现高性能](https://blog.csdn.net/adaedwa187545/article/details/143113058)
- [同一套RN代码，运行在原生鸿蒙端结果不一样？](https://blog.csdn.net/qq_15873073/article/details/144356790)

**TurboModule 开发**：
- [RN鸿蒙自定义TurboModule \| 华为开发者联盟](https://developer.huawei.com/consumer/cn/blog/topic/03169292089068014)
- [RN鸿蒙自定义TurboModule，调用原生方法](https://openharmonycrossplatform.csdn.net/693fd4420800f3458b828881.html)
- [自定义TurboModule的实现](https://gitee.com/openharmony-sig/ohos_react_native/blob/master/docs/zh-cn/TurboModule.md)

**权限处理**：
- [向用户申请权限](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/request-user-authorization)
- [鸿蒙开发（NEXT/API 12）向用户申请授权](https://blog.csdn.net/m0_70748845/article/details/142741380)
- [请求用户权限& 打开应用设置界面](https://juejin.cn/post/7414055548788604966)

**深度链接**：
- [使用Deep Linking实现应用间跳转](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/deep-linking-startup)
- [鸿蒙开发基础—— 使用Deep Linking实现应用间跳转](https://blog.csdn.net/guangxishaoji/article/details/145670748)
- [HarmonyOS App Linking：系统级深度链接能力](https://www.infoq.cn/article/2vdvlxrmx0dnuvqc5suh)

**底部导航**：
- [RN for OpenHarmony 实战TodoList 项目：底部Tab 栏](https://openharmonycrossplatform.csdn.net/695e4d136554f1331aa05a8d.html)
- [TabContent - HarmonyOS开发者文档](https://developer.huawei.com/consumer/cn/doc/harmonyos-references/ts-container-tabcontent)

**数据持久化**：
- [应用数据持久化](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/app-data-persistence)
- [Harmony之路：数据持久化——Preferences本地存储方案](https://blog.csdn.net/weixin_63328892/article/details/155754738)

**图片加载**：
- [HarmonyOS 鸿蒙Next中长网络图片列表渲染性能优化](https://bbs.itying.com/topic/694b6b67c504c50058fca5a2)
- [ImageKnife专门为OpenHarmony打造的一款图像加载缓存库](https://juejin.cn/post/7403182163632341011)

**推送通知**：
- [HarmonyOS 推送通知开发实战](https://blog.csdn.net/m0_74456227/article/details/150503912)
- [OpenHarmony教程指南-自定义通知推送](https://zhuanlan.zhihu.com/p/686028537)

### 社区资源

- **华为开发者联盟**: https://developer.huawei.com/consumer/cn/
- **OpenHarmony 官方论坛**: https://forums.openharmony.cn/
- **CSDN OpenHarmony 专区**: https://openharmonycrossplatform.csdn.net/
- **掘金 OpenHarmony 标签**: https://juejin.cn/tag/OpenHarmony

---

## 测试计划与调试

### tester 测试应用

**位置**: `/Users/wuzhao/Desktop/ty/rnoh/react-native-openharmony/tester`

**启动步骤**：
```bash
cd tester
npm run i              # 安装依赖（不是 npm i）
npm start              # 启动 Metro 服务器
# 在 DevEco Studio 中打开 tester/harmony 项目并运行
```

### 测试计划文档

**位置**: `tester/plan/`

**文档命名规范**：
- `iteration_XXX_YYY_[主题].md` - 每个迭代（50个测试）的详细计划
- `SUMMARY_ZZZ_FINAL.md` - 迭代完成后的总结报告

**迭代示例**：
| 文件 | 测试范围 | 关键内容 |
|------|----------|----------|
| iteration_901_950_image_gesture_scroll_input_style.md | 图片、手势、滚动、输入框 | Image组件、手势冲突、ScrollView |
| iteration_851_900_network_storage_worker_bridge.md | 网络、存储、Worker、Bridge | NetworkingTurboModule、AsyncStorage |
| iteration_801_850_animation_layout_rendering.md | 动画、布局、渲染 | Animated API、Yoga布局 |
| iteration_751_800_dataflow_state_events.md | 数据流、状态、事件 | Context、事件系统 |

### 常用测试命令

```bash
# 运行特定测试
npm test -- --testNamePattern="StatusBar"

# 检查类型
npm run typecheck

# 验证包
npm run verify
```

### 调试技巧

1. **查看日志**：使用 DevEco Studio 的 Hilog 工具
2. **检查 TurboModule**：确认 `getConstants()` 返回值
3. **调试布局问题**：使用 React DevTools 查看 ShadowTree
4. **性能分析**：使用 Chrome DevTools 的 Performance 面板

---

## 手势系统核心机制

### TouchDispatcher 触摸分发器

**位置**: `RNOH/TouchDispatcher.ets`

```typescript
export class TouchDispatcher {
  private static MEANINGFUL_MOVE_THRESHOLD = 1;  // 移动阈值
  private targetTagByTouchId: Map<number, Tag> = new Map();  // 触摸ID到组件Tag的映射

  public handleTouchEvent(touchEvent: TouchEvent) {
    // 1. 清理已删除的组件
    this.maybeRemoveDeletedTargets();
    // 2. 记录新的触摸目标
    this.recordNewTouchTargets(touchEvent);
    // 3. 取消已处理的触摸
    this.cancelHandledTouches(timestamp);
    // 4. 过滤无意义的移动事件
    if (this.canIgnoreEvent(touchEvent)) return;
    // 5. 发送触摸事件到 C++ 层处理
    this.rnInstance.emitComponentEvent(-1, RNOHEventEmitRequestHandlerName.Touch, touchEvent);
  }

  // 性能优化：忽略微小移动
  private isMoveMeaningful(currentTouch: TouchObject, previousTouch: TouchObject): boolean {
    const dx = currentTouch['pageX'] - previousTouch['pageX'];
    const dy = currentTouch['pageY'] - previousTouch['pageY'];
    return dx*dx + dy*dy > TouchDispatcher.MEANINGFUL_MOVE_THRESHOLD;
  }
}
```

### ResponderLockDispatcher 手势冲突解决

**位置**: `RNOH/ResponderLockDispatcher.ts`

```typescript
export class ResponderLockDispatcher {
  // 按组件 Tag -> 锁定源 -> 是否锁定的映射
  private isLockedByOriginByTag: Map<Tag, Map<Origin, boolean>> = new Map()

  public onBlockResponder(tag: Tag, origin: Origin) {
    const tags = this.componentManagerRegistry.getComponentManagerLineage(tag)
      .map(d => d.getTag())

    tags.forEach((tag) => {
      const currentNumberOfLocks = this.getTotalNumberOfLocks(tag)
      if (currentNumberOfLocks === 0) {
        // 首次锁定时，阻塞原生响应器
        this.componentCommandHub.dispatchCommand(
          tag,
          RNOHComponentCommand.BLOCK_NATIVE_RESPONDER,
          undefined
        )
      }
      this.isLockedByOriginByTag.get(tag).set(origin, true)
    })
  }
}
```

### ScrollView 嵌套滚动处理

**位置**: `RNOHCorePackage/ComponentInstances/ScrollViewComponentInstance.cpp`

```cpp
void ScrollViewComponentInstance::onNativeResponderBlockChange(bool isBlocked) {
  m_isNativeResponderBlocked = isBlocked;
  auto newEnableScrollInteraction = isEnableScrollInteraction(m_props && m_props->scrollEnabled);

  if (newEnableScrollInteraction != m_enableScrollInteraction) {
    m_enableScrollInteraction = newEnableScrollInteraction;
    m_scrollNode.setEnableScrollInteraction(m_enableScrollInteraction);
  }

  // 处理 PullToRefreshView 的父子手势冲突
  auto parent = std::dynamic_pointer_cast<PullToRefreshViewComponentInstance>(this->getParent().lock());
  if (parent) {
    if (isBlocked) {
      parent->setRefreshPullDownRation(0.0);  // 禁用下拉刷新
    } else {
      parent->setRefreshPullDownRation(1.0);  // 启用下拉刷新
    }
  }
}
```

---

## Animated API 原生驱动架构

### AnimatedNodesManager 动画核心引擎

**位置**: `RNOHCorePackage/TurboModules/Animated/AnimatedNodesManager.cpp`

```cpp
class AnimatedNodesManager {
  // 节点存储
  std::unordered_map<Tag, std::unique_ptr<AnimatedNode>> m_nodeByTag;
  // 动画驱动器
  std::unordered_map<Tag, std::unique_ptr<AnimationDriver>> m_animationById;
  // 事件驱动器
  std::vector<std::unique_ptr<EventAnimationDriver>> m_eventDrivers;
  // 需要更新的节点集合
  std::unordered_set<Tag> m_nodeTagsToUpdate;

  // 支持的节点类型
  void createNode(Tag tag, folly::dynamic const& config) {
    auto type = config["type"].asString();
    if (type == "props") {
      node = std::make_unique<PropsAnimatedNode>(config, *this);
    } else if (type == "style") {
      node = std::make_unique<StyleAnimatedNode>(config, *this);
    } else if (type == "value") {
      node = std::make_unique<ValueAnimatedNode>(config);
    } else if (type == "transform") {
      node = std::make_unique<TransformAnimatedNode>(config, *this);
    } else if (type == "interpolation") {
      node = std::make_unique<InterpolationAnimatedNode>(config, *this);
    }
    // ... 更多节点类型
  }
}
```

### SpringAnimationDriver 弹簧动画

**位置**: `RNOHCorePackage/TurboModules/Animated/Drivers/SpringAnimationDriver.cpp`

```cpp
void SpringAnimationDriver::advance(double deltaTime) {
  double c = m_damping;    // 阻尼系数
  double m = m_mass;       // 质量
  double k = m_stiffness;  // 刚度

  // 计算阻尼比
  double zeta = c / (2 * std::sqrt(k * m));
  double omega0 = std::sqrt(k / m);

  if (zeta < 1) {
    // 欠阻尼状态 - 产生震荡
    double envelope = std::exp(-zeta * omega0 * t);
    position = m_toValue - envelope * (
      (v0 + zeta * omega0 * x0) / omega1 * std::sin(omega1 * t) +
      x0 * std::cos(omega1 * t)
    );
  }

  // 超调处理
  if (m_overshootClamping && isOvershooting()) {
    m_fromValue = m_toValue;
    m_position = m_toValue;
    m_velocity = 0;
  }
}
```

### VSync 驱动机制

```cpp
void NativeAnimatedTurboModule::runUpdates() {
  auto now = std::chrono::high_resolution_clock::now();
  auto frameTime = std::chrono::duration_cast<std::chrono::nanoseconds>(
      now.time_since_epoch());

  // 运行动画更新
  auto tagsToUpdate = this->m_animatedNodesManager.runUpdates(frameTime.count());

  // 在主线程更新 Native Props
  if (m_ctx.taskExecutor->isOnTaskThread(TaskThread::MAIN)) {
    this->setNativeProps(tagsToUpdate);
  } else {
    m_ctx.taskExecutor->runTask(TaskThread::MAIN, [...]());
  }
}
```

---

## 网络与图片加载

### NetworkingTurboModule 核心功能

**位置**: `RNOHCorePackage/turboModules/Networking/NetworkingTurboModule.ts`

**支持的 HTTP 方法**：
```typescript
private REQUEST_METHOD_BY_NAME: Record<string, http.RequestMethod> = {
  OPTIONS: http.RequestMethod.OPTIONS,
  GET: http.RequestMethod.GET,
  POST: http.RequestMethod.POST,
  PUT: http.RequestMethod.PUT,
  DELETE: http.RequestMethod.DELETE,
}
```

**中文编码处理**：
```typescript
private getEncodedURI(str: string): string {
  if (this.isEncodedURI(str)) {
    return str;
  }
  return str.replace(/[\u4e00-\u9fa5]/g, (char) => encodeURIComponent(char));
}
```

### RemoteImageCache 三级缓存

**架构**：
```
内存缓存 (128个对象) → 磁盘缓存 (128个文件) → 网络下载
```

**内存缓存**：
```typescript
export class RemoteImageMemoryCache extends RemoteImageCache<RemoteImageSource> {
  // 默认容量：128个图片源对象
  // LRU淘汰策略：当超过最大容量时，删除最旧的条目
}
```

**磁盘缓存**：
```typescript
export class RemoteImageDiskCache extends RemoteImageCache<boolean> {
  private getCacheKey(uri: string): string {
    const reg = /[^a-zA-Z0-9 -]/g;
    return uri.replace(reg, '');
  }

  // 启动时恢复已有缓存文件
  constructor(maxSize: number, cacheDir: string) {
    const filenames: string[] = fs.listFileSync(cacheDir);
    filenames.forEach(filename => {
      this.set(filename, true);
    });
  }
}
```

**查询优先级**：
```typescript
public queryCache(uri: string): 'memory' | 'disk' | undefined {
  if (this.diskCache.has(uri)) {
    return 'disk';
  }
  if (this.memoryCache.has(uri)) {
    return 'memory';
  }
  return undefined;
}
```

### 图片格式支持

**支持的格式**：
- 静态图片：JPEG, PNG, GIF (单帧), BMP, WEBP
- 动画格式：GIF (多帧)
- 检测方法：`await imageSource.getFrameCount()`

**解码流程**：
```typescript
const imageSource = image.createImageSource(uri);
const frameCounter = await imageSource.getFrameCount();

if (frameCounter === 1) {
  // 静态图片 → PixelMap
  const pixelMap = await imageSource.createPixelMap();
} else {
  // 动画 GIF → 保持原格式
  this.imageSource = new ImageSourceHolder(remoteImage.getLocation());
}
```

### 网络请求优化

**请求去重机制**：
```typescript
private activeRequestByUrl: Map<string, Promise<FetchResult>> = new Map();
private activePrefetchByUrl: Map<string, Promise<boolean>> = new Map();

// 防止同一个URL的重复并发请求
// 多个组件请求同一图片时共享请求结果
```

**流式传输**：
```typescript
httpRequest.requestInStream(url, finalRequestOptions, (err, data) => {
  // 流式处理，避免内存溢出
});

httpRequest.on('dataReceive', (chunk) => {
  dataChunks.push(chunk);
  currentLength += chunk.byteLength;
  // 触发进度回调
});
```

---

## 布局系统 (Yoga Layout)

### Yoga 集成架构

RNOH 使用 Facebook Yoga 作为核心布局引擎，与鸿蒙 ArkUI 深度集成：

```
JS 层 (Flexbox 样式)
    ↓
ShadowTree (YogaLayoutableShadowNode)
    ↓
Yoga 计算引擎 (YGNode)
    ↓
LayoutMetrics 转换
    ↓
ArkUI 布局应用
```

### 核心类：YogaLayoutableShadowNode

**文件位置**: `RNOH/arkui/YogaLayoutableShadowNode.h`

```cpp
class YogaLayoutableShadowNode : public LayoutableShadowNode {
  mutable YGConfig yogaConfig_;      // Yoga 配置
  mutable YGNode yogaNode_;         // Yoga 节点
  ListOfShared yogaLayoutableChildren_;  // Yoga 子节点列表

public:
  void layoutTree(LayoutContext, LayoutConstraints) override;
  void layout(LayoutContext) override;
  void cleanLayout() override;
  void dirtyLayout() override;
};
```

### LayoutContext 结构

```cpp
struct LayoutContext {
  Float pointScaleFactor{1.0};                          // 点缩放因子
  std::vector<LayoutableShadowNode const *> *affectedNodes{}; // 受影响节点列表
  bool swapLeftAndRightInRTL{false};                    // RTL 左右交换标志
  Float fontSizeMultiplier{1.0};                        // 字体大小倍数
  Point viewportOffset{};                               // 视口偏移
};
```

**线程局部存储优化**：
```cpp
thread_local LayoutContext threadLocalLayoutContext;
```

### 布局计算流程

```cpp
void YogaLayoutableShadowNode::layoutTree(
    LayoutContext layoutContext,
    LayoutConstraints layoutConstraints) {

  // 1. 设置 Yoga 配置
  yogaConfig_.pointScaleFactor = layoutContext.pointScaleFactor;

  // 2. 转换布局约束
  auto &yogaStyle = yogaNode_.getStyle();
  yogaStyle.maxDimensions() = ...;
  yogaStyle.minDimensions() = ...;

  // 3. 处理 RTL 方向
  auto direction = yogaDirectionFromLayoutDirection(
      layoutConstraints.layoutDirection);

  // 4. 执行 Yoga 布局计算
  YGNodeCalculateLayout(&yogaNode_, ownerWidth, ownerHeight, direction);

  // 5. 更新布局指标
  if (yogaNode_.getHasNewLayout()) {
    auto layoutMetrics = layoutMetricsFromYogaNode(yogaNode_);
    setLayoutMetrics(layoutMetrics);
  }

  // 6. 递归布局子节点
  layout(layoutContext);
}
```

### 并发安全机制

**节点克隆处理**：
```cpp
void YogaLayoutableShadowNode::adoptYogaChild(size_t index) {
  if (childNode.yogaNode_.getOwner() == nullptr) {
    // 直接拥有子节点
    childNode.yogaNode_.setOwner(&yogaNode_);
  } else {
    // 需要克隆避免 ABA 问题
    auto clonedChildNode = childNode.clone({});
    replaceChild(childNode, clonedChildNode, index);
  }
}
```

**魔术数标记**：`0xBADC0FFEE0DDF00D` 用于标识所有权变更

### 布局约束处理

**LayoutConstraints 结构**：
```cpp
struct LayoutConstraints {
  Size minimumSize{0, 0};                              // 最小尺寸
  Size maximumSize{                                    // 最大尺寸
      std::numeric_limits<Float>::infinity(),
      std::numeric_limits<Float>::infinity()};
  LayoutDirection layoutDirection{LayoutDirection::Undefined};  // 布局方向

  Size clamp(const Size &size) const {
    return Size{
        std::clamp(size.width, minimumSize.width, maximumSize.width),
        std::clamp(size.height, minimumSize.height, maximumSize.height)};
  }
};
```

### 约束转换机制

| React Native 约束 | Yoga 测量模式 |
|-------------------|---------------|
| 具体宽度 + 具体高度 | Exactly + Exactly |
| 具体宽度 + undefined | Exactly + Undefined |
| undefined + 具体高度 | Undefined + Exactly |

### 布局脏检查优化

```cpp
void YogaLayoutableShadowNode::cleanLayout() {
  yogaNode_.setDirty(false);
}

void YogaLayoutableShadowNode::dirtyLayout() {
  yogaNode_.setDirty(true);
}
```

**优化原理**：只有标记为 dirty 的节点才会重新计算布局

---

## Text 组件渲染

### Text 组件架构

```
React Native JS 层
    ↓
Props 解析 (TextPropsParser)
    ↓
Shadow Tree (TextShadowNode)
    ↓
Component Instance (TextComponentInstance)
    ↓
ArkUI Layer (TextNode)
    ↓
文本测量 (TextMeasurer)
    ↓
最终渲染
```

### 核心类定义

**TextNode** - ArkUI 文本节点：
```cpp
class TextNode : public ArkUINode {
private:
    enum {
        FLAG_PADDING = 0,
        FLAG_MINFONTSIZE,
        FLAG_MAXFONTSIZE,
        FLAG_COPYOPTION,
        FLAG_ENABLE,
        FLAG_MAX
    };
    bool m_initFlag[FLAG_MAX] = {0};
    float m_minFontSize = 0.0;
    float m_maxFontSize = 0.0;
    int32_t m_testCopyOption = 0;
    std::shared_ptr<TextMeasureInfo> m_measureInfo;

public:
    void setTextContent(const std::string &content);
    void setFontColor(uint32_t color);
    void setFontSize(float size);
    void setFontWeight(int32_t weight);
    void setTextLineHeight(float lineHeight);
    void setTextDecoration(int32_t type, uint32_t color);
};
```

### TextComponentInstance 实现

**文件位置**: `RNOHCorePackage/ComponentInstances/TextComponentInstance.h`

```cpp
class TextComponentInstance : public CppComponentInstance<
    facebook::react::ParagraphShadowNode>,
                            public TextNodeDelegate {
private:
    TextNode m_textNode{};
    StackNode* m_stackNodePtr = nullptr;
    std::vector<std::shared_ptr<ArkUINode>> m_childNodes{};
    FragmentTouchTargetByTag m_fragmentTouchTargetByTag{};
    bool m_hasCheckNesting = false;

public:
    void onPropsChanged(SharedConcreteProps const& textProps) override;
    void onChildInserted(ComponentInstance::Shared const& childComponentInstance,
                        std::size_t index) override;
};
```

### TextMeasurer - 文本测量引擎

**文件位置**: `RNOH/TextMeasurer.h`

```cpp
class TextMeasurer : public facebook::react::TextLayoutManagerDelegate {
public:
    facebook::react::TextMeasurement measure(
        facebook::react::AttributedString attributedString,
        facebook::react::ParagraphAttributes paragraphAttributes,
        facebook::react::LayoutConstraints layoutConstraints) override;

    ArkUITypographyBuilder measureTypography(
        facebook::react::AttributedString const& attributedString,
        facebook::react::ParagraphAttributes const& paragraphAttributes,
        facebook::react::LayoutConstraints const& layoutConstraints);

private:
    std::shared_ptr<TextMeasureRegistry> m_textMeasureRegistry;
    std::shared_ptr<FontCollection> m_fontCollection;
};
```

### 文本属性分类

**颜色属性**：
- `foregroundColor` - 前景色
- `backgroundColor` - 背景色
- `opacity` - 透明度

**字体属性**：
- `fontFamily` - 字体系列
- `fontSize` - 字号
- `fontWeight` - 字重 (100-900)
- `fontStyle` - 字体样式 (normal/italic)
- `letterSpacing` - 字间距

**段落样式**：
- `lineHeight` - 行高
- `alignment` - 对齐方式
- `baseWritingDirection` - 书写方向

**文本装饰**：
- `textDecorationColor` - 装饰颜色
- `textDecorationLineType` - 装饰线类型
- `textDecorationStyle` - 装饰样式

### 字重映射表

```cpp
int32_t TextConversions::getArkUIFontWeight(int32_t fontWeight) {
    switch (fontWeight) {
        case (int)facebook::react::FontWeight::Weight100:
            return ArkUI_FontWeight::ARKUI_FONT_WEIGHT_W100;
        case (int)facebook::react::FontWeight::Weight400:
            return ArkUI_FontWeight::ARKUI_FONT_WEIGHT_W400;
        case (int)facebook::react::FontWeight::Weight700:
            return ArkUI_FontWeight::ARKUI_FONT_WEIGHT_W700;
        case (int)facebook::react::FontWeight::Weight900:
            return ArkUI_FontWeight::ARKUI_FONT_WEIGHT_W900;
        default:
            return ArkUI_FontWeight::ARKUI_FONT_WEIGHT_NORMAL;
    }
}
```

### AttributedString 处理

**片段结构**：
```cpp
class Fragment {
    std::string string;
    TextAttributes textAttributes;
    ShadowView parentShadowView;
    bool isAttachment() const;  // 是否为附件（Image、View）
};
```

**文本转换处理**：
1. 检查缓存
2. 处理字体缩放 (`allowFontScaling`)
3. 处理文本转换 (`textTransform`)
4. 字体大小自适应 (`adjustsFontSizeToFit`)
5. 使用 `ArkUITypographyBuilder` 测量

### 样式继承机制

```typescript
// 父 Text 组件的样式会被子 Text 组件继承
<Text style={{fontSize: 20, color: 'red'}}>
  这里的文本继承红色和 20pt 字号
  <Text style={{color: 'blue'}}>
    这里的文本继承 20pt 字号，但覆盖为蓝色
  </Text>
</Text>
```

### 文本选择功能

```typescript
// 设置文本可选性
int32_t testCopyOption = textProps->isSelectable ?
    ArkUI_CopyOptions::ARKUI_COPY_OPTIONS_LOCAL_DEVICE :
    ArkUI_CopyOptions::ARKUI_COPY_OPTIONS_NONE;

// 设置选择颜色
if (textProps->rawProps.count("selectionColor") != 0) {
    uint32_t selectionColor = textProps->rawProps["selectionColor"].asInt();
    selectionColor = (selectionColor & 0x00ffffff) | 0x33000000;
    m_textNode.setSelectedBackgroundColor(selectionColor);
}
```

### 数据检测支持

**支持的类型**：
- `address` - 地址
- `link` - 链接
- `phoneNumber` - 电话号码
- `email` - 邮箱
- `all` - 全部

---

## SafeArea 和 StatusBar

### SafeArea 架构

```
鸿蒙系统事件 (avoidAreaChange)
    ↓
SafeAreaInsetsProvider (单例)
    ↓
EventEmitter
    ↓
SafeAreaTurboModule
    ↓
React Native 事件
    ↓
SafeAreaView 组件
```

### SafeAreaInsetsProvider

**文件位置**: `RNOHCorePackage/turboModules/SafeAreaInsetsProvider.ts`

```typescript
export class SafeAreaInsetsProvider {
  private currentInsets: SafeAreaInsets = {
    top: 0,
    left: 0,
    right: 0,
    bottom: 0
  };

  public eventEmitter: EventEmitter<{
    "SAFE_AREA_INSETS_CHANGE": [SafeAreaInsets]
  }>;

  constructor(uiAbilityContext: common.UIAbilityContext) {
    // 监听窗口避让区域变化
    WindowUtils.getLastWindow(uiAbilityContext).then((window) => {
      window.on("avoidAreaChange", this.onSafeAreaChange.bind(this));
    });
  }

  private onSafeAreaChange() {
    this.createSafeAreaInsets().then((insets) => {
      this.currentInsets = insets;
      this.eventEmitter.emit("SAFE_AREA_INSETS_CHANGE", insets);
    });
  }
}
```

### Insets 计算逻辑

**多区域合并算法**：
```typescript
function getSafeAreaInsetsFromAvoidAreas(
  avoidAreas: WindowUtils.AvoidArea[],
  windowSize: {width: number, height: number}
): SafeAreaInsets {
  return avoidAreas.reduce((currentInsets, avoidArea) => {
    return {
      top: Math.max(currentInsets.top,
        avoidArea.topRect.height + avoidArea.topRect.top),
      left: Math.max(currentInsets.left,
        avoidArea.leftRect.width + avoidArea.leftRect.left),
      right: Math.max(currentInsets.right,
        avoidArea.rightRect.left > 0 ?
          windowSize.width - avoidArea.rightRect.left : 0),
      bottom: Math.max(currentInsets.bottom,
        avoidArea.bottomRect.top > 0 ?
          windowSize.height - avoidArea.bottomRect.top : 0),
    }
  }, { top: 0, left: 0, right: 0, bottom: 0 })
}
```

**区域类型**：
- `TYPE_SYSTEM` - 系统栏
- `TYPE_CUTOUT` - 刘海屏
- `TYPE_NAVIGATION_INDICATOR` - 导航栏
- `waterfallDisplayAreaRects` - 瀑布屏区域

### SafeAreaView 动态 Padding

**文件位置**: `Libraries/Components/View/SafeAreaView.harmony.tsx`

```typescript
const getPaddingTop = (inset: number, pageY: number) => {
  return Math.max(0, inset - (pageY < 0 ? pageY * -1 : pageY));
}

const getPaddingBottom = (
  insetBottom: number, insetTop: number, paddingTop: number,
  height: number, windowHeight: number, pageY: number, positionY: number
) => {
  // 处理多种边界情况：
  // 1. SafeArea 不可见或超出视口
  // 2. SafeAreaView 不在顶部且不是全高
  // 3. SafeAreaView 全屏且顶部对齐
  // 4. 嵌套场景和滚动视图

  const isVisible = positionY + height > 0 && positionY < windowHeight;
  if (!isVisible) return 0;

  const isFullHeight = height >= windowHeight;
  const isAtTop = positionY <= 0;

  if (isFullHeight && isAtTop) {
    return insetBottom;
  }

  // 计算底部 padding...
}
```

### StatusBar API 12 兼容性

**颜色动画实现**：
```typescript
// API 12 新特性
let systemBarProperties = {
  statusBarContentColor: '#E5FFFFFF',
  enableStatusBarAnimation: animated,
};

if (!animated) {
  windowInstance.setWindowSystemBarProperties(systemBarProperties);
} else {
  // 手动创建颜色动画（API 12 动画可能不工作）
  let colors = interpolateColor(this._currentColor, colorString, 20);
  let intervalId = setInterval(() => {
    windowInstance.setWindowSystemBarProperties({
      statusBarColor: colors[i]
    });
  }, 16);
}
```

### 颜色插值算法

```typescript
function interpolateColor(color1: string, color2: string, steps: number): string[] {
  let colorStartRGB = hexToRGB(color1);
  let colorStopRGB = hexToRGB(color2);

  let colorArray: string[] = [];
  for (let i = 0; i < steps; i++) {
    const ratio = i / (steps - 1);
    const r = Math.round(colorStartRGB.r +
      (colorStopRGB.r - colorStartRGB.r) * ratio);
    const g = Math.round(colorStartRGB.g +
      (colorStopRGB.g - colorStartRGB.g) * ratio);
    const b = Math.round(colorStartRGB.b +
      (colorStopRGB.b - colorStartRGB.b) * ratio);

    colorArray.push(`#${r.toString(16).padStart(2, '0')}` +
                   `${g.toString(16).padStart(2, '0')}` +
                   `${b.toString(16).padStart(2, '0')}`);
  }
  return colorArray;
}
```

### SafeArea 常见问题

**问题 1**：SafeAreaView 在滚动视图中不正确
- **原因**：动态计算依赖 `pageY` 和 `positionY`
- **解决**：使用 `onScroll` 回调更新计算

**问题 2**：刘海屏适配问题
- **原因**：`waterfallDisplayAreaRects` 未正确计算
- **解决**：使用 `display.getCutoutInfo()` 获取准确信息

**问题 3**：StatusBar 动画不生效
- **原因**：API 12 的 `enableStatusBarAnimation` 可能不工作
- **解决**：使用手动颜色插值动画

---

## ScrollView 组件

### ScrollView 架构

```
React Native JS 层
    ↓
Props 解析 (ScrollViewPropsParser)
    ↓
Shadow Tree (ScrollViewShadowNode)
    ↓
Component Instance (ScrollViewComponentInstance)
    ↓
ArkUI Layer (ScrollNode)
    ↓
RNScrollView.ets (ArkTS 封装)
    ↓
最终渲染
```

### 核心类定义

**ScrollNode** - ArkUI 滚动节点：
```cpp
class ScrollNode : public ArkUINode {
  ScrollNodeDelegate* m_scrollNodeDelegate;

public:
  void scrollTo(float x, float y, bool animated);
  void setNestedScroll(ArkUI_ScrollNestedMode scrollNestedMode);
  void setScrollOverScrollMode(std::string const& overScrollMode);
};
```

**ScrollViewComponentInstance**：
```cpp
class ScrollViewComponentInstance
    : public CppComponentInstance<facebook::react::ScrollViewShadowNode>,
      public ScrollNodeDelegate {
private:
  enum ScrollState : int32_t { IDLE, SCROLL, FLING };

  ScrollNode m_scrollNode;
  StackNode m_contentContainerNode;
  facebook::react::Size m_contentSize;
  facebook::react::Size m_containerSize;
  ScrollState m_scrollState = IDLE;
};
```

### 嵌套滚动处理

**嵌套滚动配置**：
```typescript
// RNScrollView.ets
getNestedScroll(): NestedScrollOptions {
  const isNestedInRefreshControl = this.shouldWrapWithPullToRefresh();
  if (isNestedInRefreshControl) {
    return {
      scrollForward: NestedScrollMode.SELF_ONLY,
      scrollBackward: NestedScrollMode.SELF_ONLY
    };
  }
  return {
    scrollForward: NestedScrollMode.SELF_FIRST,
    scrollBackward: NestedScrollMode.SELF_FIRST
  };
}
```

**嵌套滚动模式**：
- `SELF_ONLY`: 只有当前 ScrollView 响应滚动事件
- `SELF_FIRST`: 当前 ScrollView 优先响应，传递给父视图
- 在下拉刷新场景下禁用嵌套滚动以避免冲突

### 事件处理机制

**事件类型映射**：
| RNOH 事件 | ArkUI 事件 | 描述 |
|-----------|------------|------|
| onScroll | onScroll | 滚动事件 |
| onScrollBeginDrag | onScrollStart | 拖动开始 |
| onScrollEndDrag | onScrollStop | 拖动结束 |
| onMomentumScrollBegin | - | 动量滚动开始（状态判断） |
| onMomentumScrollEnd | - | 动量滚动结束（状态判断） |

**事件节流**：
```typescript
onScrollEvent() {
  const now = Date.now();
  if (this.allowNextScrollEvent ||
      this.isScrolling &&
      descriptor.props.scrollEventThrottle < now - this.lastScrollDispatchTime) {
    this.lastScrollDispatchTime = now;
    this.ctx.rnInstance.emitComponentEvent(
      this.tag, "onScroll", this.createScrollEvent());
  }
}
```

### LazyForEach 性能优化

**虚拟化渲染**：
```typescript
buildScrollCore() {
  Scroll(this.scroller) {
    Stack() {
      LazyForEach(this.ctx.createComponentDataSource({ tag: this.tag }),
        (descriptorWrapper: DescriptorWrapper) => {
          (this.ctx as RNComponentContext).wrappedRNComponentBuilder.builder(
            (this.ctx as RNComponentContext),
            descriptorWrapper.tag
          );
        },
        (descriptorWrapper: DescriptorWrapper) =>
          descriptorWrapper.tag.toString() + "@" + descriptorWrapper.renderKey
      );
    }
  }
}
```

**优化策略**：
1. 虚拟化渲染：只渲染可视区域内的组件
2. 数据源管理：通过 `createComponentDataSource` 管理组件数据
3. Key 管理：使用 tag + renderKey 作为唯一标识

### scrollTo 命令实现

```typescript
scrollTo(xOffset: number, yOffset: number, animated: boolean = false) {
  const animation: Animation | undefined = animated ?
    { duration: 1000, curve: Curve.Smooth } : undefined;

  const currentOffset = this.scroller.currentOffset();
  if (!animation && currentOffset) {
    this.positionBeforeScrolling = {
      x: currentOffset.xOffset,
      y: currentOffset.yOffset,
    };
    this.isScrolling = false;
  }

  this.scroller.scrollTo({ xOffset, yOffset, animation });
  this.contentOffset = { x: xOffset, y: yOffset };
}
```

### 下拉刷新集成

```typescript
private shouldWrapWithPullToRefresh() {
  const pullToRefreshTag = this.parentTag;
  if (pullToRefreshTag === undefined) return false;

  const parentDescriptor = this.ctx.descriptorRegistry.getDescriptor(pullToRefreshTag);
  return parentDescriptor?.type === RN_PULL_TO_REFRESH_VIEW_NAME;
}

build() {
  if (this.shouldWrapWithPullToRefresh()) {
    RNViewBase({ ctx: this.ctx, tag: this.parentTag }) {
      RNPullToRefreshView({ ctx: this.ctx, tag: this.parentTag }) {
        this.buildScrollCore();
      }
    }
  }
}
```

### ScrollView 常见问题

**问题 1**：嵌套滚动冲突
- **原因**：父子 ScrollView 同时响应滚动事件
- **解决**：配置 `nestedScrollEnabled` 属性

**问题 2**：LazyForEach 组件不更新
- **原因**：renderKey 未变化
- **解决**：确保 key 函数返回唯一标识

---

## Modal 组件

### Modal 架构

```
React Native JS 层
    ↓
Shadow Tree (ModalHostViewShadowNode)
    ↓
Component Instance (ModalHostViewComponentInstance)
    ↓
RNModalHostView.ets (ArkTS)
    ↓
ModalHostViewDialog (@CustomDialog)
    ↓
CustomDialogController
```

### 核心类定义

**ModalHostViewDialog**：
```typescript
@CustomDialog
struct ModalHostViewDialog {
  controller: CustomDialogController
  @State showContent: boolean = false

  aboutToAppear() {
    this.controller = new CustomDialogController({
      alignment: DialogAlignment.TopStart,
      customStyle: true,
      maskColor: Color.Transparent,
      closeAnimation: { duration: 0 },
      openAnimation: { duration: 0 },
      autoCancel: false,
    })
    this.controller.open()
  }
}
```

### 动画类型支持

**动画效果**：
```typescript
getTransitionEffect() {
  if (this.descriptor.rawProps.animationType === 'slide') {
    return TransitionEffect.move(TransitionEdge.BOTTOM)
      .animation({ duration: 500 });  // MODAL_ANIMATION_DURATION
  } else if (this.descriptor.rawProps.animationType === 'fade') {
    return TransitionEffect.OPACITY.animation({ duration: 500 });
  } else {
    return TransitionEffect.IDENTITY;
  }
}
```

**动画类型**：
- `'none'` - 无动画
- `'slide'` - 从底部滑入（500ms）
- `'fade'` - 淡入淡出（500ms）

### ModalProps 属性

```cpp
class JSI_EXPORT ModalHostViewProps final : public ViewProps {
public:
  ModalHostViewAnimationType animationType{ModalHostViewAnimationType::None};
  ModalHostViewPresentationStyle presentationStyle{
      ModalHostViewPresentationStyle::FullScreen};
  bool transparent{false};
  bool statusBarTranslucent{false};
  bool visible{false};
  bool animated{false};
  int identifier{0};
};
```

**动画类型**：
```cpp
enum class ModalHostViewAnimationType { None, Slide, Fade };
```

**显示风格**：
```cpp
enum class ModalHostViewPresentationStyle {
  FullScreen,     // 全屏
  PageSheet,      // 页面表单
  FormSheet,      // 表单表单
  OverFullScreen  // 覆盖全屏
};
```

### 显示/隐藏流程

**显示流程**：
1. 组件挂载 → `RNModalHostView.aboutToAppear()`
2. 创建 DialogController
3. 打开对话框 → `controller.open()`
4. 触发 onShow 事件

**隐藏流程**：
1. 关闭信号
2. 执行动画（如需要）
3. 关闭对话框 → `controller.close()`
4. 触发 onDismiss 事件

### 触摸事件处理

```typescript
build() {
  if (this.showContent) {
    Stack() {
      // 子组件渲染
    }.onTouch((touchEvent) => {
      if (this.showContent) {
        this.touchDispatcher.handleTouchEvent(touchEvent)
      }
    })
  }
}
```

### 折叠屏适配

```cpp
void ModalHostViewComponentInstance::resetModalPosition(
    DisplayMetrics const& displayMetrics,
    SharedConcreteState const& state) {
  FoldStatus foldStatus = static_cast<FoldStatus>(
      ArkTSBridge::getInstance()->getFoldStatus());
  auto isSplitScreenMode = ArkTSBridge::getInstance()->getIsSplitScreenMode();

  if ((foldStatus == FOLD_STATUS_EXPANDED || foldStatus == FOLD_STATUS_HALF_FOLDED)
      && isSplitScreenMode) {
    m_rootStackNode.setPosition({
        displayMetrics.screenPhysicalPixels.width /
            displayMetrics.windowPhysicalPixels.scale, 0});
  } else {
    m_rootStackNode.setPosition({0, 0});
  }
}
```

---

## TextInput 组件

### TextInput 架构

```
React Native JS 层
    ↓
Props 解析 (TextInputPropsParser)
    ↓
Shadow Tree (TextInputShadowNode)
    ↓
Component Instance (TextInputComponentInstance)
    ↓
ArkUI Layer (TextInputNode/TextAreaNode)
    ↓
RNTextInput.ets (ArkTS 封装)
    ↓
最终渲染
```

### 核心类定义

**TextInputNode**：
```cpp
class TextInputNode : public TextInputNodeBase {
private:
  uint32_t m_caretColorValue;
  bool m_autofocus{false};
  bool m_setTextContent{false};
  std::string m_textContent;
  TextInputNodeDelegate* m_textInputNodeDelegate;

public:
  void setTextContent(std::string const& textContent);
  void setCaretHidden(bool hidden);
  void setInputType(ArkUI_TextInputType keyboardType);
  void setPlaceholder(std::string const& placeholder);
};
```

**TextAreaNode**（多行输入）：
```cpp
class TextAreaNode : public TextInputNodeBase {
public:
  void setInputType(ArkUI_TextAreaType keyboardType);
  void setshowSoftInputOnFocus(int32_t enable);
};
```

### 属性处理

**文本属性**：
- `placeholder` - 占位符文本
- `placeholderTextColor` - 占位符颜色
- `maxLength` - 最大长度限制
- `selectionColor` - 选中区域颜色
- `cursorColor` - 光标颜色

**输入行为属性**：
- `keyboardType` - 键盘类型
- `secureTextEntry` - 安全文本输入（密码模式）
- `editable` - 可编辑性控制
- `caretHidden` - 光标隐藏

**键盘类型映射**：
| React Native | ArkUI TextInput | ArkUI TextArea |
|--------------|-----------------|----------------|
| default | NORMAL | NORMAL |
| email-address | EMAIL_ADDRESS | EMAIL |
| numeric | NUMBER | NUMBER |
| phone-pad | PHONE_NUMBER | PHONE_NUMBER |

### 事件处理

**事件类型**：
```cpp
enum class TextInputEventType {
  Change,            // 文本变化
  Focus,             // 获得焦点
  Blur,              // 失去焦点
  SelectionChange,   // 选择变化
  Submit,            // 提交
  ContentSizeChange  // 内容大小变化
};
```

**onChange 事件**：
```cpp
void TextInputComponentInstance::onChange(std::string text) {
  m_nativeEventCount++;
  m_content = std::move(text);
  m_eventEmitter->onChange(getOnChangeMetrics());
  m_valueChanged = true;
}
```

**onFocus/onBlur 事件**：
```cpp
void TextInputComponentInstance::onFocus() {
  this->m_focused = true;
  if (this->m_clearTextOnFocus) {
    m_textAreaNode.setTextContent("");
  }
}

void TextInputComponentInstance::onBlur() {
  this->m_focused = false;
  m_eventEmitter->onBlur(getTextInputMetrics());
  m_eventEmitter->onEndEditing(getTextInputMetrics());
}
```

### 键盘显示控制

```cpp
void TextInputNode::setshowSoftInputOnFocus(int32_t enable) {
  ArkUI_NumberValue value = {.i32 = enable};
  ArkUI_AttributeItem item = {&value, 1};
  NativeNodeApi::getInstance()->setAttribute(
      m_nodeHandle, NODE_TEXT_INPUT_SHOW_KEYBOARD_ON_FOCUS, &item);
}
```

**键盘交互流程**：
1. 聚焦触发 → 用户点击或 `focus()` 调用
2. 键盘显示 → `showSoftInputOnFocus` 控制
3. 输入处理 → `onChange` 回调
4. 失去焦点 → `blur()` 调用
5. 键盘隐藏 → 自动收起

### 光标管理

**光标显示/隐藏**：
```cpp
void TextInputNode::setCaretHidden(bool hidden) {
  if (hidden) {
    caretStyle = { width: 0 };
  } else {
    caretStyle = { width: 2 };  // 默认宽度
  }
}
```

**文本选择**：
```cpp
struct Selection {
  int start{0};
  int end{0};
};

void setTextSelection(int32_t start, int32_t end);
```

### 多行输入支持

**模式切换**：
```typescript
if (this->descriptor.props.multiline) {
  TextArea({ controller: this.areaController, ... })
} else {
  TextInput({ controller: this.inputController, ... })
}
```

**TextArea 键盘类型**：
```typescript
TextAreaType getTextAreaType(keyboardType?: string) {
  switch (keyboardType) {
    case "numberPad":
    case "numeric":
      return TextAreaType.NUMBER;
    case "emailAddress":
      return TextAreaType.EMAIL;
    default:
      return TextAreaType.NORMAL;
  }
}
```

### 自动填充支持

```cpp
void TextInputNode::setAutoFill(bool autoFill) {
  uint32_t isAutoFill = static_cast<uint32_t>(autoFill);
  ArkUI_NumberValue value = {.u32 = isAutoFill};
  NativeNodeApi::getInstance()->setAttribute(
      m_nodeHandle, NODE_TEXT_INPUT_ENABLE_AUTO_FILL, &item);
}
```

**支持的内容类型**：
- 姓名、邮箱、电话
- 密码、新密码
- 街道、城市、邮政编码

### TextInput 常见问题

**问题 1**：onChange 事件重复触发
- **原因**：程序化修改也触发事件
- **解决**：使用 `m_setTextContent` 标志过滤

**问题 2**：多行输入滚动问题
- **原因**：TextArea 未正确配置滚动属性
- **解决**：检查 `onContentScroll` 事件处理

**问题 3**：键盘遮挡输入框
- **原因**：未配置键盘避让
- **解决**：使用 `KeyboardAvoidingView` 或 ScrollView 的 `keyboardShouldPersistTaps`

---

## FlatList 组件

### FlatList 架构

```
React Native JS 层
    ↓
VirtualizedList (基类)
    ↓
FlatList (便捷封装)
    ↓
ScrollView 容器
    ↓
LazyForEach 虚拟化渲染
    ↓
CellRenderer (单项渲染)
```

### 核心类定义

**VirtualizedList** - 虚拟化列表基类：
```javascript
class VirtualizedList extends StateSafePureComponent {
  _scrollMetrics: {
    contentLength: number,
    offset: number,
    visibleLength: number,
    velocity: number,
  };
  _indicesToKeys: Map<number, string>;
  _cellRefs: Object;

  _adjustCellsAroundViewport(props, cellsAroundViewport);
  _updateCellsToRenderBatcher: Batchinator;
}
```

**FlatList** - 平铺列表：
```javascript
class FlatList extends VirtualizedList {
  static defaultProps = {
    data: [],
    renderItem: null,
    keyExtractor: (item, index) => item.key || item.id || String(index),
  };
}
```

### 虚拟化渲染机制

**渲染窗口计算**：
```javascript
// VirtualizeUtils.js
export function computeWindowedRenderLimits(
  props, maxToRenderPerBatch, windowSize, prev, getFrameMetricsApprox, scrollMetrics
) {
  const visibleBegin = Math.max(0, offset);
  const visibleEnd = visibleBegin + visibleLength;
  const overscanLength = (windowSize - 1) * visibleLength;

  // 计算预扫描区域
  const overscanBegin = Math.max(0, visibleBegin - (1 - leadFactor) * overscanLength);
  const overscanEnd = Math.max(0, visibleEnd + leadFactor * overscanLength);
}
```

**渲染窗口调整**：
```javascript
_adjustCellsAroundViewport(props, cellsAroundViewport) {
  const {contentLength, offset, visibleLength, velocity} = this._scrollMetrics;

  // 根据滚动方向调整预加载
  if (velocity > 1) {
    fillPreference = 'after';   // 向下滚动
  } else if (velocity < -1) {
    fillPreference = 'before';  // 向上滚动
  }
}
```

### 渲染批处理优化

```javascript
// 批处理避免频繁布局更新
this._updateCellsToRenderBatcher = new Batchinator(
  this._updateCellsToRender,
  this.props.updateCellsBatchingPeriod ?? 50,
);
```

**批处理限制**：
```javascript
const maxToRenderPerBatchOrDefault = (maxToRenderPerBatch) => {
  return maxToRenderPerBatch ?? 10;  // 默认每批渲染 10 个项目
};
```

### onEndReached 触发机制

**触发条件**：
```javascript
if (
  onEndReached &&
  this.state.cellsAroundViewport.last === getItemCount(data) - 1 &&
  isWithinEndThreshold &&
  this._scrollMetrics.contentLength !== this._sentEndForContentLength
) {
  this._sentEndForContentLength = this._scrollMetrics.contentLength;
  onEndReached({distanceFromEnd});
}
```

**阈值计算**：
```javascript
function getScrollingThreshold(threshold, visibleLength) {
  return (threshold * visibleLength) / 2;
}

// 默认阈值为 2（两个可见高度的滚动距离）
```

### getItemLayout 优化

**提供预定义布局信息**：
```javascript
getItemLayout: (data, index) => ({
  length: ITEM_HEIGHT,           // 项目固定高度
  offset: ITEM_HEIGHT * index,   // 累计偏移
  index: index,
})
```

**优化作用**：
1. 避免运行时测量每个项目高度
2. 支持精确的 `scrollToIndex` 定位
3. 提升滚动性能

### 键提取策略

```javascript
const defaultKeyExtractor = (item, index) => {
  return item === null || item === undefined
    ? `null-${index}`
    : typeof item.key === 'string'
      ? item.key
      : item.id;
};
```

### FlatList 最佳实践

```javascript
// 最佳实践示例
const FlatListExample = () => {
  const data = useMemo(() =>
    Array.from({length: 1000}, (_, i) => ({id: i, text: `Item ${i}`})), []);

  const renderItem = useCallback(({item}) => (
    <View style={styles.item}>
      <Text>{item.text}</Text>
    </View>
  ), []);

  return (
    <FlatList
      data={data}
      renderItem={renderItem}
      keyExtractor={item => item.id.toString()}
      getItemLayout={(data, index) => ({
        length: 60,
        offset: 60 * index,
        index,
      })}
      initialNumToRender={10}
      maxToRenderPerBatch={5}
      windowSize={10}
      onEndReached={handleLoadMore}
      onEndReachedThreshold={0.5}
    />
  );
};
```

---

## Image 组件

### Image 架构

```
React Native JS 层
    ↓
Props 解析 (ImagePropsParser)
    ↓
Shadow Tree (ImageShadowNode)
    ↓
Component Instance (ImageComponentInstance)
    ↓
RNImage.ets (ArkTS 封装)
    ↓
ImageLoaderTurboModule (网络加载)
    ↓
RemoteImageCache (三级缓存)
    ↓
最终渲染
```

### 核心类定义

**ImageNode** - ArkUI 图片节点：
```cpp
class ImageNode : public ArkUINode {
private:
  ArkUI_NodeHandle m_childArkUINodeHandle;
  ImageNodeDelegate* m_imageNodeDelegate;
  std::string m_uri;

public:
  void setSources(ImageSource const& sources);
  void setResizeMode(ImageResizeMode const& mode);
  void setTintColor(SharedColor const& color);
  void setBlur(double blur);
};
```

**RNImage ArkTS 组件**：
```typescript
export struct RNImage {
  ctx!: RNOHContext
  tag: number = 0
  @State private imageSource: ImageSourceHolder | undefined

  async updateImageSource() {
    const uri = this.descriptor.props.uri;

    // 本地资源
    if (uri.startsWith("asset://")) {
      this.imageSource = new ImageSourceHolder($rawfile(uri.replace("asset://", "")));
      return;
    }

    // Base64
    if (uri.startsWith("data:")) {
      this.imageSource = new ImageSourceHolder(uri);
      return;
    }

    // 网络图片
    const imageLoader = this.ctx.rnInstance.getTurboModule<ImageLoaderTurboModule>("ImageLoader");
    const remoteImage = await imageLoader.getRemoteImageSource(uri);
    this.imageSource = new ImageSourceHolder(remoteImage.getLocation());
  }
}
```

### 图片源类型支持

| 类型 | 前缀 | 处理方式 |
|------|------|----------|
| 本地资源 | `asset://` | `$rawfile()` 加载 |
| Base64 | `data:` | 直接创建 ImageSource |
| 网络图片 | `http://`/`https://` | RemoteImageLoader 处理 |

### 三级缓存架构

**缓存查找顺序**：
```
内存缓存 (128个对象)
    ↓ 未命中
磁盘缓存 (128个文件)
    ↓ 未命中
网络下载
```

**内存缓存**：
```typescript
export class RemoteImageMemoryCache extends RemoteImageCache<RemoteImageSource> {
  // 最大缓存数量：128
  // 采用 LRU 策略
}
```

**磁盘缓存**：
```typescript
export class RemoteImageDiskCache extends RemoteImageCache<boolean> {
  // 缓存目录：ctx.uiAbilityContext.cacheDir
  // 最大缓存数量：128
}
```

### resizeMode 映射

```typescript
getResizeMode(resizeMode: number) {
  switch (resizeMode) {
    case 0: return ImageFit.Cover;      // cover - 等比填充，可能裁剪
    case 1: return ImageFit.Contain;    // contain - 等比适应，可能留白
    case 2: return ImageFit.Fill;       // stretch - 拉伸填充
    case 3:
    case 4: return ImageFit.None;       // center, repeat
    default: return ImageFit.Cover;
  }
}
```

### 图片加载事件

```typescript
// RNImage.ets
build() {
  Image(this.imageSource.source)
    .onComplete(event => {
      this.onLoad(event)    // 加载完成
      this.onLoadEnd()      // 加载结束
    })
    .onError((event) => {
      this.dispatchOnError(event.message)
      this.onLoadEnd()
    })
}

onLoadStart() {
  this.ctx.rnInstance.emitComponentEvent(this.descriptor.tag, "loadStart", null);
}
```

### GIF 动画支持

**动画检测**：
```typescript
async updateImageSource() {
  const imageSource = remoteImage.getImageSource();
  const frameCounter = await imageSource.getFrameCount();

  if (frameCounter === 1) {
    // 静态图片 → PixelMap（性能优化）
    this.imageSource = new ImageSourceHolder(await imageLoader.getPixelMap(uri));
  } else {
    // GIF 动图 → 保持原格式
    this.imageSource = new ImageSourceHolder(remoteImage.getLocation());
  }
}
```

**支持格式**：
- 静态图片：JPEG, PNG, GIF (单帧), BMP, WEBP
- 动画格式：GIF (多帧)

### 图片加载优化

**请求去重**：
```typescript
private activeRequestByUrl: Map<string, Promise<FetchResult>> = new Map();

// 防止同一个 URL 的重复并发请求
```

**流式传输**：
```typescript
httpRequest.on('dataReceiveProgress', ({receiveSize, totalSize}) => {
  onProgress?.(receiveSize / totalSize);
});
```

---

## Switch 组件

### Switch 架构

```
React Native JS 层
    ↓
Props 解析 (SwitchPropsParser)
    ↓
Shadow Tree (SwitchShadowNode)
    ↓
Component Instance (SwitchComponentInstance)
    ↓
RNSwitch.ets (ArkTS 封装)
    ↓
Toggle (ArkUI 组件)
```

### 核心类定义

**RNSwitch ArkTS 组件**：
```typescript
@Component
export struct RNSwitch {
  ctx!: RNOHContext
  tag: number = 0
  @State private descriptor: SwitchDescriptor = Object() as SwitchDescriptor

  build() {
    RNViewBase({ ctx: this.ctx, tag: this.tag }) {
      Toggle({ type: ToggleType.Switch, isOn: this.descriptor.props.value })
        .selectedColor(Color.Pink)      // 开启状态颜色
        .switchPointColor(Color.Green)   // 滑块颜色
        .onChange((isOn) => {
          this.onChange(isOn)
        })
    }
  }

  onChange(isOn: boolean) {
    this.ctx.rnInstance.emitComponentEvent(
      this.descriptor.tag,
      "onChange",
      { value: isOn, target: this.descriptor.tag }
    );
  }
}
```

**SwitchProps 属性**：
```typescript
export interface SwitchProps extends ViewBaseProps {
  trackColor?: ColorSegments;   // 轨道颜色
  thumbColor?: ColorSegments;    // 滑块颜色
  value?: boolean;               // 开关值
  disabled?: boolean;            // 是否禁用
}
```

### C++ 层属性处理

```cpp
// SwitchComponentInstance.cpp
void SwitchComponentInstance::onPropsChanged(SharedConcreteProps const& props) {
  if (!m_props || props->onTintColor != m_props->onTintColor) {
    getLocalRootArkUINode().setSelectedColor(props->onTintColor);
  }
  if (!m_props || props->tintColor != m_props->tintColor) {
    getLocalRootArkUINode().setUnselectedColor(props->tintColor);
  }
  if (!m_props || props->thumbTintColor != m_props->thumbTintColor) {
    getLocalRootArkUINode().setThumbColor(props->thumbTintColor);
  }
  getLocalRootArkUINode().setEnabled(!props->disabled);
  if (props->value != m_toggleNode.getValue()) {
    getLocalRootArkUINode().setValue(props->value);
  }
}
```

### 事件处理机制

**防止事件循环**：
```cpp
void SwitchComponentInstance::onValueChange(int32_t& value) {
  // 防止通过 props 更新时触发事件回传
  if (m_props == nullptr || m_props->value == value) {
    return;
  }

  if (m_eventEmitter != nullptr) {
    auto onValueChange = facebook::react::SwitchEventEmitter::OnChange();
    onValueChange.value = value;
    onValueChange.target = getTag();
    m_eventEmitter->onChange(onValueChange);
  }

  m_toggleNode.setValue(m_props->value);
}
```

### ToggleNode 方法

```cpp
class ToggleNode : public ArkUINode {
public:
  ToggleNode& setSelectedColor(SharedColor const& color);
  ToggleNode& setUnselectedColor(SharedColor const& color);
  ToggleNode& setThumbColor(SharedColor const& color);
  ToggleNode& setValue(bool value);
  bool getValue();
};
```

### 受控组件实现

```typescript
// React 层使用示例
const ValuePropExample = () => {
  const [value, setValue] = useState(false);

  return (
    <View>
      <Switch value={value} onValueChange={setValue} />
      <Button onPress={() => setValue(!value)} label="Toggle" />
    </View>
  );
};
```

---

## 其他表单组件

### 组件角色映射

```cpp
const std::unordered_map<std::string, ArkUI_NodeType> NODE_TYPE_BY_ROLE_NAME = {
    {"button", ARKUI_NODE_BUTTON},
    {"checkbox", ARKUI_NODE_CHECKBOX},
    {"switch", ARKUI_NODE_TOGGLE},
    {"slider", ARKUI_NODE_SLIDER},
    {"progressbar", ARKUI_NODE_PROGRESS},
};
```

### ProgressBarAndroid

```javascript
export type ProgressBarAndroidProps = {
  ...ViewProps,
  styleAttr: 'Horizontal' | 'Normal' | 'Small' | 'Large',
  indeterminate: boolean,
  animating?: boolean,
  color?: ColorValue,
};
```

**特点**：
- 支持水平、垂直和不同尺寸样式
- 支持确定和不确定进度模式

---

## 核心架构深度解析

### RNInstance 实例管理

#### 核心接口定义

```typescript
export interface RNInstance {
  // 核心组件系统
  descriptorRegistry: DescriptorRegistry;
  componentManagerRegistry: ComponentManagerRegistry;

  // 事件系统
  cppEventEmitter: EventEmitter<Record<string, unknown[]>>;
  lifecycleEventEmitter: EventEmitter<LifecycleEventArgsByEventName>;

  // 生命周期管理
  getLifecycleState(): LifecycleState;
  subscribeToLifecycleEvents<TEventName>(...): () => void;

  // JS 通信
  callRNFunction(moduleName: string, functionName: string, args: unknown[]): void;
  emitComponentEvent(tag: Tag, eventName: string, payload: any): void;
  emitDeviceEvent(eventName: string, payload: any): void;

  // TurboModule 管理
  getTurboModule<T>(name: string): T;
  getUITurboModule<T extends UITurboModule>(name: string): T;

  // Surface 管理
  createSurface(appKey: string): SurfaceHandle;

  // 资源管理
  registerFont(fontFamily: string, fontResource: Resource | string): void;
  getAssetsDest(): string;
}
```

#### 生命周期状态

```typescript
export enum LifecycleState {
  BEFORE_CREATE,  // 初始状态，尚未创建
  PAUSED,        // 暂停状态
  READY,         // 就绪状态，可以处理 UI 更新
}
```

#### 生命周期流程

**初始化阶段**：
```typescript
public async initialize(packages: RNPackage[]) {
  // 1. 处理包，创建 TurboModule 提供者和描述符包装器工厂
  const { descriptorWrapperFactoryByDescriptorType, turboModuleProvider } =
    await this.processPackages(packages);

  // 2. 创建描述符注册表
  this.descriptorRegistry = new DescriptorRegistry({...});

  // 3. 初始化 NapiBridge，连接到 C++ 层
  this.napiBridge.onCreateRNInstance(this.envId, this.id, ...);
}
```

**运行阶段**：
```typescript
public async runJSBundle(jsBundleProvider: JSBundleProvider) {
  // 1. 设置执行状态
  this.bundleExecutionStatusByBundleURL.set(bundleURL, "RUNNING");

  // 2. 加载并执行 JS Bundle
  const jsBundle = await jsBundleProvider.getBundle();
  await this.napiBridge.loadScript(this.id, jsBundle, bundleURL);

  // 3. 更新生命周期状态
  this.lifecycleState = LifecycleState.READY;
}
```

**销毁阶段**：
```typescript
public async onDestroy() {
  // 1. 停止所有 Surface
  for (const surfaceHandle of this.surfaceHandles) {
    if (surfaceHandle.isRunning()) {
      await surfaceHandle.stop();
    }
    surfaceHandle.destroy();
  }

  // 2. 清理 TurboModule
  this.turboModuleProvider.onDestroy();

  // 3. 通知 C++ 层清理
  if (this.isFeatureFlagEnabled("ENABLE_RN_INSTANCE_CLEAN_UP")) {
    this.napiBridge.onDestroyRNInstance(this.id);
  }
}
```

#### TurboModule 管理

```typescript
export class TurboModuleProvider<TTurboModule extends UITurboModule> {
  private cachedTurboModuleByName: Record<string, TTurboModule> = {};

  getModule<T extends TTurboModule>(name: string): T {
    if (!(name in this.cachedTurboModuleByName)) {
      for (const tmFactory of this.turboModulesFactories) {
        if (tmFactory.hasTurboModule(name)) {
          this.cachedTurboModuleByName[name] = tmFactory.createTurboModule(name);
        }
      }
    }
    return this.cachedTurboModuleByName[name] as T;
  }
}
```

#### 事件发射机制

```typescript
// 组件事件
public emitComponentEvent(tag: Tag, eventName: string, payload: any) {
  this.napiBridge.emitComponentEvent(this.id, tag, eventName, payload);
}

// 设备事件
public emitDeviceEvent(eventName: string, payload: any) {
  this.napiBridge.emitDeviceEvent(this.id, eventName, payload);
}

// 生命周期事件
rnInstance.subscribeToLifecycleEvents("FOREGROUND", () => {
  console.log("App came to foreground");
});
```

---

### CppComponentsRegistry 组件注册

#### 组件绑定器架构

```cpp
class BaseComponentJSIBinder : public ComponentJSIBinder {
public:
  facebook::jsi::Object createBindings(facebook::jsi::Runtime& rt) override {
    facebook::jsi::Object baseManagerConfig(rt);
    baseManagerConfig.setProperty(rt, "NativeProps", this->createNativeProps(rt));
    baseManagerConfig.setProperty(rt, "Constants", this->createConstants(rt));
    baseManagerConfig.setProperty(rt, "Commands", this->createCommands(rt));
    baseManagerConfig.setProperty(rt, "bubblingEventTypes", this->createBubblingEventTypes(rt));
    baseManagerConfig.setProperty(rt, "directEventTypes", this->createDirectEventTypes(rt));
    return baseManagerConfig;
  }
};
```

#### Props 解析机制

```typescript
export class ViewDescriptorWrapperBase extends DescriptorWrapper {
  public get backgroundColor(): string | undefined {
    return convertColorValueToHex(this.rawProps.backgroundColor);
  }

  public get borderWidth(): Edges<number | undefined> {
    return this.resolveEdges({
      all: this.rawProps.borderWidth,
      top: this.rawProps.borderTopWidth,
      // ... 更多边框属性
    });
  }
}
```

#### DescriptorRegistry 核心

```typescript
export class DescriptorRegistry {
  private descriptorByTag: Map<Tag, Descriptor> = new Map();
  private descriptorWrapperByTag: Map<Tag, DescriptorWrapper> = new Map();
  private descriptorTagById: Map<NativeId, Tag> = new Map();

  // 应用变更（从 C++ 层同步）
  public applyMutations(mutations: Mutation[]) {
    const updatedDescriptorTags = new Set(mutations.flatMap(mutation => {
      return this.applyMutation(mutation);
    }));

    if (!this.rnInstance.shouldUIBeUpdated()) {
      updatedDescriptorTags.forEach(tag => this.updatedUnnotifiedTags.add(tag));
      return;
    }

    this.callListeners(updatedDescriptorTags);
  }

  // 监听描述符变化
  public subscribeToDescriptorChanges(tag: Tag, listener: DescriptorChangeListener) {
    if (!this.descriptorListenersSetByTag.has(tag)) {
      this.descriptorListenersSetByTag.set(tag, new Set());
    }
    this.descriptorListenersSetByTag.get(tag)!.add(listener);
    return () => this.removeDescriptorChangesListener(tag, listener);
  }
}
```

#### ComponentManager 生命周期

```typescript
export abstract class ComponentManager {
  onDestroy() {}
  abstract getParentTag(): Tag
  abstract getTag(): Tag
}

export class RNViewManager extends ComponentManager {
  protected tag: Tag;
  protected descriptorRegistry: DescriptorRegistry;
  protected componentManagerRegistry: ComponentManagerRegistry;
  protected parentTag: Tag;

  constructor(tag: Tag, ctx: RNOHContext) {
    super();
    this.descriptorRegistry = ctx.descriptorRegistry;
    this.componentManagerRegistry = ctx.componentManagerRegistry;
    this.parentTag = this.descriptorRegistry.getDescriptor(tag)?.parentTag!;
  }

  public isPointInView({x, y}: Point): boolean {
    const descriptorWrapper = this.getDescriptorWrapper()!;
    const width = descriptorWrapper.width;
    const height = descriptorWrapper.height;
    const hitSlop = this.getHitSlop();
    // ... 点击检测逻辑
  }
}
```

#### 组件命令处理

```typescript
export class RNComponentCommandHub {
  private commandHandlersByTag = new Map<Tag, Map<string, CommandHandler>>();

  public dispatchCommand(tag: Tag, commandName: string, args: unknown) {
    const handler = this.findCommandHandler(tag, commandName);
    if (handler) {
      handler(args);
    }
  }
}
```

---

### MountingManager 挂载管理器

#### 核心职责

MountingManager 负责：
1. 组件实例生命周期管理（创建、更新、销毁）
2. UI 变异处理（ShadowViewMutation）
3. 多架构协调（C-API 和 ArkTS）
4. 挂载流程控制
5. 事件和命令分发

#### 双架构设计

```cpp
// 基类接口
class MountingManager {
protected:
  using Mutation = facebook::react::ShadowViewMutation;
  using MutationList = facebook::react::ShadowViewMutationList;

public:
  virtual void willMount(MutationList const& mutations) = 0;
  virtual void doMount(MutationList const& mutations) = 0;
  virtual void didMount(MutationList const& mutations) = 0;
  virtual void finalizeMutationUpdates(MutationList const& mutations) = 0;
};

// ArkTS 架构实现
class MountingManagerArkTS final : public MountingManager {
  // 负责 ArkTS 视图树的挂载管理
};

// C-API 架构实现
class MountingManagerCAPI final : public MountingManager {
  // 负责 C-API 组件的挂载管理
};
```

#### 挂载流程三阶段

1. **willMount** - 准备工作，检查变异有效性
2. **doMount** - 执行实际挂载操作
3. **didMount** - 完成挂载，触发回调

#### 变异指令处理

```cpp
void MountingManagerArkTS::doMount(MutationList const& mutations) {
  for (auto& mutation : mutations) {
    switch (mutation.type) {
      case react::ShadowViewMutation::Create: {
        auto newChild = mutation.newChildShadowView;
        shadowViewRegistry->setShadowView(newChild.tag, newChild);
        break;
      }
      case react::ShadowViewMutation::Delete: {
        auto oldChild = mutation.oldChildShadowView;
        shadowViewRegistry->clearShadowView(oldChild.tag);
        break;
      }
      // ... 其他变异类型
    }
  }
}
```

#### ShadowTree 提交机制

```cpp
enum class CommitStatus { Succeeded, Failed, Cancelled };
enum class CommitMode { Normal, Suspended };

struct CommitOptions {
  bool enableStateReconciliation{false};
  bool mountSynchronously{true};
  std::function<bool()> shouldYield;
};

CommitStatus commit(
  const ShadowTreeCommitTransaction &transaction,
  const CommitOptions &commitOptions) const;
```

#### Surface 管理

```cpp
class SurfaceHandler {
public:
  enum class Status { Unregistered, Registered, Running };

  void start() const noexcept;
  void stop() const noexcept;
  void constraintLayout(
    LayoutConstraints const &layoutConstraints,
    LayoutContext const &layoutContext) const noexcept;
};
```

#### 布局约束应用

```cpp
struct LayoutConstraints {
  Size minimumSize{0, 0};
  Size maximumSize{
    std::numeric_limits<Float>::infinity(),
    std::numeric_limits<Float>::infinity()
  };
  LayoutDirection layoutDirection{LayoutDirection::Undefined};

  Size clamp(const Size &size) const;
};
```

#### 双架构协调机制

```cpp
class MountingManagerCAPI final : public MountingManager {
private:
  MountingManager::Shared m_arkTsMountingManager;
  std::unordered_set<std::string> m_arkTsComponentNames;
  std::unordered_set<std::string> m_cApiComponentNames;

public:
  void doMount(MutationList const& mutations) override {
    // 先处理 ArkTS 组件
    m_arkTsMountingManager->doMount(getValidMutations(mutations));

    // 再处理 C-API 组件
    for (auto const& mutation : mutations) {
      handleMutation(mutation);
    }
  }

  bool isCAPIComponent(facebook::react::ShadowView const& shadowView) {
    std::string componentName = shadowView.componentName;
    if (m_cApiComponentNames.count(componentName) > 0) {
      return true;
    }
    if (m_arkTsComponentNames.count(componentName) > 0) {
      return false;
    }
    // 动态判断并缓存
    // ...
  }
};
```

#### Scheduler 集成

```cpp
class Scheduler final : public UIManagerDelegate {
public:
  void registerSurface(SurfaceHandler const &surfaceHandler) const noexcept;
  void animationTick() const;
  void uiManagerDidFinishTransaction(
    MountingCoordinator::Shared mountingCoordinator,
    bool mountSynchronously) override;
};
```

---

## 调试工具和开发技巧

### 测试框架

**Jest 配置**：
```javascript
// jest.config.js
module.exports = {
  preset: 'react-native',
  moduleFileExtensions: ['ts', 'tsx', 'js'],
  transform: {
    '^.+\\.(js|jsx|ts|tsx)$': 'ts-jest',
  },
};
```

**测试命令**：
```bash
# 运行所有测试
npm run test

# 性能测试
npm run benchmark-fps
```

### 调试工具

**Systrace 性能追踪**：
```javascript
import { Systrace } from 'react-native';

// 同步追踪
Systrace.beginEvent('Test trace');
// 执行代码...
Systrace.endEvent();

// 异步追踪
const traceCookie = Systrace.beginAsyncEvent('Test async trace');
// 执行代码...
Systrace.endAsyncEvent('Test async trace', traceCookie);

// 计数器事件
Systrace.counterEvent('MY_SUPER_VARIABLE', 123);
```

**性能日志系统**：
```javascript
import createPerformanceLogger from 'react-native/Libraries/Utilities/createPerformanceLogger';

const perfLogger = createPerformanceLogger();

// 记录时间跨度
perfLogger.startTimespan('operationName');
// 执行操作...
perfLogger.stopTimespan('operationName');

// 记录检查点
perfLogger.markPoint('checkpoint1');
```

**HiTrace 追踪**：
```typescript
// ArkTS 端追踪
import hiTrace from '@ohos.hiTraceMeter';
hiTrace.startTrace(`myTrace`, 0);
// 执行代码...
hiTrace.finishTrace(`myTrace`, 0);
```

### 热重载配置

**Metro 配置**：
```javascript
const { mergeConfig, getDefaultConfig } = require('@react-native/metro-config');
const { createHarmonyMetroConfig } = require('react-native-harmony/metro.config');

const config = {
  transformer: {
    getTransformOptions: async () => ({
      transform: {
        experimentalImportSupport: false,
        inlineRequires: true,
      },
    }),
  },
};

module.exports = mergeConfig(
  getDefaultConfig(__dirname),
  createHarmonyMetroConfig({
    reactNativeHarmonyPackageName: 'react-native-harmony',
  }),
  config
);
```

**LAN 热重载**：
```typescript
RNApp({
  jsBundleProvider: new TraceJSBundleProviderDecorator(
    new AnyJSBundleProvider([
      MetroJSBundleProvider.fromServerIp('192.168.43.14', 8081),
    ]),
    this.rnohCoreContext.logger),
})
```

### 开发工作流

**常用命令**：
```bash
# 启动 Metro 服务器
npm run start

# 端口转发（连接设备）
hdc rport tcp:8081 tcp:8081

# 代码格式化
npm run format

# 代码检查
npm run lint:js

# 类型检查
npm run typecheck
```

### 错误处理

**LogBox 集成**：
```typescript
// LogBox 事件处理
this.rnInstance.getTurboModule<LogBoxTurboModule>(LogBoxTurboModule.NAME)
  .eventEmitter.subscribe("SHOW", () => {
    this.logBoxDialogController.open();
  });
```

**DevEco Studio 日志**：
```bash
# 在 DevEco Studio 中查看 JS 错误
# 1. 连接设备
# 2. 打开 Log 标签页
# 3. 选择 HiLog
# 4. 设置过滤条件：应用包名、Error/Warn
```

---

## 性能优化最佳实践

### 列表渲染优化

**FlatList 优化配置**：
```typescript
<FlatList
  removeClippedSubviews={true}
  data={DATA}
  renderItem={renderItem}
  keyExtractor={item => item.id}
  initialNumToRender={10}      // 初始渲染10项
  maxToRenderPerBatch={20}      // 每批渲染最多20项
  windowSize={10}               // 渲染窗口大小
  getItemLayout={(data, index) => ({
    length: 100,
    offset: 100 * index,
    index
  })}
/>
```

**SectionList 粘性头部**：
```typescript
<SectionList
  sections={DATA}
  stickySectionHeadersEnabled={true}
  removeClippedSubviews={true}
/>
```

### 图片加载优化

**三级缓存策略**：
```
内存缓存 → 磁盘缓存 → 网络下载
```

**图片预加载**：
```typescript
// 预加载单个图片
Image.prefetch('https://example.com/image.jpg');

// 批量查询缓存
await Image.queryCache([
  'https://example.com/image1.jpg',
  'https://example.com/image2.jpg'
]);
```

### 内存管理

**缓存控制**：
```typescript
// 限制缓存大小
const MAX_CACHE_SIZE = 128;

// LRU 缓存策略
set(key: string, value: T): void {
  this.data.set(key, value);

  // 检查是否超过最大尺寸
  if (this.data.size > this.maxSize) {
    const oldestKey = this.data.keys()[0];
    this.remove(oldestKey);
  }
}
```

**内存泄漏防护**：
```typescript
// 组件卸载时清理
useEffect(() => {
  const id = translateXYalue.addListener(() => {
    setNum(n => n + 1);
  });

  return () => {
    translateXYalue.removeListener(id);
    onPress.stop();  // 停止动画
  };
}, []);
```

### 布局优化

**避免过度嵌套**：
```typescript
// 优化前 - 过度嵌套
<View>
  <View>
    <View>
      <Text>内容</Text>
    </View>
  </View>
</View>

// 优化后 - 扁平结构
<View>
  <Text>内容</Text>
</View>
```

**静态样式优化**：
```typescript
// 避免：在渲染函数中创建样式
const Component = () => {
  const styles = StyleSheet.create({
    container: { width: '100%' }
  });
}

// 推荐：使用静态样式
const staticStyles = StyleSheet.create({
  container: { width: '100%' }
});
```

### 动画优化

**原生驱动动画**：
```typescript
const onPress = useRef(
  Animated.loop(
    Animated.timing(translateXYalue, {
      toValue: {x: 100, y: 100},
      useNativeDriver: true,  // 使用原生驱动
    }),
  ),
).current;
```

**动画优化技巧**：
```typescript
// 使用 useMemo 避免重复创建动画
const animation = useMemo(() => {
  const expand = Animated.timing(animWidth, {
    toValue: 300,
    duration: 1000,
    useNativeDriver: false,
  });
  const contract = Animated.timing(animWidth, {
    toValue: 100,
    duration: 1000,
    useNativeDriver: false,
  });
  return Animated.sequence([expand, contract]);
}, []);
```

### 网络优化

**请求去重**：
```typescript
// 检查是否已有相同请求正在进行
if (this.activeRequestByUrl.has(url)) {
  return this.activeRequestByUrl.get(url);
}
```

**预加载优化**：
```typescript
// 检查是否已缓存
if (this.diskCache.has(uri)) {
  return true;  // 已缓存，无需重复下载
}
```

### 渲染优化

**React.memo 避免重渲染**：
```typescript
const Item = memo(function Item({title}: ItemProps) {
  return (
    <View>
      <Text>{title}</Text>
    </View>
  );
});
```

**useCallback 和 useMemo**：
```typescript
const FlatListTest = () => {
  const renderItem = useCallback(({item}) => <Item title={item.title} />, []);
  const keyExtractor = useCallback(item => item.id, []);

  return (
    <FlatList
      data={DATA}
      renderItem={renderItem}
      keyExtractor={keyExtractor}
    />
  );
};
```

### 性能监控

**Systrace 集成**：
```typescript
export function SystraceTest() {
  return (
    <TestSuite name="Systrace">
      <TestCase.Example itShould="Performance tracing">
        <Button
          label="Start Systrace"
          onPress={() => Systrace.beginEvent('Test trace')}
        />
        <Button
          label="Stop Systrace"
          onPress={() => Systrace.endEvent()}
        />
      </TestCase.Example>
    </TestSuite>
  );
}
```

**性能测量**：
```typescript
// MeasureComponent.tsx
useEffect(() => {
  setTimeout(() => {
    const took = (drawnTss[1] - startTime).toFixed(3);
    Alert.alert(title, `${took} ms`, [{text: 'OK'}]);
  }, 1500);
}, [title, startTime]);
```

### 性能优化策略总结

| 优化类别 | 具体策略 |
|---------|----------|
| 渲染优化 | 虚拟化列表、React.memo、useMemo、useCallback |
| 内存优化 | 多层缓存、限制缓存大小、及时清理资源 |
| 网络优化 | 请求去重、图片预加载、缓存策略 |
| 动画优化 | useNativeDriver、避免频繁状态更新 |
| 布局优化 | 减少视图层级、Flexbox 布局、静态样式 |
| 性能监控 | Systrace、性能测量组件、实时监控 |

---

**注意**：此 Skill 文档会持续更新以反映 RNOH 的最新变化。建议定期查阅官方文档和源码以获取最新信息。
