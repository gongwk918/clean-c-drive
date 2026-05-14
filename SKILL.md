---
name: clean-c-drive
description: >-
  在 Windows 系统上系统化清理 C 盘空间。触发词:
  "清理C盘" "C盘满了" "C盘空间不足" "释放C盘空间" "C盘清理" "磁盘满了"
  "C盘爆了" "C盘红了" "clean c drive" "free up c drive" "disk full".
  按五阶段顺序执行: 1) 安全清理缓存/临时文件 2) 转移开发工具缓存到D盘
  3) Windows 自带清理工具处理系统保护路径 4) 卸载不用的程序 5) 按时间删除大型应用数据.
  强制约束: 每个删除/卸载前必须先扫描列清单, 详细说明每项用途与影响, 等用户明确确认才执行.
---

# Clean C Drive

系统化清理 Windows C 盘空间。**强制原则**:每次删除/卸载前必须列出候选项、说明用途与风险、等用户明确确认才执行。

## 工作准则（必须遵守）

1. **先量再删** - 任何清理动作前都要先统计大小和文件数,给用户看清楚再操作
2. **解释清楚** - 每项删除/卸载,说明它是什么、删了有什么影响、是否可逆
3. **明确确认** - 等用户具体说"清第 1、3、5 项",不要按"全部"或"建议"擅自扩大范围
4. **分批执行** - 先做无风险项,再做有风险项;先做不可逆少的,再做不可逆多的
5. **核对结果** - 每批执行后报告:本轮释放 X GB、累计释放 Y GB、剩余 Z GB
6. **使用绝对路径** - PowerShell 操作全部用绝对路径,不依赖 cwd

---

## 阶段 0: 现状评估

任务开始就跑,先看 C 盘整体瓶颈在哪。

```powershell
# C 盘容量
Get-PSDrive C | Select-Object @{N='UsedGB';E={[math]::Round($_.Used/1GB,2)}},@{N='FreeGB';E={[math]::Round($_.Free/1GB,2)}},@{N='TotalGB';E={[math]::Round(($_.Used+$_.Free)/1GB,2)}} | Format-List

# C 盘顶层各目录大小
Get-ChildItem C:\ -Force -ErrorAction SilentlyContinue | Where-Object { $_.PSIsContainer } | ForEach-Object {
    $size = (Get-ChildItem $_.FullName -Recurse -Force -ErrorAction SilentlyContinue | Measure-Object -Property Length -Sum -ErrorAction SilentlyContinue).Sum
    [PSCustomObject]@{ Name=$_.Name; SizeGB=[math]::Round($size/1GB,2) }
} | Sort-Object SizeGB -Descending | Format-Table -AutoSize
```

根据扫描结果决定接下来哪些阶段优先做。通常顺序是 1 → 2 → 4 → 3 → 5。

---

## 阶段 1: 安全可清（无风险缓存）

**特征**:删了不影响功能,大多数下次自动重建。这一类**可以批量清,但仍要先告知用户清单**。

### 候选清理项

| 类别 | 路径 | 用途 | 删除影响 |
|---|---|---|---|
| 用户临时 | `C:\Users\<user>\AppData\Local\Temp` | 系统/应用临时文件 | 无,部分活动文件会跳过 |
| 崩溃转储 | `C:\Users\<user>\AppData\Local\CrashDumps` | 程序崩溃 dump | 无,除非要排查崩溃 |
| npm 缓存 | `C:\Users\<user>\AppData\Local\npm-cache` | npm 下载缓存 | 无,下次 install 重建 |
| Go 编译 | `C:\Users\<user>\AppData\Local\go-build` | Go 编译缓存 | 无,首次构建变慢 |
| Chrome 缓存 | `<Chrome User Data>\<Profile>\Cache`,`Code Cache`,`GPUCache` | 网页资源缓存 | 无,书签/密码/历史/Cookie 不受影响 |
| 旧版应用残留 | `AppData\Local\<App>_<oldver>` | 升级后遗留 | 无,新版本不依赖 |
| 企微升级包 | `AppData\Roaming\Tencent\WXWork\upgrade` | 企微升级包暂存 | 无,自动重建 |
| 企微小程序缓存 | `AppData\Roaming\Tencent\WXWork\Applet`,`wwmapp` | 小程序运行缓存(看 LastWriteTime,几年前的可清) | 无 |
| 回收站 | 所有盘的 `$RECYCLE.BIN` | 已删除的文件 | 无,已删的彻底清掉 |

