# RNOH (React Native for OpenHarmony) 详细研究补充

本文档是对 skill.md 的补充，包含2026年1月最新研究的详细发现。

---

## 研究概述

本次研究通过以下方式进行：
1. **源码直接分析** - 阅读 RNOH 核心实现文件
2. **网络资源搜索** - 查阅2024-2025年最新技术文档
3. **实践案例研究** - 分析组件实现和常见问题

---

## 详细实现分析

### 1. MountingManagerCAPI 详细分析

**位置**: `tester/harmony/react_native_openharmony/src/main/cpp/RNOH/MountingManagerCAPI.cpp`

**关键发现**：

**Mutation 类型处理流程**：
```cpp
// 获取有效的 mutations - 分离 C-API 和 ArkTS 处理的组件
MutationList getValidMutations(MutationList const& mutations) {
    MutationList arkTSMutations{};
    for (auto const& mutation : mutations) {
        bool isArkTSMutation = false;
        switch (mutation.type) {
            case ShadowViewMutation::Create:
                isArkTSMutation = !isCAPIComponent(mutation.newChildShadowView);
                break;
            case ShadowViewMutation::Update:
                isArkTSMutation = !isCAPIComponent(mutation.newChildShadowView);
                break;
            case ShadowViewMutation::Delete:
                isArkTSMutation = !isCAPIComponent(mutation.oldChildShadowView);
                break;
            case ShadowViewMutation::Insert:
                // 如果父组件或子组件是 ArkTS 组件，整个 mutation 交给 ArkTS 处理
                isArkTSMutation = !isCAPIComponent(mutation.parentShadowView) ||
                    !isCAPIComponent(mutation.newChildShadowView);
                break;
            // ...
        }
        if (isArkTSMutation) {
            auto copyMutation = mutation;
            copyMutation.newChildShadowView.layoutMetrics.frame.origin = {0, 0};
            arkTSMutations.push_back(copyMutation);
        }
    }
    return arkTSMutations;
}
```

**组件类型判断缓存机制**：
```cpp
bool isCAPIComponent(ShadowView const& shadowView) {
    std::string componentName = shadowView.componentName;

    // 缓存已知组件类型
    if (m_cApiComponentNames.count(componentName) > 0) {
        return true;
    }
    if (m_arkTsComponentNames.count(componentName) > 0) {
        return false;
    }

    // 首次遇到该组件，检查是否在注册表中
    auto componentInstance = m_componentInstanceProvider->isContainComponentInstance(
        shadowView.tag);
    if (componentInstance) {
        m_cApiComponentNames.insert(std::move(componentName));
        return true;
    }
    m_arkTsComponentNames.insert(std::move(componentName));
    return false;
}
```

**性能追踪集成**：
```cpp
void didMount(MutationList const& mutations) {
    {
        // FABRIC_BATCH_EXECUTION_START 标记
        HarmonyReactMarker::logMarker(
            HarmonyReactMarkerId::FABRIC_BATCH_EXECUTION_START);
        m_componentInstanceProvider->clearPreallocationRequestQueue();
    }

    facebook::react::SystraceSection s(
        "#RNOH::MountingManager::didMount size = ", mutations.size());

    for (uint64_t i = 0; i < mutations.size(); ++i) {
        auto const& mutation = mutations[i];
        try {
            this->handleMutation(mutation);
        } catch (std::exception const& e) {
            LOG(ERROR) << "Mutation " << getMutationNameFromType(mutation.type)
                       << " failed: " << e.what();
        }
    }

    // FABRIC_BATCH_EXECUTION_END 标记
    HarmonyReactMarker::logMarker(
        HarmonyReactMarkerId::FABRIC_BATCH_EXECUTION_END);
}
```

### 2. ArkUINode 详细分析

**位置**: `tester/harmony/react_native_openharmony/src/main/cpp/RNOH/arkui/ArkUINode.cpp`

