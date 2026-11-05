# 全局编码规则

适用于任何工作区、任何语言、任何对话。

## 全局规则维护：发现可复用问题就回写本文件

本文件不只记录既有约定，也要持续吸收新坑。**发现下列任一情况，本轮内就更新本文件**，不要等用户再提，也不要只在对话里说一遍：

1. **工具/流程误解被用户纠正**：Agent 的工具调用方式、参数语义、路径或权限假设出错，用户指出并确认用户说法正确（例如沙箱边界、命令形态、路径解析）。记「错法 → 正确做法 → 为什么」。
2. **依赖/环境报错已解决**：项目常用依赖（pnpm/Node、Python、Gradle、插件、系统命令等）的报错解决后，汇总「报错关键行 → 根因 → 触发条件 → 解决步骤」，让下次能直接照做。
3. **跨项目共性坑**：任何不限于当前项目、其他项目大概率也会撞上的问题（编码/换行、长路径、权限与 ACL、代理与网络、版本锁定、工具 shim 失效等）。
4. **用户确认过的口径**：用户明确认可、且判断会反复用到的做法、命名或流程偏好。

写法要求：

- 写入前先搜本文件有无同主题；有则改那一条，不要堆重复条目。
- 一条一个主题，标题写问题本身；正文 2–6 行：现象 → 根因 → 做法。
- 只留可复用结论；项目专属细节（具体文件名、一次性参数）不要写进来。
- 不写密钥、token、私有 URL、个人隐私。
- 没把握是否普适时先问用户，不要猜测性写入。

不必写：一次性的调试过程、已有技能或项目文档已覆盖的内容、纯个人偏好（用户明确要求除外）。

## 单次调用的方法必须内联

- 方法/函数/helper 若只有一个调用点，就把方法体直接放到调用处，不要保留这层包装。
- 包括 private helper、内部工具函数、只在一个地方用到的分支逻辑。
- 目的：去掉无谓间接层，读调用处就能看到完整逻辑。
- 例外：public/对外 API、接口实现、生命周期回调、框架要求固定签名的方法、已明确会复用的入口、用户要求保留的 stub/占位方法。
- 内联只做结构调整：行为、日志文案、错误处理、回退逻辑保持原样。
- 重构时主动检查“只被调用一次的方法”，一并内联，不要等用户再提。

## Computer Use（桌面/浏览器操作）规则

适用于 cua-driver 一类 GUI 自动化。核心原则：**点击已发出 ≠ 点击已生效**，每次动作都要看结果核验。

### 坐标与点击

- 点击 x/y 的坐标系是「最近一次 `get_window_state` 返回图片的像素空间」，不是屏幕物理像素。窗口尺寸、截图 max_dimension 一变，旧坐标全部作废：先重新取快照，再换算。
- 不要用视觉估算选项的 y。题干行数不同、选项文字换行不同，行高和行距都会变。先截图 → 按行框底色（白/高亮）检测每行中心 → 点行中心 → 再截图核验。
- 点在两行之间的空隙会「静默失败」：不报错、无效果。所以必须点后核验选中态（高亮底色、实心圆、对勾），与预期项一致才算成功。
- 网页内容优先 `foreground` 点击；默认 `background` 走 UIA hit-test 时可能落到覆盖层/隐藏链接上，表现为「跳去别的页面/新开标签页」。出现莫名跳转就是点歪信号，立刻回退并重新定位。
- 每步动作后都验证实际效果（窗口标题、URL、题目序号、选中数量），不要拿「工具返回成功」当成功。

### 核验方式

- 状态核验用像素脚本，比反复贴截图进上下文便宜：截图写文件（不进上下文）→ 脚本按当前缩放系数（sx/sy）定位行与选中色 → 输出「第几项被选中/选中了几项」。
- 缩放系数必须由当前截图尺寸与窗口物理尺寸算出，不要写死固定物理像素，否则窗口一变全错。
- 多选必须核验「数量」和「集合」：数量对但点错项同样算错。多行选项行高不一致时，行检测的高度阈值要放宽。
- 长列表超出视口时（如 11 个多选项），用键盘 Home/End 滚到底再点，page down 在部分页面不生效。滚动后要回头确认前面已选项没被误取消。

### 提交与不可逆动作

- 提交、删除、支付类动作前先读确认框文案，再决定是否继续；用户没明确要求提交时不要自动提交。
- 系统提示「有未作答/未完成」就是前面有动作没生效，逐题回查登记状态，不要重复盲目点提交。
- 网页题目/选项顺序、行高、颜色都会随页面变动，任何「上次是 A 行第 3 个」的经验都不可直接复用，每次重新定位。

### 读题

- 有多模态能力时优先直接读图，不必要不要上 OCR；OCR 只作兜底，且不要用 OCR 文本去推断点击坐标。

## Windows：工作区 = 用户主目录时 pwsh 全部被拒

- 现象：任意 `pwsh` 调用都返回 `Windows ACL temp root must be outside the workspace: workspace=C:\Users\Jerry; temp=...\Temp`，与具体命令无关。
- 根因：工作区被设为整个用户主目录，沙箱要求的临时根落进工作区内，属配置冲突，不是权限不足或命令写错。
- 做法：确需 shell 时，对**同一条命令**单次带 `sandbox_permissions: danger-full-access` 重试（会走审批）；或改用文件工具（read/write/edit/glob/grep）完成。不要反复改命令形态试错。