### Chrome 缓存重要说明

**只删这三个子目录**:`Cache`, `Code Cache`, `GPUCache`

**绝对不删**:`Bookmarks`(书签), `Login Data`(密码), `Cookies`(登录状态), `History`(历史), `Preferences`(设置/扩展)

操作前确认 Chrome 未运行:`if (-not (Get-Process chrome -ErrorAction SilentlyContinue)) { ... }`

### 执行模板

```powershell
$initial = (Get-PSDrive C).Free
$targets = @(
    @{ Name='npm-cache';     Path='C:\Users\kang\AppData\Local\npm-cache' },
    @{ Name='CrashDumps';    Path='C:\Users\kang\AppData\Local\CrashDumps' },
    @{ Name='go-build';      Path='C:\Users\kang\AppData\Local\go-build' },
    # ...
)
foreach ($t in $targets) {
    if (Test-Path $t.Path) {
        try {
            # 清内容保留顶层目录,避免应用启动时找不到目录
            Get-ChildItem $t.Path -Recurse -Force -ErrorAction SilentlyContinue | Remove-Item -Recurse -Force -ErrorAction SilentlyContinue
            Write-Output ("[OK] {0}" -f $t.Name)
        } catch {
            Write-Output ("[WARN] {0}: {1}" -f $t.Name, $_.Exception.Message)
        }
    }
}
Clear-RecycleBin -Force -ErrorAction SilentlyContinue
$final = (Get-PSDrive C).Free
Write-Output ("释放: {0} GB" -f [math]::Round(($final-$initial)/1GB,2))
```

---

## 阶段 2: 转移到 D 盘（永久收益）

**特征**:不删除任何数据,只是改默认路径让新数据写到 D 盘,然后删 C 盘旧数据。**前提:D 盘有足够空间**。

### 转移目标对照表

| 工具 | C 盘位置 | 迁移方式 |
|---|---|---|
| npm 缓存 | `AppData\Local\npm-cache` | `npm config set cache "D:\dev-cache\npm"` |
| npm 全局包 | `AppData\Roaming\npm` | `npm config set prefix "D:\dev-cache\npm-global"` (需更新 PATH) |
| Gradle | `C:\Users\<user>\.gradle` | 环境变量 `GRADLE_USER_HOME=D:\dev-cache\.gradle` |
| Maven | `C:\Users\<user>\.m2` | 改 `~/.m2/settings.xml` 的 `<localRepository>` |
| Go 缓存 | `AppData\Local\go-build` | 环境变量 `GOCACHE=D:\dev-cache\go-build` |
| Go modules | `C:\Users\<user>\go` | 环境变量 `GOPATH=D:\dev-cache\go` |
| pip 缓存 | `AppData\Local\pip\Cache` | `pip config set global.cache-dir D:\dev-cache\pip` |
| Yarn 缓存 | - | `yarn config set cache-folder "D:\dev-cache\yarn"` |
| bun | `C:\Users\<user>\.bun` | 环境变量 `BUN_INSTALL=D:\dev-cache\.bun` |
| pnpm | - | `pnpm config set store-dir "D:\dev-cache\pnpm-store"` |
| Docker Desktop | `AppData\Local\Docker` | Settings → Resources → Disk image location |

### 标准迁移流程

1. 确认 D 盘空间
2. 创建目标目录:`New-Item -ItemType Directory -Force -Path 'D:\dev-cache\<tool>'`
3. 设置配置(npm/yarn 用 config 命令;Gradle/Go 用环境变量)
4. 验证配置生效:`<tool> config get <key>` / `[Environment]::GetEnvironmentVariable(...)`
5. 删除 C 盘旧缓存

**环境变量永久设置**(系统级):
```powershell
[Environment]::SetEnvironmentVariable('GRADLE_USER_HOME','D:\dev-cache\.gradle','User')
```

---

## 阶段 3: Windows 自带清理（系统保护路径）

PowerShell **删不掉**的系统路径,必须用 Windows 自带工具。

### 候选目标

| 路径 | 用途 | 安全删除方式 |
|---|---|---|
| `C:\$WINDOWS.~BT` | Windows 升级残留 | cleanmgr |
| `C:\Windows.old` | 旧版 Windows 备份 | cleanmgr |
| `C:\Windows\SoftwareDistribution\Download` | Windows Update 缓存(1-5GB) | cleanmgr |
| `C:\Windows\Temp` | 系统临时文件 | cleanmgr |
| 内存转储 | 系统蓝屏 dump | cleanmgr |
| 缩略图缓存 | 资源管理器缩略图 | cleanmgr |

