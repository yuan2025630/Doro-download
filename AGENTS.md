# Doro Downloader 项目总指导

本文件是本仓库及其全部子目录的项目级协作与运行规范。所有在本项目中工作的 Codex、子代理和兼容 `AGENTS.md` 的自动化工具都必须遵守；人工协作者也应以此作为实现、审查和交付基线。

用户在当前任务中的明确要求优先于本文件，但任何要求都不得隐式扩大到绕过 DRM、会员权限、验证码、平台访问控制或泄露登录凭证。若当前任务与本文件存在无法同时满足的冲突，必须先向用户说明，不得静默规避。

## 一、项目目标与基本判断

Doro Downloader 是基于 `yt-dlp`、FFmpeg/ffprobe 和 Deno 的 Windows 视频下载器。GitHub 上的开源客户端只能改善请求、解析、下载、合并和用户体验，不能关闭 Bilibili、YouTube 等平台服务器端的风控。

处理下载失败时，必须先区分以下边界，不得把所有错误笼统归为“下载失败”：

1. 应用发布包或外部工具是否完整。
2. URL、站点和视频类型是否受支持。
3. Cookie 文件是否存在、格式是否正确、是否已过期或被服务器拒绝。
4. 浏览器 Cookie 数据库是否不存在、被锁定或权限不足。
5. Windows ABE/DPAPI 是否导致 Chrome/Edge Cookie 无法解密。
6. 站点是否返回 HTTP 412、403、429、WBI、playurl 或分片错误。
7. 网络、代理、TLS/浏览器指纹是否异常。
8. 保存目录、磁盘空间、文件名和写入权限是否异常。
9. FFmpeg/ffprobe 合并、探测或后处理是否失败。
10. 任务是否被用户取消。

任何修复都必须建立在真实错误输出和可复现证据上。禁止在没有定位失败阶段时盲目增加参数、重试或 Cookie 来源。

## 二、依赖和版本管理

便携发布包必须包含且只从可信上游获取以下运行组件：

```text
tools/yt-dlp.exe
tools/ffmpeg.exe
tools/ffprobe.exe
tools/deno.exe
tools/versions.json
```

必须遵守：

- 发布前运行 `scripts/verify-tools.ps1`，确认四个可执行文件存在、非空且能输出版本。
- FFmpeg 与 ffprobe 必须来自同一构建系列。
- 默认使用经过验证的 yt-dlp 稳定版；项目可提供 nightly 通道，用于快速获得站点适配修复。
- yt-dlp 更新必须下载到临时位置、校验成功后再替换，并保留上一个可用版本用于回滚。
- 不得因为站点错误自动切换到未经验证的第三方 yt-dlp fork。
- 若当前稳定版已经是最新版本，不得声称“更新 yt-dlp”一定能解决服务端 412。
- `ffmpeg`、`ffprobe`、Deno 和 yt-dlp 的版本必须能够在诊断报告中安全显示。

参考上游：

- <https://github.com/yt-dlp/yt-dlp>
- <https://github.com/yt-dlp/FFmpeg-Builds>
- <https://github.com/denoland/deno>

## 三、Cookie 与登录态安全

Cookie 等同于登录凭证，必须按敏感数据处理。

### 强制安全边界

- 禁止把 Cookie 文件、Cookie 值、请求头、浏览器数据库内容或认证令牌提交到 Git。
- 禁止把上述内容写入日志、历史记录、测试快照、截图、异常消息、ZIP、manifest 或发布包。
- 禁止在终端输出中打印 Cookie 内容；诊断只允许记录来源名称、是否存在、文件大小、修改时间、失败阶段和脱敏错误摘要。
- 禁止读取或上传与当前站点无关的浏览器 Cookie。
- 禁止把 `bilibili-cookies.txt` 当作正式工具文件随应用分发。
- Cookie 导入、登录或刷新必须由用户主动触发，并明确提示其敏感性。
- 删除或替换 Cookie 前必须明确目标文件，禁止使用通配符递归处理用户浏览器目录。

### 推荐存储位置

新实现应优先使用：

```text
%LOCALAPPDATA%\DoroDownloader\cookies\bilibili-cookies.txt
```

为兼容旧版本，可以在新路径不存在时读取：

```text
<应用目录>\tools\bilibili-cookies.txt
```

不得自动删除、移动或覆盖旧 Cookie；迁移必须经过测试并保持可回滚。

### Cookie 获取方式

优先提供以下用户可控方式：

1. 在应用内导入 Netscape/Mozilla 格式 Cookie 文件。
2. 使用 WebView2 打开站点官方登录页面，只取得目标站点所需 Cookie。
3. 在用户明确选择时，从 Firefox 或指定浏览器读取 Cookie。
4. Chrome/Edge 数据库读取只能作为可选来源，不能作为 Windows 上唯一的认证方案。

Netscape Cookie 文件首行必须是以下之一：