**无障碍角色映射**：
```cpp
const std::unordered_map<std::string, ArkUI_NodeType> NODE_TYPE_BY_ROLE_NAME = {
    {"button", ARKUI_NODE_BUTTON},
    {"togglebutton", ARKUI_NODE_TOGGLE},
    {"search", ARKUI_NODE_TEXT_INPUT},
    {"image", ARKUI_NODE_IMAGE},
    {"text", ARKUI_NODE_TEXT},
    {"adjustable", ARKUI_NODE_SLIDER},
    {"imagebutton", ARKUI_NODE_BUTTON},
    {"checkbox", ARKUI_NODE_CHECKBOX},
    {"menuitem", ARKUI_NODE_LIST_ITEM},
    {"progressbar", ARKUI_NODE_PROGRESS},
    {"radio", ARKUI_NODE_RADIO},
    {"scrollbar", ARKUI_NODE_SCROLL},
    {"switch", ARKUI_NODE_TOGGLE},
    {"list", ARKUI_NODE_LIST},
    {"cell", ARKUI_NODE_GRID_ITEM},
    {"grid", ARKUI_NODE_GRID},
    // ... 更多映射
};
```

**全局事件接收器模式**：
```cpp
// 静态函数 - 所有 ArkUINode 共享
static void receiveEvent(ArkUI_NodeEvent* event) {
    try {
        auto eventType = OH_ArkUI_NodeEvent_GetEventType(event);
        auto node = OH_ArkUI_NodeEvent_GetNodeHandle(event);

        // 从 userData 获取实例指针
        ArkUINode* target = static_cast<ArkUINode*>(
            NativeNodeApi::getInstance()->getUserData(node));

        if (eventType == NODE_TOUCH_EVENT) {
            // Node Touch events are handled in UIInputEventHandler instead
            return;
        }

        if (eventType == NODE_ON_TOUCH_INTERCEPT) {
            auto inputEvent = OH_ArkUI_NodeEvent_GetInputEvent(event);
            target->onTouchIntercept(inputEvent);
            return;
        }

        auto componentEvent = OH_ArkUI_NodeEvent_GetNodeComponentEvent(event);
        if (componentEvent != nullptr) {
            target->onNodeEvent(eventType, componentEvent->data);
            return;
        }
    } catch (std::exception& e) {
        LOG(ERROR) << e.what();
    }
}

// 构造时注册事件接收器
ArkUINode::ArkUINode(ArkUI_NodeHandle nodeHandle) : m_nodeHandle(nodeHandle) {
    maybeThrow(NativeNodeApi::getInstance()->addNodeEventReceiver(
        m_nodeHandle, receiveEvent));
    NativeNodeApi::getInstance()->setUserData(m_nodeHandle, this);
    for (auto eventType : NODE_EVENT_TYPES) {
        this->registerNodeEvent(eventType);
    }
}
```

**脏标记机制**：
```cpp
void ArkUINode::markDirty() {
    // 标记需要重绘、布局和测量
    NativeNodeApi::getInstance()->markDirty(
        getArkUINodeHandle(), ArkUI_NodeDirtyFlag::NODE_NEED_RENDER);
    NativeNodeApi::getInstance()->markDirty(
        getArkUINodeHandle(), ArkUI_NodeDirtyFlag::NODE_NEED_LAYOUT);
    NativeNodeApi::getInstance()->markDirty(
        getArkUINodeHandle(), ArkUI_NodeDirtyFlag::NODE_NEED_MEASURE);
}
```

### 3. ComponentInstance 详细分析

**位置**: `tester/harmony/react_native_openharmony/src/main/cpp/RNOH/ComponentInstance.cpp`

**子节点插入实现**：
```cpp
void ComponentInstance::insertChild(
    ComponentInstance::Shared childComponentInstance,
    std::size_t index) {
    // 边界检查
    if (index > m_children.size()) {
        LOG(ERROR) << "index out of range, index=" << index
                   << " size=" << m_children.size();
        return;
    }

    // 设置父子关系
    auto it = m_children.begin() + index;
    childComponentInstance->setParent(shared_from_this());
    childComponentInstance->setIndex(index);

    // 子类钩子
    onChildInserted(childComponentInstance, index);

    // 插入到子节点列表
    m_children.insert(it, std::move(childComponentInstance));
}
```

### 4. ScrollViewComponentInstance 详细分析

**位置**: `tester/harmony/react_native_openharmony/src/main/cpp/RNOHCorePackage/ComponentInstances/ScrollViewComponentInstance.cpp`