### 引导用户操作

```powershell
# 直接帮用户打开磁盘清理工具
cleanmgr /d C:
# 或者只打开"清理系统文件"模式(需管理员)
Start-Process cleanmgr -ArgumentList '/d C: /VERYLOWDISK' -Verb RunAs
```

**告诉用户的步骤**:
1. 选 C 盘 → 点 **"清理系统文件"**(关键,给管理员权限才看到系统项)
2. 勾选:Windows 更新清理 / 以前的 Windows 安装 / 传递优化文件 / 临时文件 / 缩略图 / 内存转储
3. 确定

**不要试图自己删 `$WINDOWS.~BT`** - PowerShell 会直接报"系统保护路径被阻止",且会中断当前批量脚本。如果非要删,在 elevated TrustedInstaller 上下文里走 takeown + icacls,但风险大,不推荐。

---

## 阶段 4: 程序卸载

**特征**:不可逆,需要 UAC 提权。卸载前**必须**逐个确认。

### 4.1 收集已安装程序

```powershell
$paths = @(
    'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*',
    'HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*',
    'HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*'
)
$apps = foreach ($p in $paths) {
    Get-ItemProperty $p -ErrorAction SilentlyContinue | Where-Object {
        $_.DisplayName -and -not $_.SystemComponent -and -not $_.ParentKeyName -and
        $_.DisplayName -notmatch '^(KB\d+|Update for|Security Update|Hotfix)'
    }
}
# 输出: DisplayName, DisplayVersion, EstimatedSize, InstallLocation, UninstallString, QuietUninstallString
```

注册表数据不全。要补充扫 `C:\Program Files`、`C:\Program Files (x86)`、`C:\Users\<user>\AppData\Local` 的实际目录大小,才能看到真实占用。

### 4.2 ⚠️ 关键: 识别程序实际安装位置

**不是所有"列在 C 盘已装程序"的真的占 C 盘空间**。要查 `InstallLocation` 字段:

- `InstallLocation` 是 `D:\...` → 程序本体在 D 盘,C 盘只有 AppData 数据残留
- 用户**还在用**这个程序 → **不卸载,数据残留也别动**(删了配置会丢)
- 用户**不再用** → 卸载(卸载器在 D 盘),然后清 C 盘 AppData 残留

### 4.3 分组执行策略

| 组 | 特征 | 处理方式 |
|---|---|---|
| A | MSI 包(`UninstallString` 形如 `MsiExec.exe /X{GUID}`) | `msiexec /x {GUID} /qn /norestart` 静默 |
| B | EXE 卸载器,支持 `/S` `/silent` `/quiet` `/VERYSILENT` | 加参数静默调用 |
| C | EXE 卸载器,必须图形界面 | 引导用户走"设置→应用" |
| D | 无注册表入口(绿色程序/数据残留) | 直接删目录,先确认无进程 |

**MSI 退出码**:
- 0 = 成功
- 3010 = 成功但**需要重启**才能完成清理(必须告知用户)
- 1605 = 产品未安装
- 其他 = 失败,查 `%TEMP%\MSI*.log`

### 4.4 提权卸载脚本模板

把卸载脚本写到 **D 盘**(避免 C 盘 Temp 清理影响),然后用 `-Verb RunAs` 启动:

```powershell
# Write to D:\dev-cache\cleanup-admin.ps1
@'
$log = "D:\dev-cache\cleanup-admin.log"
"=== 开始: $(Get-Date) ===" | Out-File $log -Encoding UTF8

function Log($msg) { $msg | Out-File $log -Append -Encoding UTF8; Write-Host $msg }

# 删目录 (Program Files 下需要管理员权限)
$p = "C:\Program Files\<AppName>"
if (Test-Path $p) {
    try { Remove-Item $p -Recurse -Force -ErrorAction Stop; Log "[OK] $p" }
    catch { Log "[FAIL] $p : $($_.Exception.Message)" }
}

# MSI 静默卸载
$msis = @(
    @{Name='<App>'; Guid='{GUID-1}'},
    @{Name='<App2>'; Guid='{GUID-2}'}
)
foreach ($m in $msis) {
    $proc = Start-Process msiexec -ArgumentList "/x $($m.Guid) /qn /norestart" -Wait -PassThru
    Log ("[退出码={0}] {1}" -f $proc.ExitCode, $m.Name)
}

"=== 完成: $(Get-Date) ===" | Out-File $log -Append -Encoding UTF8
Write-Host "`n按任意键关闭..."
$null = $Host.UI.RawUI.ReadKey('NoEcho,IncludeKeyDown')
'@ | Out-File 'D:\dev-cache\cleanup-admin.ps1' -Encoding UTF8

