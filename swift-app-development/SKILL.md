---
name: swift-app-development
description: Swift / SwiftUI 应用开发与评审经验，来自 DeviceStatus、SwiftUtils、zoclip、SafaQrcode、KidToys 的真实提交。写或改原生 App 代码、排查「存了却读不回来 / 版本号对不上 / 时区差几小时 / 主线程 trap / 手表装不上 / 测试绿但功能不通」这类静默失败，或上架前自查时载入。
---

# Swift / SwiftUI 应用开发经验

这个技能收的不是"最佳实践"，是**编译器不拦、运行时不崩、日志里也没有**的那类失败。
它们共同的特征：代码看起来完全正确，用户只是觉得"不好用"。

## 下结论之前，先问一句

> 我判的是能力，还是我自己的假设？

四种最常被当成结论的间接证据：

| 间接证据 | 它实际证明了什么 |
|---|---|
| `try?` 没报错 | 只证明错误被变成了 `nil` |
| `#available` 通过 | 只证明系统版本够，不证明设备有这个能力 |
| 返回 `0` / `false` / `""` | 只证明函数返回了，不证明事实如此 |
| 测试绿了 | 只证明你断言的东西成立，不证明用户路径通 |

## 七个静默失败源（速查）

| # | 长什么样 | 先做这一件事 |
|---|---|---|
| 1 | **写读不对称**：存盘正常，重启后列表空了 | 列「写入点 / 读取点 / 字段表」，确认 `load` 真的被调用 |
| 2 | **同一事实散在多处**：版本号、构建号对不上 | 收敛到 build setting，代码只读 `Bundle` |
| 3 | **失败被默认值吞掉**：解码失败 → 默认值 → 下次保存覆盖真数据 | 区分「没有数据」和「数据坏了」，失败先留证 |
| 4 | **平台差异被假设抹平**：watchOS 装不上、API 没有 | 一个判定收口到一个类型，Info.plist 键用 `[sdk=...]` 限定 |
| 5 | **隔离在运行时才检查**：后台回调 `dispatch_assert_queue_fail` | 回调参数标 `@Sendable`，或调用点 `nonisolated` |
| 6 | **能力探测说谎**：显示"不支持"，用户明明有 | 编译期 / 设备 / 权限三层分开判，降级文案分开写 |
| 7 | **测试测的是间接证据**：绿着，功能不通 | 断言用户路径，别只断言内部状态存在 |

## 硬规则

### 数据
- 每个 `save` 必须有对应的 `load`，且 `load` 在启动路径上真的被调用过。
- 手写 `init(from:)` 用了 `decodeIfPresent` + 默认值，就等于"漏一行 = 永久静默丢失"；要么逐字段注释「必须登记」，要么别手写。
- 不要用 `(try? decode(...)) ?? Settings()` 一把梭：它把「首次启动」「文件损坏」「老格式」压成同一种，下一次保存会把默认值写回磁盘。
- 迁移判据用**整数格式号**（`if stored < 2`）或「无戳即迁移」，不要拿版本号字符串比大小。
- 验证方式是「写入 → 销毁实例 → 重新从磁盘构造 → 断言」，同一个对象再读一次不算。

### 版本
- 代码里只读 `Bundle.main.object(forInfoDictionaryKey:)`，不写常量；`MARKETING_VERSION` / `CURRENT_PROJECT_VERSION` 是唯一真相源。
- 构建号必须唯一且递增（`agvtool new-version -all` 或 CI 注入）；`CFBundleVersion` 只允许数字和点。
- 版本比较用 `compare(options: .numeric)`：`"1.9.0".compare("1.13.2")` 是字典序，答案是反的。

### 日期
- 时区一律 `TimeZone(identifier:)`。`TimeZone(abbreviation: "Asia/Shanghai")` 返回 **nil**，赋给 `DateFormatter` 不报错，静默回落设备时区。
- `DateFormatter` 的 `locale`（`en_US_POSIX`）、`calendar`、`timeZone` 三件套显式写全。
- 存 `Date`，不存"看起来像日期"的字符串；「今天/昨天」走 `Calendar`，不比格式化后的字符串。
- 测试注入固定时区/语言，别依赖（也别去改）`NSTimeZone.default` / `Locale.current`。