**触摸事件处理**：
```cpp
class ScrollViewTouchHandler : public UIInputEventHandler {
private:
    ScrollViewComponentInstance* m_scrollViewComponentInstance;

public:
    explicit ScrollViewTouchHandler(ScrollViewComponentInstance* rootView)
        : UIInputEventHandler(rootView->getLocalRootArkUINode()),
          m_scrollViewComponentInstance(rootView) {}

    void onTouchEvent(ArkUI_UIInputEvent* event) override {
        auto action = OH_ArkUI_UIInputEvent_GetAction(event);
        if (action == UI_TOUCH_EVENT_ACTION_UP) {
            m_scrollViewComponentInstance->onTouchEventActionUp();
        }
    }
};
```

**嵌套滚动配置**：
```cpp
ScrollViewComponentInstance::ScrollViewComponentInstance(Context context)
    : CppComponentInstance(std::move(context)),
      m_scrollNode(m_arkUINodeCtx),
      m_contentContainerNode(m_arkUINodeCtx) {
    m_touchHandler = std::make_unique<ScrollViewTouchHandler>(this);
    m_scrollNode.insertChild(m_contentContainerNode);

    // RTL 考虑
    m_scrollNode.setAlignment(ARKUI_ALIGNMENT_TOP_START);
    m_scrollNode.setScrollNodeDelegate(this);

    // 嵌套滚动模式 - 自身优先
    m_scrollNode.setNestedScroll(ARKUI_SCROLL_NESTED_MODE_SELF_FIRST);
}
```

**滚动偏移更新逻辑**：
```cpp
void updateOffsetAfterChildChange(facebook::react::Point offset) {
    if (m_scrollState != ScrollState::IDLE) {
        return;
    }

    facebook::react::Point targetOffset = {offset.x, offset.y};

    // 根据滚动方向限制边界
    if (isHorizontal(m_props)) {
        if (targetOffset.x > m_contentSize.width - m_containerSize.width) {
            targetOffset.x = m_contentSize.width - m_containerSize.width;
        }
        if (targetOffset.x < 0) {
            targetOffset.x = 0;
        }
    } else {
        if (targetOffset.y > m_contentSize.height - m_containerSize.height) {
            targetOffset.y = m_contentSize.height - m_containerSize.height;
        }
        if (targetOffset.y < 0) {
            targetOffset.y = 0;
        }
    }

    if (offset == targetOffset) {
        return;
    }

    m_scrollNode.scrollTo(
        targetOffset.x, targetOffset.y, false, m_scrollToOverflowEnabled);
    updateContentClippedSubviews();
}
```

### 5. TextInputComponentInstance 详细分析

**位置**: `tester/harmony/react_native_openharmony/src/main/cpp/RNOHCorePackage/ComponentInstances/TextInputComponentInstance.cpp`

**内容大小变化处理**：
```cpp
void onContentSizeChange(float width, float height, bool multiline) {
    // 验证 multiline 状态一致性
    if (multiline == m_multiline){
        m_contentSizeWidth = width;
        m_contentSizeHeight = height;
        m_eventEmitter->onContentSizeChange(getOnContentSizeChangeMetrics());
    }
}
```

**文本选择变化处理**：
```cpp
void onTextSelectionChange(int32_t location, int32_t length) {
    if (m_isControlledTextInput &&
        m_hasLatestControlledValueChangeBeenProcessed) {
        m_caretPositionForControlledInput = m_selectionStart;
        if (m_valueChanged) {
            m_hasLatestControlledValueChangeBeenProcessed = false;
        }
    }

    if (m_textWasPastedOrCut) {
        m_textWasPastedOrCut = false;
    } else if (m_valueChanged) {
        std::u16string key;
        bool noPreviousSelection = m_selectionLength == 0;
        bool cursorDidNotMove = location == m_selectionLocation;
        bool cursorMovedBackwardsOrAtBeginningOfInput =
            (location < m_selectionLocation) || location <= 0;

        if (!cursorMovedBackwardsOrAtBeginningOfInput &&
            (noPreviousSelection || !cursorDidNotMove)) {
            auto utfContent = boost::locale::conv::utf_to_utf<char16_t>(m_content);
            if (location > 0 && location <= utfContent.size()) {
                int length = std::max(location - m_selectionLocation, 1);
                length = std::min(length, location);
                // ... 处理键盘事件
            }
        }
    }
}
```

