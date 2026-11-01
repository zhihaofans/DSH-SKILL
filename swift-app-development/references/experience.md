# 原生 Swift / SwiftUI 应用开发经验

> **v1.0** —— 从这台机器上的几个 Swift 工程里沉出来的：
>
> - **DeviceStatus**（iOS 26 / watchOS 26 / macOS 26，单 target 多平台，Swift 6 + 默认 MainActor 隔离，SwiftUtils 的宿主工程）
> - **SwiftUtils**（自研工具库，SPM；DateUtil / DeviceUtil / NetworkUtil / ClipboardUtil）
> - **zoclip**（macOS 剪贴板历史）
> - **SafaQrcode**（Safari 扩展 + 二维码）
> - **KidToys**（iOS/watchOS 儿童向小工具）
>
> 适用场景：**SwiftUI 原生 App、数据主要存本地、无后端或弱后端、目标上架 App Store**。
>
> **证据规则**：每条都标出处。
> `[DeviceStatus]` `[SwiftUtils]` 这类是能在仓库里翻到 commit / 文件 / 行号的；
> `[通用]` 是 Swift 生态里公认会咬人、但这台机器上还没踩到的。
> 不写教科书条目 —— 只写**编译器不拦、运行时不崩、日志里也没有**的那类。

---

## 0. 一句话总纲

这类应用的难点**不在功能**，而在七个「静默失败」源：

| # | 静默失败源 | 本机证据 |
|---|---|---|
| 1 | **写读不对称**：存进去了，读的时候没读 | zoclip `ccfaaa4`「修复保存数据后重启应用不加载数据」 |
| 2 | **同一事实散在多处**：版本号 / 构建号 / 缓存名 | SafaQrcode `bd82375`「修复版本号不统一的 bug」 |
| 3 | **失败被默认值吞掉**：`try?` + `??` + `catch {}` | [通用]（Swift 写法天然纵容） |
| 4 | **平台差异被假设抹平**：`#available`、单 target、watchOS | DeviceStatus `d13cab7`「改成手表上是独立 App」 |
| 5 | **隔离在运行时才检查**：MainActor 回调 trap | [DeviceStatus] README §2，`dispatch_assert_queue_fail` |
| 6 | **能力探测说谎**：API 存在 / 返回 0，都不等于事实 | [SwiftUtils] `ece7fda`、`df31ffe` |
| 7 | **测试测的是间接证据**：绿了，但用户路径没通 | [DeviceStatus] 测试演进，README §3 |

除了第 5 条，其余都是同一批病根换了个语言；**第 5 条是 Swift 特有的**——
其他语言里"在错误的队列上执行"最多是警告，Swift 6 是直接 `fatalError` 级别地 trap 给你看。

### 为什么 Swift 的坑更容易被「类型正确」掩盖

下面这些写法**全部编译通过**，而且看起来都很合理：

```swift
let settings = (try? JSONDecoder().decode(Settings.self, from: data)) ?? Settings()
let level = device.getBatteryLevel()   // -1（读不到）和 0%（真的没电）都是 Int
if #available(iOS 17, *) { useNewAPI() }           // 新系统 ≠ 有摄像头
formatter.timeZone = TimeZone(abbreviation: "Asia/Shanghai")   // nil，不报错
```

类型系统只保证「形状对」，不保证「语义对」。而语义错了，UI 照样渲染出一个**看起来很合理**的数字。
这就是为什么这份文档里一半的篇幅在讲「怎么验证」，而不是「怎么写」。

---

## 一、数据层：写路径写了，读路径忘了

### 现象

> zoclip `ccfaaa4`「修复保存数据后重启应用不加载数据等 bug」

剪贴板历史存盘一切正常，退出重进列表永远是空的。没有崩溃、没有日志、没有报错。

### 根因

```swift
private init() {
    loadHistory()   // ← 修复前没有这一行
    start()
}
```

`saveHistory()` 一直在被调用，文件也确实写出来了；**只是启动时没人读它**。
这和纯前端那份文档里 normalize「漏登记一个字段」是同一类：
**写入是一条独立的路，读取是另一条，两条路之间没有编译器检查**。

### 做法：先做「写读对称」审计

改数据层之前，先列三个清单，缺一不可：

| 清单 | 查什么 | 本机反例 |
|---|---|---|
| 写入点 | 每个 `save` / `encode` / `try context.save()` 在哪儿 | `saveHistory()` |
| 读取点 | 每个 `load` / `decode` / `fetch` 在哪儿，**谁调用它** | `loadHistory()` 存在但没人调 |
| 字段表 | 每个字段在 encode / decode 各出现一次 | Codable 手写 `init(from:)` 最常漏 |

**Codable 的「逐字段重建」等价物**，就是手写解码初始化器：

```swift
init(from decoder: Decoder) throws {
    let c = try decoder.container(keyedBy: CodingKeys.self)
    theme     = try c.decodeIfPresent(String.self, forKey: .theme) ?? "system"
    webSearch = try c.decodeIfPresent(Bool.self, forKey: .webSearch) ?? false
    // 新增字段忘了加一行 → 存进去也读不回来，而且不报错
}
```

只要用了 `decodeIfPresent` + 默认值，**漏一行 = 永久静默丢失**。
所以要么给每个字段加注释「必须在此登记」，要么干脆别手写（见下一条）。

### `try?` + `??` 是最危险的默认值

```swift
// ❌ 三种情况被压成同一种：没有文件 / 解码失败 / 老格式
let settings = (try? decoder.decode(Settings.self, from: data)) ?? Settings()
```

问题不是"回落到默认值"本身，而是**下一次保存会把默认值写回磁盘**——
用户的真实数据在这时候就真的没了。

```swift
// ✅ 区分「没有数据」和「数据坏了」
guard let data = try? Data(contentsOf: url) else { return .fresh }   // 首次启动
do {
    return try decoder.decode(Settings.self, from: data)
} catch {
    // 坏数据先留证，再决定：备份 .bak / 上报 / 让用户选择
    try? data.write(to: url.appendingPathExtension("bak"))
    throw StorageError.corrupted(error)
}
```

