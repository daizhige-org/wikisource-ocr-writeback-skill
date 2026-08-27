# wikisource-ocr-writeback

一个给 coding agent 用的 skill：把 Wikisource 上一部**已有扫描件的书**逐页做 OCR，然后写回 `Page:` 名字空间。

适用前提是这本书的扫描件（PDF/DjVu）已经在 Wikimedia Commons 上，并且在 Wikisource 建好了 `Index:` 页。剩下的活——逐幅取图、识别、存盘、写回、排版——由 agent 按这份 skill 来做。

## 它解决什么

逐页 OCR 加写回本身不难，难的是一堆只有踩过才知道的细节：

- 缩略图宽度超过原图渲染宽度会 HTTP 400，报错还看不出原因
- 写回要写 `page.body`，写 `page.text` 会把 `<pagequality>` 结构冲掉
- 设质量等级是 `page.ql`，名字更像的 `page.quality_level` 是只读的
- 点线目录、竖排、表格，Google OCR 会彻底崩掉，得换成看图录入
- OCR 引擎会偷偷把繁体规范成简体（或反过来），破坏底本用字

这份 skill 把这些连同排版约定一起写死了，另附三个可直接改用的脚本。

## 安装

**使用 `npx skills`（推荐）**：

```bash
# 全局安装（所有项目及支持的 agent 可用）
npx skills add daizhige-org/wikisource-ocr-writeback-skill -g

# 或安装到当前项目
npx skills add daizhige-org/wikisource-ocr-writeback-skill
```

**手动安装（Claude Code）** —— 放进 skills 目录即可，重启后自动发现：

```bash
# 只给自己用
git clone https://github.com/daizhige-org/wikisource-ocr-writeback-skill \
  ~/.claude/skills/wikisource-ocr-writeback

# 或只给某个项目用
git clone https://github.com/daizhige-org/wikisource-ocr-writeback-skill \
  <项目目录>/.claude/skills/wikisource-ocr-writeback
```

**其他 agent** —— `SKILL.md` 就是一份普通的 Markdown 说明，没有专有格式。把它拼进 `AGENTS.md`、`.cursorrules` 之类的指令文件，或者开工前直接丢进对话里，一样能用。

## 依赖

| 用途 | 装什么 |
|---|---|
| 写回 Wikisource | `pip install pywikibot`，并配好 `user-config.py` |
| 看图录入时渲 PDF | `poppler`（提供 `pdftoppm`；macOS `brew install poppler`） |
| 切图放大核对数字 | `pip install pillow` |

