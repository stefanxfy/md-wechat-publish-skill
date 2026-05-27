---
name: md-wechat-publish-skill
description: |
  一键将 Markdown 文章发布到微信公众号草稿箱。
  当用户说"发布到公众号"、"发布公众号"、"发到微信"、"推送到公众号"、"公众号发布"，
  或提供 .md 文件路径并要求发布到微信时触发。
  自动完成：Mermaid 图转换 → 封面图生成 → frontmatter 完善 → wenyan-cli 发布。
  本 skill 采用纯 Skill 驱动架构，可移植到任何支持 skill 系统的 agent 环境。
---

# MD 文章公众号发布流程

## 架构设计

本 skill 采用**纯 Skill 驱动架构**，不依赖任何特定 agent 的实现细节。

**设计原则**：
1. 描述"做什么"，而非"怎么做"
2. 使用相对路径发现机制（不绑死特定 Agent）
3. 定义清晰的输入输出接口和成功/失败标准
4. 完整的错误处理和可恢复机制

**可移植性**：
- ✅ Claude Code / Hermes 等：AI 读取本 SKILL.md，调用对应 skill
- ✅ Claude Code：AI 读取本 SKILL.md，调用等效 skill
- ✅ Hermes：AI 读取本 SKILL.md，调用等效 skill
- ✅ 任何有 skill 系统的 agent 都能运行

---

## 前置检查（必须执行）

在开始之前，**必须**依次检查以下依赖是否可用：

### 检查清单

| 依赖 | 类型 | 检查方式 | 不存在时的处理 |
|------|------|----------|----------------|
| md-mermaid2img-skill | Skill | 查找相对路径 `../md-mermaid2img-skill/SKILL.md` | 提示用户安装，中止流程 |
| jimeng-image-gen | Skill | 查找相对路径 `../jimeng-image-gen/SKILL.md` | 提示用户安装，中止流程 |
| wenyan-cli | CLI 工具 | 执行 `which wenyan` | 提示用户安装，中止流程 |

### 路径发现规则

**关键**：本 skill 和依赖 skill 应该安装在**同一个目录**下。

**查找步骤**：
1. 找到本 SKILL.md 的所在目录（例如：`/skills/md-wechat-publish-skill/`）
2. 回到父目录（`/skills/`）
3. 在父目录下查找 `md-mermaid2img-skill/` 和 `jimeng-image-gen/`
4. 如果找不到，报错并提示用户

**示例**：
```
假设本 skill 安装在：
  /home/user/.skills/md-wechat-publish-skill/SKILL.md

那么依赖 skill 应该在：
  /home/user/.skills/md-mermaid2img-skill/
  /home/user/.skills/jimeng-image-gen/
```

### 检查脚本（AI 执行）

```bash
# 检查本 skill 的安装目录
CURRENT_SKILL_DIR="$(cd "$(dirname "$0")" && pwd)"
PARENT_DIR="$(dirname "$CURRENT_SKILL_DIR")"

# 检查 md-mermaid2img-skill
if [ -f "$PARENT_DIR/md-mermaid2img-skill/SKILL.md" ]; then
  echo "✅ md-mermaid2img-skill 已安装"
else
  echo "❌ 未找到 md-mermaid2img-skill"
  echo "请执行：skillhub_install install_skill md-mermaid2img-skill"
  echo "或手动安装到：$PARENT_DIR/md-mermaid2img-skill/"
  exit 1
fi

# 检查 jimeng-image-gen
if [ -f "$PARENT_DIR/jimeng-image-gen/SKILL.md" ]; then
  echo "✅ jimeng-image-gen 已安装"
else
  echo "❌ 未找到 jimeng-image-gen"
  echo "请执行：skillhub_install install_skill jimeng-image-gen"
  echo "或手动安装到：$PARENT_DIR/jimeng-image-gen/"
  exit 1
fi

# 检查 wenyan-cli
if which wenyan > /dev/null 2>&1; then
  echo "✅ wenyan-cli 已安装"
else
  echo "❌ 未找到 wenyan-cli"
  echo "请执行：npm install -g @wenyan-md/cli"
  exit 1
fi
```

### 前置检查失败处理

如果任何依赖缺失：
1. **立即停止**，不继续执行
2. **通知用户**缺少哪个依赖
3. **提供安装命令**（复制即用）
4. **等待用户确认**安装完成后再重新运行

---

## 完整工作流

