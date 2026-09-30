# 隐私安全审计与开源发布报告

> 审计对象：五大工资理财配置公式计算器
> 审计时间：2026-09-30
> 审计范围：源码、配置、注释、示例数据、文档、`.gitignore` 规则、Git 提交历史

---

## 一、审计结论（TL;DR）

**可以安全开源，无阻塞项。**

- **密钥类风险：0 项。** 全量扫描未发现任何 API Key、Token、密码、私钥、数据库连接串、Bearer 凭据。
- **个人隐私类风险：2 项，均已修复。** 一处本机绝对路径（含系统用户名）已脱敏；工作区元数据目录已加入忽略规则。
- **待人工确认项：1 项。** Git 提交身份未配置，需在推送前设置（建议使用 GitHub noreply 邮箱）。
- **Git 历史：无需清洗。** 本目录此前不是 Git 仓库，不存在历史提交，因此**不存在「已提交的密钥」需要回溯清理**——这是最理想的开局。

---

## 二、审计方法与覆盖面

| 维度 | 做法 |
| --- | --- |
| 密钥模式扫描 | 匹配 `AKIA…` / `sk-…` / `ghp_…` / `xox…-` / `AIza…` / JWT / `-----BEGIN … PRIVATE KEY` / `api_key\|secret\|password\|token` 赋值语句 |
| 连接串扫描 | 匹配 `mongodb(+srv)://` / `postgres(ql)://` / `mysql://` / `redis://` / `amqp://` / URL 内嵌凭据 `user:pass@host` |
| 个人信息扫描 | 匹配邮箱正则、中国大陆手机号 `1[3-9]\d{9}` |
| 基础设施泄露 | 匹配 IPv4 地址、`127.0.0.1` / `localhost:port`、内网域名（`.oa.com` 等）、`C:\Users\` 与 `/Users/`、`/home/` 本机路径 |
| 第三方外联 | 提取全部 `src=` / `href=` / `http(s)://` 引用，检查 CDN、字体、统计脚本 |
| 客户端存储 | 检查 `fetch` / `XMLHttpRequest` / `WebSocket` / `cookie` / `localStorage` / `sessionStorage` / `eval` |
| 隐藏目录 | 单独扫描 `.workbuddy-ai/` 等隐藏目录（默认会被 ripgrep 跳过，需显式包含） |
| 版本历史 | `git rev-parse` 确认仓库归属；对暂存内容逐 blob 复扫 |

---

## 三、隐私风险清单

| 编号 | 风险类别 | 位置 | 等级 | 处置 | 状态 |
| --- | --- | --- | --- | --- | --- |
| R-01 | 本机绝对路径（泄露系统用户名） | `.workbuddy-ai/memory/2026-09-30.md` 第 4 行 | 🟡 中 | 改写为仓库内相对路径 `index.html`；该目录同时被忽略，双重保险 | ✅ 已修复 |
| R-02 | 工作区元数据目录（AI 会话记忆、可能含内情与路径） | `.workbuddy-ai/` 整个目录 | 🟡 中 | 新增 `.gitignore` 规则屏蔽，并已验证 `git ls-files` 不含其任何文件 | ✅ 已修复 |
| R-03 | 提交身份可能暴露个人真实姓名 / 私人邮箱 | Git 配置（`user.name` / `user.email` 均未设置） | 🟡 中 | 未擅自代填身份（避免错误归属与历史重写）；推送前请手动配置，建议用 GitHub noreply 邮箱 | ⏳ 待人工确认 |
| R-04 | README 中的克隆地址占位符 | `README.md` 第九节 `<你的用户名>` | 🟢 低 | 已替换为真实仓库地址 `red-fire-99/salary-formula-calculator` | ✅ 已修复 |
| R-05 | 许可证版权人署名 | `LICENSE` | 🟢 低 | 采用中性署名 `salary-formula-calculator contributors`，未写入个人姓名 | ✅ 已处理 |
| R-06 | 密钥 / Token / 私钥 / 数据库连接串 | 全项目 | ✅ 无 | 扫描 0 命中 | ✅ 通过 |
| R-07 | 个人邮箱 / 手机号 | 全项目 | ✅ 无 | 扫描 0 命中 | ✅ 通过 |
| R-08 | 内网域名 / IP 地址 | 全项目 | ✅ 无 | 扫描 0 命中（`localhost` 仅出现在 README 的本地预览说明中，属公开通用地址） | ✅ 通过 |
| R-09 | 第三方外联与追踪 | `index.html` | ✅ 无 | 零 `src` / `href` 外部引用、零网络 API 调用，完全离线可用 | ✅ 通过 |
| R-10 | 客户端持久化与指纹 | `index.html` | ✅ 无 | 无 Cookie、无 `localStorage` / `sessionStorage`，刷新即清空 | ✅ 通过 |
| R-11 | 示例数据中的真实个人信息 | 全项目 | ✅ 无 | 示例工资金额（5000/8000/12000/20000/30000）为通用档位，非真实个人数据 | ✅ 通过 |

