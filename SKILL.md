---
name: wikisource-ocr-writeback
description: "给 Wikisource 上一部已有扫描件的书逐页做 OCR 并写回。用 ocr.wmcloud.org（Wikimedia 官方 OCR，可选 Google／Tesseract／Transkribus 引擎）识别 Commons 上的 PDF/DjVu，结果存盘，再用 pywikibot 以用户帐号写进 Page: 名字空间。也包括 OCR 读不动时（点线目录、表格、竖排、模糊数字）改成本地高清渲图＋自己看图录入的办法，以及录入时的用字规矩和 ProofreadPage 排版约定。触发词：OCR 写回、ocr.wmcloud.org、wmcloud OCR、Wikisource 逐页识别、Page 名字空间批量建页、目录/索引格式化。"
---

# Wikisource 扫描书的 OCR 与写回

**做什么**：一部书的扫描件已经传到 Wikimedia Commons、也在 Wikisource 建好了目录页，现在要把每一幅扫描图的文字弄出来，填进对应的页面里。

## 名词表

先把术语讲清楚:

| 词 | 意思 |
|---|---|
| **幅 / 幅号** | 扫描件里的第几张图，从 1 数起。**跟书上印的页码不是一回事**——封面、扉页、目录都占幅号，所以第 20 幅上印的可能是「第 5 页」。本文说「页号」时一律指幅号。 |
| **`Index:` 页** | 这本书在 Wikisource 上的总目录页，名字跟 Commons 上的文件名一样。它管着全书的分卷、幅号与印刷页码的对应关系。 |
| **`Page:` 页** | 一幅扫描图对应一个页面，地址是 `Page:<书名>.pdf/<幅号>`。文字就填在这里。 |
| **`<pagelist>`** | `Index:` 页里的一段标记，规定「第几幅印的是第几页」。想知道幅号怎么对应印刷页码，读它。 |
| **校对质量等级** | 每个 `Page:` 页都有个 0–4 的状态，标这页校到什么程度了。**机器 OCR 出来的、没人逐字核对过的，一律标 1（未校對）**。 |
| **ProofreadPage** | 管着上面这套机制的 MediaWiki 扩展，pywikibot 里对应 `ProofreadPage` 这个类。 |
| **点线目录** | 目录里「条目名……页码」那种，中间用点或省略号连起来的排法。OCR 的重灾区。 |

## 动手之前：几条不能破的规矩

- **必须有能用的 pywikibot 登录配置。** 去找 `user-config.py`（通常在用户别的项目目录里），然后用 `PYWIKIBOT_DIR=<那个目录>` 指过去。**不要去打开 `user-password.py` 看内容**——pywikibot 自己会读，你不需要看见密码。确认配置里有 `usernames['wikisource']['zh'] = '<用户名>'` 这样的条目就行。
- **开跑前先验一次权限，别跑到一半才发现建不了页。** 登录后查 `site.userinfo['rights']`，`edit`、`createpage`、`pagequality` 三项都要在：

  ```bash
  PYWIKIBOT_DIR=<配置目录> python3 -c "
  import pywikibot
  s = pywikibot.Site('zh', 'wikisource'); s.login()
  r = set(s.userinfo['rights'])
  print(s.user(), {x: x in r for x in ['edit', 'createpage', 'pagequality']})
  "
  ```

  少了 `createpage` 就是这个 BotPassword 没勾「创建、编辑和移动页面（createeditmovepage）」——**改已有页面会正常，一到建新页就失败**，而一部书的 `Page:` 页绝大多数是新建的。停下来让用户回 Special:BotPasswords 给同一个 BotPassword 补勾（补勾不换密码），别自己想办法绕。少了 `edit` 则是没勾「编辑存在的页面（editpage）」，`pagequality` 也来自这一条。
- **找不到配置就停下来问用户**，不要自己去申请或编造 bot 用户密码。（怎么申请 BotPassword、要勾哪些权限、密码文件什么格式，README 里有，可以直接指给用户看。）
- **默认不覆盖已经有正文的页面。** 别人写过的、或者用户自己写过的，跳过。除非用户明确说要覆盖。
- 每次编辑之间等 2–3 秒，别让服务器过载。
- 编辑摘要要声明这是机器识别的，比如 `OCR (Google, via ocr.wmcloud.org)`。
- 先跟用户说清**打算做多少**（整本还是先试十来页）、**要不要覆盖**，别默认一口气跑几百页。

## 一、先摸清这本书的结构