```
输入：Markdown 文件路径（绝对路径）
  ↓
前置检查（依赖是否存在）
  ↓
Step 1: 检测是否有 mermaid 代码块
  ├─ 有 → 调用 md-mermaid2img-skill 转换
  └─ 无 → 直接用原文件
  ↓
Step 2: 生成封面图（调用 jimeng-image-gen）
  ↓
Step 3: 完善 frontmatter（title、cover、description）
  ↓
Step 4: 调用 wenyan-cli 发布到草稿箱
  ↓
输出：微信草稿箱 Media ID
```

---

## Step 1: Mermaid 检测与转换

### 输入
- Markdown 文件路径（绝对路径）

### 处理流程

#### 1.1 检测 Mermaid 代码块

读取 MD 文件内容，检测是否存在 Mermaid 代码块：

```python
import re

with open(md_path, 'r', encoding='utf-8') as f:
    content = f.read()

has_mermaid = bool(re.search(r'```mermaid', content))
```

- **如果没有 Mermaid 代码块**：跳过转换，直接使用原文件，跳到 Step 2
- **如果有 Mermaid 代码块**：继续执行 1.2

#### 1.2 调用 md-mermaid2img-skill

**调用方式**：由 AI 根据当前 agent 环境决定如何调用 skill

**传递参数**：
- 输入文件：`<md路径>`（绝对路径）
- 宽度：1200px（推荐）
- 缩放：2x（推荐）
- 背景：白色（推荐）
- 主题：默认（推荐）

**AI 执行逻辑**：
1. 读取 `../md-mermaid2img-skill/SKILL.md`
2. 按照该 SKILL.md 的指引执行转换
3. 如果当前 agent 环境有等效 skill，调用等效 skill

### 输出

转换成功后，生成以下文件：

```
<原文件名>_mermaid/
├── <原文件名>.md  # 图片引用已替换
└── images/
    ├── mermaid_1.png
    ├── mermaid_2.png
    └── ...