**等级说明：** 🔴 高 = 会导致密钥泄露或直接可被利用；🟡 中 = 泄露个人 / 环境信息；🟢 低 = 不影响安全，属发布前完善项；✅ = 扫描通过，无风险。

---

## 四、修复说明

### R-01 本机绝对路径脱敏

**修复前**（记忆文件中的原始写法，含系统用户名）：

```
- 交付：单文件零依赖网站 `C:/Users/<系统用户名>/WorkBuddy AI/2026-09-30-02-34-50/index.html`，双击即可运行。
```

**修复后**：

```
- 交付：单文件零依赖网站 `index.html`（仓库根目录），双击即可运行。
```

**修复理由**：文档与笔记中记录本机绝对路径，会在协作、截图、粘贴到 Issue 时连带泄露操作系统用户名与目录结构。改用仓库内相对路径后，信息量不减少，泄露面归零。

### R-02 工作区元数据目录屏蔽

`.workbuddy-ai/` 存放 AI 会话记忆与工作区状态，包含本机路径、会话摘要等内容，**属于本机环境数据，不属于项目源码**。已加入 `.gitignore`，并已用 `git ls-files` 与 `git status --ignored` 双向验证：该目录被正确忽略，不会进入任何提交。

### 密钥读取方式的核查结论

用户要求「确认所有密钥改为从环境变量或配置读取」。核查结论是：**本项目不存在任何密钥，也不存在需要读取配置的运行时逻辑**——它是纯静态、纯客户端页面，没有服务端、没有第三方 API 调用、没有数据库。因此：

- 无需将任何硬编码密钥迁移到环境变量（不存在可迁移对象）；
- 为避免「无密钥却摆一个 `.env` 让人以为要填」的误导，仍提供了 `.env.example`，其中明确声明本项目无需任何密钥，并给出未来若引入服务端 / 统计脚本时的命名约定（`SITE_URL` / `ANALYTICS_ENDPOINT` / `ANALYTICS_API_KEY`）；
- 真实 `.env` 已被 `.gitignore` 忽略（`.env.*` 通配 + `!.env.example` 白名单），确保示例文件能提交、真实配置不能提交。

### Git 历史敏感内容的处理建议

**当前结论：无需处理。** 本目录此前不是 Git 仓库（`git rev-parse --show-toplevel` 报 `not a git repository`），不存在历史提交，因此不存在「密钥已进入历史」的问题。

**若将来误将密钥提交，按以下顺序处理：**

