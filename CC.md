NIP-CC
======

寻宝（Geocaching）事件
---------------------

`draft` `optional`

本 NIP 定义了在 Nostr 上进行 geocaching（寻宝）的 event kind。这些 event 允许用户以去中心化方式创建、分享和记录 geocache。

## Geocache 列表事件（Kind 37516）

Geocache 列表 event 是 kind `37516` 的 addressable event，结构如下：

```json
{
  "kind": 37516,
  "content": "<cache description>",
  "tags": [
    ["d", "<cache-identifier>"],
    ["name", "<cache-name>"],
    ["g", "<geohash>"],
    ["D", "<1-5>"],
    ["T", "<1-5>"],
    ["S", "<size>"],
    ["t", "<type>"],
    ["n", "<type-modifier>"],
    ["hint", "<plaintext hint>"],
    ["mission", "<key quest mission>"],
    ["image", "<image-url>"],
    ["r", "<relay-url>"],
    ["verification", "<verification-pubkey-hex>"]
  ]
}
```

列表 event 需要包含有关 cache 的所有信息，以及寻找 cache 所需的相关信息。这些信息包括 `name`、位置（`g`）、难度（`D`）和地形（`T`）评分，以及尺寸（`S`）。cache 类型（`t`）是可选的；如果未指定，默认值为 `traditional`。

cache 类型由各个客户端决定，常见类型包括 `traditional`、`multi` 和 `mystery`。客户端应根据自身实现需求决定支持哪些 cache 类型。