### 并发（Swift 6 最独特的一条）
- `SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor` + 回调式 API = 闭包隐式继承主线程隔离，后台队列回调直接 trap。**编译通过，只在走到那条路径时崩。**
- 根治在库侧：回调参数 `@escaping @Sendable`；过渡期调用点标 `nonisolated`；新代码用 `async` API。
- 限流 / 去重这类闸门用 `actor`，并且**先占位再 `await`**——`await` 会让出 actor，在它之后才改状态的设计一定会漏。

### 视图
- `body` 里不写会触发 `body` 重算的副作用；视图之间不互调，都读同一个 Store。
- 联动更新走单一入口（本机例子：`refreshLocalGroups()` 被 `.task` 和 `.onChange(of: colorScheme)` 共用，网络分组单独异步取）。
- 判断标准：删掉任意一个 Section，其余代码照常编译、照常工作。

### 能力探测
- 三层缺一不可：`#available` / 设备（`AVCaptureDevice.default(...)`）/ 权限（`authorizationStatus` + 真调一次）。
- 平台差异收口成一个类型（`SystemClipboard.isSupported` 用 `#if os(watchOS)`），调用方不散落条件编译。
- "读不到"要么返回 `Optional`，要么返回**有文档、测试钉住**的哨兵值，展示层必须能区分 `0` 和「未知」。
- 没有替代 API 时（如 watchOS 7 起 `contextMenu` 整体 deprecated 且无替代重载），判断"这个功能在手表上有没有意义"，没有就不挂。

### 多平台
- 单 target 多平台只能产出一个 bundle id → **做不出 companion watch app，只能 watch-only**（Apple 要求 companion id 是 `<父 id>.xxx`）。
- `INFOPLIST_KEY_*` 的平台专属键必须加 `[sdk=watchos*]` 之类的限定，否则会污染其他平台的 Info.plist。

### 依赖与工程
- `Package.resolved` 必须提交；升级依赖要读 changelog。
- 库仓库测逻辑，宿主工程测集成（跨平台条件编译分支只有宿主测得到）。
- 库里的 `NSApp?.x` 这类可选链会在非 GUI 进程静默返回 nil —— "能调用"不等于"行为对"。

### 测试
- 不要只断言 `!isEmpty`；结构、唯一性、取值域都要断。
- 依赖环境的值（深色模式、时区）只能钉契约，并在注释里写明"其余平台人工验证"。
- 异步用例要真的等到；UI 测试断 `isHittable`，不是 `exists`。
- 每类 bug 写「它回来就会红」的用例，而不是「验证代码写对了」。

### 发布
- archive 之后**回读**产物里的版本号，别只信命令行参数。
- 隐私清单 `PrivacyInfo.xcprivacy` 与所用 Required Reason API 对得上（漏了是 ITMS-91053）。
- TestFlight 冒烟要含「断网首启不白屏」，不是"能打开就算"。

## 完整版在哪

本技能目录下的 `references/experience.md` 是长文（每条的**现象 / 根因 / 做法 / 验证方式** + 43 项上线前检查清单）。
只读你需要的章节，别整篇灌进上下文：

| 你在做什么 | 去读这一节（标题可 grep） |
|---|---|
| 改持久化 / 模型 / 迁移 | `## 一、数据层：写路径写了，读路径忘了` |
| 发版、对版本号 | `## 二、版本：只允许存在一处` |
| 日期格式、时区、相对时间 | `## 三、日期与时区：timeZone = nil 不会报错` |
| 回调、actor、限流、MainActor | `## 四、并发：编译通过，运行时 trap` |
| SwiftUI 状态、无限刷新 | `## 五、视图层：body 里不要写副作用` |
| 相机 / 权限 / 平台能力 | `## 六、能力探测：#available 通过 ≠ 设备具备` |
| 网络、密钥、降级 | `## 七、无后端直连第三方 API` |
| 单 target、watchOS、离线 | `## 八、多平台：单 target 的边界，比你想的硬` |
| 布局、Dynamic Type、图标 | `## 九、UI 与可访问性` |
| SPM、宿主工程、拆 target | `## 十、依赖与工程结构` |
| 要不要自己写算法 | `## 十一、什么时候自己写（对应前端那份的"自研解码器"）` |
| 写测试、假绿 | `## 十二、测试：绿了不代表对` |
| 上架、签名、隐私清单 | `## 十三、发布与上架` |
| 上线前逐项核对 | `## 十四、上线前检查清单` |

## 别照搬

长文里带 `[通用]` 标记的条目是 Swift 生态里公认会咬人、但本机还没踩到的；
带项目名 + commit 的条目是真实发生过的，可以直接当案例引用。两者别混着说成"我们踩过"。