OCR 用的是 [ocr.wmcloud.org](https://ocr.wmcloud.org)（Wikimedia 官方服务），不需要 API key。

## 怎么用

装好之后，直接用大白话说要干什么，agent 会自己找到这份 skill：

> 帮我把 Commons 上的《某某文集》做 OCR 写回 zh.wikisource，Index 页已经建好了，先试前 20 幅。

它会先读 `Index:` 页搞清楚幅号和印刷页码的对应，列出已存在的页面决定跳过哪些，然后跟你确认范围和要不要覆盖，再开跑。

## 登录：帐号、权限、密码文件

写回需要你自己的 Wikisource 帐号。**不需要**专门去申请「机器人」身份——那是给高频批量作业的，要走社群批准流程；这份 skill 的节奏（每次编辑间隔 2–3 秒）用普通帐号就行。

推荐用 **BotPassword**：绑在你自己帐号上的一组独立密钥，自助创建、即时生效、无需任何审批，而且可以只给它一小撮权限。万一泄露，吊销它不影响你的主密码。

### 一、申请 BotPassword

1. 登录后打开 [`Special:BotPasswords`](https://zh.wikisource.org/wiki/Special:BotPasswords)
2. 起个名字（下面假设叫 `ocr`），勾选权限（见下表）
3. 保存后会显示 **登录名 `你的用户名@ocr`** 和 **一串 32 位密码**

密码**只显示这一次**，当场存好。丢了就回同一页面重置。

> BotPassword 只能走 API，不能用来登网页界面——这是设计如此，不是出错。

### 二、勾哪些权限

按 Special:BotPasswords 上的中文标签，**下面打「必须」的两条都要勾，缺一不可**：

| 权限（括号内是内部名） | 要不要 | 为什么 |
|---|---|---|
| 基本权限 `(basic)` | 自动包含 | 读页面、拿 token |
| **编辑存在的页面 `(editpage)`** | **必须** | 改已有的 `Page:` 页。ProofreadPage 的「设校对等级」（`pagequality`）也挂在这一条下面，不必另找 |
| **创建、编辑和移动页面 `(createeditmovepage)`** | **必须** | **新建页面只由这一条授予。** 一部书的 `Page:` 页绝大多数是新建的，漏了它，改已有页面正常、一到建页就失败 |
| 大量操作（机器人）访问权限 `(highvolume)` | 不用 | 只有帐号真有 bot 标记时才有意义 |
| 编辑受保护的页面 / 上传新文件 / 删除 | **别勾** | 这份 skill 完全用不到 |

这两条互不覆盖：名字里都有「编辑」，但 `editpage` 只管改已存在的页面，`createeditmovepage` 只管新建（和移动）。**只勾其中一条，作业跑到一半必然出错**——只勾前者是建页时报权限不足，只勾后者是改已有页面时报权限不足。

给多了则是白白扩大泄露后的破坏面，所以除这两条外别的都不要勾。

配好之后，开跑前先自检一遍实际拿到的权限：

```bash
PYWIKIBOT_DIR=<配置目录> python3 -c "
import pywikibot
s = pywikibot.Site('zh', 'wikisource'); s.login()
r = set(s.userinfo['rights'])
print(s.user(), {x: x in r for x in ['edit', 'createpage', 'pagequality']})
"
```

三项都是 `True` 才算齐；`createpage` 是 `False` 就是漏勾了「创建、编辑和移动页面」，回 Special:BotPasswords 给同一个 BotPassword 补勾即可（补勾不会换密码）。

### 三、两个配置文件

pywikibot 读两个文件，放在同一个目录里（下面叫「配置目录」）：

**`user-config.py`** —— 谁、在哪个站：

```python
family = 'wikisource'
mylang = 'zh'
usernames['wikisource']['zh'] = '你的用户名'      # 不带 @ocr 后缀
password_file = 'user-password.py'
```

> **最容易漏的一步**：`password_file` 默认是 `None`。不写这一行，pywikibot 根本不会去看密码文件，只会在终端反复问你要密码。文件名不是固定的，`user-password.py` 只是惯例，路径按相对于 `user-config.py` 所在目录解析。

**`user-password.py`** —— 密码本体，每行一个 Python 元组：

```python
('你的用户名', BotPassword('ocr', '那串32位密码'))
```

注意这里拆成两段：**用户名不带后缀**，`@` 后面的 `ocr` 作为 `BotPassword()` 的第一个参数，pywikibot 自己拼成 `你的用户名@ocr` 去登录。

不用 BotPassword 而是明文主密码时，格式是下面几种之一（越往下限定得越窄，同一用户名以最后匹配到的那行为准）：

```python
('用户名', '密码')                              # 所有站点
('wikisource', '用户名', '密码')                # 某个 family
('zh', 'wikisource', '用户名', '密码')          # 某个具体站点
```

不过 Wikimedia 各站现在都不建议明文主密码，用 BotPassword。

### 四、几条硬性约束

这些是 pywikibot 代码里真会拦的，踩了直接报错：

- **文件权限必须是 `600`**（仅本人可读写），否则启动时报权限错：
  ```bash
  chmod 600 user-password.py
  ```
- **不能是符号链接**，得是真文件
- 单行不超过 250 字符，`, ` 不超过 4 处
- pywikibot **10.7.1 起**密码行按字面量解析（不再当 Python 代码执行），所以只能老老实实写元组，别在里面做拼接或调函数

验证配好没有：

```bash
PYWIKIBOT_DIR=<配置目录> python3 -m pywikibot.scripts.login
```

打出 `Logged in on zh:wikisource as 你的用户名` 就成了。

### 五、agent 与密码的边界

skill 里写死了两条：

- agent 只通过 `PYWIKIBOT_DIR` 指向配置目录，**不去读 `user-password.py` 的内容**——密码由 pywikibot 自己读，agent 不需要也不应该看见
- 找不到配置就停下来问你，**不自己申请或编造** bot 密码

配置目录建议放在这个 repo 之外。仓库的 `.gitignore` 已经挡了 `user-config.py`、`user-password.py` 和登录 cookie（`*.lwp`），算第二道保险。

## 几条内置的规矩

skill 里写死了这些规矩，不用每次交代：

- **默认不覆盖已有正文**，除非你明确说要覆盖
- 纯 OCR 的页面一律标**质量等级 1（未校對）**，不冒充已校对
- 每次编辑之间等 2–3 秒，编辑摘要老实写明是机器识别
- 开跑前先跟你确认做多少页，不默认一口气跑几百页
- **照原书用字录入**：繁简混排、异体字、旧字形一律照录，不做规范化，不用转换工具批量转；可疑的字照录后单独列清单给你判断

## 配套脚本

在 `scripts/` 下，用前先改开头那几个常量（书名、md5 前缀等）。

| 脚本 | 干什么 |
|---|---|
| `ocr_fetch.py` | 逐幅调 ocr.wmcloud.org，结果落盘缓存，重跑不会重复请求 |
| `push_pages.py` | 用 pywikibot 写回 `Page:` 名字空间，默认跳过已有正文 |
| `format_toc.py` | 点线目录／双栏索引的 wikitext 生成器骨架 |

脚本里的示例内容（书名、目录条目）都是编的，换成你自己那本书的即可。

## 说明

写这份 skill 的实践对象是中文 Wikisource 上民国到 1950 年代的铅印书，排版约定和模板名（`Dotted TOC page listing`、`multicol`）是按 zh.wikisource 的习惯写的。换到别的语言站点，接口和 pywikibot 那部分照用，模板部分要改。