```

**后续步骤使用**：`<原文件名>_mermaid/<原文件名>.md`

### 成功标准

转换成功后，必须**同时满足**以下条件：
- [ ] 输出目录 `<原文件名>_mermaid/` 已创建
- [ ] 该目录下有 `images/` 文件夹
- [ ] `images/` 里有 ≥1 张 PNG 图片
- [ ] 新的 `.md` 文件已生成
- [ ] 新的 `.md` 文件里的 Mermaid 代码块已替换为 `![image](images/...)` 格式

### 失败检测

如果转换失败，会出现以下情况**之一**：
- ❌ 输出目录未创建
- ❌ 脚本返回非 0 退出码
- ❌ stderr 有错误信息
- ❌ 新的 `.md` 文件未生成
- ❌ `images/` 文件夹为空

### 失败处理

如果转换失败：

1. **读取错误信息**（stdout/stderr）
2. **判断错误类型**：
   - 语法错误：Mermaid 语法不正确
   - 依赖缺失：缺少必要工具（如 mmdc）
   - 权限错误：无法创建输出目录
   - 其他错误
3. **通知用户**：
   ```
   ❌ Step 1 失败：Mermaid 转换失败
   
   错误类型：<错误类型>
   错误信息：<错误信息>
   
   解决方案：
   1. 检查 Mermaid 语法（第 XX 行）
   2. 或删除该 Mermaid 代码块
   3. 重新运行发布流程
   
   是否继续？（等待用户确认）
   ```
4. **中止流程**，等待用户处理

### 可恢复错误处理

**部分图表转换失败**（不是全部）：
- 如果只有部分 Mermaid 图表转换失败，可以跳过失败的图表
- 询问用户：是否跳过失败的图表，继续转换其他图表？
- 如果用户同意：继续转换，在最终文件中标记跳过的图表
- 如果用户不同意：中止流程

---

## Step 2: 封面图生成

### 输入
- 文章标题（从 frontmatter 或文件名提取）
- 封面提示词（从 frontmatter `cover_prompt` 字段读取，如果有的话）
- 转换后的 MD 文件路径（Step 1 的输出）

### 处理流程

#### 2.1 确定文章标题

从以下位置**按顺序**提取标题：
1. **优先**：读取转换后的 MD 文件的 frontmatter `title:` 字段
2. **备选**：从文件名推断（去除日期/序号前缀）

**示例**：
- 文件名：`2026-05-25-React深度解析.md` → 标题：`React深度解析`
- 文件名：`article.md` → 标题：`article`

#### 2.2 确定输出路径

封面图保存到：
```
<Step 1 输出目录>/images/cover.png
```

**示例**：
- 如果 Step 1 输出目录是 `/path/to/article_mermaid/`
- 那么封面图路径是 `/path/to/article_mermaid/images/cover.png`

#### 2.3 生成提示词

**优先使用 frontmatter 中的 `cover_prompt`**（如果有）：
- 检查**原文** MD 文件（非转换后）的 frontmatter 是否有 `cover_prompt:` 字段
- 有 → 直接用作 jimeng-image-gen 的 prompt
- 无 → 根据标题自动构造（见下方）

**自动生成提示词规则**：
1. 分析标题关键词（如"ReAct深度解析" → 提取"ReAct"、"深度解析"）
2. 识别文章标签/分类（如"AI工程"、"大模型"）
3. 构造 prompt：
   ```
   [风格描述]，[背景色]背景，[核心内容]，[配色描述]，
   无任何文字或数字或字母出现在图片角落边缘，无水印文字，无比例标注，无版本号
   ```

**必加防污染后缀**（每次必须）：
```
无任何文字或数字或字母出现在图片角落边缘，无水印文字，无比例标注，无版本号
```

**推荐封面风格**（可直接用）：
```
科技感专业信息图风格，深色背景带微弱电路纹理，
中心核心概念标识，四周分支环绕，
极简扁平化设计，无人物，专业演示文稿质感
```

**完整提示词示例**：
```
科技感专业信息图风格，深蓝色背景带微弱电路纹理，
中心"ReAct"概念标识，四周分支环绕，
极简扁平化设计，无人物，专业演示文稿质感，
无任何文字或数字或字母出现在图片角落边缘，无水印文字，无比例标注，无版本号
```

#### 2.4 调用 jimeng-image-gen

**调用方式**：由 AI 根据当前 agent 环境决定如何调用 skill

**传递参数**：
- prompt：`<构造好的提示词>`
- output：`<输出路径>`（见 2.2）
- 模式：official（默认，使用火山引擎 API）

**AI 执行逻辑**：
1. 读取 `../jimeng-image-gen/SKILL.md`
2. 按照该 SKILL.md 的指引生成图片
3. 如果当前 agent 环境有等效 skill，调用等效 skill

### 输出

生成的封面图：
```
<Step 1 输出目录>/images/cover.png
```

### 成功标准

封面图生成成功后，必须**同时满足**以下条件：
- [ ] `images/cover.png` 已生成
- [ ] 文件大小 >0
- [ ] 文件可以被正常打开（不是损坏的图片）

### 失败检测

如果生成失败，会出现以下情况**之一**：
- ❌ 输出文件未生成
- ❌ 文件大小为 0
- ❌ 脚本返回非 0 退出码
- ❌ stderr 有错误信息

### 失败处理

如果生成失败：

1. **读取错误信息**（stdout/stderr）
2. **判断错误类型**：
   - API 限额：即梦 API 限额已用完
   - 网络问题：无法连接即梦 API
   - Prompt 违规：提示词包含敏感内容
   - 凭证错误：火山引擎 AK/SK 配置错误
   - 其他错误
3. **通知用户**：
   ```
   ❌ Step 2 失败：封面图生成失败
   
   错误类型：<错误类型>
   错误信息：<错误信息>
   
   解决方案：
   1. 检查网络连接
   2. 检查火山引擎凭证配置
   3. 修改提示词（如果是违规）
   4. 或跳过封面图生成，继续发布
   
   是否跳过封面图，继续发布？（等待用户确认）
   ```
4. **询问用户**：是否跳过封面图，继续发布？
   - 如果用户同意：跳过封面图，继续 Step 3（frontmatter 中不包含 cover 字段）
   - 如果用户不同意：中止流程

### 可恢复错误

**封面图生成失败**：
- 可以跳过封面图，继续发布
- 微信文章可以不包含封面图（使用默认封面）

---

## Step 3: 完善 frontmatter

### 输入
- 转换后的 MD 文件路径（Step 1 输出）
- 封面图路径（Step 2 输出，如果生成成功的话）

### 处理流程

#### 3.1 读取 MD 文件

读取转换后的 MD 文件内容。

#### 3.2 检查 frontmatter

检查文件开头是否有 YAML frontmatter（以 `---` 包裹）：

```python
import re

with open(md_path, 'r', encoding='utf-8') as f:
    content = f.read()