**判断标准**：`catch` 里如果只有注释或空语句，这里迟早会吞掉一次真实故障。

### 迁移：用整数格式号，不要用字符串版本

```swift
enum SaveFormat: Int { case v1 = 1, v2 = 2 }   // 只增不减

if stored.format < SaveFormat.v2.rawValue {
    migrateV1toV2(&stored)
}
```

字符串比较的坑在 Swift 里一模一样，而且**有实测**：

```swift
"1.9.0".compare("1.13.2")             // .orderedDescending  ❌ 字典序
"1.9.0".compare("1.13.2", options: .numeric)   // .orderedAscending ✅
```

补一条：迁移判据最好用**「无戳即迁移」**——字段不存在就跑一次，写完戳就永远不再跑。
不要用「当前版本 < 目标版本」当判据，版本号是给人看的，不是给代码分支用的。

### UserDefaults 的三个静默坑

1. **`register(defaults:)` 只在内存里生效**，不落盘。用户从没改过的键，读出来有值，但磁盘上没有。
   如果别处按"磁盘上有没有这个键"做判断，两边会打架。
2. **key 写成散字符串**。拼错了不报错，只是永远读默认值。收口成 `enum DefaultsKey { static let theme = "settings.theme" }`。
3. **钥匙串的生命周期和 App 不一致**。UserDefaults 随 App 删除清空，钥匙串条目通常不会。
   产品上"卸载重装 = 干净状态"的假设会翻车，要么卸载时清理，要么接受它。

### SwiftData / Core Data：别把"容器创建失败"降级成"内存库"

```swift
// ❌ 启动失败 → 悄悄退回内存库 → 用户看到空数据 → 以为是 App 把数据吃了
let container = (try? ModelContainer(for: Item.self))
    ?? (try! ModelContainer(for: Item.self, configurations: .init(isStoredInMemoryOnly: true)))
```

`[通用]` 但很常见。正确姿势是：失败就显式报错、把旧库文件备份、给用户一个"修复/重建"的入口。
**任何"失败后静默换一种存储"的代码，都在制造一次数据丢失的观感。**

### 验证方式

```
写入 → 销毁内存中的实例 → 重新从磁盘构造 → 断言值还在
```

重点是**重新构造**，不是同一个 `@StateObject` 再读一次：
`zoclip` 那个 bug，在同一个进程里怎么点都正常，只有重启才暴露。

---

## 二、版本：只允许存在一处

### 现象

> SafaQrcode `bd82375`「中英文本地化，修复版本号不统一的 bug」

`project.pbxproj` 里四个构建配置的 `MARKETING_VERSION` 两套值并存（`1.0` 与 `0.0.1`）。
打包出来的版本和 App Store Connect 上填的对不上，排查时还得挨个 target / 配置去看。

### 根因

版本号是**构建配置**，不是源码常量。它天然散落在：

- `MARKETING_VERSION`（对用户可见的版本）
- `CURRENT_PROJECT_VERSION`（构建号）
- `Info.plist`（如果手写而不是用 `GENERATE_INFOPLIST_FILE`）
- 代码里硬编码的 `"1.0.0"` 字符串
- 每个 target 各有一份（App / 扩展 / watch App / 测试）

**单 target 多平台**（DeviceStatus 的形态）还好，只有一份；一旦拆出扩展或 watch target，就是 N 份。

### 做法

**① 代码里永远读 Bundle，不写常量**

```swift
enum AppVersion {
    static var marketing: String {
        Bundle.main.object(forInfoDictionaryKey: "CFBundleShortVersionString") as? String ?? "—"
    }
    static var build: String {
        Bundle.main.object(forInfoDictionaryKey: "CFBundleVersion") as? String ?? "—"
    }
}
```

**② 构建号必须是「唯一且递增」的**

App Store Connect 拒绝重复的 `(bundle id, CFBundleVersion)`。
本地开发"每次都填 1"在第一次上传后就会失效。用 `agvtool` 或 CI 注入：

```bash
agvtool new-version -all "$(date +%Y%m%d%H%M)"   # 单调递增，无需人肉记
```

**③ `CFBundleVersion` 只允许数字和点**

`1.0.0-beta` 这种会直接被拒，`[通用]` 但每年都有人中招。语义版本留给 `MARKETING_VERSION`。

**④ 三处一致性核对**

| 位置 | 怎么查 |
|---|---|
| 源码 tag | `git describe --tags` |
| 打包产物 | `plutil -p DeviceStatus.app/Info.plist \| grep -E "Version"` |
| 线上 | TestFlight / App Store Connect 显示的版本 |

**⑤ 比较版本用 `.numeric`**（见第一章的实测），不要 `a < b`。

### 什么时候跳 MAJOR

判据是**用户感知**，不是 diff 大小：

| 变了什么 | 跳什么 | 本机实例 |
|---|---|---|
| 运行形态变了（可安装 / 独立窗口 / watch-only） | MAJOR | DeviceStatus `d13cab7` 把 watch 改成独立 App（还没发版，但正是这类改动） |
| 数据格式不兼容 | MAJOR | — |
| 加功能（向后兼容） | MINOR | — |
| 修 bug | PATCH | — |

### 一条纪律

**功能没测通之前不动版本号。** 顺序固定：改代码 → 跑测试 → 全绿 → bump。
反过来的结果是 changelog 里全是补丁，回看时分不清哪一版是功能、哪一版是在修上一版引入的问题。

---

## 三、日期与时区：`timeZone = nil` 不会报错

### 现象

> SwiftUtils `ece7fda`「修复日期时区、isYesterday 缺陷」

同一个函数，在开发机上输出正确，换台机器或换个时区就偏几小时甚至一天。
更隐蔽的是 `isYesterday` 把方向写反了：本该 `-1`，写成了 `+1`。编译通过、测试没覆盖时完全看不出来。