```bash
# 读 Index 页的原始内容，里面有 pagelist：总共多少幅、怎么分卷、幅号对应第几页
curl -s "https://zh.wikisource.org/w/index.php?title=Index:<书名>.pdf&action=raw"

# 列出已经存在的 Page: 页，用来决定哪些该跳过
# （ns=104 就是 Page: 名字空间的编号）
curl -s -G "https://zh.wikisource.org/w/api.php" \
  --data-urlencode "action=query" --data-urlencode "format=json" \
  --data-urlencode "generator=allpages" --data-urlencode "gapnamespace=104" \
  --data-urlencode "gapprefix=<书名>.pdf" --data-urlencode "gaplimit=500"
```

## 二、拿到每一幅的图片地址

OCR 服务只肯读 `upload.wikimedia.org`（或测试站 `upload.wikimedia.beta.wmflabs.org`）上的图，别处的图它不收。多页 PDF/DjVu 里单独一幅的缩略图，地址长这样：

```
https://upload.wikimedia.org/wikipedia/commons/thumb/<h1>/<h1h2>/<文件名>/page<幅号>-<宽度>px-<文件名>.jpg
```

`<h1>/<h1h2>` 是文件名 md5 的第 1 位、前 2 位。**别自己算**，问一次 API 就有了，顺带还能拿到原图宽度：

```bash
curl -s -G "https://commons.wikimedia.org/w/api.php" \
  --data-urlencode "action=query" --data-urlencode "format=json" \
  --data-urlencode "titles=File:<书名>.pdf" \
  --data-urlencode "prop=imageinfo" --data-urlencode "iiprop=size|url" \
  --data-urlencode "iiurlwidth=800"
```

返回里的 `thumburl` 就是现成的模板，照着换幅号和宽度即可。

> **坑**：请求的宽度不能超过原图渲染宽度（返回里的 `imageinfo.width`，老扫描件常在 1200 上下）。超了直接 HTTP 400，而且报错信息看不出原因。**960px 一般都安全。**

## 三、让 OCR 读图

接口是 `GET https://ocr.wmcloud.org/api.php`（写成 `/api` 也一样），返回 JSON，识别出的文字在 `text` 字段里。

| 参数 | 说明 |
|---|---|
| `engine` | 用哪个引擎：`google`（默认，中文最好用）／`tesseract`／`transkribus` |
| `image` | 上一步那个缩略图地址，**必须 URL-encode** |
| `langs[]` | 语言提示。用 Google 引擎认中文时可以不给 |
| `psm` | 只有 tesseract 认，控制它怎么切分页面 |
| `crop` / `line_id` / `rotate` | 分别是只认某一块、Transkribus 的行模型、旋转 |

想知道有哪些引擎和语言可选：`/api/models`、`/api/available_langs`。接口文档在 `https://ocr.wmcloud.org/api/doc`。

实际跑用 `scripts/ocr_fetch.py`——它会重试，也会把结果存盘，重跑不会重复请求。

## 四、写回 Wikisource

用 `scripts/push_pages.py`。三个容易踩的地方：

- **要写 `page.body`，不要写 `page.text`。** `page.text` 是整页的原始内容，包含 `<noinclude><pagequality …/></noinclude>` 那一圈结构；直接覆盖会把它冲掉。`page.body` 只动正文那部分。
- **设质量等级用 `page.ql = 1`。** 名字更像的那个 `page.quality_level` 是只读的，赋值会抛 `AttributeError`。
- 运行时要带上配置目录：`PYWIKIBOT_DIR=<配置目录> python3 scripts/push_pages.py <起幅> <止幅>`

## 五、OCR 读不动的时候：自己看图录入

Google 引擎认**普通正文**够用，但下面这些它会彻底崩掉，别硬试：

- **点线目录和索引**：条目名和页码会被拆到不同行、顺序还打乱，结果完全没法用。
- 竖排、表格、双行小注。
- 老铅印的**模糊数字**：笔画断掉之后，`0` 经常被读成 `(`、`)` 或 `:`，`9` 被读成 `(`。200 dpi 下页码几乎必错。

这些情况改走这条路：

```bash
# 下原始 PDF——比缩略图清楚太多了
curl -sL -o book.pdf "https://upload.wikimedia.org/wikipedia/commons/<h1>/<h1h2>/<文件名>.pdf"

# 渲成图片：正文 200 dpi 够用，核对模糊数字时上 400 dpi
pdftoppm -f <起幅> -l <止幅> -r 200 -jpeg -jpegopt quality=92 book.pdf img/p
```

poppler 有时会报 `Unknown segment type in JBIG2 stream` 之类的错，一般不影响输出，确认图能打开就继续。

