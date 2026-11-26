# 以 Supermium 方式在 Windows 7 上运行高版本 Chromium 与 DeepSeek Harness Desktop

实测环境：VirtualBox 7.2.22 + Windows 7 旗舰版 SP1（6.1.7601.28117 x64，含 ESU），2 vCPU / 8 GB。
所有结论均为实机验证，命令与环境记录见文末。

---

## 一、起点：Windows 7 上跑高版本 Node.js

### 1.1 官方边界

- `BUILDING.md` 自 v18 起只把 `>= Windows 10 / Server 2016` 列为 Tier 1，Win7 已不在支持表。
- `src/node_main.cc` 有硬校验：

```cpp
if (!IsWindows8Point1OrGreater() && !(IsWindowsServer() && IsWindows8OrGreater()) &&
    GetEnvironmentVariableA("NODE_SKIP_PLATFORM_CHECK", nullptr) != 1) {
  // "Node.js is only supported on Windows 8.1, Windows Server 2012 R2, or higher."
  exit(ERROR_EXE_MACHINE_TYPE_MISMATCH);
}
```

v24 起阈值改为 `IsWindows10OrGreater()`，文案改为 `Windows 10, Windows Server 2016`。

### 1.2 实测矩阵（官方 win-x64 zip）

| 版本 | 结果 | 失败原因 |
|---|---|---|
| 12.22.12 | ✅ | — |
| 14.21.3 | ❌ `0xC0000139` | 缺 `GetHostNameW@WS2_32` |
| 16.20.2 / 18.20.8 | ⚠️ 打印平台检查后 `exit 216` | 设 `NODE_SKIP_PLATFORM_CHECK=1` 后 ✅ |
| 20 / 22 / 24 官方版 | ❌ `0xC0000139` | 缺 `GetSystemTimePreciseAsFileTime@KERNEL32`（加载器阶段失败，SKIP 无效） |
| 26 | ❌ `0xC0000005` | 额外缺 `WaitOnAddress` / `WakeByAddress*` |

关键点：`0xC0000139` 发生在 **main() 之前**，所以 `NODE_SKIP_PLATFORM_CHECK` 对 20+ 完全无效。

### 1.3 两条修复路线

**A. 二进制等长改名**（`GetSystemTimePreciseAsFileTime` → `GetSystemTimeAsFileTime`）
v22 / v24 / v25 可行（配 SKIP），v26 因 `WaitOnAddress` 无解。

**B. VxKex 兼容层**（推荐）

```
"C:\Program Files\VxKex\KexCfg.exe" /EXE:"<abs path to exe>" /ENABLE:1
```

v14 – v26 全部 EXIT=0，且**不需要** SKIP；x86 同样有效。VxKex 无性能惩罚（v26 比 v18+SKIP 快约 2 倍），时钟精度无可测退化。

> 注意：`FilterFullPath` 绑死单个绝对路径，无法按文件名批量匹配；每个 exe 路径都要单独 enable。

### 1.4 结论

**官方 MSI 可按官方流程装**（`msiexec /i node-v24.22.3-x64.msi /qn` EXIT=0），但装完必须对 `node.exe` 启用 VxKex 才能运行。npm / npx / 原生模块（sqlite3）全部验证通过。

---

## 二、转折：Supermium 的思路

Supermium 之所以能在 Win7 跑高版本 Chromium，核心不是打补丁，而是**在源码级把平台的静态依赖彻底剥掉**，让二进制对 Win7 的导入表零要求。

对应到 Electron 就是 `Alex313031/electron-legacy`：

| 指标 | 官方 Electron | electron-legacy |
|---|---|---|
| PE `OSver` / `SubsysVer` | 10.0 / 10.0 | **5.2 / 5.2** |
| Win7 加载器 | 直接拒绝（`不是有效的 Win32 应用程序`） | 放行 |
| 静态导入 | 含 `GetSystemTimePreciseAsFileTime`、`WaitOnAddress` | 仅 8 个基础 DLL，无任何 Win8+ 导入 |
| 解决方式 | 需 VxKex / 改 PE | 源码级 `patches/node/01..18` |