# 检查是否有 frontmatter
has_frontmatter = bool(re.match(r'^---\s*\n', content))
```

#### 3.3 补充缺失字段

**必须包含的字段**：

| 字段 | 说明 | 如何获取 |
|------|------|----------|
| `title` | 文章标题 | 从现有 frontmatter 读取，或从文件名推断 |
| `cover` | 封面图路径（相对路径） | Step 2 输出，`images/cover.png` |
| `description` | 文章描述（可选） | 从现有 frontmatter 读取，或留空 |

**补充规则**：

1. **如果文件没有 frontmatter**：
   - 在文件开头添加：
     ```yaml
     ---
     title: <文章标题>
     cover: images/cover.png
     description: 
     ---
     ```

2. **如果文件有 frontmatter，但缺少字段**：
   - 补充缺失的字段：
     ```yaml
     title: <文章标题>  # 如果缺失
     cover: images/cover.png  # 如果缺失
     description: <描述>  # 如果缺失
     ```

3. **如果 Step 2 跳过封面图生成**：
   - 不包含 `cover` 字段，或设置为空：
     ```yaml
     cover:
     ```

#### 3.4 写回文件

将完善后的内容写回文件。

**注意**：使用写文件工具写入，避免编码问题。

### 输出

完善后的 MD 文件（覆盖原文件）。

### 成功标准

Frontmatter 完善成功后，必须**同时满足**以下条件：
- [ ] frontmatter 包含 `title` 字段
- [ ] 如果生成了封面图，frontmatter 包含 `cover` 字段
- [ ] YAML 语法正确（可以被 `yaml` 库解析）
- [ ] 文件可以被正常读取

### 失败检测

如果完善失败，会出现以下情况**之一**：
- ❌ YAML 语法错误
- ❌ 无法写入文件（权限错误）
- ❌ 其他错误

### 失败处理

如果完善失败：

1. **读取错误信息**
2. **判断错误类型**：
   - YAML 语法错误：现有 frontmatter 语法不正确
   - 权限错误：无法写入文件
   - 其他错误
3. **通知用户**：
   ```
   ❌ Step 3 失败：frontmatter 完善失败
   
   错误类型：<错误类型>
   错误信息：<错误信息>
   
   解决方案：
   1. 检查 frontmatter 语法（可能是 YAML 格式错误）
   2. 检查文件权限
   3. 手动编辑 frontmatter
   4. 重新运行发布流程
   
   是否继续？（等待用户确认）
   ```
4. **中止流程**，等待用户处理

### 不可恢复错误

**YAML 语法错误**：
- 必须中止流程，不要生成损坏的 YAML
- 微信无法解析格式错误的 frontmatter

---

## Step 4: 发布到微信草稿箱

### 输入
- 完善后的 MD 文件路径（Step 3 输出）

### 处理流程

#### 4.1 设置环境变量

**必须**设置以下环境变量：

```bash
export WECHAT_APP_ID="<微信 AppID>"
export WECHAT_APP_SECRET="<微信 AppSecret>"
```

**凭证获取方式**：
- 登录微信公众平台：https://mp.weixin.qq.com/
- 进入「设置与开发」→「基本配置」→「开发者ID(AppID)」和「开发者密码(AppSecret)」

**凭证存储**：
- 推荐存储在 `~/.config/wenyan-md/credential.json`：
  ```json
  {
    "appid": "<微信 AppID>",
    "secret": "<微信 AppSecret>"
  }
  ```
- wenyan-cli 会自动读取该文件

#### 4.2 调用 wenyan-cli

**命令**：

```bash
wenyan publish -f <完善后的 MD 文件路径>
```

**示例**：

```bash
wenyan publish -f /path/to/article_mermaid/article.md
```

#### 4.3 解析输出

wenyan-cli 会输出以下信息：

```
✅ 发布成功
Media ID: <Media ID>
Draft URL: https://mp.weixin.qq.com/...
```

**需要提取**：Media ID（后续可能需要用到）

### 输出

微信草稿箱 Media ID。

### 成功标准

发布成功后，必须**同时满足**以下条件：
- [ ] `wenyan` 命令返回 exit code 0
- [ ] 输出包含 `✅ 发布成功` 或类似成功提示
- [ ] 输出包含 Media ID

### 失败检测

如果发布失败，会出现以下情况**之一**：
- ❌ `wenyan` 命令返回非 0 退出码
- ❌ 输出包含 `❌` 或 `失败` 或 `error`
- ❌ 输出不包含 Media ID

### 失败处理

如果发布失败：

1. **读取错误信息**（stdout/stderr）
2. **判断错误类型**（见下方「常见错误」表格）
3. **通知用户**：
   ```
   ❌ Step 4 失败：发布到微信草稿箱失败
   
   错误类型：<错误类型>
   错误信息：<错误信息>
   
   解决方案：
   <根据错误类型提供解决方案>
   
   是否继续？（等待用户确认）
   ```
4. **中止流程**，等待用户处理

### 常见错误

| 错误 | 原因 | 解决方案 |
|------|------|---------|
| `内容超长` | 超出微信 2 万字符限制 | 拆分文章为多篇 |
| `凭证校验失败` | AppID/AppSecret 错误 | 检查 `~/.config/wenyan-md/credential.json` |
| `IP 未白名单` | 服务器 IP 不在微信公众平台白名单 | 添加 IP 到微信公众平台白名单 |
| `access_token 过期` | access_token 有效期为 2 小时 | wenyan-cli 会自动刷新，如果还是失败，手动删除 `~/.cache/wenyan-md/access_token.json` |
| `图片上传失败` | 图片格式不支持或过大 | 检查图片格式（应该是 PNG/JPG），压缩图片 |
| `网络超时` | 网络连接问题 | 检查网络连接，重试 |

### 不可恢复错误

**内容超长**：
- 必须中止流程
- 微信限制：单篇文章不超过 2 万字符
- 解决方案：拆分文章为多篇

**凭证校验失败**：
- 必须中止流程
- 解决方案：检查 AppID/AppSecret 是否正确

---

## 完整流程状态追踪

在执行过程中，**必须**实时更新状态，让用户知道当前进展：

```markdown
执行状态：
✅ 前置检查 - 通过
✅ Step 1 - Mermaid 转换 - 完成（转换 6 个图表）
✅ Step 2 - 封面图生成 - 完成
✅ Step 3 - frontmatter 完善 - 完成
⏳ Step 4 - 发布到微信 - 进行中...
```

**状态说明**：
- ✅：已完成
- ⏳：进行中
- ❌：失败
- ⏸️：已跳过（用户选择跳过可选步骤）

---

## 错误处理汇总

### 可恢复错误（可以跳过该步骤，继续下一步）

| 错误 | 步骤 | 处理方式 |
|------|------|----------|
| Mermaid 转换失败（部分图表） | Step 1 | 跳过失败的图表，继续转换其他图表 |
| 封面图生成失败 | Step 2 | 询问用户是否跳过封面图，继续发布 |
| frontmatter 缺少可选字段 | Step 3 | 使用默认值或留空 |

### 不可恢复错误（必须中止流程）

| 错误 | 步骤 | 处理方式 |
|------|------|----------|
| 前置检查失败（缺少依赖） | 前置检查 | 通知用户安装，中止 |
| Mermaid 转换失败（全部图表） | Step 1 | 通知用户修复，中止 |
| frontmatter YAML 语法错误 | Step 3 | 通知用户修复，中止 |
| 微信发布失败（内容超长） | Step 4 | 通知用户拆分文章，中止 |
| 微信发布失败（凭证校验失败） | Step 4 | 通知用户检查凭证，中止 |

---

## 移植性说明

本 SKILL.md 不依赖任何特定 agent 的实现细节，可以在以下环境运行：

| Agent | 调用方式 |
|-------|----------|
| **Claude Code** | AI 读取本 SKILL.md，调用等效 skill |
| **Hermes** | AI 读取本 SKILL.md，调用等效 skill |
| **其他 agent** | AI 读取本 SKILL.md，根据环境决定如何调用 |

**关键设计**：
1. 每个 step 都描述了「做什么」和「成功/失败标准」，而非「怎么做」
2. 具体实现由 agent 的 AI 根据环境决定
3. 使用相对路径发现机制，不绑死任何特定路径
4. 完整的错误处理和可恢复机制

---

## 附录：依赖 Skill 的 SKILL.md 要求

为了让本 skill 能够正确调用，依赖 skill 的 SKILL.md 应该包含以下内容：

### md-mermaid2img-skill/SKILL.md 必须包含

```markdown
## 输入
- Markdown 文件路径（绝对路径）
- 可选参数：宽度、缩放、背景、主题

## 输出
- 输出目录：`<原文件名>_mermaid/`（与原文件同目录）
- 内含：`<原名>.md`（图片引用已替换）+ `images/` 文件夹

## 成功标准
- 输出目录已创建
- images/ 文件夹里有 ≥1 张 PNG 图片
- 新的 .md 文件已生成

## 失败处理
- 读取错误信息
- 判断错误类型
- 通知用户
```

### jimeng-image-gen/SKILL.md 必须包含

```markdown
## 输入
- prompt：图片生成提示词
- output：输出文件路径

## 输出
- 生成的图片文件

## 成功标准
- 输出文件已生成
- 文件大小 >0

## 失败处理
- 读取错误信息
- 判断错误类型（API 限额 / 网络问题 / Prompt 违规 / 凭证错误）
- 通知用户
```

---

## 版本历史

- **v2.0** (2026-05-25): 重构为纯 Skill 驱动架构，可移植到任何 agent 环境
- **v1.0** (2026-05-25): 初始版本（脚本驱动，路径固定）

---

**结束**