然后**用 Read 工具直接看这些图**，一页页录入。数字拿不准就把那一列切出来放大再看：

```python
from PIL import Image
im = Image.open("img/p-429.jpg")          # 400 dpi 下大约 3308×5026
im.crop((1150, 3350, 1700, 4700)).resize((1100, 2700)).save("num.jpg")
```

一次别塞太多张图进去，**两张一批**比较稳；页码那一栏单独切出来复核一遍。

## 六、用字：原书印什么，就录什么

**这条最容易被无意中破坏，单独拎出来讲。**

- **照着排面录，一个字都不要「顺手规范化」。** 中华民国及中国大陆 1950 年代的排印本经常繁简混用——同一页上「書」和「书」并存、「記」和「记」并存，还有「暫」「規」「認眞」这类当时通行的写法。这是底本的真实面貌，**不要统一成简体，也不要统一成繁体**。
- **绝对不要拿转换工具批量转换。** 一转就把底本信息抹平了，而且不可逆。
- **异体字和旧字形照录**：「眞／真」「爲／為／为」「體／体」「鬬／鬥」，印的是哪个就写哪个。
- **注意 OCR 会偷偷改字。** Google 引擎有把繁体规范成简体（或反过来）的倾向，尤其在混排的页面上。所以 OCR 出来的每一页都要对着图核一遍用字，不能只看句子通不通顺。看图录入时同样要盯住——凭语感打字很容易顺手打成自己习惯的那个写法。
- **可疑的字照录，不要擅自「修正」。** 笔画断掉、疑似漏印、明显的排印错误，一律先按看到的样子录进去，然后在给用户的汇报里单独列一张清单，让用户自己判断。

原因很简单：Wikisource 目标是希望忠实于底本。校勘意见可以写在讨论页或注释里，但正文必须是原书的样子。

## 七、排版约定

**点线目录条目**——照抄 zh.wikisource 上做熟了的书目怎么写，比如 `Index:魯迅全集01 (1948).pdf`：

```
{{Dotted TOC page listing|entrytext=記客歲冬日與友人夜話西山舊事|pagetext=(5)|symbol=……|spaces=0|col3-width=4em|hi=1em}}
```

- `hi=1em` 是折行时的悬挂缩进，让长条目换行后跟原书对齐一致。
- `col3-width` 是页码那一列的宽度，**全书必须统一**（页码是三位数就用 `4em`），否则页码列参差不齐。
- 子条目（「附：…」「又：…」）外面套一层 `{{left|offset=1em|…}}`，再下一层就是 `2em`。

**页码要不要括号，看排面。** 原书目录印成「( 154 )」就写 `pagetext=(154)`；索引里印的是光秃秃的数字，就写 `pagetext=154`。

**双栏索引**：用 `{{multicol}}` … `{{multicol-break}}` … `{{multicol-end}}` 包起来，栏内保留原书用空行做的分组。

**条目跨页**：有的条目文字在 A 幅末尾起头，点线和页码到 B 幅才出现。按印刷实况分开处理——A 幅末尾留一行纯文本（**不套模板**），B 幅用模板接上剩下的文字和页码。

**标题类**：章节标题 `{{c|{{larger|总　　类}}}}`；书名 `{{x-larger|…}}`；横线 `{{rule|30em}}`。

## 八、保存之前先试渲染

批量写回之前，拿 API 空跑一遍渲染，确认模板没写错——不然错的模板会一次性铺满几十页，收拾起来很烦：

```bash
curl -s -X POST "https://zh.wikisource.org/w/api.php" \
  -d "action=parse&format=json&contentmodel=wikitext&prop=text" \
  --data-urlencode "text@out/13.txt"
```

## 九、产出怎么组织

**把条目存成结构化的数据，wikitext 由函数生成**（骨架见 `scripts/format_toc.py`），不要直接手写一堆 wikitext。

这样做的好处是：用户回头说「页码改成带括号」「缩进再进一层」「`col3-width` 换成 5em」时，改一个函数重跑就行，不用几十页手动改一遍。

## 配套脚本

都在本 skill 的 `scripts/` 下，用之前先改开头那几个常量（书名、md5 前缀这些）。

| 脚本 | 干什么 |
|---|---|
| `ocr_fetch.py` | 逐幅调 ocr.wmcloud.org，结果存盘缓存 |
| `push_pages.py` | 用 pywikibot 写回 `Page:` 名字空间 |
| `format_toc.py` | 点线目录／双栏索引的 wikitext 生成器骨架 |