**取消按钮模式处理**：
```cpp
void onBlur() {
    this->m_focused = false;

    // 根据焦点状态调整取消按钮
    if (m_props->traits.clearButtonMode ==
        facebook::react::TextInputAccessoryVisibilityMode::WhileEditing) {
        m_textInputNode.setCancelButtonMode(
            facebook::react::TextInputAccessoryVisibilityMode::Never);
    } else if (
        m_props->traits.clearButtonMode ==
        facebook::react::TextInputAccessoryVisibilityMode::UnlessEditing) {
        m_textInputNode.setCancelButtonMode(
            facebook::react::TextInputAccessoryVisibilityMode::Always);
    }

    m_eventEmitter->onBlur(getTextInputMetrics());
    m_eventEmitter->onEndEditing(getTextInputMetrics());
}
```

### 6. ImageComponentInstance 详细分析

**位置**: `tester/harmony/react_native_openharmony/src/main/cpp/RNOHCorePackage/ComponentInstances/ImageComponentInstance.cpp`

**URI 处理常量**：
```cpp
const std::string RAWFILE_PREFIX = "resource://RAWFILE/assets/";
const std::string INVALID_PATH_PREFIX = "invalidpathprefix/";
const std::string RESFILE_PREFIX = "file:///data/storage/el1/bundle/";
const std::string BASE_64_PREFIX = "data:";
const std::string BASE_64_MARK = ";base64,";
const std::int32_t BASE_64_FORMAT_LENGTH = 8;
const std::string BASE_64_STANDARD_PREFIX = "data:image/png;base64,";
const std::string RESFILE_PATH = "/resources/resfile/assets/";

const std::unordered_set<std::string> validImageTypes = {
    "png", "jpeg", "jpg", "gif", "bmp", "webp"
};
```

**Base64 URI 处理**：
```cpp
bool isValidMimeType(const std::string& mimeType) {
    if (mimeType.empty()) {
        return false;
    }

    if (mimeType.substr(0, BASE_64_MIME_TYPE_LENGTH) != "image/") {
        return false;
    }
    std::string imageType = mimeType.substr(BASE_64_MIME_TYPE_LENGTH);

    return validImageTypes.find(imageType) != validImageTypes.end();
}

std::string processBase64Uri(const std::string& uri) {
    size_t base64Pos = uri.find(BASE_64_MARK);
    if (base64Pos == std::string::npos) {
        return uri;
    }
    size_t mimeStart = BASE_64_PREFIX.length();
    std::string mimeType = uri.substr(mimeStart, base64Pos - mimeStart);

    if (base64Pos <= mimeStart || !isValidMimeType(mimeType)) {
        // 只有 MIME 类型非法时才改为 image/png
        return BASE_64_STANDARD_PREFIX +
            uri.substr(base64Pos + BASE_64_FORMAT_LENGTH);
    }

    return uri;
}
```

**图片源设置**：
```cpp
void setSources(facebook::react::ImageSources const& sources) {
    // 确定的 layoutMetrics 对于选择最佳源至关重要
    if (m_layoutMetrics == facebook::react::EmptyLayoutMetrics) {
        return;
    }

    auto uri = m_deps->imageSourceResolver->resolveImageSource(
        *this, m_layoutMetrics, sources);

    if (uri.rfind(BASE_64_PREFIX, 0) == 0 &&
        uri.find(BASE_64_MARK) != std::string::npos) {
        uri = processBase64Uri(uri);
    }

    this->getLocalRootArkUINode().setSources(
        uri, getAbsolutePathPrefix(getBundlePath())
    );
}
```

### 7. TurboModuleFactory 详细分析

**位置**: `tester/harmony/react_native_openharmony/src/main/cpp/RNOH/TurboModuleFactory.cpp`