### 根因（实测，不是推测）

```swift
TimeZone(abbreviation: "Asia/Shanghai")   // nil  ← 传了 IANA 标识符给「缩写」API
TimeZone(identifier: "Asia/Shanghai")     // 正常
```

把 `nil` 赋给 `formatter.timeZone`，`DateFormatter` 不报错，**静默回落到设备当前时区**。
代码看起来"设了时区"，实际没设。

还有一个更阴的：

```swift
TimeZone(abbreviation: "CST")   // America/Chicago ← 不是中国标准时间
```

缩写有歧义，Foundation 按自己的表解析。**能用 `identifier` 就别用 `abbreviation`。**

### 做法

**① 时区一律 `TimeZone(identifier:)`，日期格式化三件套显式写全**

```swift
let f = DateFormatter()
f.locale = Locale(identifier: "en_US_POSIX")   // 否则 12 小时制 / 佛历 / 和历 会改格式
f.calendar = Calendar(identifier: .gregorian)
f.timeZone = TimeZone(identifier: "Asia/Shanghai")
f.dateFormat = "yyyy-MM-dd"
```

**② 存 `Date`（绝对时刻），不存"看起来像日期"的字符串**

存字符串的唯一理由是要和外部系统交换（API / 备份文件），那种情况用 ISO 8601 并显式带时区。

**③ `Date` 本身没有时区**

它只是时间轴上的一个点。**"今天/昨天"是日历概念，必须经过 `Calendar`**：

```swift
Calendar.current.isDateInToday(date)        // ✅
fmt.string(from: date) == fmt.string(from: .now)   // ❌ 差一秒就错
```

`Calendar.current` 的时区、周起始日都跟用户设置走。要跨地区一致，就显式给 `Calendar`。

**④ `DateFormatter` 不是 `Sendable`**

Swift 6 下把它做成全局可变单例、或在 `Task` 之间传递，会报并发错误。
要么每次新建（便宜到可以忽略），要么放进 `@MainActor` 的上下文里复用。

**⑤ 测试不要依赖机器设置**

```swift
@Test("固定时区后输出可断言")
func formattingIsStableAcrossZones() {
    let f = DateFormatter()
    f.locale = Locale(identifier: "en_US_POSIX")
    f.timeZone = TimeZone(identifier: "Asia/Shanghai")!
    f.dateFormat = "yyyy-MM-dd HH:mm"
    #expect(f.string(from: Date(timeIntervalSince1970: 0)) == "1970-01-01 08:00")
}
```

注意这里**不依赖**全局状态：`NSTimeZone.default`、`Locale.current` 都是进程级的，
在测试里改它们（或者依赖它们）会被同进程的其他用例污染，并行执行时表现为"随机红"。`[通用]`
确实需要换语言环境时，用启动参数（`-AppleLocale zh_CN -AppleLanguages "(zh-Hans)"`），别在测试里改全局。

### 验证方式

和纯前端那份文档一样：**遍历所有展示点，断言不出现原始格式**。
SwiftUI 里可以给格式化函数加参数化测试，把所有形态过一遍：

```swift
@Test(arguments: [Date(timeIntervalSince1970: 0), .now, .distantPast])
func formattedDateNeverLeaksISO(_ date: Date) {
    let text = DateDisplay.string(date)
    #expect(!text.contains("T00:00:00Z"))
}
```

---

## 四、并发：编译通过，运行时 `trap`

这是 Swift 6 最独特、也最容易被"编译通过"骗过去的一条。

### 现象

> [DeviceStatus] README §2：`SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor` + 回调式 API

```
dispatch_assert_queue_fail
  ← _swift_task_checkIsolatedSwift
  ← closure #1 in DeviceInfoProvider.networkTypeLabel()
  ← NetworkUtil.swift:46   (com.apple.root.background-qos)
```

`NWPathMonitor` 在后台队列回调，闭包却被推断成 `@MainActor`（因为整个 App 默认 MainActor 隔离），
运行时隔离检查直接 trap。**不是编译错误，编译器不会提醒。**

### 为什么这条比别的语言狠

其他生态里"在非主线程碰 UI"通常是警告或未定义行为；Swift 6 是**确定性的崩溃**，
但只在真的走到那条路径时崩。于是：

- 单测跑 `snapshot()` 不崩（不碰网络回调）
- 界面手动点一下可能崩（网络状态变化时）
- 崩溃日志里只有一个 `dispatch_assert_queue_fail`，没有业务线索

### 做法

**① 根治在库侧：回调参数标 `@escaping @Sendable`**

```swift
public func getNetworkType(
    callback: @escaping @Sendable (NetworkType?) -> Void,
    fail: @escaping @Sendable (Error) -> Void
)
```

调用方就不会再隐式继承 actor 隔离，同时把"回调在哪个队列"变成类型上可见的约定。

**② 过渡期在调用点标 `nonisolated`**

```swift
private nonisolated static func networkTypeLabel() async -> String { ... }
```

[DeviceStatus] 当前就是这么绕的，注释里写明了根因。

**③ 新代码用 `async` API，不要用回调**

回调 API 在 Swift 6 下要求调用方理解隔离继承规则；`async` 函数天然带 `Sendable` 边界。

**④ 打开严格并发，把运行时的坑前移到编译期**

```
SWIFT_VERSION = 6.0
SWIFT_STRICT_CONCURRENCY = complete
SWIFT_DEFAULT_ACTOR_ISOLATION = MainActor
```

**⑤ 限流这类"闸门"，用 `actor` 而不是 `DispatchQueue` + 时间戳**

纯前端那份文档里踩的是"并发读到同一个旧时间戳，闸门形同虚设"。Swift 里同样：