实测 `electron.exe --version` → `v44.10.2`，**无需 VxKex、无需 SKIP、无需改 PE 头**。
内部版本：`node 24.22.2 / v8 15.2.124.45-electron.0 / chrome 152.0.7977.169`。

---

## 三、把 DSH Desktop 套进 electron-legacy

### 3.1 官方 DSH 为什么不能直接跑

`DeepSeek Harness.exe` 的 PE 头是 `OSver=10.0 / SubsysVer=10.0`，Win7 加载器在用户代码执行前就拒绝，VxKex 无从下手。

### 3.2 套壳步骤

1. 用 electron-legacy 的 `electron.exe` 替换主程序（复制为 `DSHTest.exe`，走打包模式）；
2. 保留 DSH 的 `resources\app.asar` + `app.asar.unpacked` + `runtime`；
3. 启动后依次遇到并解决：
   - `DSH_DESKTOP_PRIMARY_RUNTIME_DIR is required for an unpackaged launch` → 必须打包模式运行；
   - `node-addon-require-builtin unsupported (unsupported Electron runtime fingerprint ...)` → 见 3.3；
   - `ENOENT ... office-skills\office\SKILL.md` → 补齐 runtime 目录。

### 3.3 指纹门（关键补丁）

`node-addon-require-builtin` 内置白名单写死「Electron 版本 ↔ V8 串」，Node 版本还以整数三元组硬编码：

- V8 串 `15.2.124.28-electron.0` → `15.2.124.45-electron.0`（等长替换，偏移 297520）
- Node 三元组 `(24,21,0)` → `(24,22,2)`（两处）

补丁后指纹通过，`dsh-desktop:boot` 继续执行。

### 3.4 渲染：Win7 上必须切 D3D9

| 启动参数 | 结果 |
|---|---|
| 无参数 | **白屏**（GPU 进程反复崩，`exit_code=34`） |
| `--disable-gpu` | ✅ 可用（全软件渲染） |
| `--use-angle=d3d9` | ✅ 可用且**保留 GPU 加速**（推荐） |

`capturePage()` 在 d3d9 / swiftshader / in-process 三种模式产出的 PNG 完全一致（含中文排版），说明崩溃只影响合成画面，不影响渲染正确性。

已在 `main.js` 顶部硬编码：

```js
app.commandLine.appendSwitch("use-angle", "d3d9");
app.commandLine.appendSwitch("disable-gpu-process-crash-limit"); // GPU 崩溃后不要永久禁用
app.commandLine.appendSwitch("disable-gpu-watchdog");
```

后两条针对 `child-process-gone`（GPU，exitCode `0xC0000005`）：避免 Chromium 崩几次后永久关闭 GPU 并留下空窗。

---

## 四、无边框外观（去 Aero）

### 4.1 成因

主窗口与欢迎窗口都声明了 `titleBarStyle: "hidden"` + `titleBarOverlay`（WCO 需 Win10 1809+），Win7 静默回退到原生 Aero 边框；前端 `data-windows-titlebar` 又是无条件打上的，于是「自定义标题栏」与「Aero 边框」叠加。

### 4.2 做法

- 主窗：`frame: false, thickFrame: false`，并禁用 `setTitleBarOverlay`（否则抛 `TypeError: Titlebar overlay is not enabled`）；
- 欢迎窗：`backgroundColor` 调亮 + 按系统主版本分流（≥10 才用 acrylic/WCO）；
- 自绘按钮：preload 注入三枚 46×40 SVG 按钮（统一 1px 描边，`-webkit-app-region: no-drag`），`ipc` 回主进程处理 minimize / maximize / close，用 `window-state` 事件回推切换还原图标。

**取舍：`thickFrame` 同时管辖 Aero 边框与原生最小化动画，二者不可兼得。** 本项目选择去边框、放弃原生动画。

### 4.3 登录页 vs 主界面

- 登录页（welcome）：不给最大化按钮，且 `maximizable: false`——最大化后内容不铺满、右侧会露桌面；
- 主界面：完整四态（最小化 / 最大化 / 还原 / 关闭）。

---

## 五、工具链修复