**TurboModule 创建流程**：
```cpp
SharedTurboModule create(
    std::shared_ptr<facebook::react::CallInvoker> jsInvoker,
    const std::string& name,
    std::shared_ptr<EventDispatcher> eventDispatcher,
    std::shared_ptr<MessageQueueThread> jsQueue,
    std::shared_ptr<facebook::react::Scheduler> scheduler,
    RNInstance::SafeWeak instance) const {

    DLOG(INFO) << "Creating Turbo Module: " << name;

    // 1. 确定 TurboModule 运行线程
    auto arkTSTurboModuleThread =
        this->findArkTSTurboModuleThread(name).value_or(TaskThread::JS);

    // 2. 获取 ArkTS TurboModule 环境
    auto arkTSTurboModuleEnvironment =
        this->getArkTSTurboModuleEnvironmentByTaskThread(arkTSTurboModuleThread);

    // 3. 构建上下文
    Context ctx{
        .jsInvoker = jsInvoker,
        .instance = instance.lock(),
        .safeInstance = instance,
        .arkTSMessageHub = m_arkTSMessageHub,
        .env = arkTSTurboModuleEnvironment.napiEnv,
        .arkTSTurboModuleInstanceRef =
            arkTSTurboModuleThread == TaskThread::JS
            ? NapiRef{}
            : this->maybeGetArkTsTurboModuleInstanceRef(
                  name, arkTSTurboModuleThread, arkTSTurboModuleEnvironment),
        .turboModuleThread = arkTSTurboModuleThread,
        .taskExecutor = m_taskExecutor,
        .eventDispatcher = eventDispatcher,
        .jsQueue = jsQueue,
        .scheduler = scheduler
    };

    // 4. UIManager 特殊处理
    if (name == "UIManager") {
        HarmonyReactMarker::logMarker(
            HarmonyReactMarkerId::CREATE_UI_MANAGER_MODULE_START);
        auto uiManagerModule = std::make_shared<UIManagerModule>(
            ctx, name, std::move(m_componentBinderByString));
        HarmonyReactMarker::logMarker(
            HarmonyReactMarkerId::CREATE_UI_MANAGER_MODULE_END);
        return uiManagerModule;
    }

    // 5. 委托给其他 delegates
    auto result = this->delegateCreatingTurboModule(ctx, name);

    // 6. 验证 ArkTS 侧实现
    if (result != nullptr) {
        auto arkTSTurboModule =
            std::dynamic_pointer_cast<const ArkTSTurboModule>(result);
        if (arkTSTurboModule != nullptr && !ctx.arkTSTurboModuleInstanceRef) {
            std::vector<std::string> suggestions = {
                "Have you linked a package that provides this turbo module on the ArkTS side?"
            };
            if (!m_featureFlagRegistry->isFeatureFlagOn("WORKER_THREAD_ENABLED")) {
                suggestions.push_back(
                    "Is this a WorkerTurboModule? If so, it requires the Worker thread to be enabled.");
            }
            throw FatalRNOHError(
                std::string("Couldn't find Turbo Module '")
                    .append(name)
                    .append("' on the ArkTS side."),
                suggestions);
        }
        return result;
    }

    // 7. 验证 C++ 侧实现
    if (ctx.arkTSTurboModuleInstanceRef) {
        m_taskExecutor->runTask(
            arkTSTurboModuleThread,
            [tmRef = std::move(ctx.arkTSTurboModuleInstanceRef)] {}
        );
        std::vector<std::string> suggestions = {
            "Have you linked a package that provides this turbo module on the CPP side?"
        };
        throw FatalRNOHError(
            std::string("Couldn't find Turbo Module '")
                .append(name)
                .append("' on the CPP side."),
            suggestions);
    }

    return this->handleUnregisteredModuleRequest(ctx, name);
}
```

**Worker 线程检测**：
```cpp
std::optional<TaskThread> findArkTSTurboModuleThread(
    const std::string& turboModuleName) const {
    if (m_featureFlagRegistry->isFeatureFlagOn("WORKER_THREAD_ENABLED")) {
        auto workerArkTSTurboModuleEnv =
            this->getArkTSTurboModuleEnvironmentByTaskThread(TaskThread::WORKER);
        if (this->hasArkTSTurboModule(
                turboModuleName,
                TaskThread::WORKER,
                workerArkTSTurboModuleEnv.napiEnv,
                workerArkTSTurboModuleEnv.arkTSTurboModuleProviderRef)) {
            return TaskThread::WORKER;
        }
    }
    auto mainArkTSTurboModuleEnv =
        this->getArkTSTurboModuleEnvironmentByTaskThread(TaskThread::MAIN);
    if (this->hasArkTSTurboModule(
            turboModuleName,
            TaskThread::MAIN,
            mainArkTSTurboModuleEnv.napiEnv,
            mainArkTSTurboModuleEnv.arkTSTurboModuleProviderRef)) {
        return TaskThread::MAIN;
    }
    return std::nullopt;
}
```