```swift
// ❌ 两个调用方可能同时通过：都读到了旧的 next
actor BadGate {
    private var next = ContinuousClock.now
    func pass(interval: Duration) async throws {
        if ContinuousClock.now < next { try await Task.sleep(until: next, clock: .continuous) }
        next = ContinuousClock.now   // ← 睡眠期间别人已经进来了
    }
}

// ✅ 先占位，再等待；占位是同步的，actor 保证串行
actor RateLimiter {
    private let gap: Duration
    private var next: ContinuousClock.Instant

    init(gap: Duration) { self.gap = gap; self.next = .now }

    func waitTurn() async throws {
        let now = ContinuousClock.now
        let start = max(now, next)
        next = start.advanced(by: gap)      // 先在 actor 内把位置占住
        if start > now {
            try await Task.sleep(until: start, clock: .continuous)
        }
    }
}
```

关键不是 `actor` 关键字，而是**「占位」必须发生在 `await` 之前**。
`await` 会让出 actor，任何在 `await` 之后才改状态的设计都会漏。

### 验证方式

- CI 里至少跑一次"会走回调路径"的集成测试（[DeviceStatus] 的 `SwiftUtilsIntegrationTests` 就是干这个的）。
- 崩溃上报里按 `dispatch_assert_queue_fail` 建告警。
- 模拟器上手动切飞行模式，是这类 bug 成本最低的复现手段。

---

## 五、视图层：`body` 里不要写副作用

纯前端那份文档的"渲染函数之间禁止互调"，在 SwiftUI 里的等价物是：
**`body` 里改会触发 `body` 重算的状态**。

### 症状

- `AttributeGraph: cycle detected`（循环依赖）
- 控制台刷 `Publishing changes from within view updates is not allowed`
- 界面抖一下、列表整片重建、CPU 跑满

### 规则：单向、单一入口

```
数据层（Service / Store） → 计算层（filter / sort / 格式化） → 视图层（只读）
```

- 视图只读数据、只写自己的临时状态
- **视图之间不互调**：A 需要 B 的数据，让它们都读同一个 Store，而不是让 A 调 B
- 联动更新走单一入口

[DeviceStatus] 的正面例子（`ContentView`）：

```swift
private func refreshLocalGroups() {
    groups = DeviceInfoProvider.snapshot(displayScale: displayScale)
}

private func reload() async {
    refreshLocalGroups()                            // 本地部分
    networkGroup = await DeviceInfoProvider.loadNetworkGroup()   // 异步部分单独
    hasLoaded = true
}
```

`refreshLocalGroups()` 被 `.task` 和 `.onChange(of: colorScheme)` 两处复用，
但它**只做一件事**：重采本地分组。网络分组由 `reload()` 单独负责。
所以"切深浅色"不会顺带再开一次 `NWPathMonitor`。

**判断标准**：删掉任意一个 Section，其余代码照常编译、照常工作。

### 几个具体的坑

| 写法 | 后果 |
|---|---|
| `@State` 用外部值初始化 | 只生效一次，外部再变不刷新（用 `.id()` 或 `@Binding`） |
| `@ObservedObject` 在 View 里创建 | 每次重建都新建对象，状态丢（用 `@StateObject`） |
| `ForEach` 用 `\.self` 且元素不是稳定 id | 列表身份错乱、动画乱飞 |
| `.onAppear` 里做重活 | 视图复用时重复执行（用 `.task`，它会随视图消失而取消） |
| `onChange` 里改触发它的那个值 | 无限循环，且 `AttributeGraph` 不一定报错 |

---

## 六、能力探测：`#available` 通过 ≠ 设备具备

纯前端那份文档讲的是 `BarcodeDetector` 存在但没摄像头。Swift 里同样有三层，缺一层就说谎。

| 层 | 判什么 | 怎么写 | 判错的后果 |
|---|---|---|---|
| 编译期 | 这个 SDK 有没有 API | `#available(iOS 17, *)` / `#if os(watchOS)` | 用了就崩，或者代码走不到 |
| 设备 | 这台机器有没有硬件 | `AVCaptureDevice.default(.builtInWideAngleCamera, for: .video, position: .back)` | 显示一个必然失败的按钮 |
| 权限 | 用户给不给 | `AVCaptureDevice.authorizationStatus(for:)` + 真调一次 | 报错但分不清是权限还是硬件 |

三条都要过，**但降级理由要写对**，不能三合一成"不支持"。

### 本仓库的正面例子：把平台差异收口成一个类型

```swift
enum SystemClipboard {
    static var isSupported: Bool {
        #if os(watchOS)
        return false
        #else
        return true
        #endif
    }
}
```

好处不是"省了几行 `#if`"，而是**判定只存在一处**。
将来加平台、改判定，只改这里，调用方一行不动。

### 反过来：失败值必须文档化，否则就是"说谎"

[DeviceStatus] 的集成测试里有两条，看着不起眼，其实是这一章的核心：

```swift
// -1 表示当前环境读不到（模拟器、无电池的 Mac 台式机），是文档化的失败值
#expect(level == -1 || (0...100).contains(level))

// macOS 与 watchOS 没有可上架的公开亮度 API，约定返回 0
#expect(brightness == 0)
```

**API 存在、返回 0，不代表 0 是事实。**
把 `0%` 电量、`0` 亮度当成真实值展示给用户，和"宣称用户没有摄像头"是同一类错误：
**用间接证据下了结论**。

规则：凡是"读不到"的情况，要么返回 `Optional`，要么返回一个**有文档、测试钉住**的哨兵值，
并且展示层必须能区分"0"和"未知"。

### watchOS 的 API 缺口：没有替代品时，只能不挂

> [DeviceStatus] 实测：`contextMenu(menuItems:)` 从 watchOS 7 起整体 deprecated，
> SDK 里带 `preview:` 的重载在 watchOS 上 `unavailable`。

也就是说，**没有"迁移到新 API"这条路径**。正确做法是判断"这个功能在手表上有没有意义"：

```swift
#if !os(watchOS)
.contextMenu { Button("复制") { copy(item) } }
#endif
```

