---
name: flutter-upgrade
description: Upgrade a Flutter project's SDK version (via FVM) and its pub dependencies, generically — covering .fvmrc switching, environment/Dart constraint bumps, which packages block a major-version upgrade (compatibility probing), API usage changes after upgrade, dart fix, and end-to-end verification. Use when the user asks to upgrade Flutter, bump the SDK, upgrade plugins/dependencies, migrate to a newer Dart version, fix analyze errors after an upgrade, or resolve `pub upgrade` version-solving conflicts, in any Flutter/Dart project.
---

# Flutter 升级（通用，FVM 管理）

适用于任意 Flutter app / plugin / package。所有 Flutter/Dart 命令必须走 `fvm` —— 裸 `flutter`/`dart` 会用系统 SDK，版本可能完全不同。

**升级分两条独立轨道，一次只做一条：**

| 轨道 | 触发条件 | 风险 |
|---|---|---|
| **A. SDK 升级**（FVM） | 明确要换 Flutter 版本 | 中：Gradle/Kotlin、`dart fix`、平台目录、语言模式迁移 |
| **B. 依赖升级**（pubspec） | 要做插件升级，或 A 之后需要跟进 | 高：大版本破坏性变更、API 改名、基础包版本天花板 |

两条轨道混做，出问题时无法归因。**做完一条、验证通过，再做下一条。**

---

## 0. 前置勘察（每个项目先跑）

```bash
pwd && ls pubspec.yaml
cat .fvmrc 2>/dev/null || echo "NO .fvmrc"     # 无输出 = 该项目未用 FVM 固定版本
fvm flutter --version                          # 当前实际 SDK
git status --short                             # 必须干净
```

判定项目形态，决定后续走哪条路：

| 现象 | 含义 | 处理 |
|---|---|---|
| 无 `.fvmrc` | 未固定 FVM 版本 | 先 `fvm use <version>` 固定，再做升级 |
| `.fvmrc` 版本是 `custom_*_ohos_*` | OHOS 定制 SDK | **不可走标准升级**，见 A1 |
| `pubspec.yaml` 只有 `dependencies:` 且无 `flutter:` SDK 依赖 | 纯 Dart 包，不是 Flutter 项目 | 用 `fvm dart` 而非 `fvm flutter` |
| 有 `melos.yaml` | monorepo | 需在 workspace 根执行，见 A7 |
| `environment.sdk: ">=2.x <3.0.0"` | Dart 2 项目 | 先做**空安全/lang 版本迁移**，见 A5 |

> ⚠️ **无 `.fvmrc` 时 `fvm flutter` 不会报错，而是静默回退到全局默认版本。**
> 同一台机器上，带 `.fvmrc` 的项目和没有 `.fvmrc` 的项目会解析出**不同**的 SDK 版本，且不报错。
> **这会让升级验证得出错误结论**——先在每个项目里 `fvm use <version>` 固定版本。

> ⚠️ FVM 版本首次使用会自动下载，可能超过 2 分钟，别当成卡死。

---

## 轨道 A：SDK 升级

### A1. 切换版本

```bash
fvm list                       # 已安装 / 可切换版本
fvm install <version>          # 未安装时
fvm use <version>              # 写入 .fvmrc 并重建 .fvm/flutter_sdk 软链
fvm flutter --version          # 必须显示新版本，否则后续全是错的
```

**`.fvmrc` 是版本来源**（不是旧的 `.fvm/fvm_config.json`；老项目可能还残留后者，`fvm use` 会以 `.fvmrc` 为准）。

**OHOS 定制版本**（`.fvmrc` 形如 `custom_3.27.4_ohos_dev`）：这类是华为鸿蒙定制分支，不能 `fvm install` 拉取。升级必须换用对应的新 OHOS 分支，且 `ohos/` 目录、`oh-package.json5` 等需同步，**不要按标准流程处理**。

**IDE SDK 路径**：`fvm use` 后 IDE 通过 `.fvm/flutter_sdk` 软链取 SDK。若 IDE 起不来，重新 `fvm use <version>` 即可。切勿直接删 `~/fvm/versions/<version>`，会让 IDE 的 Dart SDK 路径断掉。

