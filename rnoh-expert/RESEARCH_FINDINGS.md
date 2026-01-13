# RNOH (React Native for OpenHarmony) Research Findings

## Research Summary

This document contains comprehensive research findings about RNOH (React Native for OpenHarmony) gathered from codebase exploration and web research in January 2026.

---

## Key Sources

### Official Documentation
- [RNOH Official Repository](https://gitee.com/openharmony-sig/ohos_react_native) - Main repository with source code and documentation
- [RNOH Third-Party Library Documentation](https://react-native-oh-library.github.io/usage-docs/) - Usage guides for third-party libraries
- [New Architecture Documentation](https://github.com/react-native-oh-library/doc/blob/master/zh-cn/new-architecture.md) - Fabric/TurboModule architecture details

### Developer Resources (2025)
- [鸿蒙NEXT架构浅析](https://blog.csdn.net/m0_74823264/article/details/145808080) - February 2025 architecture analysis
- [React Native跨平台鸿蒙开发高级应用原理](https://ai6s.net/69265768791c233193d04253.html) - November 2025 advanced principles
- [鸿蒙中RNOH架构介绍](https://juejin.cn/post/7473777534543740937) - February 2025 architecture overview
- [React Native for OpenHarmony一文搞定](https://openharmonycrossplatform.csdn.net/69294dbe2087ae0db79d3aae.html) - November 2025 guide
- [使用React Native开发HarmonyOS应用TOP问题集锦](https://www.infoq.cn/article/sjtlpdkaiLvrlSr5PFwZ) - Common issues and solutions
- [RN鸿蒙自定义TurboModule](https://developer.huawei.com/consumer/cn/blog/topic/03169292089068014) - December 2024 TurboModule guide
- [OpenHarmony使用ContentSlot占位组件](https://blog.csdn.net/m0_68635815/article/details/155262925) - November 2025 ContentSlot guide

### Performance & Architecture
- [为什么跨平台框架可以适配鸿蒙](https://guoshuyu.cn/home/wx/fo.html) - Cross-platform framework technical principles
- [Yoga Flexbox布局终极指南](https://openharmonycrossplatform.csdn.net/693fe0ac96fa167eeecddec0.html) - Flexbox layout guide
- [鸿蒙开源三方组件--yoga](https://www.cnblogs.com/HarmonyOS/p/14705069.html) - Yoga component for HarmonyOS

### HarmonyOS APIs
- [XComponent Documentation](https://developer.huawei.com/consumer/cn/doc/harmonyos-guides/napi-xcomponent-guidelines) - Custom rendering documentation
- [JSVM-API Overview](https://gitee.com/openharmony/docs/blob/39467f023bec8cfca8ec2f97b99039b1dbd141e5/en/application-dev/napi/jsvm-introduction.md) - JSVM API reference

---

## Architecture Highlights

### Version Information
- **RNOH Version**: 0.72.102
- **React Native Base**: 0.72.5 (Fabric new architecture)
- **HarmonyOS SDK**: 5.0.0.61+ (SP1)
- **Latest Release**: v5.0.0.813 (December 26, 2024)

### Core Architecture Layers

```
React Native JS Layer (Components, APIs)
         ↓ JSI
React Common C++ Layer (ShadowTree, Scheduler, Yoga)
         ↓
RNOH Adaptation Layer (C++ ~1073 files, ArkTS ~51 files)
         ↓ N-API
OpenHarmony OS Layer (ArkUI, Native C-API)
```

### Key Components

#### 1. Component Rendering System
- **Fabric-based architecture** - New React Native rendering system
- **Three rendering phases**: Render (JS) → Diff (JS) → Execute (MAIN)
- **C-API components**: Direct ArkUI C-API integration for performance
- **ArkTS components**: Flexible implementation via ArkTS layer

#### 2. TurboModule Architecture
- **cxxTurboModule**: Pure C++ implementation for platform-independent functionality
- **ArkTSTurboModule**: ArkTS-based implementation for HarmonyOS system APIs
- **Dual-thread support**: MAIN thread for UI operations, WORKER thread for heavy tasks
- **JSI-based**: Direct JavaScript-C++ interface without bridge overhead

#### 3. ContentSlot (New Architecture)
- **Replaces XComponent** NODE type (which is no longer evolving)
- **Better performance**: Superior to XComponent in memory and performance
- **Two-step process**: createSurface → startSurface with attachToNodeContent

---

## Key Implementation Details

### Thread Model
- **MAIN**: UI operations, component lifecycle, ArkTS TurboModules
- **JS**: JavaScript runtime, ShadowTree computation, Yoga layout
- **BACKGROUND**: Experimental (not recommended for production)
- **WORKER**: TurboModule execution to prevent UI blocking

### Component Types
1. **C-API Components** (Performance priority)
   - Direct C++ to ArkUI C-API connection
   - No cross-language overhead
   - Properties diff optimization
   - Examples: StackNode, TextNode, ImageNode, TextInputNode

2. **ArkTS Components** (Flexibility priority)
   - Implemented through ArkTS layer
   - Registered via arkTsComponentNames
   - Suitable for HarmonyOS-specific features

### JavaScript Engine Options
- **Hermes** (default): Meta's JavaScript engine
- **JSVM**: OpenHarmony's V8-based JavaScript engine

---

## Migration from RN 0.51/0.59

### Key Changes
1. **TurboModule** replaces NativeModules
2. **Event listener** removal: `subscription.remove()` instead of `removeEventListener`
3. **setNativeProps deprecated**: Use React state management
4. **Date format**: ISO 8601 format required
5. **Platform detection**: `PlatformUtils.isHarmony()`
6. **File extensions**: `.harmony.tsx` for OpenHarmony

---

## Performance Optimizations

### C-API Advantages
- Minimal C layer with reduced cross-language calls
- Direct native API usage
- No data type conversions needed
- Property diff optimization
- Component pre-creation in background threads

### Parallelization Features
- ShadowTree parallelization (experimental)
- Worker thread for TurboModules
- Parallel component creation

---

## Development Best Practices

1. **Prioritize C-API components** for performance
2. **Use cxxTurboModule** for pure logic, ArkTSTurboModule for system APIs
3. **Check `.harmony.js` patches** for implementation examples
4. **Use ContentSlot** instead of XComponent
5. **Verify flexbox properties** against CSS specifications
6. **Use MCP tools** (harmonyos-ui) for UI debugging

---

## Known Issues & Solutions

| Issue | Cause | Solution |
|-------|-------|----------|
| zIndex failure | Fabric stricter CSS rules | Check positioning, use get_ui_tree |
| Layout issues | Yoga integration differences | Verify flexbox, parent container height |
| Gesture anomalies | Event dispatch changes | Check event listener registration |
| Animated lag | Architecture differences | Use NativeAnimatedTurboModule |
| Component not showing | ContentSlot connection | Verify attachToNodeContent |

---

## Tooling & Debugging

### Metro Bundler
- Hot reloading support
- Development bundle serving

### React DevTools
- Component tree inspection
- Props debugging

### HarmonyOS UI Debugging (MCP)
```bash
list_windows          # Check foreground app
get_ui_tree [pid]     # Get UI tree structure
list_abilities        # List Ability states
screenshot [path]     # Capture screenshot
```

---

## Project Structure

```
/Users/wuzhao/Desktop/ty/rnoh/ohos_react_native/
├── react-native-harmony/              # Core SDK (NPM)
│   ├── Libraries/                     # 36 .harmony.js patches
│   └── index.js
├── react-native-harmony-cli/          # CLI tool
├── react-native-harmony-hvigor-plugin/ # Build plugin
├── react-native-harmony-sample-package/ # Example module
├── tester/                            # Test app
│   └── harmony/react_native_openharmony/
│       └── src/main/
│           ├── cpp/                   # ~1073 C++ files
│           └── ets/                   # ~51 ArkTS files
└── docs/                              # Documentation
    ├── zh-cn/                         # Chinese docs
    └── en/                            # English docs
```

---

## Research Completion Notes

This research compiled information from:
- Direct codebase exploration
- Official documentation and repositories
- Developer articles from 2024-2025
- Architecture analysis papers
- Performance optimization guides

**Status**: Comprehensive understanding of RNOH architecture, implementation details, and best practices achieved.

**Last Updated**: January 12, 2026