1. **立即轮换密钥**——这是第一步且不可省略。密钥一旦进入历史即视为已泄露，无论后续是否重写历史。
2. **尚未推送时**：`git reset --soft HEAD~1` 撤销提交，从暂存区移除敏感文件，补全 `.gitignore` 后重新提交。
3. **已推送、但仓库是私有且刚推送**：使用 [`git filter-repo`](https://github.com/newren/git-filter-repo) 重写历史：
   ```bash
   git filter-repo --path path/to/secret.env --invert-paths
   git push --force --all && git push --force --tags
   ```
   或使用 [BFG Repo-Cleaner](https://rtyley.github.io/bfg-repo-cleaner/) 处理超大仓库。
4. **已公开推送**：重写历史无法保证他人已克隆的副本被清除，务必以「密钥已泄露」为前提完成轮换，并在必要时联系平台清理缓存与 Fork。
5. **预防**：启用 GitHub 的 **Secret Scanning** 与 **Push Protection**（仓库 Settings → Code security），让平台在推送阶段直接拦截密钥。

---

## 五、新增 / 修改的文件列表

### 新增（6 个）

| 文件 | 作用 |
| --- | --- |
| `.gitignore` | 屏蔽工作区数据、密钥、依赖、构建产物、系统与编辑器文件 |
| `.gitattributes` | 统一换行符为 LF，避免跨平台协作时整文件被标记为改动 |
| `.env.example` | 环境变量示例与约定（明确标注本项目运行时无需任何密钥） |
| `LICENSE` | MIT 许可证，版权人为中性署名 |
| `README.md` | 项目简介、功能特性、五大公式、安装运行、使用示例、项目结构、贡献指南、免责声明 |
| `PRIVACY-AUDIT.md` | 本审计报告 |

### 修改（1 个）

| 文件 | 改动 |
| --- | --- |
| `.workbuddy-ai/memory/2026-09-30.md` | 移除含系统用户名的本机绝对路径，改为仓库内相对路径（该文件本身被忽略，不会入库） |

### 未改动

| 文件 | 说明 |
| --- | --- |
| `index.html` | **代码零改动。** 原始代码即通过全部隐私扫描，无需脱敏；功能、UI、公式逻辑保持原样 |

### 依赖声明

**无运行时依赖、无构建依赖。** 全部功能由原生 HTML / CSS / JavaScript 实现，不引用任何框架、UI 库或 CDN。因此**刻意不创建** `package.json` / `requirements.txt`——为空依赖的项目添加依赖清单文件属于形式主义，反而会让人误以为需要 `npm install`。该决策已写入 README 第四节，若你希望保留一个占位 `package.json`（例如为了 `npm start` 快捷预览），可随时告知我补上。

---

## 六、正式推送 GitHub 前的最终检查项

### A. 身份与仓库（必做）

- [ ] **配置提交身份**（当前未配置，这是唯一未完成项）：
  ```bash
  git config user.name  "你的 GitHub 用户名"
  git config user.email "你的ID+用户名@users.noreply.github.com"   # GitHub 提供的匿名邮箱
  ```
  > 在 GitHub → Settings → Emails 勾选 *Keep my email addresses private*，即可拿到该 noreply 邮箱，避免私人邮箱被永久写入公开提交记录。
- [ ] **确认忽略规则生效**：`git status --ignored --short` 应显示 `.workbuddy-ai/` 被忽略
- [ ] **确认入库清单**：`git ls-files` 应恰好只有 6 个文件——`.env.example` `.gitattributes` `.gitignore` `LICENSE` `PRIVACY-AUDIT.md` `README.md` `index.html`（含本报告共 7 个）
- [ ] **完成首次提交**：`git commit -m "chore: initial release of salary formula calculator"`

### B. 内容自检（必做）

- [ ] 在 GitHub 仓库 Settings → Code security 中开启 **Secret Scanning** 与 **Push Protection**
- [ ] 替换 `README.md` 中的克隆地址占位符 `<你的用户名>` 为真实地址
- [ ] 决定 `PRIVACY-AUDIT.md` 是否随仓库公开；若不想公开，执行 `git rm --cached PRIVACY-AUDIT.md` 并加入 `.gitignore`
- [ ] 确认没有把 `.workbuddy-ai/`、`.env`、任何 `*.key` / `*.pem` 误加入提交
- [ ] 复查提交信息本身不含个人信息（姓名、邮箱、内部项目名、本机路径）

### C. 功能验收（必做）

- [ ] 双击 `index.html` 在 Chrome / Edge / Firefox / Safari 各验证一次
- [ ] 桌面（≥1000px，3 列）、平板（640–999px，2 列）、手机（≤375px，1 列）三档布局正常
- [ ] 输入 `12000` → 五项结果分别为 ¥2,400 / ¥36,000 / ¥6,000 / ¥3,600,000 / ¥720,000
- [ ] 输入 `-5000`、`abc`、`1.2.3`、空值，确认提示语与 `—` 占位表现符合 README 第五节
- [ ] 输入框连续打字时千分位与光标位置正常，光标不跳到末尾
- [ ] 断网打开页面，确认功能完全可用（验证零外联）

### D. 发布动作（可选）

- [ ] 在 GitHub 仓库 About 处补充描述与 Topics（如 `finance` `calculator` `vanilla-js` `zero-dependency`）
- [ ] Settings → Pages 开启 GitHub Pages，Source 选 `main` 分支根目录，即可获得在线演示链接
- [ ] 在 README 顶部补上在线演示地址与仓库地址

---

## 七、复现本次审计

```bash
# 1. 密钥与凭据模式
grep -rInE 'AKIA[0-9A-Z]{16}|sk-[A-Za-z0-9]{20,}|ghp_[A-Za-z0-9]{36}|xox[baprs]-|-----BEGIN|(api[_-]?key|secret|password|token)\s*[:=]' .

# 2. 个人与环境信息（注意：需显式包含隐藏目录）
grep -rInE '[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}|1[3-9][0-9]{9}|[A-Za-z]:[/\\]Users|/(Users|home)/[A-Za-z0-9._-]+' . --include='*' 

# 3. 第三方外联
grep -nEo 'https?://[^"'"'"' )]+|src="[^"]+"|href="[^"]+"' index.html

# 4. 入库清单与忽略验证
git ls-files && git status --ignored --short
```