# 启动提权窗口
Start-Process pwsh -ArgumentList '-NoProfile','-ExecutionPolicy','Bypass','-File','D:\dev-cache\cleanup-admin.ps1' -Verb RunAs
```

**关键点**:
- 用 `Start-Process msiexec -Wait -PassThru` 同步执行并取 `ExitCode`
- 日志写到 D 盘(不要写 `C:\Temp`,会被阶段 1 清掉或被自删)
- 结尾 `ReadKey` 让窗口保留,用户看完结果再关闭
- 脚本启动后告诉用户:"请在 UAC 弹窗点'是',新窗口会显示进度,完成后按任意键关闭"

### 4.5 卸载后扫残留

```powershell
$residues = @(
    'C:\Program Files\<App>',
    'C:\Program Files (x86)\<App>',
    'C:\Users\<user>\AppData\Local\<App>',
    'C:\Users\<user>\AppData\Roaming\<App>',
    'C:\ProgramData\<App>'
)
```

如果 MSI 有 3010 退出码,**强制提醒用户重启**才会真正释放(VMware/Pulse Secure 等典型)。

### 4.6 一些已知的需小心的程序

- **Sangfor / EasyConnect / SangforVNC / VDI / iSecStar / iSecPrint / secoresdk** - 深信服系列,**公司装的话别动**,会失去内网访问能力
- **HP/Huawei/Lenovo/Dell 厂商工具** - 涉及驱动,可能影响硬件功能
- **NVIDIA 驱动** - 卸载用 DDU,不要直接删目录
- **Visual C++ Redistributable** - 多版本是正常的,被无数程序依赖,不要清理

---

## 阶段 5: 按时间清理大型应用数据

**特征**:数据量大、用户希望保留近期数据。**最高风险**,必须每个具体路径逐项确认。

### 典型应用

| 应用 | 数据可能位置 | 关键子目录 |
|---|---|---|
| 微信新版(xwechat 4.0) | `D:\xwechat_files\<wxid>` 或 `<Documents>\xwechat_files` | `msg`,`FileStorage`,`Image`,`Video` |
| 微信旧版 | `<Documents>\WeChat Files\<wxid>` | `FileStorage`,`Image`,`Video`,`Voice` |
| 企业微信 | `<Documents>\Tencent Files\<id>` | 同上结构 |
| 钉钉 | `D:\DingTalkAppData\<id>` 或 `<Documents>\DingTalkAppData` | 文件接收/缓存目录 |
| 飞书 | `AppData\Local\Lark` | 媒体缓存 |
| QQ | `<Documents>\Tencent Files\<QQ号>` | `FileRecv`,`Image` |
| 网盘 | 自定义目录 | 同步缓存 |

**重要**:先用 Test-Path 探,**别假设**位置。新版微信可能整个不在 C 盘。

### 核心原则

- **配置/数据库不能删**:`.db`, `.dat`, `Msg/` 索引目录, `config/` 等
- **只删媒体文件目录**:`FileStorage`, `Image`, `Video`, `Voice`, `Cache`, `Files`
- **按 LastWriteTime 过滤**:超过用户指定时间的才删

### 执行模板

```powershell
$mediaPath = 'D:\xwechat_files\wxid_xxx\msg\file'  # 必须是用户确认过的具体路径
$cutoff = (Get-Date).AddMonths(-12)  # 用户指定的时间界限

# 第一步:量(必做)
$old = Get-ChildItem $mediaPath -Recurse -File -Force -ErrorAction SilentlyContinue |
    Where-Object { $_.LastWriteTime -lt $cutoff }
$count = $old.Count
$sizeGB = [math]::Round(($old | Measure-Object Length -Sum).Sum / 1GB, 2)
Write-Output "$count 个文件,共 $sizeGB GB,最早 $(($old | Sort-Object LastWriteTime | Select -First 1).LastWriteTime)"

# 第二步:展示几个示例文件让用户判断范围对不对
$old | Sort-Object Length -Descending | Select -First 5 FullName, Length, LastWriteTime