这些要求是广为人知的，并遵循现有标准，例如 [geocaching.com](https://www.geocaching.com/help/index.php?pg=kb.chapter&id=97) 上列出的标准。

这些 event 被假定由 cache 提交者拥有，核心细节应由该提交者维护。不过，社区日志也应提供有关 cache 当前状态和有效性的上下文。

### Content 字段

content 字段包含 cache 描述以及有关该 cache 的任何附加信息。

### 标签

- `d`（必需）- cache 的唯一标识符
- `name`（必需）- cache 的人类可读名称
- `g`（必需）- cache 位置的 geohash。为了允许邻近搜索，请包含不同精度级别（3-9 个字符）的多个 geohash 标签
- `D`（必需）- 表示谜题/寻找难度的 1-5 整数（可索引）
- `T`（必需）- 表示地形难度的 1-5 整数（可索引）
- `S`（必需）- 下列之一：`micro`、`small`、`regular`、`large`、`other`（可索引）
- `t`（可选）- cache 类型，常见值包括：`traditional`、`multi`、`mystery`。如果未指定，默认值为 `traditional`
- `n`（可选）- 影响生命周期、认领语义或奖品性质的类型修饰符。见[类型修饰符](#type-modifiers)。可以存在多个 `n` 标签，但每个修饰符类别最多一个
- `hint`（可选）- 帮助寻找 cache 的明文提示
- `mission`（可选）- 明文“Key Quest”任务，寻找者应完成该任务以合法认领 cache（例如密码短语、谜语答案或要携带的物品）。一个 treasure MUST NOT 包含超过一个 `mission` 标签；如果存在多个，客户端 SHOULD 使用第一个并忽略其余标签。当存在该标签时，客户端 SHOULD 将 found-log 提交限制为具有物理到场证明的寻找者（通常是 cache 位置处的验证密钥）。任务完成情况 MAY 记录为 [NIP-GD](NIP-GD.md) Good Deed event，其 `a` 标签引用该 cache
- `image`（可选）- 与 cache 相关的图片 URL
- `r`（可选）- 用于日志的首选 relay URL
- `verification`（可选）- 用于验证在此 cache 处找到记录的十六进制编码公钥
- `F`（可选）- 锁定 first-to-find 获胜者。仅当 treasure 带有 `first-to-find` `n` 修饰符时有效。格式：`["F", "<winner-pubkey-hex>"]`。见[类型修饰符 › `first-to-find`](#claim-semantics)。SHOULD 最多存在一个 `F` 标签

## Found Log 事件（Kind 7516）

Found log event 记录对 geocache 的成功访问：

```json
{
  "kind": 7516,
  "content": "<log message>",
  "tags": [
    ["a", "37516:<pubkey>:<d-tag>"]
  ]
}
```

### 标签

- `a`（必需）- 对正在记录的 geocache 的引用
- `image`（可选）- 访问时拍摄的照片
- `verification`（可选）- 嵌入式验证 event（见 Verified Finds 小节）

## 评论日志事件（Kind 1111）

未找到类日志使用 comment event（kind `1111`），遵循 NIP-22 评论结构：

```json
{
  "kind": 1111,
  "content": "<log message>",
  "tags": [
    ["A", "37516:<pubkey>:<d-tag>"],
    ["K", "37516"],
    ["P", "<cache-owner-pubkey>"],
    ["a", "37516:<pubkey>:<d-tag>"],
    ["k", "37516"],
    ["p", "<cache-owner-pubkey>"],
    ["t", "<log-type>"]
  ]
}
```

这些 event 按照 NIP-22 评论线程模型，通过人工报告捕获关于 cache 的失败、备注和状态相关信息；在该模型中，geocache 列表同时是根内容和父内容。

评论日志类型包括 `dnf`（did not find，未找到）、`note`（有帮助或中性的上下文）以及 `maintenance`（cache 需要维护）。如果不存在 `t` 标签，则该评论被假定为一般备注。

cache 所有者可以使用 `t` 标签中的 `archived` 标签值正式归档 cache，从而在不完全删除的情况下保留 cache 历史。

### 标签

`A`/`K`/`P`（root）和 `a`/`k`/`p`（parent）标签遵循 [NIP-22](https://github.com/nostr-protocol/nips/blob/master/22.md)。对于这些顶级评论，root 和 parent 相同：它们引用 geocache 列表（`37516:<pubkey>:<d-tag>`）、kind `37516` 以及 cache 所有者的 pubkey。

- `t`（可选）- 日志类型：`dnf`、`note`、`maintenance`、`archived`。如果省略，假定为 `note`
- `image`（可选）- 访问时拍摄的照片

## Geocache 验证事件（Kind 7517）

验证 event 提供某人实际到达 geocache 的加密证明。这些 event 由 cache 的验证私钥签名。

```json
{
  "kind": 7517,
  "content": "Geocache verification for <finder-npub>",
  "tags": [
    ["a", "<finder-pubkey-hex>:<geocache-naddr>"]
  ]
}
```

### Content 字段

content 字段必须遵循静态格式：`"Geocache verification for <finder-npub>"`，其中 `<finder-npub>` 是找到该 cache 的人的 NIP-19 编码公钥（npub）。

### 标签

- `a`（必需）- 复合标识符，包含寻找者的十六进制 pubkey 以及正在验证的 geocache naddr

### 用法

当寻找者获得 cache 的验证私钥访问权限时（通常通过 cache 位置处的二维码），会创建验证 event。该 event 必须由 cache 的验证私钥签名，并且可以：

1. 作为 JSON 字符串嵌入到 Verified Found Log event（kind 7516）的 `verification` 标签中。
2. 作为独立 event 发布到 relay。
3. 同时嵌入和发布，以实现冗余。

## 已验证找到记录

启用验证的 geocache 可以提供寻找者实际到达该 cache 的加密证明。这通过由 cache 验证密钥签名的验证 event（kind 7517）实现。

### 验证流程

当 cache 具有包含公钥的 `verification` 标签时，寻找者可以通过以下方式创建已验证日志：

1. 获取 cache 的验证私钥（通常通过 cache 位置处的二维码）
2. 创建由该密钥签名的验证 event（kind 7517）
3. 将验证 event 嵌入到自己的日志条目中

### 验证校验

要校验一次已验证找到记录：

1. 检查验证 event 是否由预期的验证公钥签名
2. 验证 `a` 标签中的寻找者 pubkey 是否与日志作者匹配
3. 确认 `a` 标签中的 geocache naddr 正确引用目标 cache
4. 使用标准 Nostr 验证方式校验 event 签名

<a id="type-modifiers"></a>

## 类型修饰符

Geocache 列表 MAY 包含一个或多个 `n` 标签，用于通过额外类型修饰符对 geocache 进行分类。不同于描述 cache *如何被找到*的 `t` cache-type 标签，`n` 修饰符描述 cache 发布后*如何表现*，即其生命周期、认领语义或奖品性质。

`mission` 标签在更广义上也是一种类型修饰符，但它使用自己的专用标签，因为它携带 payload（任务文本）。客户端 SHOULD 在展示用途（徽章、过滤器等）上以一致方式处理 `mission` 和 `n` 修饰符。

### 规则

1. 每个修饰符值只属于一个**类别**（见下文）。
2. 一个 geocache SHOULD 在每个类别中最多包含一个 `n` 标签。如果同一类别中存在多个值，客户端 SHOULD 使用第一次出现的值并忽略其余值。
3. 来自不同类别的修饰符可以自由组合。除非某个特定修饰符的定义另有说明，否则任意组合都是有效的。
4. 客户端 SHOULD 忽略无法识别的 `n` 值，以便在定义新修饰符时保持向前兼容。

### 类别和修饰符

<a id="claim-semantics"></a>

#### 认领语义

影响 treasure 认领解释方式的修饰符。

- `first-to-find` — 单次认领 geocache 列表。第一个已验证 found log（带有有效嵌入式 kind 7517 的 kind 7516）构成独占认领。后续已验证 found log 仍然是到达该位置的有效记录，但不构成额外认领。一旦存在任何有效的已验证 found log，客户端 SHOULD 将 geocache 列表渲染为实际上已归档，隐藏提交找到记录的交互，并突出显示获胜寻找者。要求 geocache 列表 event 上存在 `verification` 标签。

  确定获胜日志（锁定前的暂定规则）：
  - 获胜日志是 `created_at` 值最早的已验证 found log。
  - 如果 `created_at` 相同，则通过 event `id` 的升序字典序比较打破平局。
  - 所有已验证日志都是物理到场的证据（生成它们需要访问二维码）；独占认领只归属于最早的一条。

  锁定获胜者（`F` 标签）：
  - 一旦 geocache 列表创建者确认认领，创建者 SHOULD 发布 geocache 列表 event 的新修订版，该修订版同时归档列表（添加 `["t", "archived"]`）并通过追加 `F` 标签锁定获胜者：
    `["F", "<winner-pubkey-hex>"]`
  - 该值是获胜寻找者的 pubkey（小写十六进制）。可以通过查询引用该 treasure 的 `a` 标签且作者匹配 `F` pubkey 的已验证 found log，恢复具体的获胜日志。
  - SHOULD 最多存在一个 `F` 标签。如果存在多个，客户端 SHOULD 使用第一个。
  - 当存在 `F` 标签时，客户端 MUST 将独占认领归属于 `F` 标签中的 pubkey，无论当前哪条已验证 found log 看起来最早。由于 `created_at` 由作者提供且可伪造，这可以防止已锁定的认领被后来带有伪造更早时间戳的日志取代。

#### 奖品性质

描述实体 treasure *是什么*的修饰符。

- `art` — geocache 本身是一件实体艺术作品（版画、雕塑、贴纸、zine、绘制物、壁画、装置等）。该作品是否可以带走、原地观看、只能拍照，或以其他方式互动，由其他修饰符以及 treasure 的 `content` 描述决定。

### 向前兼容

未来版本的本 NIP 或补充 NIP 中 MAY 定义新的修饰符。也 MAY 引入新的类别。实现本 NIP 的客户端 SHOULD 忽略未知的 `n` 值，而不是拒绝该 event。

## Geocache 策展列表事件（Kind 37517）

策展列表 event 是 kind `37517` 的 addressable event，用于将 geocache 组织进策展集合。客户端可以将这些集合呈现为冒险、路线、寻宝活动或任何其他主题体验。

```json
{
  "kind": 37517,
  "content": "<full description>",
  "tags": [
    ["d", "<list-identifier>"],
    ["title", "<list-title>"],
    ["description", "<short summary>"],
    ["image", "<banner-image-url>"],
    ["g", "<geohash>"],
    ["theme", "<page-theme>"],
    ["map", "<map-style>"],
    ["a", "37516:<pubkey>:<d-tag>"],
    ["a", "37516:<pubkey>:<d-tag>"]
  ]
}
```

### Content 字段

content 字段包含策展列表的完整描述，包括规则、提示、叙事，或创建者希望提供的任何其他上下文。

### 标签

- `d`（必需）- 列表的唯一标识符
- `title`（必需）- 列表的人类可读名称
- `a`（必需，1+）- 对 geocache 列表 event（kind 37516 或 37515）的引用。顺序会被保留且具有意义
- `description`（可选）- 在浏览/卡片视图中显示的简短摘要
- `image`（可选）- 横幅图片 URL
- `g`（可选）- 列表中心位置的 geohash。为便于发现，请包含多个精度级别（3-6 个字符）
- `theme`（可选）- 列表的默认页面主题。显示列表时，客户端应应用该主题，除非用户已明确选择其他主题。支持值：`adventure`、`mojave`
- `map`（可选）- 列表的默认地图样式。客户端显示列表时应将其作为初始地图样式，但允许用户更改。支持值：`original`、`dark`、`satellite`、`adventure`

### 跨作者引用

策展列表可以引用任何公开 geocache，而不受作者限制。`a` 标签使用标准 Nostr addressable event 坐标（`<kind>:<pubkey>:<d-tag>`），允许单个列表横跨来自多个创建者的 geocache。

## 客户端

为了获得最佳 Geocaching 体验，实现 geocaching 支持的客户端应：

- 支持提示编码（例如 ROT13），以防止剧透。
- 从最近的日志模式判断 cache 状态。多个 DNF 条目和/或维护备注将表明该 cache 存在问题。
- 在可用时，将日志发布到 cache 的 `r` 标签指定的 relay。
- 在接受 cache 提交前，校验 geohash 精度是否满足最低要求（8+ 个字符，micro cache 为 9+）。

## 示例

### 基本 Cache

```json
{
  "kind": 37516,
  "content": "The first Nostr treasure, left in the aftermath of Oslo Freedom Forum!",
  "tags": [
    ["d", "first-treasure-1748619568668"],
    ["name", "First Treasure"], 
    ["g", "u4x"],
    ["g", "u4xs"],
    ["g", "u4xsu"],
    ["g", "u4xsu6"],
    ["g", "u4xsu6r"],
    ["g", "u4xsu6ry"],
    ["g", "u4xsu6ryb"],
    ["D", "1"],
    ["T", "1"],
    ["S", "small"],
    ["t", "traditional"],
    ["hint", "In the branches"],
    ["image", "https://blossom.primal.net/74efe01a767b27dead71b8a9bb8278a108360438e78e55194ed9ab14a9382dd3.jpg"]
  ]
}
```

### 已验证 Cache

```json
{
  "kind": 37516,
  "content": "High-security treasure requiring physical verification!",
  "tags": [
    ["d", "verified-treasure-1748619568669"],
    ["name", "Verified Treasure"], 
    ["g", "u4xsu6ry"],
    ["D", "3"],
    ["T", "2"],
    ["S", "small"],
    ["t", "traditional"],
    ["hint", "Look for the secret code"],
    ["verification", "6805d4e5c0df48b4f76e2fdcb67a2acb1d97567b01c6fe17a236dc32f34f1c07"]
  ]
}
```

### 带 Key Quest 的 Cache

```json
{
  "kind": 37516,
  "content": "Solve the riddle to claim your prize.",
  "tags": [
    ["d", "key-quest-treasure-1748619568670"],
    ["name", "Riddle of the Old Oak"],
    ["g", "u4xsu6ry"],
    ["D", "4"],
    ["T", "2"],
    ["S", "small"],
    ["t", "mystery"],
    ["hint", "Count the rings on the fallen log"],
    ["mission", "Bring a token of nature you found along the way"],
    ["verification", "6805d4e5c0df48b4f76e2fdcb67a2acb1d97567b01c6fe17a236dc32f34f1c07"]
  ]
}
```

### First-to-Find 艺术品

一个单次认领 treasure，其中 cache 本身是一件实体艺术品。第一个已验证寻找者是独占认领者；作品如何交付（带回家、拍照等）由 cache content 描述。

```json
{
  "kind": 37516,
  "content": "Hand-pulled linocut, edition of 1, signed on the back next to the QR. Whoever finds it keeps it.",
  "tags": [
    ["d", "linocut-aftermath-1748619568671"],
    ["name", "Aftermath (Linocut #1)"],
    ["g", "u4xsu6ry"],
    ["D", "2"],
    ["T", "2"],
    ["S", "small"],
    ["t", "traditional"],
    ["n", "first-to-find"],
    ["n", "art"],
    ["hint", "Behind glass, but not in a frame"],
    ["image", "https://blossom.primal.net/example-linocut.jpg"],
    ["verification", "6805d4e5c0df48b4f76e2fdcb67a2acb1d97567b01c6fe17a236dc32f34f1c07"]
  ]
}
```

### Found Log 记录

```json
{
  "kind": 7516,
  "content": "Found it! Great hiding spot.",
  "tags": [
    ["a", "37516:0461fcbecc4c3374439932d6b8f11269ccdb7cc973ad7a50ae362db135a474dd:first-treasure-1748619568668"]
  ]
}
```

### DNF Log

```json
{
  "kind": 1111,
  "content": "Searched for 30 minutes but couldn't find it. Maybe it's missing?",
  "tags": [
    ["A", "37516:0461fcbecc4c3374439932d6b8f11269ccdb7cc973ad7a50ae362db135a474dd:first-treasure-1748619568668"],
    ["K", "37516"],
    ["P", "0461fcbecc4c3374439932d6b8f11269ccdb7cc973ad7a50ae362db135a474dd"],
    ["a", "37516:0461fcbecc4c3374439932d6b8f11269ccdb7cc973ad7a50ae362db135a474dd:first-treasure-1748619568668"],
    ["k", "37516"],
    ["p", "0461fcbecc4c3374439932d6b8f11269ccdb7cc973ad7a50ae362db135a474dd"],
    ["t", "dnf"]
  ]
}
```

### Note Log 记录

```json
{
  "kind": 1111,
  "content": "Lots of muggles around during the day. Best to visit in the evening.",
  "tags": [
    ["A", "37516:0461fcbecc4c3374439932d6b8f11269ccdb7cc973ad7a50ae362db135a474dd:first-treasure-1748619568668"],
    ["K", "37516"],
    ["P", "0461fcbecc4c3374439932d6b8f11269ccdb7cc973ad7a50ae362db135a474dd"],
    ["a", "37516:0461fcbecc4c3374439932d6b8f11269ccdb7cc973ad7a50ae362db135a474dd:first-treasure-1748619568668"],
    ["k", "37516"],
    ["p", "0461fcbecc4c3374439932d6b8f11269ccdb7cc973ad7a50ae362db135a474dd"],
    ["t", "note"]
  ]
}
```

### 验证事件

```json
{
  "kind": 7517,
  "content": "Geocache verification for npub1qc0lc5lxnhxnfxlw2lxkv4x4vp6xsf4d5qwvlhfx6qmz6x4nfhqd8h2z3",
  "tags": [
    ["a", "0461fcbecc4c3374439932d6b8f11269ccdb7cc973ad7a50ae362db135a474dd:naddr1qqxnzd3e8q6n2dfk8qcnjve48qmnsw3jsqgswaehxw309aex2mrp0yhx6tpdsek6w309aex2mrp0yh56tnwdus8vatjvs6kzdrz956k7tjzw6qzypzgd2dmgxhxf34hnlw2y03nckr8f4g6mw9flxqq65v94zkp77rqfgrf8"]
  ],
  "pubkey": "6805d4e5c0df48b4f76e2fdcb67a2acb1d97567b01c6fe17a236dc32f34f1c07",
  "created_at": 1672531200,
  "sig": "a1b2c3d4e5f6789012345678901234567890abcdef1234567890abcdef123456789012345678901234567890abcdef1234567890abcdef1234567890abcdef12"
}
```

### 已验证 Found Log

```json
{
  "kind": 7516,
  "content": "Found it! Great hiding spot.",
  "tags": [
    ["a", "37516:0461fcbecc4c3374439932d6b8f11269ccdb7cc973ad7a50ae362db135a474dd:verified-treasure-1748619568669"],
    ["verification", "{\"kind\":7517,\"content\":\"Geocache verification for npub1qc0lc5lxnhxnfxlw2lxkv4x4vp6xsf4d5qwvlhfx6qmz6x4nfhqd8h2z3\",\"tags\":[[\"a\",\"0461fcbecc4c3374439932d6b8f11269ccdb7cc973ad7a50ae362db135a474dd:naddr1qqxnzd3e8q6n2dfk8qcnjve48qmnsw3jsqgswaehxw309aex2mrp0yhx6tpdsek6w309aex2mrp0yh56tnwdus8vatjvs6kzdrz956k7tjzw6qzypzgd2dmgxhxf34hnlw2y03nckr8f4g6mw9flxqq65v94zkp77rqfgrf8\"]],\"pubkey\":\"6805d4e5c0df48b4f76e2fdcb67a2acb1d97567b01c6fe17a236dc32f34f1c07\",\"created_at\":1672531200,\"sig\":\"a1b2c3d4e5f6789012345678901234567890abcdef1234567890abcdef123456789012345678901234567890abcdef1234567890abcdef1234567890abcdef12\"}"]
  ]
}
```

### Geocache 策展列表

```json
{
  "kind": 37517,
  "content": "Explore the festival grounds and find all hidden treasures before the jousting tournament!",
  "tags": [
    ["d", "ren-fest-hunt-1748619568670"],
    ["title", "Texas Ren Fest Treasure Hunt"],
    ["description", "Find all the hidden treasures at the festival!"],
    ["image", "https://blossom.primal.net/banner-example.jpg"],
    ["g", "9vk"],
    ["g", "9vk5"],
    ["g", "9vk5b"],
    ["g", "9vk5b7"],
    ["theme", "adventure"],
    ["map", "adventure"],
    ["a", "37516:0461fcbecc4c3374439932d6b8f11269ccdb7cc973ad7a50ae362db135a474dd:first-treasure-1748619568668"],
    ["a", "37516:0461fcbecc4c3374439932d6b8f11269ccdb7cc973ad7a50ae362db135a474dd:verified-treasure-1748619568669"]
  ]
}
```
