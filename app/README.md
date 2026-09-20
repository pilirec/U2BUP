# U2BUP 0.1.0

Rust + Tauri + Vue 的本机录播工作台原型。第一阶段面向现有 LiveRec 历史素材，原始录像保留，成品写入独立目录。

## 当前可用

- 本地素材扫描、FFprobe 参数缓存、XML 历史资料、房间别名聚合。
- 素材搜索、直播间/日期/类型筛选、跨页批量选择、缩略图、MP4 预览。
- B 站直播间当前资料抓取，保留历史标题，记录失败原因。
- 显示标题的模板、普通替换、正则替换，预览后应用；不改磁盘原文件名。
- 按直播间、标题、连续时间、完整探测流参数生成合并计划。画幅切换保留时间顺序。
- 时长与体积双重预算；未知时间、未知时长、超限单文件明确阻止。历史 MP4 默认排除，选择纳入时单独处理。
- 真正执行 FFmpeg 流复制合并，显示进度、取消、原计划重试；完成记录、媒体校验与抽样解码。
- SQLite 持久化素材、计划、任务与成品；启动时恢复中断状态，已登记产物会重新验证后复用。
- Tauri Windows 桌面入口与同一套本机 WebUI，认证会话按端口隔离。

## 尚未实现

YouTube OAuth/上传/管理、biliLive-tools 实时录制集成、AI、单文件自动切割、自动弹幕时间映射、任意格式浏览器播放、文件系统批量重命名、远程或多用户访问。macOS/Linux 尚未构建验证。

原型的输出验证是流参数、时长/体积及片段连接处抽样解码，不是全片完整解码。文件变更判断使用路径、大小和 mtime，不是全库内容哈希。SQLite 初版采用版本化 JSON 文档表，后续会迁移到关系型领域表。

## Windows 便携包

`dist/U2BUP-0.1.0-windows-x64/` 包含桌面程序、独立 Web 服务与 `resources/ffmpeg.exe`、`resources/ffprobe.exe`。

双击 `U2BUP.exe`。首次启动选择包含主播子目录的素材根目录；点击“扫描 LiveRec”。桌面数据存放于系统应用数据目录 `io.u2bup.desktop/data`，成品在同级 `exports`，根目录选择记录在 `library-root.txt`。

也可从 PowerShell 运行包内的：

```powershell
.\Start-Web.ps1 -Library 'C:\path\to\LiveRec'
```

终端会打印带本机会话令牌的启动链接。打开该链接，令牌交换为 HttpOnly 会话 cookie 后会从地址栏移除。数据目录的 `connection.json` 也保存本次启动链接；不要对外分享该文件或 `.local` 数据。

运行需要系统 WebView2。已在本机 Windows 验证；便携目录不是完成全平台安装认证的正式发行版。

## 从源码运行 WebUI

需要 Rust/MSVC、Node、FFmpeg 和 FFprobe。锁文件已纳入源码，首次安装后：

```powershell
.\Start-Web.ps1 -Build
```

后续只启动已有构建：

```powershell
.\Start-Web.ps1
```

手动构建：

```powershell
npm --prefix web ci
npm --prefix web run build
cargo build -p u2bup-server --locked
.\target\debug\u2bup-server.exe --library '..\LiveRec' --port 4173
```

Web 静态资源由 Rust 二进制嵌入，发行运行不需要 Node/Vite。开发时先构建 Web 再编译 Rust。服务默认只监听 127.0.0.1，拒绝跨来源 API 调用，不应直接暴露公网。

命令行参数：`--library`、`--data`、`--output`、`--port`、`--ffmpeg`、`--ffprobe`。同一个数据目录只允许一个服务实例。桌面和独立 Web 服务默认使用不同数据目录；若要查看同一任务，浏览器应连接桌面服务的 `connection.json` 链接。

## 构建桌面和便携包

```powershell
npm --prefix web run build
cargo build -p u2bup-desktop --locked
.\target\debug\u2bup-desktop.exe --library '..\LiveRec'
```

```powershell
.\scripts\build-portable.ps1
```

`stage-media.ps1` 的默认媒体组件来源为本机 Scoop FFmpeg，可以用 `-FfmpegDir` 指定另一发行目录。生成的媒体清单包含 SHA-256 和版本。发行模式桌面拒绝静默使用 PATH 中的 FFmpeg。

`desktop/tauri.bundle.conf.json` 提供 NSIS 安装包配置；要生成安装包，先准备组件，然后在 `desktop` 目录执行 Tauri CLI build 并传入该配置。便携包构建不等于 NSIS 安装包已测试。

## 验证

```powershell
cargo test -p u2bup-core -p u2bup-server --locked
cargo clippy -p u2bup-core -p u2bup-server --all-targets -- -D warnings
cargo build -p u2bup-server --locked
node scripts/smoke.mjs
```

`smoke.mjs` 在 `.local/smoke/<时间>/` 创建独立的合成素材，测试真实 FFmpeg 处理、中文/单引号路径、未知时长、切换画幅、源文件 SHA-256、鉴权和原计划重试，不修改 LiveRec。每次留下报告和产物供检查。

真实 LiveRec 验收记录见 `../docs/BUILD-STATUS.md`。