```text
# HTTP Cookie File
# Netscape HTTP Cookie File
```

不得建议用户安装来源不明的 Cookie 扩展。若提及扩展，必须引用 yt-dlp 官方文档认可的方式并附带凭证风险说明。

参考：<https://github.com/yt-dlp/yt-dlp/wiki/FAQ#how-do-i-pass-cookies-to-yt-dlp>

## 四、Bilibili 访问与回退策略

Bilibili 的 HTTP 412 是服务器拒绝请求，不等同于 FFmpeg 错误，也不保证能通过客户端代码永久消除。

### 来源计划

新实现的推荐顺序为：

```text
本地 Cookie 文件
→ 用户选定的 Firefox/浏览器来源
→ Chrome
→ Edge
→ 匿名
```

当前实现若尚未支持用户选择浏览器，可以保持已测试的兼容顺序，但任何调整都必须由测试固定并与 README 一致。

### 允许继续下一来源的失败

只有与凭据或站点拒绝相关的失败才允许切换来源，例如：

- Cookie 文件过期、无效或被服务器拒绝。
- 浏览器 Cookie 数据库不存在、被锁定、复制失败或权限不足。
- ABE/DPAPI Cookie 解密失败。
- 需要新鲜 Cookie。
- Bilibili 网页、WBI 或 playurl 请求返回可归因于凭据/风控的拒绝。
- HTTP 412，并且计划中还有尚未尝试的合法来源。

### 必须立即停止的失败

以下失败不得通过更换 Cookie 来源反复请求站点：

- 用户取消。
- 网络明确中断或超时达到策略上限。
- 保存目录或磁盘写入失败。
- 视频分片已经开始下载后的确定性媒体错误。
- FFmpeg/ffprobe 合并、探测或后处理失败。
- 程序内部参数、JSON 解析或不变量错误。

### 重试限制

- 每个来源每个阶段默认只尝试一次。
- 不得无限快速重试 HTTP 412、403 或 429。
- Bilibili 重试应有明确的等待时间和总次数上限。
- 多次 412 后必须停止，并提示用户稍后重试、刷新登录态或在合法范围内更换正常网络环境。
- 不得使用高频代理轮换、伪造账户、私有 API 逆向或其他方式规避平台访问控制。

已知上游问题：

- <https://github.com/yt-dlp/yt-dlp/issues/14830>
- <https://github.com/yt-dlp/yt-dlp/issues/12013>

## 五、浏览器指纹与请求参数

对于已验证需要浏览器指纹的站点，可以使用 yt-dlp 支持的 impersonation，例如：

```text
--impersonate chrome
```

必须遵守：

- 使用前验证当前 yt-dlp 构建确实有可用 impersonation target。
- impersonation 是辅助措施，不能替代有效 Cookie，也不能保证消除服务器端风控。
- 不得为了“像浏览器”而硬编码过期 User-Agent；若 Cookie 与 User-Agent 需要匹配，应由同一登录流程生成或明确记录来源。
- 不得全局强制不必要的 impersonation，以免损害下载速度和稳定性。
- 命令参数必须通过 `ProcessStartInfo.ArgumentList` 或等价 argv API 逐项传递，禁止拼接 shell 命令。

参考：<https://github.com/yt-dlp/yt-dlp#network-options>

## 六、发布与打包必须原子化

禁止直接删除正在运行或可能被占用的正式发布目录后再就地发布。打包必须使用 staging 流程：

1. 解析并验证 staging、正式发布目录、ZIP 和 manifest 的绝对路径。
2. staging 必须位于明确允许的产物目录中，且不能是仓库根目录、用户主目录或磁盘根目录。
3. 在新的 staging 目录执行 `dotnet publish`。
4. 只复制明确列出的运行工具和文档。
5. 断言 staging 和 ZIP 中不存在 Cookie、下载视频、日志、`.part` 或 `.ytdl` 文件。
6. 运行工具版本检查、构建检查和 EXE 启动检查。
7. 所有检查通过后再以可恢复方式替换正式发布目录。
8. 失败时删除或保留明确标记的 staging，不得破坏上一次可用发布包。
9. 生成 ZIP 和 SHA-256 manifest，并验证 ZIP 可重新解压运行。

发布脚本必须在任一步骤失败时返回非零退出码，不得继续打印“发布成功”。

## 七、错误分类和用户提示

核心层必须保存稳定的失败阶段，界面只显示安全、简洁、可操作的摘要。

至少区分：

```text
MissingTools
InvalidUrl
UnsupportedSite
BrowserDatabaseUnavailable
CookieDecryption
FreshCookiesRequired
Webpage
Wbi
PlayurlApi
MediaFragment
Network
FileSystem
PostProcessing
Cancelled
Unknown
```

具体枚举名称可以与当前代码兼容，但语义不得混淆。

错误分类必须优先匹配更具体的文本。例如，Cookie 数据库被锁或复制失败不能因为同一段日志包含 `decrypt` 就误报为 DPAPI 解密错误。