---

## .harmony.js 补丁分析

### View.harmony.js 补丁

**位置**: `react-native-harmony/Libraries/Components/View/View.harmony.js`

**关键补丁**：
```javascript
/*
* RNOH patches:
* - imports
* - disable view flattening (collapsable prop) for box-only and none pointer events
*/

// View 组件的 collapsable 补丁
collapsable={["box-only", "none"].includes(newPointerEvents) ? false : collapsable}
```

**说明**：RNOH 禁用了 pointerEvents 为 "box-only" 或 "none" 时的 view flattening 优化，确保事件处理正确。

### ScrollView.harmony.js 补丁

**位置**: `react-native-harmony/Libraries/Components/ScrollView/ScrollView.harmony.js`

**关键补丁**：
```javascript
//RNOH: patch - fix imports
import type { HostComponent } from "react-native/Libraries/Renderer/shims/ReactNativeTypes";
// ... 其他导入
import Platform from "../../Utilities/Platform";
import TextInputState from "../TextInput/TextInputState.harmony";  // 使用 harmony 版本
```

**说明**：ScrollView 使用了 harmony 专用的 TextInputState 导入，并修复了导入路径。

### TextInput.harmony.js 补丁

**位置**: `react-native-harmony/Libraries/Components/TextInput/TextInput.harmony.js`

**关键补丁**：
```javascript
const OS = "ios" // RNOH: patch - replaced occurrences Platform.OS with OS

// RNOH: patch - updated imports
import type {HostComponent} from 'react-native/Libraries/shims/ReactNativeTypes';

// ... 后续代码使用 OS 而不是 Platform.OS
if (OS === 'android') {
    AndroidTextInput = require(...);
} else if (OS === 'ios') {
    RCTSinglelineTextInputView = require(...);
}
```

**说明**：TextInput 使用固定的 OS 常量而非 Platform.OS，确保平台特定代码的正确性。

---

## 常见问题深度分析

### zIndex 失效问题

**根本原因**：
1. Fabric 架构对 CSS 规范更严格
2. zIndex 只影响同一 stacking context 中的元素
3. 需要配合 position 属性使用

**解决方案**：
```css
/* 错误 */
.child {
  zIndex: 10;  /* 可能不生效 */
}

/* 正确 */
.parent {
  position: 'relative';
}
.child {
  position: 'absolute';
  zIndex: 10;
}
```

**调试步骤**：
1. 使用 `get_ui_tree` 查看 drawRegion
2. 验证 position 属性是否正确设置
3. 检查是否在同一 stacking context

### SafeAreaView 问题

**鸿蒙特定问题**：
- 官方 SafeAreaView 主要针对 iOS/Android
- 鸿蒙屏幕安全区适配可能不完整

**推荐方案**：
```javascript
// 使用 react-native-safe-area-context
import { SafeAreaProvider, SafeAreaView } from 'react-native-safe-area-context';

function App() {
  return (
    <SafeAreaProvider>
      <SafeAreaView style={{ flex: 1 }}>
        {/* 内容 */}
      </SafeAreaView>
    </SafeAreaProvider>
  );
}
```

**配置步骤**：
1. 安装包：`ohpm install @react-native-oh-tpl/react-native-safe-area-context`
2. 手动配置 CMakeLists.txt
3. 在 PackageProvider 中注册

### setNativeProps 废弃影响

**为什么废弃**：
- Fabric 架构不支持直接操作 native props
- 只能生效一次，后续更新被忽略
- 与 React 声明式模型冲突

**迁移方案**：
```javascript
// 旧代码
class MyComponent extends React.Component {
  handlePress = () => {
    this.refs.myComponent.setNativeProps({
      style: { opacity: 0.5 }
    });
  }

  render() {
    return <View ref="myComponent" style={{ opacity: 1 }} />;
  }
}

// 新代码
function MyComponent() {
  const [opacity, setOpacity] = useState(1);

  const handlePress = () => {
    setOpacity(0.5);
  };

  return <View style={{ opacity }} onPress={handlePress} />;
}
```

