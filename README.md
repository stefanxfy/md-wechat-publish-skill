# md-wechat-publish-skill

Skill 工具集，包含多个自动化技能，专注 AI 工程内容的创作与发布工作流。

---

## 📦 技能列表

| 技能 | 功能 | 版本 |
|------|------|------|
| [md-wechat-publish-skill](#md-wechat-publish-skill) | 一键将 Markdown 文章发布到微信公众号草稿箱 | v2.0 |
| [md-mermaid2img-skill](https://github.com/stefanxfy/md-mermaid2img-skill) | Mermaid 图表转 PNG/ SVG 图片 | - |
| [jimeng-image-gen](#jimeng-image-gen) | 即梦 AI 图片生成（需配合 md-wechat-publish-skill 使用） | - |

---

## md-wechat-publish-skill (v2.0)

> **架构升级 (2026-05-25)**: v2.0 重构为**纯 Skill 驱动架构**，完全可移植到任何 Agent 环境（Claude Code、Hermes 等）。

一键将 Markdown 文章发布到微信公众号草稿箱，自动完成：**Mermaid 图转换 → 封面图生成 → frontmatter 完善 → wenyan-cli 发布**。

### 🚀 v2.0 核心改进

1. **纯 Skill 驱动**（非脚本驱动）
   - 改前：`python3 ~/.skills/.../scripts/xxx.py`（硬编码路径）
   - 改后：描述需求和参数，让 AI 根据环境调用对应 skill
   - 示例：
     ```markdown
     调用 `md-mermaid2img-skill`，参数：
     - 输入文件：<md路径>
     - 宽度：1200px
     - 缩放：2x
     ```

2. **可移植性**（相对路径发现机制）
   - 问题：不同 agent 的 skill 安装路径不同（如 `~/.skills/`、`~/.openclaw/skills/` 等）
   - 解决方案：相对路径发现（不绑死特定路径）
   - 查找规则：
     1. 找到本 SKILL.md 的所在目录（例如：`/skills/md-wechat-publish-skill/`）
     2. 回到父目录（`/skills/`）
     3. 在父目录下查找 `md-mermaid2img-skill/` 和 `jimeng-image-gen/`
     4. 如果找不到，报错并提示用户

3. **完整的检查机制**
   - 前置检查：依赖是否存在（检查 md-mermaid2img-skill、jimeng-image-gen、wenyan-cli）
   - 成功标准：每个 step 明确定义（如输出目录已创建、文件大小 >0、退出码为 0 等）
   - 失败检测：如何判断失败（输出目录未创建、退出码非 0、stderr 有错误信息等）
   - 错误处理：区分可恢复错误（可以跳过该步骤，继续下一步）和不可恢复错误（必须中止流程）

4. **清晰的流程控制**
   - 成功时：记录输出路径，继续执行下一步
   - 失败时：立即停止 + 分析原因 + 通知用户 + 提供解决方案 + 等待用户处理
   - 可恢复错误：询问用户是否跳过（如部分 Mermaid 转换失败、封面图生成失败）

### 效果

```
输入：本地 .md 文件路径（绝对路径）
  → Step 1: 检测 mermaid 代码块 → 调用 md-mermaid2img-skill 转换
  → Step 2: 生成封面图（调用 jimeng-image-gen）
  → Step 3: 完善 frontmatter（title / cover / description）
  → Step 4: 调用 wenyan-cli 发布到微信草稿箱
输出：Media ID（发布成功）
```

### 安装

#### 前提依赖

| 依赖 | 说明 | 安装方式 |
|------|------|---------|
| **Agent 运行环境** | 支持 Claude Code、Hermes 等有 skill 系统的 Agent |
| **md-mermaid2img-skill** | Mermaid 图转图片 | [GitHub](https://github.com/stefanxfy/md-mermaid2img-skill) |
| **jimeng-image-gen** | 即梦 AI 生成封面图 | 需配合 md-wechat-publish-skill 使用 |
| **wenyan-cli** | 微信草稿箱发布 CLI | 见下方安装 |

#### 安装步骤

```bash
# 1. 克隆本仓库
git clone https://github.com/stefanxfy/myskills.git
cd myskills

# 2. 安装 md-wechat-publish-skill
mkdir -p ~/.skills/md-wechat-publish-skill
cp -r md-wechat-publish-skill/* ~/.skills/md-wechat-publish-skill/

# 3. 安装依赖 skill（md-mermaid2img-skill）
#    参见：https://github.com/stefanxfy/md-mermaid2img-skill
#    若未安装，执行：
#    skillhub_install install_skill md-mermaid2img-skill
#    或手动克隆：
#    git clone https://github.com/stefanxfy/md-mermaid2img-skill.git ~/.skills/md-mermaid2img-skill

# 4. 安装 wenyan-cli
npm install -g @wenyan-md/cli --ignore-engines

# 5. 配置微信凭证
mkdir -p ~/.config/wenyan-md
cat > ~/.config/wenyan-md/credential.json << 'EOF'
{
  "appId": "你的AppID",
  "appSecret": "你的AppSecret"
}
EOF
```

> ⚠️ 微信凭证（AppID / AppSecret）需在 [微信公众平台](https://mp.weixin.qq.com) 获取，并配置IP白名单。

#### 依赖检查

```bash
# 检查 skill 目录
ls ~/.skills/md-wechat-publish-skill/
ls ~/.skills/md-mermaid2img-skill/
ls ~/.skills/jimeng-image-gen/

# 检查 CLI 工具
which wenyan
which mmdc
```

### 使用方式

#### 自动触发（推荐）

在 AI Agent 对话中直接说：

```
发布到公众号 / 发公众号 / 推送到公众号
```

并附上文章路径，skill 将自动完成全流程。

#### 手动调用

```bash
# 设置微信凭证（环境变量方式）
export WECHAT_APP_ID=wx9cb6938231b854fc
export WECHAT_APP_SECRET=你的AppSecret

# 发布文章（需先完成 mermaid 转换和封面图生成）
wenyan publish -f /path/to/your/article_mermaid/article.md
```

#### 微信草稿箱文章上限说明

微信草稿 API 有 **2 万字符**硬限制（原文渲染后 HTML 长度），超出部分无法发布。
超长文章需**拆篇发布**，拆分为多篇后分别发布。

### 工作流详解（v2.0 Skill 驱动方式）

#### Step 1 - Mermaid 转换

**调用方式**：描述需求，让 AI 根据环境调用 `md-mermaid2img-skill`

```
调用 `md-mermaid2img-skill`，参数：
- 输入文件：<md文件路径>
- 输出目录：<原文件名>_mermaid/
- 宽度：1200px
- 缩放：2x
```

**成功标准**：
- 输出目录已创建
- 所有 ````mermaid` 代码块都已转换为 `.png` 图片
- 原 md 文件已更新（图片链接替换代码块）

**失败检测**：
- 输出目录未创建 → 不可恢复错误，中止流程
- 部分 mermaid 转换失败 → 可恢复错误，询问用户是否跳过

#### Step 2 - 封面图生成

**调用方式**：描述需求，让 AI 根据环境调用 `jimeng-image-gen`

```
调用 `jimeng-image-gen`，参数：
- 提示词：根据文章标题和 frontmatter 的 `cover_prompt` 字段构造
- 输出路径：`images/cover.png`（相对于 md 文件路径）
- 防污染后缀：必须添加"无任何文字或数字或字母出现在图片角落边缘，无水印文字，无比例标注，无版本号"
```

**成功标准**：
- `images/cover.png` 文件已生成
- 文件大小 > 0

**失败检测**：
- 输出文件未生成 → 可恢复错误，询问用户是否跳过（使用默认封面或不添加封面）
- 图片上有多余文字/水印 → 重新生成，添加更强的防污染后缀

#### Step 3 - frontmatter 完善

**调用方式**：AI 直接读取 md 文件，检查并补充 frontmatter 字段

**需完善的字段**：
- `title`：优先保留原文已有值，无则根据文件名生成
- `cover`：固定为 `images/cover.png`（相对于 md 文件路径）
- `description`：从文章第一段自动提取，120~180 字符，去除 Markdown 语法

**成功标准**：
- frontmatter 包含所有必需字段（title、cover、description）
- 字段值符合要求（description 长度 120~180 字符）

#### Step 4 - wenyan-cli 发布

**调用方式**：描述需求，让 AI 调用 `wenyan-cli`

```
调用 `wenyan-cli`，参数：
- 命令：`publish`
- 文件：`-f <md文件路径>`
- 凭证：环境变量 `WECHAT_APP_ID` 和 `WECHAT_APP_SECRET`（或在 `~/.config/wenyan-md/credential.json` 配置）
```

**成功标准**：
- 命令退出码为 0
- 输出包含 Media ID（格式：`k72KB1lohlDk-...`）

**失败检测**：
- 退出码非 0 → 不可恢复错误，分析错误信息并提示用户
- 内容超长（>2万字符）→ 不可恢复错误，建议拆篇发布

### 错误处理

| 错误 | 原因 | 解决方案 | 可恢复？ |
|------|------|---------|----------|
| `内容超长，超出小绿书模式限制` | 微信 2 万字符限制 | 将文章拆分为多篇 | ❌ 不可恢复 |
| `Could not find a suitable point for the given distance` | Mermaid 图表语法过于复杂 | 简化图表或去掉该图表 | ✅ 可恢复（跳过） |
| `ENOENT: no such file or directory` | 封面图路径错误 | 检查 `images/cover.png` 是否存在 | ✅ 可恢复（重新生成） |
| 凭证校验失败 | AppID/AppSecret 错误或 IP 未白名单 | 检查配置或添加白名单 | ❌ 不可恢复 |
| 部分 Mermaid 转换失败 | 图表语法错误或不支持 | 跳过该图表，继续下一步 | ✅ 可恢复 |

### 移植性说明

v2.0 采用**纯 Skill 驱动架构**，完全可移植到任何有 skill 系统的 Agent 环境：

| Agent | 调用方式 |
|-------|----------|
| **Claude Code** | AI 读取本 SKILL.md，调用等效 skill |
| **Hermes** | AI 读取本 SKILL.md，调用等效 skill |
| **其他 agent** | AI 读取本 SKILL.md，根据环境决定如何调用 |

**关键设计决策**：
- ✅ **描述"做什么"**，而非"怎么做"（不依赖特定实现）
- ✅ **相对路径发现**，不绑死特定 agent 路径（如 `~/.skills/`）
- ✅ **完整错误处理**，区分可恢复错误和不可恢复错误
- ✅ **清晰流程控制**，成功时继续，失败时中止+通知用户

### 相关项目

| 项目 | 链接 |
|------|------|
| md-mermaid2img-skill | [github.com/stefanxfy/md-mermaid2img-skill](https://github.com/stefanxfy/md-mermaid2img-skill) |
| wenyan-cli | [github.com/crycotcus/wenyan-cli](https://github.com/crycotcus/wenyan-cli) |
| 即梦 AI | [jimeng.jianying.com](https://jimeng.jianying.com) |

---

## md-mermaid2img-skill

将 Markdown 中的 Mermaid 图表代码块渲染为 PNG / SVG 图片。
独立技能，可单独安装使用。

**GitHub**: [stefanxfy/md-mermaid2img-skill](https://github.com/stefanxfy/md-mermaid2img-skill)

---

## jimeng-image-gen

即梦 AI 图片生成工具，提供 Python 脚本调用火山引擎即梦 API。
被 md-wechat-publish-skill 依赖用于生成文章封面图。

> 注意：此技能需配合 md-wechat-publish-skill 使用，单独使用需提前配置火山引擎 AK/SK。

---

## License

MIT

---

## v2.0 更新日志 (2026-05-25)

### 架构重构

- **Skill 驱动**：从脚本驱动重构为纯 Skill 驱动架构
  - 改前：`python3 ~/.skills/.../scripts/xxx.py`（硬编码路径）
  - 改后：描述需求和参数，让 AI 根据环境调用对应 skill

- **可移植性**：相对路径发现机制，不绑死特定 agent 路径
  - 查找规则：找到本 SKILL.md 的所在目录 → 回到父目录 → 在父目录下查找依赖 skill
  - 支持 Claude Code、Hermes 等任何有 skill 系统的 agent 环境

- **完整检查机制**：
  - 前置检查：依赖是否存在
  - 成功标准：每个 step 明确定义
  - 失败检测：如何判断失败
  - 错误处理：区分可恢复错误和不可恢复错误

- **清晰流程控制**：
  - 成功时：记录输出路径，继续执行下一步
  - 失败时：立即停止 + 分析原因 + 通知用户 + 提供解决方案 + 等待用户处理
  - 可恢复错误：询问用户是否跳过

### 文档更新

- 更新 README.md，添加 v2.0 架构说明
- 更新工作流详解，改为 Skill 驱动方式描述
- 添加移植性说明表格
- 添加 v2.0 更新日志