## npm 禁用 git 依赖（EALLOWGIT）

- 现象：`npx github:owner/repo` 之类安装失败，报 `npm error code EALLOWGIT` / `Fetching packages of type "git" have been disabled`。
- 根因：npm 侧 `allow-git=none`（本机为 `none`），所有 git 协议依赖被策略禁用。
- 做法：`git clone --depth 1 <repo>` → `npm pack <cloned-dir> --pack-destination <dir>` → 用产出的 `.tgz` 安装（profile 依赖写 `file:`，或装器用 `--source <tgz>`）。不要擅自改用户的 npm 策略。

## pnpm / npx shim 指向失效路径

- 现象：`pnpm.cmd` 或 `npx pnpm` 报 `'"...\node_modules\pnpm\pnpm"' is not recognized`，但 `pnpm --version` 之前还能用。
- 根因：shim 记的是 store 里已不存在的链接路径，常见于 pnpm 升级、缓存清理或换版本之后。
- 做法：改用 `corepack pnpm ...`（按 `packageManager` 取版本，必要时联网下载）。项目声明了 `packageManager` 时，corepack 是首选。
- 补充：安装器若报 `pnpm was not found on PATH`，先定位 DSH 自带的 pnpm（`%APPDATA%\DSH Desktop\runtime-commands\generations\*\bin\pnpm.cmd`），但该 shim 常常也是坏的，仍优先 corepack。

## Gradle 在受限文件沙箱下报 native-platform.dll 加载失败

- 现象：`gradle` 启动即失败，报 `Could not initialize native services. > Failed to load native library 'native-platform.dll' for Windows ...`，与构建脚本无关；`-Dorg.gradle.native=false` 无效。
- 根因：Gradle 要往 `GRADLE_USER_HOME`（默认 `~/.gradle`，在工作区外）解压 native 库并建缓存锁；文件沙箱只放行工作区时被拒，但报错文案指向 native 库，容易误判为环境损坏。
- 做法：对**同一条** gradle 命令单次带 `sandbox_permissions: danger-full-access` 重试；或把 `GRADLE_USER_HOME` 指到工作区内可写目录（首次会重新解析依赖，离线环境不可行）。
- 同一命令在工作区策略为 `danger-full-access` 时能正常构建，可据此确认是该原因而非 Gradle 损坏。
## night-config 解析空 YAML 抛 NPE

- 现象：加载配置时报 `ParsingException: YAML parsing failed`，cause 为 `NullPointerException: Cannot invoke "java.util.Map.entrySet()" because "map" is null`，定位在 `YamlParser.parse`。
- 根因：`snakeyaml.load()` 对空文档（0 字节或只有注释）返回 `null`，night-config 未判空就调 `map.entrySet()`。
- 做法：解析前判断文件是否含实际键值（存在非空行且不以 `#` 开头），没有就跳过 `load()`，用默认值，随后 `save()` 会写成合法 YAML。首次 `createNewFile()` 出来的空配置文件必中这一条。
## Ktor：ContentNegotiation 会污染 WebSocket 握手

- 现象：同一个 `HttpClient` 上既装 `ContentNegotiation` 又用 `WebSockets`，建连抛 `Serializer for class 'DefaultClientWebSocketSession' is not found`；看着像漏加 `@Serializable`，但那个类是 Ktor 内部的，补不了。
- 根因：`ContentNegotiation` 的请求拦截以 `Content-Type` 为触发条件；若 `defaultRequest` 里全局设了 `Content-Type: application/json`，WS 升级请求也会被当成待序列化 body 送进 kotlinx.serialization。
- 做法：全程只 `setBody(字符串)` 的客户端不要装 `ContentNegotiation`；HTTP 与 WS 混用且 HTTP 需要类型化序列化时，拆成两个 client，WS 那个只装 `WebSockets`；`defaultRequest` 只放鉴权头，`Content-Type` 按请求设。
- 附带：`setBody(非 @Serializable 数据类)` 在装了 `ContentNegotiation` 的客户端上必抛同款异常，先 `toJson` 成字符串再发。
## Windows：DSH 沙箱下 MSBuild 建 Temp 目录失败，cmake-rs crate 编译挂
- 现象：`cargo check/build` 编译依赖 cmake-rs 的 crate（flate2 zlib-ng、libz-sys 等）时 build script panic：`command did not execute successfully, got: exit code: 1` 或 `0xe0434352`；MSBuild 只打一行版本横幅、0 error，或报 `MSB1025 ... 对路径 %TEMP%\MSBuildTemp... 的访问被拒绝`。
- 根因：MSBuild 启动要在 `%TEMP%` 建 `MSBuildTemp<guid>`，DSH 文件沙箱下该创建被拒且错误常不显示，看起来像编译器/VS 损坏。
- 做法：同一条命令带 `danger-full-access`，并把 `$env:TEMP`、`$env:TMP`、`$env:TMPDIR` 指到工作区内先建好的目录再跑 cargo（只放行不改 TEMP 仍可能间歇失败）。不要去改 CMake/VS 配置。
