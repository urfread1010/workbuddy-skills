---
name: personal-folder-cleanup
description: 清理用户个人目录（Downloads / Desktop 等）里的污染资源：只读扫描分类 → 用户确认 → 垃圾送回收站、自有内容迁移到 Documents。适用于"桌面/下载太乱了，帮我收拾一下"类请求。
---

# 个人目录清理（Downloads / Desktop）

## 触发
用户说"清理桌面/下载""Downloads 里很多垃圾""把下载区收拾干净"等。本机用户为 Windows（31310），用 Git Bash / PowerShell。

## 铁律（个人目录高风险，必须遵守）
1. **scan = 只读**：第一遍只列清单（路径+大小+日期），绝不移动/删除/重命名。
2. **删除只送回收站，不用 rm/del**：用下面 ctypes 调 `SHFileOperationW` + `FOF_ALLOWUNDO`，可还原。
3. **迁移用 shutil.move**（同盘原子改名），不丢数据；**迁移前对每个文件做重复比对**（见下"文件级重复比对"），绝不静默覆盖或产生双份。
4. **先分类给用户确认**：把项目分档（见下），列出精确清单 + ⚠️ 警告，等用户说"执行"再动手。
5. 系统目录（Desktop/Downloads/Documents/Home）**绝不递归删/清空**。

## 分类四档
- 🟢 A 明确污染（零风险删）：`._cache_*.exe`、`.crdownload`、`~$*.docx`（Office 锁）、`desktop.ini`、`temp/` 空目录、重复的同名 `Installer.exe`
- 🟡 B 安装包残留（需用户确认已装）：`WorkBuddy*.exe`、`*.msi`、`SteamSetup.exe`、`ChatGPT Installer*.exe`、各 `Installer.exe`。注意 `.exe` 文件名可能含空格（如 `._cache_ChatGPT Installer.exe`），glob 分类时别误判
- 🔵 C 自有内容（迁移到 Documents/下载整理/）：`.md` 思考稿、`.lua` 脚本、课程/海报 html+jpeg、个人子目录（银行流水/工资/评价表/项目文件夹等）
- ⚪ D 边界模糊（等用户定）：表情包、AI 生成图（用户常直接删）

## 回收站删除（Python ctypes，已验证可跑）
```python
import os, ctypes
from ctypes import wintypes, c_void_p, byref
def recycle(paths):
    class S(ctypes.Structure):
        _fields_=[("hwnd",wintypes.HWND),("wFunc",wintypes.UINT),
                  ("pFrom",wintypes.LPCWSTR),("pTo",wintypes.LPCWSTR),
                  ("fFlags",wintypes.WORD),("aborted",wintypes.BOOL),
                  ("hm",c_void_p),("title",wintypes.LPCWSTR)]
    buf=ctypes.create_unicode_buffer("\0".join(paths)+"\0\0")
    st=S(); st.wFunc=3  # FO_DELETE
    st.pFrom=ctypes.cast(buf, ctypes.c_wchar_p)  # 必须 cast，否则报类型错误
    st.fFlags=0x40|0x10|0x4|0x400  # ALLOWUNDO|NOCONFIRM|SILENT|NOERRORUI
    return ctypes.windll.shell32.SHFileOperationW(byref(st))  # 返回 0 = 成功
```
- `pFrom` 必须是 `c_wchar_p`（cast 自 buffer），直接赋值 c_wchar_Array 会 TypeError。
- 多文件用 `\0` 分隔 + 双 `\0` 结尾。
- 返回非 0 → 回退方案：移到 `C:\Users\31310\._downloads_trash\<时间戳>\`，并报告。

## 重复目录处理（重要）
迁移脚本若因"目标已存在"跳过，可能留下**源和目标两份**。执行后必须比对：
- 比对两目录的 `{相对路径: 大小}` 集合。
- 若源是子集（"仅源有"为空）→ 源是冗余副本，直接回收站删除，零损失。
- 若有差异 → 停下人工确认，绝不自动覆盖。

## 文件级重复比对（高频坑，已踩 2 次）
同一文件常**既在 Downloads 根目录、又已在某次已迁的目标子目录里**。迁移每个文件前必须：
```python
import hashlib, os
def md5(p):
    h=hashlib.md5()
    with open(p,"rb") as f:
        for b in iter(lambda:f.read(1<<20),b""): h.update(b)
    return h.hexdigest()