用户提示必须回答：

1. 失败发生在哪个阶段。
2. 已尝试过哪些不含敏感信息的来源。
3. 用户现在可以采取什么合法操作。
4. 重试是否安全、是否值得。

禁止在界面展示整段 `--verbose` 调试日志。完整诊断若需要保存，必须经过脱敏，并由用户主动导出。

## 八、成熟项目模式的采用原则

可以借鉴 Seal、YTDLnis 等项目已经验证的产品模式：

- 应用内 Cookie 导入、刷新和删除。
- 获取信息失败后提供明确的登录/刷新入口。
- 用户完成登录后自动重试一次。
- yt-dlp stable/nightly 更新通道及回滚。
- Cookie 与匹配 User-Agent 的一致管理。
- 可读的任务日志、重试和取消。

参考：

- <https://github.com/JunkFood02/Seal>
- <https://github.com/deniscerri/ytdlnis>

借鉴行为和架构时必须核对许可证，不得未经审查复制大段第三方实现。上游项目自身仍可能受到站点风控，不能把“其他 GUI 能下载”当作永久保证。

## 九、实现工作流

所有功能、修复和发布工作必须遵循：

1. 读取完整错误和相关日志，建立稳定复现。
2. 检查 Git 状态和最近变更，保护用户未提交内容。
3. 明确根因和失败组件，不得先猜测修复。
4. 为根因创建最小失败测试或可重复诊断用例。
5. 一次只改变一个关键变量。
6. 实现针对根因的最小修复，不顺带重构无关代码。
7. 运行目标测试、完整核心测试、Release 构建和工具验证。
8. 涉及发布时额外运行包内容、ZIP、Cookie 泄露和启动检查。
9. 涉及真实下载时使用公开、非 DRM、此前未下载的短视频验收。
10. 交付时报告实际运行的命令、退出码、输出文件和仍存在的限制。

若连续三个修复假设都失败，必须停止继续堆叠补丁，重新评估架构并向用户说明。

## 十、强制验证门槛

### 源码门槛

```powershell
dotnet run --project .\tests\DoroDownloader.Tests\DoroDownloader.Tests.csproj -c Release
dotnet build .\DoroDownloader.sln -c Release --nologo
powershell -NoProfile -ExecutionPolicy Bypass -File .\scripts\verify-tools.ps1
git diff --check
```

要求：

- 核心测试全部通过。
- Release 构建无项目警告和错误。
- 四个工具验证通过。
- `git diff --check` 通过。
- 不相关的用户改动仍然保留。

### 发布门槛

- 发布目录包含应用和全部必需工具。
- 发布目录与 ZIP 中不存在任何 Cookie 文件。
- ZIP 和 manifest 成功生成并完成哈希验证。
- 从最终发布目录启动 `DoroDownloader.exe`，进程至少稳定运行三秒。
- 关闭验证进程后无残留文件锁。
- 上一次可用发布包未因失败流程被破坏。

### 真实下载门槛

- 使用一个公开、非会员、非 DRM、非直播的短视频。
- 元数据解析成功。
- 下载退出码为 0。
- 最终文件存在且非空。
- ffprobe 检测到预期的视频流和音频流。
- 报告视频 ID、标题、分辨率、编码、时长、文件大小和绝对路径。
- 验收日志不包含 Cookie、请求头或浏览器数据库内容。

真实网络测试可能因站点或环境波动失败。此时必须准确报告失败阶段，不得为了得到绿色结果隐藏失败或改用未经授权的规避手段。

## 十一、完成定义

只有同时满足以下条件，才能声称下载功能或发布包“可用”：

1. 根因已经被证据确认。
2. 相应自动化测试通过。
3. Release 构建和工具验证通过。
4. 最终发布包通过内容与启动验证。
5. 在任务明确要求或改动涉及下载链路时，真实公开短视频验收通过。
6. Cookie、媒体和敏感日志未进入 Git 或发布包。
7. 已知服务端限制、未解决上游 issue 和适用边界已明确告知用户。

不得仅凭“代码来自 GitHub”“测试替身通过”“程序能打开”或“某次解析成功”宣称整个下载流程可靠。

## 十二、明确禁止事项

- 禁止绕过 DRM、会员权限、付费墙、验证码或账户访问控制。
- 禁止高频轮换代理、伪造账户或逆向私有 API 以规避风控。
- 禁止提交、打包、记录或展示 Cookie 和认证令牌。
- 禁止使用 `git reset --hard`、`git checkout --` 或 `git clean -fdx` 清理用户工作区，除非用户明确要求并确认目标。
- 禁止对仓库根目录、用户主目录、磁盘根目录或未验证路径执行递归删除。
- 禁止在打包失败、验证未执行或真实测试失败时声称工作完成。
- 禁止为了追求下载成功而吞掉非零退出码或把失败历史记录成成功。

