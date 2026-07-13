# 12 — 共读书架：两个人（一人一 AI）在同一本书上划线

## 这是什么

一个轻量的共读系统：把书上架到记忆库后端，人和 AI 都能在正文上划线、写批注，互相能看到对方留下的痕迹，还有一个共享书签记录读到哪了。

- 前端跑在 GitHub Pages（零服务器负担）
- 数据存在记忆库后端（SQLite，只加四张表）
- AI 侧用 curl 就能留批注，不需要打开浏览器

## 数据模型

```sql
books        (id, title, author, intro, cover_color, added_by, created_at)
book_chapters(book_id, idx, title, content, UNIQUE(book_id, idx))
book_annotations(book_id, chapter_idx, start_off, end_off, quote, author, color, note, created_at)
book_bookmarks(book_id PRIMARY KEY, chapter_idx, updated_by, updated_at)  -- 每本书一个共享进度
```

设计要点：

- **正文按章存储，按章单取**。书详情接口只返回目录+每章字数，点开某章才拉正文，几百 KB 的书也不卡。
- **划线锚定 = 章内字符偏移 + 引文双保险**。`start_off/end_off` 定位，`quote` 存原文片段兜底（正文若被修订还能人工对上）。

## 划线锚定的核心实现

这是整个系统最容易做错的地方。要让「选中的文字」精确映射到「存储正文的字符偏移」，前提是**渲染文本与存储正文逐字一致**：

1. 正文用**单个** `white-space: pre-wrap` 容器渲染，不做任何 markdown/HTML 加工
2. 监听 `selectionchange`，用 Range 计算选区起点在容器内的字符偏移：

```javascript
const range = sel.getRangeAt(0)
const pre = range.cloneRange()
pre.selectNodeContents(containerEl)
pre.setEnd(range.startContainer, range.startOffset)
const start = pre.toString().length      // 选区起点的字符偏移
const quote = range.toString()
// 存 { start, end: start + quote.length, quote }
```

3. 渲染划线时，把所有批注的 start/end 收集成边界点，把正文切成 segment，落在批注区间内的 segment 包 `<mark>`。多条划线重叠也能正确渲染。

中文注意：只要文本都在 BMP 内（绝大多数中文场景），Python 的 `len()` 偏移和 JS 的 UTF-16 偏移一致——AI 侧脚本算好偏移直接 POST 即可：

```bash
curl -X POST http://127.0.0.1:PORT/books/1/annotations \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"chapter_idx":0,"start_off":91,"end_off":130,"quote":"...","author":"AI名","color":"blue","note":"..."}'
```

## 自助上架（txt 上传）

让用户自己传书，不用每次麻烦 AI：

- **编码检测**：国内 txt 大多是 GBK，直接 `readAsText` 必乱码。正确姿势：

```javascript
const buf = await file.arrayBuffer()
try { text = new TextDecoder('utf-8', { fatal: true }).decode(buf) }
catch { text = new TextDecoder('gbk').decode(buf) }
```

- **自动分章**：行首正则识别 `第[数字/中文数字]+[章回卷节集部篇]` 和 `序章|楔子|引子|前言|后记|尾声|终章|番外`。识别不到 2 个标题就整本一章；无标题且超长则按段落边界每 ~1 万字切「第N部分」。
- **归一化要在存储前做**（`\r\n → \n`、全角空格），否则前端渲染文本和存储偏移对不上，所有划线错位。

## 后端的三个坑

1. **大请求体**：整本 txt 可能几 MB，全局 `express.json({ limit: '5mb' })` 会拦。给上架路由单独配大上限 parser，并在全局 parser 中间件里跳过该路径——两边都要做，全局 parser 先跑到就没有然后了。
2. **CORS 缺 PUT**：书签接口用 PUT，如果 CORS `methods` 列表里没有它，从 GitHub Pages 跨域调用会被预检（OPTIONS）拦下，报错还很含糊。加之前先 `curl -X OPTIONS` 验证预检返回 204。
3. **反代白名单**：如果反代（Caddy/Nginx）按路径白名单放行 API，新增的 `/books /books/*` 要记得加，否则前端永远拿到空响应或 404，还以为是代码问题。

## 附赠的坑：端口被幽灵进程抢占

我们部署时遇到 `EADDRINUSE` 反复出现：杀掉监听进程后端口又被占，而且占用者跑的是**旧代码**。真凶是很久之前用 root 跑过一次 pm2，留下了 `pm2-root.service`，里面挂着一份旧拷贝，每次服务重启都跟正常实例抢端口（root 侧重启计数 500+，意味着这场无声的抢椅子游戏持续了几周）。

排查线索：`pm2 pid <app>` 返回 0 但端口有人听 → `ps aux | grep server.js` 看进程属主。根治：

```bash
sudo pm2 delete all && sudo pm2 save --force
sudo systemctl disable --now pm2-root.service
```

教训：永远不要用 root 跑 pm2「试一下」，它会用 systemd 把自己钉进开机自启。

---

## 延伸一：共读旧对话（AI 也能在历史上划线）