# 若目标已存在同名文件：
if os.path.getsize(src)==os.path.getsize(tgt) and md5(src)==md5(tgt):
    recycle(src)          # 一致 → 回收源副本，零损失
else:
    shutil.move(src, tgt.replace(".xlsx","__Downloads副本.xlsx"))  # 不一致 → 改名保两份
```
- 也适用于目录：比对 `{相对路径: 大小}` 集合判断是否为子集。
- 绝不先 `mv` 再比，先比再决定，避免产生双份后难清理。

## 执行顺序
1. 只读扫描，生成四档清单 → 呈现给用户 + ⚠️ 警告。
2. 用户确认范围后 → 先迁移（C 档），再回收站删除（A/B/AI图），最后删空目录（`软件/` 等）。
3. 终检：列出剩余项，确认目标项已清空；生成执行日志。

## 已知坑
- `SHFileOperationW` 的 `pFrom` 字段类型必须用 `ctypes.cast(buf, ctypes.c_wchar_p)`，否则 `TypeError: incompatible types, c_wchar_Array_xxxx instead of c_wchar_p`。
- 脚本若在"回收站"步骤前崩溃，"迁移"步骤可能已跑完——重跑时会因"目标已存在"跳过，留下重复目录，务必比对。
- 桌面 `books` 用 Git Bash `mv` 可能 Permission denied（OneDrive/应用锁），改用 PowerShell `robocopy /MOV /E` 逐文件搬。
- 空壳目录被应用锁死（如 Typora 锁 `books\markdown`）无法删除，记录待用户关闭应用后再清，不擅自杀进程。

## AppData 大目录缓存清理（最常见的"隐形胖子"）
AppData（Local/Roaming/LocalLow）是应用私仓，极易膨胀到几十 GB。先只读扫描按"是什么/能不能清"分档，再分批送回收站。
- **只读扫描**：`os.walk` 聚合每个顶层子目录体积，输出 Top N（同 disk_scan 思路）。区分：
  - 🟢 纯缓存（放心送回收站，可还原）：`Notion`/`ima.copilot`/`AMD`(显卡驱动缓存)/`Google`(Chrome 缓存)/`npm-cache`/`ms-playwright`/`Cypress`/各 AI 助手缓存/`JianyingPro`(剪映草稿+缓存)。
  - 🔴 真实数据（别动或先确认）：`Tencent`(微信/QQ 聊天+收的文件)/`LarkShell`(飞书，云端有同步副本可清)/`baidu`/`Quark`(网盘，可能含你下载的文件)/`OneDrive` 同步副本(在 `Microsoft` 内)。
  - ⚪ 看情况：`dev.leaguetoolkit`(游戏 MOD)、`Code`(VS Code 配置+扩展)、`Programs`(便携程序)。
- **两段式送回收站（关键）**：
  1. 先尝试把**整个文件夹**整体移回收站（`SHFileOperationW` + `FOF_ALLOWUNDO`）。
  2. 若返回 `code 32`（sharing violation，某文件被进程锁）→ 整体移动会**整目录失败**，改**逐文件移动、锁住的单文件跳过**（walk 收集全部文件，逐个 `SHFileOperationW` 移，失败/锁住的记入 locked 列表继续）。
  - 这样能回收 **99% 体积**，仅剩极少数被运行中的程序/服务锁住的小文件（如 ima 的 `ima_context_menu.dll` 被 ima 程序锁、AMD 的 `DxCache`/`RGStats.db` 被 `AMDRSServ` 服务锁——标准用户杀不掉该服务）。
  - 锁文件重启或关掉对应程序后，二次跑同一脚本即可清掉。
- **务必提醒用户**：回收站只是中转，**手动清空回收站才真正释放空间**；且 `Microsoft` 内可能含 OneDrive 同步副本、`baidu`/`Quark` 可能含下载文件，清空回收站前让用户自查。
- `Docker`(WSL vhdx)、`Programs` 等"还得用/含已装程序"的，按用户节奏最后再动。
- **⚠️ 致命坑（已踩，勿犯）**：**绝不对 `AppData\Local\Microsoft` 用逐文件搬迁**。该目录含 Explorer 自身的缓存/缩略图，逐文件搬（数万文件、需几十分钟）时 Explorer 一边开着一边被挪 → **文件管理器白屏卡死**。正确做法：用**子目录级整体搬**（每个 top-level 子目录一次 SHFileOperation，秒级完成，不拖住 Explorer）。被运行中的程序/服务锁住的子目录（Edge/Office/Windows/IE/GameDVR/AMD 服务）会干净地整目录失败，不白屏；关掉对应程序或重启后再搬即可。
- **命名坑**：英雄联盟工具包真实目录是 `dev.leaguetoolkit.manager`（Local+Roaming），不是 `dev.leaguetoolkit`；扫描大小对应的是 `.manager`，写错名字会 SKIP 误判已删。

## 系统级超大目录清理（如 $WINDOWS.~BT / WinSxS）—— 高危易卡死
- **首选官方方法：磁盘清理 (cleanmgr, 管理员)**：Win+R → `cleanmgr` → 选 C: → 点"清理系统文件" → 再选 C: → 勾 **"以前的 Windows 安装(们)"** + **"临时 Windows 安装文件"** → 确定 → 删除文件。系统内部处理 TrustedInstaller 所有权，不卡死、无需脚本。若这些勾选项不出现（已过 10 天回滚期），再走下方手动法。
- **核心难点 = TrustedInstaller 所有权**：`$WINDOWS.~BT`/WinSxS 里的文件 owner 是 `NT SERVICE\TrustedInstaller`，即便管理员**裸 `rd` 也会逐文件 Access Denied 删不掉**。必须先用 `takeown` 把 owner 改成 Administrators，再 `icacls` 赋权，最后 `rd`。
- **关键教训（已踩坑，务必记住）**：`takeown /R` 对 8 万+ 文件会很慢（10~20 分钟），但它是"慢"不是"卡死"。**绝对不要给它设 5 分钟之类的短超时**——会把夺权中途掐断，导致深层文件仍是 TrustedInstaller 所有，rd 全拒（表现为"ALL DONE 但仍在、一堆拒绝访问"）。正确做法：让 takeown 跑完，仅留一个很大的安全上限（如 40 分钟）防真挂起即可。加 `/SKIPDLV` 可显著加速递归。
- **手动三段式（管理员 PowerShell）**：
  1. 先杀锁：`Get-Process TiWorker | Stop-Process -Force` + `Stop-Service TrustedInstaller,wuauserv -Force`（TiWorker 锁 WinSxS 时 takeown 会无限等待，必须先杀；服务停不掉可忽略）。
  2. 夺权：`takeown /F "<路径>" /R /D Y /SKIPDLV`（**不要短超时**，Start-Process + `.Wait(2400000)` 仅作 40min 兜底）。
  3. 赋权：`icacls "<路径>" /grant *S-1-5-32-544:F /T /C /Q`（*S-1-5-32-544 = Administrators SID）。
  4. 删除：`cmd /c rd /s /q "\\?\<路径>"`（`\\?\` 长路径前缀绕 260 字符限制）。
  5. **兜底 Pass2**：若仍残留，再 takeown+icacls+rd 一次（极少数锁文件仍删不掉则提示重启后重跑）。
- **首选"逐文件夹、自底向上、可中断续跑"模式（针对超大树，避免"一次全删又白等"）**：`Get-ChildItem <根> -Recurse -Directory | Sort-Object { ($_.FullName -split '\\').Count } -Descending` 取全部目录、按深度降序，逐个 `takeown /R /D Y /SKIPDLV` → `icacls /grant *S-1-5-32-544:F /T /C /Q` → `cmd /c rd /s /q "\\?\<dir>"`。自底向上删，父目录后处理时子已空，takeown 工作量最小；**每删一个就永久消失**，中途 Ctrl+C 或关窗口都不丢进度，重跑即从剩余继续。比"整片 takeown 完再整片 rd"省心、可观察、可恢复。
- 脚本里把**简单确定的步骤（删小目录、清注册表、清环境变量）前置执行**，最后再攻坚大目录——单点失败也不致全盘失败。
- 此类操作需管理员：本机 `.bat` 自提权重拉不可靠（UAC 静默失败、窗口闪退）、管理员 PowerShell 才持久，由用户手动 Win+X → 终端(管理员) → 粘贴 `-File` 脚本运行。