手表上没有剪贴板，菜单本来也是空的——删掉是净化，不是妥协。
同类情况还有 SwiftUtils `50457d1`「ViewEx.showShareTextView 新增对 watchOS 支持」：
库得单独为 watchOS 补一条实现，而不是假设"iOS 能用，watch 也能用"。

### 测试：四种组合，一个都不能少

| 组合 | 期望 |
|---|---|
| 有 API + 有设备 | 走原生实现 |
| 无 API + 有设备 | **仍可用**（走自研/降级实现） |
| 有 API + 无设备 | 不显示入口，文案改成"换一条路" |
| 无 API + 无设备 | 同上 |

只测第一种，就会漏掉"用户明明有摄像头却被判成扫不了"这种最伤人的 bug。
`[通用]` 实现上把探测抽成协议，测试注入四种 stub：

```swift
protocol CameraProbing { var hasCamera: Bool { get } }
```

---

## 七、无后端直连第三方 API

前端那份文档的四步（CORS → 预检 → 限流 → 降级），在原生 App 里的对应物：

### ① CORS 换成 ATS

浏览器拦的是跨域，App 拦的是 `App Transport Security`：默认只允许 HTTPS，
明文 HTTP 要开例外并在审核时解释。**别为了联调方便长期关 ATS**，那是给自己埋上架风险。

### ② 密钥不能硬编码

二进制里 `strings` 一下就能捞出来。放 Keychain（至少不随二进制被拷走），或者放自建代理。
`[通用]` 但属于"一旦被发现就是事故"的级别。

### ③ 限流闸门用 actor（见第四章），并且**验证实际间隔**

不是"我写了 sleep"，而是"记录每次请求的时间戳，断言相邻间隔 ≥ 阈值"。

### ④ 付费接口默认关闭

设置页写清单价、什么情况不计费；开关打开但没配 key 时要提示，不能"看起来开了其实没生效"。

### ⑤ 失败要有降级，而且降级要如实标注

```
联网查证失败 → 回退本地估算，并标「估算」而不是「查证」
图片下载失败 → 回退占位图 + 可重试入口
```

### 几个 URLSession 的静默坑 `[通用]`

| 坑 | 后果 | 做法 |
|---|---|---|
| `URLSession.shared` 默认缓存策略 | 拿回旧响应 | 需要新数据时 `.reloadIgnoringLocalCacheData` |
| 只 `try?` 不检查 `HTTPURLResponse.statusCode` | 4xx/5xx 被当成成功 | 显式判 `200..<300` |
| `Task.cancel()` 不取消 URLSession | 白跑一次请求 | `withTaskCancellationHandler` 里 `task.cancel()` |
| 回调在后台队列 | Swift 6 下可能 trap（见第四章） | 用 `async` API |
| 无超时 / 无限重试 | 用户等到放弃 | `timeoutIntervalForRequest` + 退避重试上限 |

### 依赖越少越好

> [SwiftUtils] `d267e0d`「HttpUtil: 新增纯 Foundation 实现并为剥离 Alamofire 做准备」

这是正确的方向：一个网络工具库为了 `get/post` 拖进整个 Alamofire，
在"这个 App 只是要看几个数字"的场景里是纯负债。能自己写 20 行 `URLSession` 就别引库。

---

## 八、多平台：单 target 的边界，比你想的硬

### 现象

> [DeviceStatus] `d13cab7`「改成手表上是独立 App」

装不上手表，报 `Missing WKCompanionAppBundleIdentifier key`。

### 根因：一个 target 只能产出一个 bundle id

> 系统要求 companion 的 bundle id 必须是 `<父 id>.xxx`。
> 单 target 多平台（Supported Destinations）产出的所有平台产物都是同一个 bundle id，
> **所以做不出 companion watch app，只能做 watch-only。**

实测过两条路：

| 方案 | 结果 |
|---|---|
| `WKWatchOnly = YES` | ✅ 安装、启动都正常 |
| `WKCompanionAppBundleIdentifier = me.zzhhoo.DeviceStatus`（自引用） | ❌ 被 Apple 明确拒绝 |

**产品后果**：手表上是独立 App，只能看手表自己的信息，拿不到配对 iPhone 的电量、存储。
要"在手表上看手机状态"，必须拆独立 watchOS target（`.watchkitapp` 后缀、`TARGETED_DEVICE_FAMILY = 4`、
`SKIP_INSTALL = YES`、`WKCompanionAppBundleIdentifier` 指向父 id、再嵌进 iOS App），
然后用 WatchConnectivity 同步数据。**这个决策要在做界面之前定**，不是上架前才发现。

### Info.plist 的键要按 SDK 限定作用域

```
"INFOPLIST_KEY_WKApplication[sdk=watchos*]" = YES;
"INFOPLIST_KEY_WKWatchOnly[sdk=watchos*]" = YES;
```

`[sdk=...]` 不是可选装饰：不加的话这些键会污染 iOS / macOS 的 Info.plist，
表现为"iOS 包被识别成 watch App"这类诡异问题。

### watchOS 的现实约束

| 约束 | 影响 |
|---|---|
| 没有剪贴板 | 复制功能要收口（[DeviceStatus] 的 `SystemClipboard`） |
| 后台执行窗口短 | 别指望像 iOS 那样长跑任务；`WKApplicationRefreshBackgroundTask` 是尽力而为 |
| 屏幕小、Always-On | 纯黑更省电；内容密度要降；颜色语义和 iPhone 不同 |
| 独立运行 | 没有配对的 iPhone 也能启动，所有数据必须自给自足 |

### 离线与生命周期

原生 App 不像 PWA 要自己写 Service Worker，但"离线"换了个面孔：

- **首启断网**：本地数据能读，但远程配置 / 模板 / 图片下载失败 → 必须有兜底，不能白屏。
- **切回前台**：`scenePhase == .active` 时重采一次数据（[DeviceStatus] 用它重采本地分组）。
- **后台被杀**：迁移、清理这类工作不能只挂在"下次启动"，要能中断后续跑。
- **`BGTaskScheduler` 不保证执行**：它是"有机会就跑"，不是定时器。

