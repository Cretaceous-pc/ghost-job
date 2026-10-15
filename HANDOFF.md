# 交接说明：Boss 活跃度筛选 + 关键词失焦修复

> 给下一个会话（人或 agent）看的现状说明。**这个文件只存在于本地分支，请不要带进任何提给上游的 PR。**

## 一句话现状

`ghost-job`（fork 自 [InfernalAzazel/ghost-job](https://github.com/InfernalAzazel/ghost-job)）上做了两件事：给自动投递加了 **Boss 活跃度筛选**（PR 已提），修了 **关键词输入失焦丢失**（待提 PR）。上游 PR、Issue、本地分支、v0.6.9 安装包都在下面。

## 远程与分支拓扑

| 远程 | 地址 | 角色 |
|---|---|---|
| `origin` | `https://github.com/Cretaceous-pc/ghost-job` | 我的 fork |
| `upstream` | `https://github.com/InfernalAzazel/ghost-job` | 上游主仓库 |

```
main                  556aeaf  [origin/main]                上游 v0.6.8，未改动
└── 添加活跃度筛选      1c8b802  [origin/添加活跃度筛选]      ← PR #8 的 head（已推送）
    └── 修复关键词失焦丢失 4b8a215  [origin/修复关键词失焦丢失] ← 当前分支（已推送）
pr-a-screenshots      abcefc4  [origin/pr-a-screenshots]    ← 只放 PR 截图，别删
```

## 提交清单（从下往上看）

| 提交 | 内容 | 关键文件 |
|---|---|---|
| `1481b45` feat: 支持按 Boss 活跃度筛选岗位 | 活跃度档位归一 + 过滤器 + 配置项 + 详情解析 + 老库补列 | `job/boss/filters.py`、`job/boss/jobs.py`、`job/models/__init__.py`、`job/models/{job,search}.py`、`job/ui/pages/config.py`、`job/ui/state/{config,boss}.py` |
| `1c8b802` feat: 岗位管理支持按 Boss 活跃度筛选 | 列表活跃度列 + 档位筛选 + 详情弹窗 | `job/models/job.py`、`job/ui/pages/jobs.py`、`job/ui/state/jobs.py` |
| `590dfcf` fix: 关键词输入后离开输入框也会保存 | 4 个关键词框失焦即提交 | `job/ui/pages/config.py`（`_tag_input`） |
| `4b8a215` chore: 发布 v0.6.9 | 版本号 + release notes | `job/__init__.py`、`.github/release-notes/v0.6.9.md` |

## 上游 PR / Issue

| 对象 | 链接 | 状态 |
|---|---|---|
| Issue #7 | https://github.com/InfernalAzazel/ghost-job/issues/7 | 已开（作者 Cretaceous-pc） |
| **PR #8** | https://github.com/InfernalAzazel/ghost-job/pull/8 | OPEN，head=`添加活跃度筛选`，16 文件 +735/−16 |
| PR 截图 | `https://raw.githubusercontent.com/Cretaceous-pc/ghost-job/pr-a-screenshots/pr-7-screenshots/*.png` | 托管在 `pr-a-screenshots` 分支，**删了图就裂** |

**待办**：关键词失焦修复还没提 Issue / PR。上游规则是「先开 Issue 再提 PR」，且「一个 PR 只做一件事」，所以它应该：

```bash
git switch -c 修复关键词失焦丢失-独立 main
git cherry-pick 590dfcf          # 只摘这一个提交，别带 HANDOFF.md 和版本号
```

## 关键实测事实（省得重新挖）

数据来源：本机 `~/.ghost-job/chrome-profile` 的 Chrome 缓存，我解压出 **433 个真实 `job/detail.json` 响应** + 7 个 `joblist.json` 响应 + 50 张 `/job_detail/*.html` 页面。

| 事实 | 依据 |
|---|---|
| 活跃度文案 = `job/detail.json` → **`zpData.bossInfo.activeTimeDesc`** | 433 个真实响应 |
| 在线标记 = `zpData.bossInfo.bossOnline`（布尔） | 同上 |
| **列表接口 `joblist.json` 没有活跃度文案**，每条只有 `bossOnline` | 真实 key 清单（47 键） |
| `bossOnline=true` **只**出现在「刚刚活跃」（51/127），今日/3日内/本周/本月全为 `false` | 交叉表 → **不能拿它预筛** |
| 实测 12 种措辞：刚刚活跃 127、半年前活跃 116、3日内活跃 39、本月活跃 31、今日活跃 25、5月内活跃 17、2月内活跃 17、3月内活跃 17、本周活跃 14、2周内活跃 13、4月内活跃 9、近半年活跃 3（+5 例 null） | 统计脚本 |
| DOM 侧：`.job-boss-info h2.name` 里是 `<span class="boss-active-time">半年前活跃</span>`，在线时换成 `<span class="boss-online-tag">在线</span>` | 50 张真实详情页 |
| BOSS 搜索 URL **不支持**活跃度参数（服务端只下发 薪资/经验/学历/规模/融资/类型） | 缓存里的真实 URL + 调研 |
| `job/card.json` 是扁平结构，字段在 `zpData.activeTimeDesc` | 调研；代码里用 `AliasChoices` 兼容两种形状 |

实现要点：**文案归一成 13 个档位再比对**（不做逐字枚举，枚举会漏掉「近半年活跃」↔「半年前活跃」这类互不包含的写法）；**读不到就放行**（实测约 1%，不误杀）。

## 环境与常用命令

```powershell
# 依赖（首次）——必须挂 Clash 代理，否则会卡死
$env:HTTP_PROXY='http://127.0.0.1:7897'; $env:HTTPS_PROXY='http://127.0.0.1:7897'; $env:ALL_PROXY='http://127.0.0.1:7897'
uv sync

uv run pytest -q                      # 210 passed
uvx ruff check job tests              # All checks passed
uvx ruff format --check job tests     # 64 files already formatted

# 起界面做人工验证：必须对齐后端端口，否则一直卡在「正在启动」
uv run reflex run --backend-port 18380     # → http://localhost:3000
# 原生桌面窗口（首次要 cargo build，几分钟）
uv run python -m job
```

### ⚠️ reflex-desktop 的端口坑

`DesktopPlugin` 用 `crc32(identifier)` 算出**固定内嵌后端端口 18380**（`reflex_desktop/config.py` 的 `default_port()`，范围 17600–18599）并**在编译期烧进前端**；而 `reflex run` 默认后端在 8000，于是浏览器一直卡在「Ghost Job 正在启动」、控制台是 `ws://127.0.0.1:18380/_event` 连接被拒。**必须 `--backend-port 18380`**（或用 `REFLEX_DESKTOP_DEV_BACKEND_URL` 覆盖）。

## 验证证据（都实跑过）

- `uv run pytest -q` → **210 passed**（新增 36 个用例：档位归一/兜底换算、多选判定、未知放行、详情解析两种形状、卡片流程「活跃度不符不投递且不进 AI」、按档位筛选、老库迁移）
- `uvx ruff check` / `format --check` 全绿
- `uv run reflex compile` → App compiled successfully
- **老库迁移**：拿本机 `ghost-job.db` 的**副本**（458 条岗位 / 140 已投递 / 84 条聊天）跑迁移，11 项验收点全 PASS：数据零丢失、两列补齐、重复启动幂等。原库只多了两列，行数据未动
- **真实界面断言**：配置页卡片存在、下拉 13 档位、勾选状态回读正确；岗位管理按「刚刚活跃」筛选后表格 3 条 = 「共 3 条」；详情弹窗显示「Boss 活跃度：刚刚活跃」；控制台**零错误**
- **关键词失焦**：修复后实测「输入 → 点别处 → 刷新」标签仍在、输入框自动清空、库里已落盘；测试数据已还原

## 打包（v0.6.9，已完成）

产物在 `tauri/src-tauri/target/release/bundle/`：

| 文件 | 大小 | SHA256 |
|---|---|---|
| `nsis/Ghost Job_0.6.9_x64-setup.exe` | 82.3 MB | `b1c9cc441c018d8612610753a98bd60c8af5552cff8a5a51c2c983a65d84b824` |
| `msi/Ghost Job_0.6.9_x64_en-US.msi` | 127 MB | `7f882b35ee5671675fc015db0595b62fc9b776d256db7a81f9e4970d220e7c47` |

**完整步骤**（照 `release.yml` 的 Windows 流程，一步都不能少）：

1. `uv run reflex-desktop doctor --bundle`
2. `uv export --frozen --no-dev --no-hashes --no-emit-project -o requirements.txt` ← **漏了这步会打出「装得上但一启动就崩」的包**（`/requirements.txt` 已在 gitignore 里）
3. `uv run reflex export --frontend-only`
4. 改 `tauri/src-tauri/tauri.conf.json`：`version = "0.6.9"`，并把资源里的 `python/python/python3*.dll` 换成 4 条 `win-dlls/<dll> → <dll>` 映射
5. `cd tauri/src-tauri && cargo tauri icon ../../assets/icon.png`
6. 装配内嵌运行时，再把 4 个 DLL 复制到 `tauri/src-tauri/win-dlls/`
7. `uv run reflex-desktop build --bundle --skip-export`

### 两个真实踩坑（已规避，别再踩）

- **必须设 `PYTHONUTF8=1`**：`reflex_desktop/plugin.py` 的 `_substitute_tokens` 用 `path.read_text()` 不指定编码，中文 Windows 下按 cp936 去读 UTF-8 的 `src/main.rs` → `UnicodeDecodeError`，`reflex export` 直接失败。这是 reflex-desktop 的可移植性 bug，值得给上游提。
- **依赖标记会骗人**：`tauri/src-tauri/site-packages/.reflex_desktop_deps` 存的是依赖集指纹。第一次装配时（没有 `requirements.txt`）指纹里只有 reflex 基础依赖，之后再打包就会打印 `embedded deps already installed (skipping pip)` 跳过安装，于是打出来的包一启动就是 `ModuleNotFoundError: No module named 'sqlalchemy'`。补上 `requirements.txt` 后**必须删掉这个标记文件**再重打。重打用 `%TEMP%\rebundle-069.ps1`——它**不删 `tauri/`**，保留 1.7 GB 的 cargo 缓存。

### 产物验证（本机实跑过）

- `target/release/` 下 `ghost-job.exe`(18.4 MB) 与 `python312.dll`(6.6 MB) 同目录 —— 等价于 CI 的 `python312.dll is not next to ghost-job.exe` 检查
- 直接启动编译产物：进程存活，**6 秒内** `http://127.0.0.1:18380/ping` 返回 `HTTP 200 "pong"`（内嵌后端 + 应用依赖齐全）
- 辅助脚本：`%TEMP%\build-069.ps1`（全流程）、`rebundle-069.ps1`（只重打包）、`verify-built-exe.ps1`（产物自检）、`diag-built-exe.ps1`（抓启动失败 stderr）
- ⚠️ Tauri 打包会去 GitHub 下 NSIS / WiX，网络不通时用 `ghproxy.net` 镜像 + `robocopy /E` 手工放置（本机以前踩过；本次直连成功）

## 上手须知（踩过的坑）

- `docs/` 被 `.gitignore` 忽略 —— 往仓库里放东西别放 `docs/`，`git add` 会静默什么都不加（第一次就因此推了个空分支）
- PowerShell 里给 `gh` 传 `--jq` 表达式会被拆坏，改用 `--json` + `ConvertFrom-Json`
- 已选中的多选下拉**没有 placeholder**（`rx.cond(selected.length() > 0, "", placeholder)`），用 patchright 定位要按字段标签找，不能按 placeholder
- 关键词输入框刻意**不做受控组件**（受控回写会打断拼音输入），失焦提交走 `el.form.requestSubmit()` 复用原生表单提交
- 本机数据库备份：`%TEMP%\ghost-job-db-backup-20261015-103930.db`（确认无误后可删）
- 临时脚本都在 `%TEMP%`：`ghost-job-shots.py`（截图）、`ghost-job-migrate-check.py`（迁移验收）、`ghost-job-verify-fix.py`（失焦验证）、`ghost-job-repro-keyword.py`（丢失复现）