### A2. 修订 `environment` 约束

**按顺序判断，停在第一个成立的：**

1. `fvm flutter pub get` 能过 → **不动**（宽区间兼容性更好）。
2. 过不了 → 看报错指向哪个固定版本，才收紧下界。
3. 用户明确要求 → 改成 `sdk: ^<当前Dart版本>` + `flutter: ^<当前Flutter版本>`。
4. 仅当要发 pub.dev 包时 → 下界应设为“实际用到的新特性所需的最小版本”（此时可用 [Pana](https://pub.dev/packages/pana) 的 `--dependency-overrides` 思路验证）。

**禁止**把宽区间顺手改成 caret —— 那是风格改动，不是升级需要，会制造无谓 diff。

### A3. 跨语言版本（Dart 2 → 3，`|` 划分符）

`environment.sdk` 的写法决定语言版本模式：

| 写法 | 含义 |
|---|---|
| `">=2.16.2 <3.0.0"` | Dart 2 模式 |
| `">=2.12.0 <4.0.0"` | Dart 2/3 双支持 |
| `">=3.0.0 <4.0.0"` | Dart 3，**关闭 `|` 划分符启发式** |
| `"^3.11.0"` | Dart 3，启用全部新特性 |

**升级到 Dart 3 时会报 `The language version must be ...` / `| is being used as an or` 之类的解析错误** —— 这是因为 Dart 3 移除了联合类型惯用法。修复方式：

```bash
fvm dart fix --dry-run     # 先预览，会提示类型划分符相关修复
fvm dart fix --apply
```

### A4. 平台目录迁移

```bash
# 看现状，不写
grep -nE 'compileSdk|minSdk|targetSdk' android/app/build.gradle* 2>/dev/null
grep -nE 'kotlin|com.android.application' android/settings.gradle* android/build.gradle* 2>/dev/null
cat android/gradle/wrapper/gradle-wrapper.properties 2>/dev/null
```

**准则：**
- `compileSdk = flutter.compileSdkVersion` 这类引用 `flutter.*` 变量的写法，SDK 升级**通常自动跟随**，不用改。
- 只有当 `flutter build apk` / `flutter build ios` 报 AGP/Kotlin/Gradle 不兼容时，**只改报错指向的那一个版本号**。
- iOS：`ios/Podfile` 的 `platform :ios, 'x.y'` 低于新 SDK 最低要求时才调高。
- Web：`dart:html` 与 wasm 互斥。若依赖了基于 `dart:html` 的包，`build web` 会打印 `WebAssembly incompatibility` 警告，**这是预期噪声**，不要为消除它切 wasm。

### A5. 自动修复

```bash
fvm dart fix --dry-run
fvm dart fix --apply
```

只应用机械化改动。`dart fix` 改不动的（API 语义变更）留给手工，进入 A6。

### A6. 手工修复

```bash
fvm flutter analyze --no-fatal-infos
```

逐条修，**只修报错指向的**。常见升级期错误类型：

| 报错 | 原因 | 处理 |
|---|---|---|
| `The method 'X' isn't defined` | 方法改名/删除 | 查 CHANGELOG，替换新 API |
| `Too many positional arguments` | 参数改 named | 改调用处 |
| `The named parameter 'x' isn't defined` | 参数改名/移除 | 查新签名 |
| `The argument type 'A' can't be assigned to 'B'` | 类型体系变更 | 通常需改类型标注或加转换 |
| `Will never be true` / `unnecessary_non_null_assertion` | 空安全推论变严格 | 删多余 `!` |

### A7. monorepo（`melos.yaml` 存在）

在 **workspace 根**执行，不要在每个包内单独跑：

```bash
melos bs                      # bootstrap，等价于所有包的 pub get
melos exec -- fvm flutter analyze
```

`melos.yaml` 里的 `environment` 块也要同步新 SDK 约束，否则 `melos bs` 会报错。

---

## 轨道 B：依赖升级

### B1. 先做兼容性预判（最关键）

**大版本不一定装得上**，取决于 SDK 钉死了哪些基础包（`meta`、`collection`、`material_color_utilities`、`vector_math` 等）。先探测，别直接改 pubspec。

**探测法 —— 在项目根建临时副本，绝不改真实文件：**

```bash
# 1) 建临时副本
mkdir -p .probe_tmp && cp pubspec.yaml .probe_tmp/
cp pubspec.lock .probe_tmp/ 2>/dev/null

# 2) 塞 override 强行拉目标大版本
sed -i '' 's|^dependency_overrides:|dependency_overrides:\n  <目标包>: ^<目标大版本>|' .probe_tmp/pubspec.yaml
#    Linux 用: sed -i '...'

# 3) 解析
(cd .probe_tmp && fvm flutter pub get)

# 4) 用完立刻删
rm -rf .probe_tmp
```

**判读结果：**

| 输出 | 含义 | 处理 |
|---|---|---|
| `Got dependencies!` 或 `! <pkg> <ver> (overridden)` | **仅说明与 SDK 无冲突** | ⚠️ 不等于能升，必须再做 B1.5 的传递依赖检查 |
| `Because X depends on <基础包> ^A ... every version of flutter_test from sdk depends on <基础包> B` | **SDK 天花板**，放弃该大版本 | 记录原因，不绕 |
| `Because X depends on pkg ^A and Y depends on pkg ^B, version solving failed` | 两个普通包互相冲突 | 一并升级 Y，或让解析器自己算 |
| `could not find package <path依赖> at "../xxx"` | **路径问题，不是版本冲突** | 副本深一层，补 `<pkg>: path: ../../xxx` override 后重试 |

> ⚠️ **必须区分「路径错误」和「版本冲突」**。副本目录深一层，`path: ../sibling` 这类相对路径会失效，报错看起来却像依赖问题，容易误判成“该包不可升”。

**发现 SDK 天花板时不要用 `dependency_override` 长期绕开** —— 会破坏 `flutter_test`，且下次升 SDK 更麻烦。

> 🔴 **override 探测法的最大陷阱：它会强行压过传递依赖的约束。**
>
> `dependency_overrides` 的语义是“无视其他包的约束，就用这个版本”。所以
> `Got dependencies!` **只证明「跟 Flutter SDK 不冲突」，完全不证明「跟你的其他依赖兼容」**。
> 实测踩过：`archive`、`equatable` 探测全过，真正 `pub add` 时立刻失败 ——
> ```
> Because <app> depends on excel ^4.0.6 which depends on archive ^3.6.1, archive ^3.6.1 is required.
> Because <app> depends on excel ^4.0.6 which depends on equatable ^2.0.0, equatable ^2.0.0 is required.
> ```
>
> **探测通过后，必须走 B1.5 再确认一遍。**

### B1.5. 传递依赖检查（override 探测通过后必做）

override 探测只能告诉你“SDK 层面没冲突”。真正能否升级，还取决于**谁在依赖这个包、它们要求的版本区间是什么**。

```bash
# 1) 找出谁依赖目标包
fvm flutter pub deps --style=tree | grep -B5 -A2 '<目标包>'

# 2) 直接问解析器（最可靠，不依赖人眼判断）
fvm flutter pub add <目标包>:^<目标版本>
#    失败会明确告诉你：Because <A> depends on <pkg> ^<旧> ...
#    成功后再决定是否保留，不满意就 git checkout -- pubspec.yaml
```

**或者不联网、不碰 pubspec 的纯查询法** —— 直接看上游包声明的约束：

```bash
# 本地缓存里的上游 pubspec
grep -A10 '^dependencies:' ~/.pub-cache/hosted/<prefix>/<上游包>-<版本>/pubspec.yaml

# 或者查 pub 上最新版的约束（确认上游是否已经放开）
curl -s https://pub.dev/api/packages/<上游包> | \
  python3 -c "import sys,json; d=json.load(sys.stdin); print(d['latest']['version'], d['latest']['pubspec'].get('dependencies',{}))"
```

**判断规则：**

| 情况 | 结论 |
|---|---|
| 目标包的**直接**使用者都是你自己 | 探测通过即可升 |
| 有上游包（如 `excel` 依赖 `archive`）且约束是 `^旧大版本` | **被上游卡死**。查上游最新版是否放开；没放开就等，别用 override |
| 上游包已是 pub 上最新版，约束仍然旧 | 无解，记录为“上游未跟进” |

**不要为了升一个包去 override 上游的约束**：上游代码是照着旧 API 写的，强制升会引入编译错误，而且会让 `pub get` 的解析结果变得不可预测。

### B2. 分级升级

```bash
# ① 小版本/补丁：先做，风险最低
fvm flutter pub upgrade --dry-run
fvm flutter pub upgrade

# ② 大版本：先预演，再应用
fvm flutter pub upgrade --major-versions --dry-run
fvm flutter pub upgrade --major-versions
```

**dry-run 输出读法：**
- `> pkg 1.2.3 (was 1.2.2)` → **会被改**
- `  pkg 1.2.3 (4.0.0 available)` → **受约束限制升不动**
- `- pkg` → 不再被依赖

**注意：`--major-versions` 会改写 `pubspec.yaml` 里的 `^` 约束**，不只是改 lock 文件。diff 要 review。

### B2.5. 联邦插件（federated plugin）升级后必须 clean

`file_picker`、`url_launcher`、`shared_preferences`、`connectivity_plus` 等都已联邦化 ——
主包只留接口，各平台实现在独立包里，通过 `.dart_tool/` 里的 `web_plugin_registrant.dart`
等注册器文件装配。**这些注册器不会随包结构变化自动更新**，`analyze` 也查不出来。

```bash
# 升完任何联邦插件后
fvm flutter clean && fvm flutter pub get
grep -i <pkg> .dart_tool/flutter_build/*/web_plugin_registrant.dart   # 确认 import 路径正确
```

### B3. 升级后的 API 变更排查

`pub upgrade` 本身不会报错，问题在编译期暴露：

```bash
fvm flutter analyze --no-fatal-infos
fvm dart fix --dry-run
fvm dart fix --apply
fvm flutter analyze --no-fatal-infos       # 修复后再确认
```

**查 CHANGELOG —— 本地 pub 缓存里就有，不用联网：**

```bash
# 找到包位置和缓存前缀
ls ~/.pub-cache/hosted/                     # 常见: pub.dev 或镜像前缀

PKG=<包名>; PREFIX=<上面列出的前缀>
ls -d ~/.pub-cache/hosted/$PREFIX/$PKG-*/ | sort -V | tail -3
grep -niE 'breaking|migrat' -A15 ~/.pub-cache/hosted/$PREFIX/$PKG-<旧版本>/CHANGELOG.md | head -60
```

**样例：某包 2.1.0 → 3.0.0 的 CHANGELOG 破坏点长这样**

```
3:- **BREAKING** fix!: do not test for `runtimeType` in object comparison
4:- **BREAKING** fix!: avoid custom `toString` override
5:- **BREAKING** refactor!: remove `EquatableMixin`
```

→ 每条 `BREAKING` 都对应一处代码要改。拿被删/被改的符号名去全项目 grep，就是要动的点。

**重点关注 `BREAKING CHANGES` / `Migration` 段落**，并在代码里 grep 受影响 API 的实际用法：

```bash
grep -rn --include='*.dart' '<被改的类名或方法名>' lib/
```

### B4. 联动约束

有些包被其他包依赖，**不能单独升**。先用依赖树确认：

```bash
fvm dart pub deps --style=compact      # 看谁依赖谁
fvm flutter pub deps --style=tree      # 树状更直观
```

典型联动：工具类包（Excel、PDF、图片处理等）往往同时依赖编码、序列化、集合工具包，升级时要整组评估。遇到 `version solving failed` 先看 `pub deps` 找出共同依赖者。

---

## 验证（不可跳过）

按成本从低到高，任一步失败就停：

```bash
# 1) 依赖能解析
fvm flutter pub get

# 2) 静态分析干净
fvm flutter analyze --no-fatal-infos

# 3) 机械化修复已应用
fvm dart fix --dry-run          # 期望 "Nothing to fix!"

# 4) 真正编译（能抓到 analyze 抓不到的集成问题）
fvm flutter build web --no-web-resources-cdn     # Web 项目
# 或按项目平台：fvm flutter build apk --debug / fvm flutter build macos --debug

# 5) 运行时冒烟（插件行为变化的唯一验证手段，可选）
fvm flutter run -d chrome --debug
```

**不通过就不许声称完成。** 某步因环境无法执行（无设备、内网不通）时，必须明确报告“该步未执行”及原因。

> ⚠️ **第 4 步和第 5 步不可互相替代。** 实际升级中常见：`analyze` 全程零问题，
> `build web` 报联邦插件路径错误（见 Gotchas）；`build web` 通过后，运行时又冒出
> `Overlay is null` 和 `ListTile` 断言。**Flutter SDK 大版本升级会新增断言**，
> 这类问题只有**真正跑起来**才暴露。

---

## 常见升级问题与解法

以下是升级过程中反复出现的几类问题，按发现阶段排列。遇到报错先在这里对号入座。

### 1. 版本探测阶段

| 现象 | 根因 | 解法 |
|---|---|---|
| 临时副本里报 `could not find package <X> at "../X"` | 副本目录比原工程深一层，`path: ../X` 这类相对路径失效 | 副本的 `dependency_overrides` 里补上 `<X>: {path: ../../X}`，这是**路径问题不是版本冲突** |
| 某个大版本始终升不上，报 `incompatible with flutter_test from sdk` | Flutter SDK 的 `flutter_test` 钉死了基础包（`meta`、`collection` 等）的版本，目标包要求更高 | 这是**硬天花板**，放弃该大版本；不要用 `dependency_override` 长期绕开，会破坏 `flutter_test` |
| 两个普通包互相冲突 `version solving failed` | 共同依赖者约束不一致 | 先 `pub deps` 找出共同依赖者，一并升级或降级其一 |

### 2. 编译阶段

| 现象 | 根因 | 解法 |
|---|---|---|
| `Undefined name 'XxxWeb'` / `xxx_web.dart (No such file or directory)` | 插件改成了**联邦插件**（federated plugin）结构，主包已无平台实现文件；`.dart_tool/` 里的 plugin registrant 仍是旧路径。**`analyze` 不报错** | `flutter clean && flutter pub get` 重新生成 registrant |
| 构建产物异常 / 平台注册类报错 | 构建缓存与新的包结构不匹配 | 同上，`flutter clean` |
| 编译时间明显变长 | 无缓存全量编译 | 正常，clean 后首次必慢 |

**联邦插件是高频陷阱**：`file_picker`、`url_launcher`、`shared_preferences`、`connectivity_plus`、`device_info_plus` 等都已联邦化。规则很简单——**升级任何插件后，只要它涉及平台实现，就先 `flutter clean`**，成本远低于排查报错。

### 3. API 变更阶段

| 现象 | 典型原因 | 解法 |
|---|---|---|
| `The method 'X' isn't defined` / `The getter 'X' isn't defined` | 类/方法被删除或改名 | 查缓存里的 CHANGELOG，找替代 API |
| `The named parameter 'x' isn't defined` | 参数被移除或改名 | 同上；常见于“主包里删掉了已废弃参数，拆到平台 Options 类” |
| `Too many positional arguments` | 位置参数改成了具名参数 | 改调用处 |
| `The argument type 'A' can't be assigned to 'B'` | 返回类型变了（如 `Future<T>` → `Future<T?>`，`List<X>` 取代包装类） | 跟随新签名调整 |

**通用排查流程**：报错 → `grep -n 'BREAKING' -A15 ~/.pub-cache/hosted/<prefix>/<pkg>-<newver>/CHANGELOG.md` → 在项目里 `grep -rn '<旧API名>' lib/` → 逐个替换。

### 4. 运行时阶段

| 现象 | 根因 | 解法 |
|---|---|---|
| `Overlay is null`（toast 类插件） | `init()` 传了 widget 树顶部的 context（`navigatorKey.currentContext`），向上找不到 `Overlay` | 改用 Navigator 内部的 context；并在**调用时**才 `init`，不要在文件顶层 |
| 插件在 `MaterialApp.builder` 里“配置了但没生效” | 该 API 返回的是 `TransitionBuilder`，**返回值没被使用**就等于没调用 | `widget = PluginBuilder()(context, widget);` |
| `ListTile background color or ink splashes may be invisible` | Flutter 新版断言：`ListTile` 外层有带背景色的 `ColoredBox`/`Container`，墨水效果会被遮挡 | 中间插一层 `Material(type: MaterialType.transparency)` |
| 其他新断言（assertion was thrown） | **Flutter 小版本升级也会加断言**，不是只有大版本才 breaking | 读断言里的 `To fix this...` 提示，按提示改 |

> ⚠️ **运行时问题只有跑起来才暴露。** `analyze` 和 `build` 都通过，不代表 app 能正常用。
> 升级后至少把主要页面的核心操作点一遍。

### 升级前评估改动面

动手前先确认每个要升的包在项目里被用得多深：

```bash
for pkg in <要升的包...>; do
  echo "$pkg: $(grep -rl "package:$pkg/" lib | wc -l | tr -d ' ') 个文件"
done
```

用到文件数为 0 的包可以放心升；超过 10 个文件的，先读 CHANGELOG 再决定。

### 四条硬经验

1. **联邦插件升级必须 `flutter clean`** —— 构建缓存不会随包结构自动更新。
2. **analyze 通过 ≠ 能编译 ≠ 能运行** —— 三级验证缺一不可。
3. **小版本也会 breaking** —— 断言、lint、运行时行为都可能在次版本变化。
4. **override 探测通过 ≠ 能升** —— 探测只验证 SDK 层面，传递依赖约束要靠 B1.5 确认。

---

## 回滚

```bash
git status --short                  # 先看改了什么
git checkout -- pubspec.yaml pubspec.lock
fvm flutter pub get
```

SDK 回滚：`fvm use <旧版本>`（先 `git checkout -- .fvmrc`）。

**升级前必做**：`git stash` 或先提交一个基线 commit。没有退路不要开始。

---

## Gotchas

- **`fvm` 不是 `flutter`**：裸命令用系统 SDK，版本可能不同，analyze 结果不可信。
- **没有 `.fvmrc` 的项目**：`fvm flutter` 会**静默**回退到全局默认版本，不报错。同一机器上不同项目可能解析出不同 SDK，先 `fvm use <version>` 固定版本，否则升级结论不可信。
- **`pub add` 不支持 `dependency_overrides:` 前缀**：`fvm flutter pub add dependency_overrides:<包>:^<版本>` 直接失败。要加 override 就编辑 pubspec 或用 B1 副本法。
- **探测副本的路径依赖会断**：副本深一层，`path: ../sibling` 失效。报 `could not find package` 时补 path override，别当版本冲突。
- **`--major-versions` 会改 pubspec.yaml**：不只改 lock，`^` 约束下界会变。
- **`dart fix` 只做机械修复**：API 改名它多半修不了，会留在 analyze 里。
- **SDK 钉死基础包**：`flutter_test` 固定 `meta` 等版本，任何要求更高版本的包都会被卡住。这是“能否升级”的硬天花板。
- **`flutter_localizations` 钉死 `intl`**：`intl` 的版本由 Flutter SDK 决定，自己改约束必然 `version solving failed`。报错提示 `intl is pinned to version X by flutter_localizations`，直接放弃。
- **`dependency_overrides` 探测通过不代表能升**：override 是无视所有约束强行指定，`Got dependencies!` 只证明“跟 SDK 不冲突”。真正升级走 `pub add` / `pub upgrade` 时会被上游包的旧约束卡住（如工具包依赖 `archive ^3.6.1`）。必须先做 B1.5。
- **上游包没出新版 = 死路**：用 `curl -s https://pub.dev/api/packages/<包>` 看 `latest` 版本及其依赖约束；最新版仍卡旧版本就只能等上游，不要 override。
- **`pub add` / `pub upgrade` 失败后 pubspec 可能已被改写**：务必 `git checkout -- pubspec.yaml` 还原再继续。
- **`pubspec.lock` 的 `sdks:` 块自动生成**，手改会被下次 `pub get` 覆盖。
- **analyzer 插件写了机器绝对路径**：`analysis_options.yaml` 的 `plugins:` 下若写了绝对路径，换机器或 CI 会 analyze 失败。改成版本号约束。
- **macOS 的 `sed -i` 需要 `''`**：`sed -i '' 's/...'`；Linux 是 `sed -i 's/...'`。
- **联邦插件升级后必须 `flutter clean`**：`.dart_tool/` 里的 `web_plugin_registrant.dart` 不会自动跟着包结构更新，会指向已删文件。`analyze` 完全不报，只有编译才炸。
- **Flutter 小版本升级会新增断言**：次版本也会加新的 debug 断言（如 `ListTile` 被 `ColoredBox` 遮挡的告警）。升级后务必跑起来点一遍，静态检查看不到。
- **`FToastBuilder()` 是返回值，不是副作用**：`MaterialApp.builder` 里写成 `FToastBuilder();` 等于没调用。必须 `widget = FToastBuilder()(context, widget);`
- **`FToast` 的 context 不能用 `navigatorKey.currentContext`**：它在 widget 树顶部，向上找不到 `Overlay`。用 `navigatorKey.currentState!.overlay!.context` 或 Navigator 内部的 context，且在**调用时**才 `init`（顶层 init 时 Navigator 可能还没建好）。
- **`analyze` 突然快很多时先验证分析器真的跑了**：写一个带明显错误的临时文件，确认它会报错，再相信 "No issues found"。

---

## Troubleshooting

| 现象 | 原因 | 处理 |
|---|---|---|
| `fvm flutter --version` 卡住 | 版本未安装，自动下载 | 等待；确认 `.fvmrc` 指向正确 |
| `pub get` 报 `incompatible with flutter_test from sdk` | SDK 钉死的基础包版本冲突 | 放弃该大版本（B1 判读规则） |
| `pub get` 在副本里报 `could not find package X at "../X"` | 相对 path 在子目录失效 | 加 `X: {path: ../../X}` override |
| `pub get` 报内网 git 拉不到 | 私有依赖仓库不可达 | 检查网络/VPN，重试 |
| `analyze` 报 analyzer 插件路径错误 | `analysis_options.yaml` 的 `plugins:` 写了机器绝对路径 | 改成版本号约束 |
| `build web` 报 `Undefined name 'XXXWeb'` / `xxx_web.dart (No such file or directory)` | 联邦插件结构变了，构建缓存里的 plugin registrant 是旧的 | `fvm flutter clean && fvm flutter pub get` 后重建 |
| 运行时 `Overlay is null`（fluttertoast 等 overlay 类库） | `init()` 用了 widget 树顶部的 context，向上找不到 Overlay | 改用 Navigator 内部 context，且在调用时才 `init` |
| 运行时 `ListTile background color or ink splashes may be invisible` | Flutter 3.44 新断言：ListTile 外层有带背景色的 ColoredBox/Container | `ListTile` 外包一层 `Material(type: MaterialType.transparency)` |
| `dart fix --apply` 后仍有大量错误 | API 真实变更 | 按 B3 手工修 |
| Dart 2 项目报 `|` 解析错误 | Dart 3 移除了联合类型划分符 | 见 A3，`dart fix` + 手工 |
| `melos bs` 报环境约束错误 | `melos.yaml` 的 `environment` 未同步 | 同步后再 `melos bs` |
| IDE 找不到 Dart SDK | `.fvm/flutter_sdk` 软链失效 | `fvm use <version>` 重建 |
| `fvm install` 拉不到版本 | 网络/镜像问题 | 检查 `FLUTTER_STORAGE_BASE_URL` 镜像配置 |

---

## 报告模板

```
## 升级总结
- 项目：<path>（类型：app / plugin / package / monorepo）
- SDK：<旧> → <新>（轨道 A）
- 依赖：N 个变更，M 个大版本（轨道 B）
- 放弃：<pkg> <ver>（原因：...）
- 手工 API 调整：<file>:<line> — <说明>

## 验证
- [ ] fvm flutter pub get
- [ ] fvm flutter analyze --no-fatal-infos → No issues
- [ ] fvm dart fix --dry-run → Nothing to fix
- [ ] fvm flutter build <platform> → 成功
- [ ] 运行时冒烟

## 未执行项
- <项>（原因：...）

## 回滚方式
git checkout -- pubspec.yaml pubspec.lock && fvm flutter pub get
```