验证方式：**真机开飞行模式冷启动**，Release 配置跑一遍。
Debug 和 Release 的优化、断言、内存行为都不同，很多问题只在 Release 出现。

---

## 九、UI 与可访问性

### 尺寸与适配

| 项 | 要求 | 原因 |
|---|---|---|
| 点击区 | iOS ≥ 44×44pt | 手指精度；用 `.contentShape` + `.frame(minWidth:minHeight:)` 补齐 |
| 字号 | 用语义字号（`.body` / `.headline`） | Dynamic Type 自动缩放；写死 `size:` 只在确有设计约束时用 |
| 安全区 | `safeAreaInset` / 让系统处理 | `ignoresSafeArea` 慎用，watchOS 的圆角和曲面会吃掉内容 |
| 窄屏 | `ViewThatFits` / `Layout` 协议 | `HStack` 不会自动换行，装不下就挤压或截断 |

`[通用]` 但可以直接借用前端那份文档的方法论：**窄屏溢出要用脚本量，不靠肉眼**。
SwiftUI 里对应的是多设备 Preview + 快照测试，重点看 `360pt` 和 `393pt` 两档。

### 深色模式：语义色 + 触发器分离

[DeviceStatus] 的写法值得抄：颜色和判断都交给系统/库，视图只用 `colorScheme` 当**刷新触发器**。

```swift
.onChange(of: colorScheme) { refreshLocalGroups() }
```

判断值来自 `AppUtil().isDarkColorMode()`，视图只负责"外观变了 → 重采一次"。
**如果视图自己算颜色，测试就永远测不到真实的取值。**

### 图标：程序化生成，一套代码出全套

[DeviceStatus] 的 `Scripts/generate-appicons.py` 就是这个思路：
1024 母版 + 参数化缩放 + 内容收在安全区内，一次生成三平台需要的所有尺寸。

三个坑：

1. **maskable 要内缩**，否则被圆形/方形裁掉边缘（前端那份文档里的同一课）。
2. **中文字体不一定有**，`ImageFont.truetype` 失败要 `try/except` 降级成纯图形，不能让整个脚本挂掉。
3. **一套图多平台有代价**：三平台共用 asset catalog 时，macOS 的 `actool` 不认识 watch 专属的
   `watch-marketing` slot，会报 `unassigned child` 警告（[DeviceStatus] 已知无害警告）。
   拆成平台各自的 catalog 能消掉，但要按 SDK 配 `ASSETCATALOG_COMPILER_APPICON_NAME`，收益不大。

另外一条 [DeviceStatus] README 记录的经验：Xcode 27 的 Icon Composer `.icon` 目前**不被 actool 当 app icon set**，
换用时要同时调整构建设置，别以为拖进去就完事。

### emoji 是内容，不是图标

前端那份文档说"界面图标用 SVG，玩法符号可以用 emoji"。Swift 里对应：
**界面图标用 SF Symbols**（随系统版本更新、自动适配字号/粗细/深浅色），
emoji 只用在"它本身就是内容"的地方（水果、动物这类玩法符号）。
watchOS 上 emoji 的字形和 iPhone 差异尤其明显。

---

## 十、依赖与工程结构

### SPM 依赖就是"外链"

前端那份文档说"任何 CDN 都是单点故障"，Swift 里对应的是"任何第三方库都是供应链"。

- `Package.resolved` **必须提交**（[DeviceStatus] 已提交）。不提交 = 每个人解析到的版本可能不同。
- 版本规则要读懂：`upToNextMinorVersion` 意味着 `0.0.25` 可以自动升到 `0.0.26`，但不会到 `0.1.0`。
- **升级要读 changelog**。[DeviceStatus] 从 SwiftUtils `0.0.24` → `0.0.25` 升的就是 macOS 深色模式判定，
  升完那一行从"临时用 `colorScheme` 顶替"改回"走库"。

### 宿主工程要真的测到库

[DeviceStatus] 作为 SwiftUtils 的宿主，测试分工写得很清楚（`SwiftUtilsIntegrationTests` 开头注释）：

> 库自身的细粒度单元测试在 SwiftUtils 仓库里，这里不重复实现细节，
> 只钉住「在这个宿主 App 的三个平台上，公开 API 必须满足的可观测约定」。

这是正确姿势：**库仓库测逻辑，宿主工程测集成**。
只有库测试，测不到"链接进三平台 App 之后"的条件编译分支；只有宿主测试，又会重复。

### 库的坑会变成 App 的坑

> [DeviceStatus] README §3：`AppUtil.isDarkColorMode()` 的 macOS 分支曾硬编码返回 `false`，
> 0.0.25 才修好；修好之后仍然**没有单元测试**，而且新实现用 `NSApp?` 可选链——
> `NSApp` 为 nil（非 GUI 进程）时静默返回 `false`，看起来仍像"永远是浅色"。

三条教训：

1. **"能调用"不等于"行为对"**。集成工程适合钉跨平台契约，但钉不住语义是否正确。
2. **可选链是静默失败的温床**。`NSApp?.x` 编译通过，返回 nil 时逻辑继续走。
3. **返回值要有"我不知道"这个状态**，而不是用 `false` / `0` / `""` 冒充。

### 单 target vs 多 target

| | 单 target 多平台 | 多 target |
|---|---|---|
| 配置 | 一份设置，`[sdk=...]` 限定差异 | 每个 target 各一份，容易漂 |
| watch | 只能 watch-only | 可做 companion，能拿手机数据 |
| 扩展 | 不好做 | 天然 |
| 适合 | 信息展示类、工具类 | 需要扩展 / 手表联动 / 独立 bundle id |

判断依据：**"这个东西还要长多久，以及要不要和别的设备互通"**。