# 第三步:用户明确"删"才执行
# $old | Remove-Item -Force -ErrorAction SilentlyContinue
```

---

## 强制工作流(每次任务)

```
1. 阶段 0: 扫现状 → 报告整体情况
2. 跟用户确认:从哪个阶段开始?(通常优先 1→2→4)
3. 进入阶段 N:
   a. 扫描候选项 → 列表 + 大小 + 用途说明
   b. 给清单,等用户挑(具体说"清 1,3,5"或"不清"或"暂跳过这个阶段")
   c. 执行被选中的项,记录每项结果
   d. 核对:本轮释放、累计释放、当前剩余
4. 进入下一阶段
5. 结束时给汇总:起点剩余 X GB → 终点 Y GB,累计释放 Z GB
```

---

## 已知坑（实战经验）

### 1. `$WINDOWS.~BT` 是死路
PowerShell 报"Remove-Item on system path is blocked",且**整个批量脚本会中断**。如果它在删除列表里,前面的目标也会被一起跳过。**永远把它放到 cleanmgr 阶段处理,绝不放进 PowerShell 批量**。

### 2. 删 Temp 会自删脚本输出
Claude Code 任务输出文件就在 `C:\Users\<user>\AppData\Local\Temp\claude\...\tasks\` 下。删 `Temp` 会把脚本自己的输出文件也删了,导致读不到输出。**这不代表脚本失败**——脚本通常已执行完。处理方法:
- 让 Bash/PowerShell 调用读不到输出时,立即用单独的命令查 PSDrive 状态 + 关键目录大小核对
- 或者把日志写到 D 盘

### 3. Chrome 缓存目录区分
- `Cache`/`Code Cache`/`GPUCache` → 缓存,可删
- `Bookmarks`/`Login Data`/`Cookies`/`History`/`Preferences` → 用户数据,**绝不删**
- 如果整个 `User Data\<Profile>` 都删了 → 失去所有书签/密码,只能从 Google 同步恢复

### 4. MSI 退出码 3010
表示"成功但需要重启"。**任何卸载收尾如果出现 3010,必须明确告诉用户重启**,否则用户会以为"卸了但空间没释放"。

### 5. 微信数据可能不在 C 盘
新版微信(`xwechat` 4.0)默认把用户数据存在 `<盘>:\xwechat_files\<wxid>`,**很可能就在 D 盘**。先 `Test-Path` 探多个候选位置,不要假设在 C 盘。

### 6. 程序本体 vs 数据残留
查 `InstallLocation` 字段。常见情况:程序装在 D 盘(`D:\软件\xxx`)但 C 盘 `AppData\Local\<app>` 有几百 MB 数据。**如果程序还在用,数据残留不能删**(删了配置/登录态会丢)。

### 7. D 盘做工作目录
- 临时脚本写 `D:\dev-cache\*.ps1`(不要写 `C:\Temp\`,会被自己清掉)
- 日志写 D 盘
- 迁移目标也放 D 盘
- 这样阶段 1 的清理不会影响阶段 4 的工作

### 8. 一些 ENG 不知道的中文 Windows 命名
- "已安装的应用" = `ms-settings:appsfeatures` URI
- 磁盘清理 = `cleanmgr.exe`
- 中文用户名/路径要用引号或 escape

---

## 快速命令清单

```powershell
# 看 C 盘容量
Get-PSDrive C | Select Used,Free,@{N='UsedGB';E={[math]::Round($_.Used/1GB,2)}},@{N='FreeGB';E={[math]::Round($_.Free/1GB,2)}}

# 扫顶层目录大小
Get-ChildItem C:\ -Force | ? PSIsContainer | % { $s=(gci $_.FullName -R -Force -EA 0 | measure Length -Sum -EA 0).Sum; [PSCustomObject]@{N=$_.Name;GB=[math]::Round($s/1GB,2)} } | sort GB -Desc

# 打开"已安装的应用"
Start-Process 'ms-settings:appsfeatures'

# 启动磁盘清理(管理员)
Start-Process cleanmgr -ArgumentList '/d C:' -Verb RunAs

# 清空回收站
Clear-RecycleBin -Force

# 提权运行脚本
Start-Process pwsh -ArgumentList '-NoProfile','-ExecutionPolicy','Bypass','-File','D:\dev-cache\xxx.ps1' -Verb RunAs
```