### TurboModuleRegistry 迁移

**注意事项**：
1. 必须使用 `getEnforcing()` 而非 `get()`
2. TurboModule 可能在运行时不存在
3. 需要确保 C++ 和 ArkTS 双侧实现

```javascript
// 正确的用法
import { TurboModuleRegistry } from 'react-native';

const MyModule = TurboModuleRegistry.getEnforcing('MyModule');
// 如果 MyModule 不存在，这里会抛出错误

// 带检查的用法
const MyModule = TurboModuleRegistry.get('MyModule');
if (MyModule != null) {
  // 可选使用
}
```

---

## 性能优化建议

### 1. 优先使用 C-API 组件

C-API 组件性能优势：
- 直接操作 ArkUI C-API
- 无跨语言调用开销
- 属性 diff 优化
- 可在子线程预创建

### 2. TurboModule 线程选择

| 场景 | 推荐线程类型 | 原因 |
|------|-------------|------|
| 纯计算逻辑 | cxxTurboModule (JS) | 无跨语言开销 |
| 网络请求 | ArkTSTurboModule (WORKER) | 不阻塞主线程 |
| 文件操作 | ArkTSTurboModule (WORKER) | 不阻塞主线程 |
| UI 操作 | ArkTSTurboModule (MAIN) | 必须在主线程 |

### 3. 减少重渲染

```javascript
// 使用 memo 和 useMemo
const MyComponent = React.memo(function MyComponent({ data }) {
  const processedData = useMemo(() => {
    return expensiveOperation(data);
  }, [data]);

  return <View>{processedData}</View>;
});
```

### 4. 优化布局计算

```javascript
// 避免深层嵌套
// 不好
<View>
  <View>
    <View>
      <View>
        <Text>内容</Text>
      </View>
    </View>
  </View>
</View>

// 较好
<View style={{ flexDirection: 'row' }}>
  <Text>内容</Text>
</View>
```

---

## 调试技巧

### 使用 MCP 工具

```bash
# 1. 检查前台应用
list_windows

# 2. 获取 UI 树
get_ui_tree

# 3. 查看特定进程的 UI 树
get_ui_tree <pid>

# 4. 截图对比
screenshot /tmp/before.png
# ... 操作
screenshot /tmp/after.png
```

### 性能追踪

```cpp
// 在代码中添加性能标记
HarmonyReactMarker::logMarker(
    HarmonyReactMarkerId::CUSTOM_MARKER,
    "Custom operation"
);

// 或使用 SystraceSection
facebook::react::SystraceSection s("CustomOperation");
// ... 操作代码
```

### 日志调试

```cpp
// 使用 glog
LOG(INFO) << "Info message";
LOG(WARNING) << "Warning message";
LOG(ERROR) << "Error message";

// DLOG (仅调试模式)
DLOG(INFO) << "Debug info";
```

---

## 扩展阅读

### 官方资源

1. [RNOH 官方仓库](https://gitee.com/openharmony-sig/ohos_react_native)
2. [RNOH 三方库文档](https://react-native-oh-library.github.io/usage-docs/)
3. [华为开发者联盟](https://developer.huawei.com/consumer/cn/blog/topic/03169292089068014)

### 技术文章 (2024-2025)

1. [鸿蒙NEXT架构浅析](https://blog.csdn.net/m0_74823264/article/details/145808080)
2. [React Native跨平台鸿蒙开发高级应用原理](https://ai6s.net/69265768791c233193d04253.html)
3. [鸿蒙中RNOH架构介绍](https://juejin.cn/post/7473777534543740937)
4. [React Native for OpenHarmony一文搞定](https://openharmonycrossplatform.csdn.net/69294dbe2087ae0db79d3aae.html)
5. [使用React Native开发HarmonyOS应用TOP问题集锦](https://www.infoq.cn/article/sjtlpdkaiLvrlSr5PFwZ)

### 视频资源

1. [HarmonyOS 开发者论坛](https://developer.huawei.com/consumer/cn/forum)
2. [RNOH 技术分享会](https://gitcode.com/openharmony-sig/ohos_react_native)

---

**更新日期**: 2026年1月12日

**版本**: 1.0