| 组件 | 问题 | 修复 |
|---|---|---|
| Python 3.12.14 | 缺 `api-ms-win-core-path-l1-1-0.dll`（`0xC0000135`） | 对 `python.exe` 启用 VxKex（KxBase 实现了 `PathCch*`） |
| Node 24.21.0（捆绑） | `0xC0000139` | 启用 VxKex + `/WINVERSPOOF:WIN10` |
| ripgrep | 14.x 崩 `0xC0000005` | 换成 **13.0.0** |
| sharp (libvips) | 缺 `WaitOnAddress` 一族 | 给宿主 exe 启用 VxKex |
| PowerShell | 2.0，无 `Get-CimInstance` | 装 .NET 4.8 → WMF 5.1（`PS 5.1.14409.1005`） |
| Git / 7-Zip / LibreOffice | 缺失 | 静默装 2.46.2 / 22.01 / 7.3.7.2（控制台调用必须用 `soffice.com`，`soffice.exe` 会停在 `Press Enter`） |

---

## 六、安装包设计

- 工具：Inno Setup 6.7.4（直链仅 GitHub releases 可用）
- 载荷：约 1.66 GB / 7868 文件
- 多语言：自建 `chinese.isl`（`LanguageID=$0804`、`LanguageCodePage=936`），**`LanguageDetectionMethod=locale`** 才能真正跟随系统语言；`uilanguage` 在本机不生效，`registry` 不是合法值
- 术语策略：`License Agreement` / `Information` / `GPL-2.0` / `MPL-2.0` / `VxKex` 保持英文，其余界面中文
- 两个独立向导页（替代弹框）：
  1. **环境检测**：OS 版本、.NET/PS/工具链有无、VxKex 与兼容性 shim、已有 IFEO
  2. **覆盖安装**：已装版本 → 目标版本、数据不删说明；无旧版本时自动跳过
- 关键坑：
  - `Kex64\*.dll` **必须复制到 exe 同目录**，否则 `丢失 kxnt.DLL`；
  - 先决包不能用 `Flags: dontcopy`（否则 `CreateProcess failed; code 2`），要在安装开始时解压到 `{tmp}`；
  - `TStrings.LoadFromFile` 按 ANSI 读文件 → 中文说明文件要额外提供 GBK 版本；
  - `.iss` 必须是带 BOM 的 UTF-8，且不能用 PowerShell `-replace` 改（会把表达式字面写进文件）。
- `管理 dsh 命令` 落点已从 `{app}\resources\runtime\cli\bin` 迁到 `%LOCALAPPDATA%\DeepSeek Harness\cli-bin`（改 `command-manager.js` 一处 + 安装器 `[Dirs]` 预建目录）。

---

## 七、已知限制