---

## 十一、什么时候自己写（对应前端那份的"自研解码器"）

三个条件同时成立才动手：

1. 系统能力缺失，且这件事在核心路径上；
2. 有可靠的验证手段，能把"对不对"变成自动化断言；
3. 代码量可控。

本机的两个例子：

| 自己写的东西 | 为什么 | 验证 |
|---|---|---|
| 构建号读取（`sysctlbyname("kern.osversion")`） | 系统没有公开 API 拿 `kern.osversion` | 与 `sw_vers -buildVersion` 比对 |
| 占位图标生成（Python + Pillow） | 没有"生成占位图"这种系统能力，尺寸又多，手工必漏 | 生成后 `actool` 能过、三平台都能构建 |

反例也说清楚：二维码这类事情**不要自己写**。CoreImage 有 `CIQRCodeGenerator`，Vision 有
`VNDetectBarcodesRequest`，`[通用]` 自己实现编解码器只在"系统方案在你的目标平台上不可用"时才值得
（SwiftUtils 里那次 `cc0a4fe 修复 NSImage 二维码太模糊` 属于渲染细节，不是重写编解码）。

自研的代码边界要写清楚，[DeviceStatus] 里 `sysctl` 那段就是典型：

```swift
var buffer = [UInt8](repeating: 0, count: size)
guard sysctlbyname("kern.osversion", &buffer, &size, nil, 0) == 0 else { return "未知" }

// sysctl 给的是以 NUL 结尾的 C 字符串：先截掉 NUL，再按 UTF-8 解码。
// 不走 String(cString:)——它对 [CChar] 已在新 SDK 标记 deprecated。
let build = String(decoding: buffer.prefix { $0 != 0 }, as: UTF8.self)
```

这段同时体现了两条：

- **C 接口的边界要自己封**（长度、NUL、编码、失败值）；
- **SDK 的弃用要跟**（`String(cString:)` 这条就是编译器给的警告，改了就没噪音了）。

---

## 十二、测试：绿了不代表对

### 本仓库在用的：Swift Testing

```swift
@Suite("设备信息采集")
struct DeviceInfoProviderTests {
    @Test("分组齐全、每个条目都有标题和值")
    func snapshotIsWellFormed() { ... }

    @Test("包含预期的分组")
    func snapshotHasExpectedGroups() {
        let titles = makeSnapshot().map(\.title)
        for expected in ["系统", "硬件", "存储", "电池", "屏幕", "应用"] {
            #expect(titles.contains(expected), "缺少分组「\(expected)」")
        }
    }
}
```

`@Test(arguments:)` 还能把上面那个循环写成参数化用例（同一个用例跑六遍，失败时能看出是哪一组）。

比 XCTest 好的地方：`#expect` 失败信息自带表达式、参数化测试、`@Suite(.serialized)` 控制并发。
[DeviceStatus] 现在 2 个 suite / 17 个用例。

### 五种「假绿」

**① 只断言"存在"**

```swift
#expect(!groups.isEmpty)   // 对应前端那份文档里的 caches.keys() 断言，同一个病
```

"有分组"不代表"每个分组都有内容、标题不重复、值不是空串"。
这类断言要往上加：结构、唯一性、取值域。

**② 依赖运行环境的值**

```swift
// 深色模式的值取决于系统外观，测不了固定结果
#if os(watchOS)
#expect(value == "是")     // 钉住可确定的契约
#else
#expect(value == "是" || value == "否")   // 只钉形状
#endif
```

[DeviceStatus] 的处理是对的：**能钉的钉死，钉不住的写清楚为什么钉不住**，
并在注释里指明"其余平台由人工切换外观验证"。假装能测，比不测更糟。

**③ 测试和生产共享同一个错误假设**

纯前端那份文档里，透视测试写成了双线性映射，改了两轮算法都没用，最后发现是尺子歪了。
Swift 里常见的是：mock 的语义和真实 API 不一致（mock 返回 `nil` 表示失败，真实 API 返回空数组表示失败）。

**④ 异步没真等**

```swift
func testLoads() async {
    Task { await store.load() }     // ❌ 没等，断言跑在加载之前
    #expect(store.items.count == 3)
}
```

**⑤ UI 测试里"存在"≠"能点"**

`exists` 为真但 `isHittable` 为假（被遮挡、在屏幕外、`opacity: 0`），
测试照样绿，用户照样点不到。

### 每类 bug 写「能复现它」的测试

| bug 类型 | 怎么测 | 本机对应 |
|---|---|---|
| 重启丢数据 | 写入 → 重建 store → 断言 | zoclip `ccfaaa4` |
| 迁移失败 | 构造老结构 → 迁移 → 断言新结构 | [通用] |
| 时区偏移 | 注入固定 `TimeZone` 断言输出 | SwiftUtils `ece7fda` |
| 版本号漂移 | 读 Info.plist 断言与配置一致 | SafaQrcode `bd82375` |
| 能力误判 | 注入 stub 跑四种组合 | [通用] |
| 并发隔离 | 集成测试走真实回调路径 | [DeviceStatus] README §2 |

### 编译期能拦的，别留给测试

```
SWIFT_VERSION = 6.0
SWIFT_STRICT_CONCURRENCY = complete
SWIFT_TREAT_WARNINGS_AS_ERRORS = YES
```

[DeviceStatus] 还没开 `-warnings-as-errors`，但已经吃了两次弃用警告的亏
（`String(cString:)`、`contextMenu`）。开了以后这类问题在 CI 就红，不用等到某次翻代码才发现。

---

## 十三、发布与上架

### 一条命令写齐，然后回读校验

对应前端那份的 `node tools/stamp.js`：

```bash
xcodebuild -project DeviceStatus.xcodeproj -scheme DeviceStatus \
  -configuration Release -destination 'generic/platform=iOS' \
  -archivePath build/DeviceStatus.xcarchive archive \
  CURRENT_PROJECT_VERSION="$BUILD" MARKETING_VERSION="$VER"
```