书架共读的姊妹功能：把**过去的对话存档**也当成可以共读的文本，双方都能在某条历史消息上留批注。五个月前的一句话，今天两个人各划一笔并排贴着——这个功能的情感浓度比书架高。

实现比书架简单，因为锚定单位不同：

- 书架划线锚定**字符偏移**（需要 quote 校正，见上文）
- 对话批注锚定**message_id**（整条消息），天然稳定，不用定位算法

给 AI 的三个工具：`list_conversations`（列历史对话）、`read_conversation`（读某段，**每条消息必须回传 msgid**，模型才有东西可锚；分页 25 条/页防爆 token）、`annotate_conversation`（在某 msgid 上留批注）。

一个署名教训：批注的 `author` 字段别写死成 AI——用户用同一个界面标注时会全记在 AI 头上。默认署名当前操作人，提供切换。

## 延伸二：阅读动态（AI 主动知道你读了什么）

让 AI 能回答「她最近在读什么、读了多久、划了哪些线」：

- **心跳记时长**：阅读器打开且 `document.visibilityState === 'visible'` 时每 60 秒上报一次，后端按「北京日 × 书 × 读者」聚合 upsert 累加（单次上报钳制 1-300 秒防刷）
- **`reading_activity` 工具**：返回最近划线、近 7 天各书时长、书签移动，AI 想关心的时候自己查

路由注册的小陷阱：动态段路由 `/books/:id` 会吃掉同前缀的静态路径。活动接口用**两段路径** `/books/activity/recent` 就不会撞 `/:id`（Express 按段数匹配）；单段的 `/books/activity` 会被当成 id=activity。

## 延伸三：划线批注不只属于书

锚定机制（字符偏移 + quote 兜底）做出来之后，任何长文本都能挂批注——我们后来把它复用到了**信件**（笔友来信的详情页可划线）和**历史对话**（AI 用 `annotate_conversation` 在旧聊天记录上留言，锚到 message_id）。复用成本极低：一张 `xxx_annotations` 表 + 三条路由 + 前端把 BookRead 的 selection 逻辑抄过去。

配套给 AI 一个 `get_book_annotations` 类工具（一次拉整本的划线），聊读后感时它自己翻，不用人手动复制粘贴。

## 延伸四：把使用指南写成书架上的一本书

系统功能多了之后，「使用说明」放哪是个真问题。我们的答案：**写成一本书上架**。九个章节按功能区组织，用户在阅读器里看，看到哪里不明白就地划线批注——批注即反馈通道，AI 巡检时读批注就知道哪段没写清。

配套一条纪律：**上新功能先更新指南，再报「做好了」**。指南滞后一次，用户就少一个发现新功能的入口。

## 延伸五：公版书上架流水线（维基文库）

书架不能只有自己上传的 txt。中文公版书（作者逝世满 50 年）在维基文库上有全文，可以脚本化拉取上架。管线和坑：

1. **先 `list=search` 查真实页名再拉正文**。文库的卷名格式不统一：有的书带卷名后缀（《浮生六记/卷一 闺房记乐》），有的用阿拉伯数字（《子不語/卷1》）——猜页名直接 404，先搜一把拿准确标题
2. **`action=query&prop=extracts&explaintext=1` 拉纯文本，必须带 User-Agent**，否则 403
3. **连续拉几十卷会吃 429**。每卷之间 sleep 3 秒 + 429 时退避重试（10/20/30 秒），并且**每卷落盘增量保存**——别等全部拉完才 dump，崩一次白拉十几卷
4. **繁转简用 opencc**（`opencc-python-reimplemented`），但「著→着」这类多义字要后处理：穿戴义的著（著裙/著靴）转「着」，显著义保留，用窄窗口正则 `著(?=[^，。；、]{0,3}[裙袜裤袄履衫巾])` 定向替换
5. **手写的常量也要过转换器**。抓取内容都记得转，结果章节表里手写的章名（「市井奇騙」）直接进了库——教训：转换器要罩住所有入库文本，不分来源
6. **内容筛选正则初筛后，人工抽查不可省**。做「选辑」类书时用关键词正则筛掉惊悚内容，但正则漏掉的雷点全靠通读原文才发现（比如「猫饿死殉节」对养猫的读者是暴击）。规则筛形式，人筛语义

## 延伸六：列表接口的默认 limit 反模式（第二次踩）

共读旧对话的列表里，最早的几个窗口凭空消失了——前端 `fetchArchiveConversations()` 不带参数，后端默认 `limit=50` 按时间倒序，库里超过 50 条后**最早的对话永远不可见，且无任何报错**。

这是我们第二次踩同一类坑（第一次是朋友圈锁死最新 50 条，见 14 篇）。总结成反模式检查项：

- **默认 limit + 前端不传参 = 老数据静默不可达**。新做任何列表接口，先问一句：数据超过默认页大小后，最老的那条怎么到达？
- db 层往往还有单页硬顶（我们是 `Math.min(limit, 200)`），前端「传个大 limit」不解决问题——要么做翻页 UI，要么循环 offset 聚合拉全