1. **DSH 宿主沙箱在 Win7 不可用**。`AssignProcessToJobObject` 在 Win7/2008R2/Vista 上要求「进程不能已属于某个 Job」，否则 `ERROR_ACCESS_DENIED`；嵌套 Job 要 Windows 8 才支持（[MSDN](https://learn.microsoft.com/en-us/windows/win32/api/jobapi2/nf-jobapi2-assignprocesstojobobject)）。官方给的出路是 `CreateProcess` 加 `CREATE_BREAKAWAY_FROM_JOB`，或对 PCA 兼容性 Job 的进程先 breakaway。
   → 该模块不在本安装包内（`windows-acl-run` 属宿主运行时），本移植版无法修；当前表现为「受限模式无法执行任何命令」。
2. **GPU 进程仍可能崩**（`0xC0000005`，VxKex + Chromium 152 典型症状）。已加崩溃限制绕过，退回软件合成，不空窗但可能偶发卡顿。
3. **平台已过保**：Win7 于 2020-01 停止支持，Electron 44 / Chromium 152 官方要求 Win10+。此方案属兼容性回退，官方 nightly 升级可能再次破坏。
4. 无法承诺与原版完全一致：`electron-legacy` 与官方 Electron 44.7.0 存在小版本差异，已通过指纹补丁对齐。

---

## 八、复现要点

```bat
:: 环境（VirtualBox）
VBoxManage sharedfolder add windows7 --name=dsh --hostpath="<共享目录>" --transient --automount

:: 每个需要运行的 exe 都要单独启用兼容层
"C:\Program Files\VxKex\KexCfg.exe" /EXE:"C:\Program Files\DeepSeek Harness\DeepSeek Harness.exe" /ENABLE:1
"C:\Program Files\VxKex\KexCfg.exe" /EXE:"...\runtime\primary-runtime\dependencies\node\bin\node.exe" /ENABLE:1
"C:\Program Files\VxKex\KexCfg.exe" /EXE:"...\runtime\primary-runtime\dependencies\python\python.exe" /ENABLE:1

:: 编译安装包
"C:\Program Files (x86)\Inno Setup 6\ISCC.exe" "E:\WorkSpace\_innosetup\dsh_win7.iss"
```

**静默安装**：`DSHSetup.exe /VERYSILENT /SUPPRESSMSGBOXES /NORESTART /COMPONENTS="main,vxkex"`

---

*本报告基于 2026-11 的实机测试。VxKex 上游未声明开源许可证（`license: null`、无 LICENSE 文件），随包分发前请自行评估授权风险，详见 `licenses\VxKex-NOTICE.txt`。*

## 九、2026-11-24 收尾修复记录

### 9.1 命令管理「Command-manager process failed.」的根因
`resources\runtime\cli\command-manager.js` 使用 ESM 语法（首行 `import { execFile, spawn } from "node:child_process"`），但同目录缺少 `package.json`。Node 对 `.js` 默认按 CommonJS 解析，遇到 `import` 在**解析阶段**即抛 `SyntaxError`，一行都不执行；主进程 `worker()` 用 `execFile` 调它，得不到任何 stdout，于是落到

```js
throw new CommandWorkerError("EIO", "Command-manager process failed.");
```

修复：补回 `resources\runtime\cli\package.json`，内容 `{"type":"module"}`。

### 9.2 命令 worker 的双通道启动
`main.js` 的 `worker()` 原来只认捆绑的 `node.exe`（24.21.0），而该二进制在 Win7 上必须靠 VxKex 的 IFEO 项才能启动，一旦 IFEO 丢失就整体失效。现改为：

```js
try {
  stdout = (await promisify(execFile)(node, workerArgs, workerOptions)).stdout;
} catch (bundledNodeError) {
  stdout = (await promisify(execFile)(process.execPath, workerArgs, {
    ...workerOptions, env: { ...env, ELECTRON_RUN_AS_NODE: "1" }
  })).stdout;
}
```

Electron 运行时是 electron-legacy（PE 子系统 5.2），原生可跑，因此命令管理不再依赖兼容层。

### 9.3 向导外观
- 位图改 2 倍图（`wizard.bmp` 328×628、`small.bmp` 128×128），配合 `WizardImageStretch=yes`；
- `WizardSizePercent` 由 118 调回 **100**——这是「图片发糊」的真正原因（整体放大 18% 导致位图重采样）；
- 右上角小图标改为**纯白底 alpha 合成**，消除与标题栏的色差；
- 移除完成页重复的女仆图，只保留左侧栏一处。

### 9.4 已知限制（未变）
- Win7 无嵌套 Job，沙箱只能「完全权限」运行；
- 捆绑 Python/Node 需 VxKex（安装器会写 IFEO，命令管理已不再依赖它）；
- `libreoffice-kit.exe --help` 无输出，Office→PDF 待真实任务验证。

### 9.5 office-export-pdf / office-render 在 Win7 不可用（已确证）

测试走文档化 CLI，而非直接跑 exe：

```
node <cli> convert --input <docx> --output <pdf>
# exit=1，1.1 秒
# {"code":"failed","error":"LibreOffice helper returned an invalid response (exit 3221225785)."}
```

`3221225785 = 0xC0000139 STATUS_ENTRYPOINT_NOT_FOUND`。耗时 1.1 秒说明引擎在**加载阶段**就失败，未进入文档解析；输出无 PDF、无 `%PDF-` 签名；直接运行 `libreoffice-kit.exe --help`（CWD 指向 bin）同样返回 `-1073741511`。

根因：该 sidecar 是 LibreOfficeDev 26 引擎（`@deepseek-ai/libreoffice-kit-win32-x64@0.1.5`，178 MB，2026-11-04 构建），官方仅支持 Win10+。bin 目录内已自带 VxKex 垫片（`KexDll.dll`、`Kx*.dll` 11 个、`dwrw10.dll`，以及 Win10 的 `ucrtbase.dll` / `msvcp_win.dll`），且与 node.exe / python.exe 旁逐字节同尺寸——后两者在本机可用，说明垫片机制有效；因此结论是**该二进制缺少的入口点超出 VxKex 现有实现**，非配置问题。

替代路径（均不经过 DSH office 技能）：
1. 自备 LibreOffice 7.3/7.5，`soffice.com --headless --convert-to pdf`；
2. 用 Python（fpdf2 / reportlab）直接排版生成，中文需嵌入系统字体（微软雅黑/宋体）；
3. 换 Win10/11 或 Linux 运行 DSH。

## 十、这一版，我们把 DeepSeek Harness 尽量改得顺眼了

前面九节讲的是「能不能跑」，这一节讲的是「跑起来像不像话」。说白了：能让它跑是及格，让它像个正常桌面软件才是目标。

### 10.1 它终于不再顶着一圈 Windows 7 的 Aero 玻璃边框

Electron 在主窗口上原本写的是 `titleBarStyle: "hidden"` + `titleBarOverlay`，而后者要 Windows 10 1809 以上才认。Win7 拿到这个参数只会礼貌地「不理解，但当没听见」，然后照旧画一圈 Aero 边框和标题栏。

我们直接不让它有机会画：主窗口与欢迎窗口都载入 `frame: false, thickFrame: false`。代价很明确——`thickFrame` 同时管着边框和原生最小化动画，去掉边框就没有系统那条「缩到任务栏」的动画了。这是一次取舍，不是 bug。

### 10.2 三颗按钮，是我们自己画的

边框没了，系统的最小化/最大化/关闭也跟着没了。于是我们在 preload 里注入一组自绘按钮：三枚 46×40 的内联 SVG，统一 1px 描边，`-webkit-app-region: no-drag` 保证能点，点击后 `ipc.send("dsh-win7-window-control", …)` 交给主进程处理；窗口最大化/还原时，主进程再回推 `dsh-win7-window-state`，让图标在「方框」和「双叠方框」之间切换。

顺带把最大化整个拿掉了——因为在 Win7 上它最大化后内容不铺满、右侧会露出一条桌面，与其留着碍眼，不如不给。

### 10.3 它现在有个鲸鱼图标，任务栏和托盘都是

可执行文件用 `rcedit` 换掉了图标与版本信息（`ProductName=DeepSeek Harness`、`FileVersion=0.2.2.0`），体积从 247,651,328 B 微调到 247,629,824 B，PE 头仍是 `5.2/5.2/2`——没有为了好看而牺牲「Win7 能加载」这条底线。

托盘那个图标曾经消失过一次，原因很朴素：`resources\tray.ico` 被覆盖安装清掉了，而代码里 `new Tray(...)` 老老实实包在 try/catch 里，文件不在就静默失败。补回那个 15,698 B 的图标后，鲸鱼又回到通知区域了。

### 10.4 它不再渲染成一块白板

虚拟机的 VBoxSVGA 撑不住 Chromium 152 的 GPU 合成，GPU 进程一次次以 `exit_code=34` 崩掉，然后界面白得干干净净。

现在主进程起来就 `app.disableHardwareAcceleration()`，从源头上放弃硬件加速；想验硬件路径的话，`DSH_WIN7_GPU=1` 仍会走 `--use-angle=d3d9`。这不是「修好了」，而是「承认这台机器的显卡靠不住，所以不去打扰它」。

### 10.5 Office 预览：从「不可用」到真的能看

这大概是这一版最折腾的一处。

界面那句「Office 预览不可用」只是文案，真正的闸门在宿主 RPC：`officeToPdf.generation()` 返回 `ok: false`，客户端就抛错。而它背后的 `@deepseek-ai/dsh-office-to-pdf` 把转换全权交给 `libreoffice-kit`，kit 又去调一个 Win32 sidecar——那个 sidecar 静态依赖 `BCryptKeyDerivation` 等 Win8+ 导出，在 Win7 上根本加载不起来。

于是我们给 `dsh-office-to-pdf` 补了一条退路：`converter.render()` 失败就转调系统 LibreOffice 7.3 的 `soffice.com --headless --norestore --nolockcheck`，产出的 PDF 再按源文件基名改回约定的输出路径。中间踩了四个坑，每个都很具体：

1. `__dshLog` 里写了 `require("node:fs")`，而那是 ESM 文件——`require` 未定义，异常被 `catch {}` 吞掉，所以日志永远是空的，`existsSync` 也因此永远说「soffice 不存在」；
2. `--convert-to pdf --outdir` 产出的是**源文件基名.pdf**，不是我们以为的固定名；
3. 回退成功分支返回了空对象，客户端读 `result.missingFonts` 直接 `is not iterable`；
4. kit 的 `index.js` 被我早前塞进一句指向不存在文件的 `import`，导致整个模块 `ERR_MODULE_NOT_FOUND`，服务从未激活。

四条都清掉之后，一份 3 页的中文报告能完整渲染出来，`pdf size=99316`。**这是实测通过，不是「应该可以」。**

### 10.6 命令管理从「静默报错」变成「装好了」

点「管理 dsh 命令」原本弹「无法更新命令」。根因不在权限，而在模块系统：`resources\runtime\cli\command-manager.js` 是 ESM，但同目录缺 `package.json`，Node 按 CommonJS 解析，第一行就 `SyntaxError`。

补一个 23 字节的 `{"type":"module"}`，弹窗就变成「Desktop 命令已安装」，还顺手把落点从不可写的 `{app}\resources\runtime\cli\bin` 挪到了 `%LOCALAPPDATA%\DeepSeek Harness\cli-bin`。

### 10.7 设置里多了一项「兼容设置」

这一项折腾得最久，因为**试错了五次**：塞进「通用设置」页末尾（锚点命中了外层滚动容器，看不见）、塞进「当前版本」行后（断言没匹配上）、克隆左侧导航（`bili 设置` 文本三种写法都没命中）、加进原生 `Menu`（应用启动期反复 `setApplicationMenu` 把我们的项冲掉）、全局快捷键（注册成功了，但没人按）。

最后的正解其实很土：**在左侧「更多」下面克隆一行，改掉文案**。点击行为不调闭包里的 `openPanel()`，而是去点一个已经绑好事件的隐藏按钮——`document.getElementById("dsh-win7-fab").click()`。这样绕开了 IIFE 作用域，也不需要理解应用的内部状态。

中途我犯过一个必须记下来的错误：把一个探针直接拼进 `main.js`，嵌套引号转义错了，主进程 `SyntaxError` 起不来。**给 main.js 追加代码，先过 `node --check`。**

### 10.8 侧栏这一行，现在两种语言都认

```js
var __zh = /^zh/i.test(navigator.language || "");
var __lbl = __zh ? "兼容设置" : "Compatibility";
```

中文系统显示「兼容设置」，英文系统显示 `Compatibility`，字号、图标、间距与相邻项完全一致；图标是内联 SVG 常量（齿轮，`scale(0.84)` 居中缩放到与「更多」的点阵同视觉尺寸），换图标只需改那一行字符串。

### 10.9 安装包里，不该有的东西都清掉了

对 `_payload` 做了三轮扫描，删掉三类垃圾：两份 89,072 B 的 kit 旧备份、一个孤立且被误 import 的 `win7-soffice.js`、两个空的 unpacked 目录（空目录会遮蔽 asar 内条目，是早期试验的残留）。

清理后 7,924 个文件 / 1,743 MB，没有 `.bak/.old/.orig/.rej/~/.DS_Store/Thumbs.db`。剩下的大文件都是硬需求：LibreOffice MSI 332 MB、主程序 236 MB、`app.asar` 203 MB、kit sidecar 170 MB、.NET 4.8 116 MB、Node 89 MB、Git 65 MB、WMF 65 MB、ICU 26 MB。

第三方包里的 `test`/`doc` 目录我**没有删**——几 MB 的收益，换不来「某个包 require 到自己 doc 路径就崩」的风险。

### 10.10 安装程序的三个警告，修了两个

- `NeedReboot` 原本只有中文，英文系统会回退成中文 → 补了英文条目；
- `KexDll.dll` 装进 `{sys}` 却带 `ignoreversion` → 去掉该标志（系统目录不该用这个）；
- `PrivilegesRequired=admin` 但脚本写 `HKCU`/`localappdata` —— 这条**保留**，因为 VxKex 必须写 `Program Files` 和 IFEO，这是有意为之，不是疏漏。

最终产物 **888,794,021 B**，`Successful compile (204.485 sec)`，此前的两条警告不再出现。

### 10.11 关于 kit 的 IFEO：写进去了，但注定不生效

安装器原本给四个可执行文件写了 VxKex 的 IFEO 项（`DeepSeek Harness.exe`、`node.exe`、`python.exe`、`libreoffice-kit.exe`），注册表实测四条齐全、`VerifierDlls = kexdll.dll` 也在。但它依赖的 `KexDll.dll` 必须能被**目标进程的加载器**找到，而 kit 的 sidecar 是从 `resources\...\libreoffice-kit\bin\` 启动的，和 `KexDll.dll` 不在同一目录 —— 于是这条配置只是"看起来该生效"。

这一版把 `libreoffice-kit.exe` 那条**主动删掉**了，只留下真正有效的三条。Office 转换改走系统 LibreOffice 的 `soffice.com --headless` 回退链（实测 9 秒/次、字体嵌入完整、中文无乱码）。

留个后手：如果以后把 `KexDll.dll` 复制进 sidecar 目录，这条 IFEO 是可以恢复的 —— 前提是那时 kit 的 sidecar 本身还能在 Win7 上加载。

---

## 十一、系统要求（Windows 7 补丁与前置条件）

安装程序会在「环境检测」页逐项列出下列内容，并区分四个等级：`[√] 已满足`、`[!] 建议补齐`、`[ ] 可选/未安装`、`[×] 影响功能`。

### 11.1 硬性要求

| 项目 | 要求 | 不满足的后果 |
|---|---|---|
| 系统 | **Windows 7 SP1（6.1.7601）x64** | 安装器直接拒绝；非 SP1 时 .NET 4.8 / PowerShell 5.1 无法安装 |
| 兼容层 | 随包安装的 VxKex API Extensions 1.2.2 | Electron 44 / Chromium 152 静态导入 Win8+ API，无兼容层则进程启动即失败 |
| 运行时 | 随包 electron-legacy 44.10.2（PE 子系统 5.2） | 官方 Electron 44 的 PE 头是 10.0，Win7 加载器拒绝加载 |

### 11.2 与功能相关的补丁

| 补丁 | 作用 | 影响的功能 |
|---|---|---|
| .NET Framework 4.8 | PowerShell 工具链 | `.ps1` 脚本、`office-pdf.cmd` |
| WMF 5.1（KB3191566） | PowerShell 5.1 | 同上（实测 `5.1.14409.1005`） |
| LibreOffice 7.3.7 | Office 预览 / PDF 导出 | `.docx/.xlsx/.pptx` 预览（走 `soffice.com --headless` 回退，约 9 秒/份） |
| TLS 1.2（KB3140245 + 注册表 `DefaultSecureProtocols`） | 现代 TLS | 登录、在线更新 |

### 11.3 行业标准补丁（本项目未逐项验证，但强烈建议）

| 补丁 | 原因 |
|---|---|
| **KB4474419 + KB4490628** | SHA-2 代码签名支持。2020 年后签名的 .NET / Git 等安装包没有它会报「签名无效」，**升级安装前最容易踩** |
| KB2533623 | `SetDefaultDllDirectories`，Chromium 系加载 DLL 需要 |
| KB2670838 | Platform Update（D2D / DWrite / D3D11.1）。默认软件渲染不依赖；改用 `DSH_WIN7_GPU=1` 的 D3D9 硬件加速时需要 |

> 说明：本项目测试机为 `6.1.7601.28117`（已含上述大部分补丁）。**纯净未打补丁的 SP1 上的边界未做逐项实测**，安装器只做检测与提示，不代替实际验证。