**校验的不是"我传了参数"，而是"产物里真的是这个值"**：

```bash
plutil -p build/DeviceStatus.xcarchive/Products/Applications/DeviceStatus.app/Info.plist \
  | grep -E "CFBundleShortVersionString|CFBundleVersion"
```

这和前端那份文档的"校验语义，不是字符串"是同一条：
`grep` 之前先确认那一行长什么样，别假设。

### 上架前必须人工处理的（[DeviceStatus] 清单）

| 项 | 说明 |
|---|---|
| 签名 | 本机只有 `Apple Development`；上传需要 Distribution 证书，Xcode 自动管理即可 |
| App Store Connect | 建 app 记录、三平台截图、隐私政策 URL、App 隐私标签 |
| 隐私清单 | `PrivacyInfo.xcprivacy` 的 Required Reason API 理由码必须齐 |
| 本地化 | `knownRegions` 补 `zh-Hans`，显示名用 `InfoPlist.strings` |
| 版本 | `MARKETING_VERSION` / `CURRENT_PROJECT_VERSION` 每次上传唯一递增 |

### 隐私清单：漏了会被机器拦（ITMS-91053）

[DeviceStatus] 声明了四类：

| 类别 | 理由码 | 对应场景 |
|---|---|---|
| `SystemBootTime` | `35F9.1` | 开机时长 |
| `DiskSpace` | `85F4.1` | 存储容量 |
| `FileTimestamp` | `C617.1` | 读时间戳 |
| `UserDefaults` | `CA92.1` | 用户偏好 |

**用到就声明，没用到就删**。声明了不用的 API 不会拦，但会让人怀疑清单是抄的。

### 出口合规与沙盒

- `ITSAppUsesNonExemptEncryption = NO`（只用了系统 HTTPS）能免掉每次上传的合规问答。
- Mac App Store 需要 `ENABLE_APP_SANDBOX` + `ENABLE_HARDENED_RUNTIME`。
- Mac 上架强校验 `LSApplicationCategoryType`。

### TestFlight 冒烟（对应前端那份的"线上冒烟"）

- 版本号是新的
- 关键功能真的能跑（不是"能打开就算"）
- 首启断网不白屏
- 深色 / 浅色都看一眼
- 0 条崩溃、0 条未捕获异常
- 不产生意外的付费调用

---

## 十四、上线前检查清单

```
数据
□ 每个 save 都有对应的 load，且 load 真的被调用
□ 每个字段在 encode / decode 各出现一次
□ 解码失败与「没有数据」分开处理，失败时保留原始文件
□ 迁移用整数格式号，不用字符串版本比较
□ 迁移判据是「无戳即迁移」，不是「版本号大小」
□ 写 → 重建 store → 断言 的测试存在
□ UserDefaults 的 key 已收口成常量
□ 清空数据有二次确认

版本
□ 版本只来自 build setting，代码里没有硬编码
□ 所有 target 的 MARKETING_VERSION / CURRENT_PROJECT_VERSION 一致
□ CFBundleVersion 只含数字和点，且单调递增
□ 版本比较用 compare(options: .numeric)
□ 功能测试全绿之后才 bump

并发
□ 回调式 API 的参数标了 @Sendable，或调用点 nonisolated
□ 严格并发已打开（complete）
□ 限流 / 去重是「先占位再 await」
□ 集成测试覆盖真实回调路径

平台
□ 每个「平台差异」都收口在一个类型里，没散落在 View
□ 编译期 (#available) / 设备 / 权限 三层探测分开
□ 「读不到」有 Optional 或文档化的哨兵值，展示层能区分
□ watchOS 上的功能逐个确认过（剪贴板、后台、屏幕尺寸）
□ Info.plist 的平台专属键用了 [sdk=...] 限定

日期
□ 时区一律 TimeZone(identifier:)
□ DateFormatter 的 locale / calendar / timeZone 三件套显式
□ 存 Date 不存日期字符串
□ 「今天/昨天」走 Calendar，不比字符串
□ 测试不依赖机器时区 / 语言

UI
□ 点击区 ≥ 44pt
□ 语义字号 + Dynamic Type 实测（最大字号）
□ 深色 / 浅色都看过
□ 窄屏（360pt）无截断 / 溢出
□ 图标是 SF Symbols，emoji 只作内容
□ App Icon 三平台齐全，maskable 有内缩

测试
□ 没有只断言 !isEmpty 的用例
□ 依赖环境的值只钉契约，并注明人工验证
□ 异步用例真的等到了
□ UI 测试断言的是 isHittable，不是 exists
□ CI 跑 -warnings-as-errors

发布
□ archive 里的版本号回读校验过
□ 隐私清单与所用 API 对得上
□ TestFlight 冒烟通过（含断网首启）
□ dSYM 已保留
□ 三处版本一致（tag / archive / TestFlight）
```

---

## 最后

这份东西里最值钱的不是"最佳实践"，是那几个**真的静默过一次**的 commit。

它们共同的特征是：**编译通过、运行不崩、日志干净**，只是"用户觉得不好用"。

原生开发里，最容易骗过自己的四个"间接证据"：

1. **`try?` 没报错，不代表成功** —— 它只是把错误变成了 `nil`。
2. **`#available` 通过，不代表设备具备** —— 它只说明系统版本够。
3. **返回 0 / false / ""，不代表事实是 0 / false / ""** —— 失败值没文档化就会冒充数据。
4. **测试绿了，不代表用户路径通了** —— 只测内部状态，等于只证明"装上了"。

四个是同一件事：**把间接证据当成了结论**。

所以每次说「支持」「不支持」「没问题」之前，先问一句：

> 我判的是能力，还是我自己的假设？

---

*本文档 v1.0，2026-10-18。基于 DeviceStatus / SwiftUtils / zoclip / SafaQrcode / KidToys 的实践与 commit 记录整理。*
