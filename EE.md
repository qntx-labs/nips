> __Warning__  `unrecommended`：已被 [Marmot Protocol](https://github.com/marmot-protocol/marmot) 取代

NIP-EE
======

使用 Messaging Layer Security（MLS）协议的 E2EE 消息
---------------------------------------------------

`final` `unrecommended` `optional`

本 NIP 标准化了如何在 Nostr 中使用 [MLS Protocol](https://www.rfc-editor.org/rfc/rfc9420.html)，以实现高效的 E2EE（端到端加密）直接消息和群组消息。

## 背景

最初，Nostr 中的一对一直接消息（DM）通过 [NIP-04](04.md) 中定义的方案实现。该 NIP 不再推荐使用，因为它虽然加密了消息内容（提供了不错的机密性），但会泄露大量有关会话参与方的元数据（完全缺乏隐私）。

随着 [NIP-44](44.md) 的加入，我们有了一个更新的加密方案，它改进了机密性保证，但并未进一步定义如何使用该加密方案执行直接消息。因此，它对隐私几乎没有带来差异。

最近，[NIP-17](17.md) 将 [NIP-44](44.md) 加密与 [NIP-59](59.md) gift-wrapping 结合起来，把加密的直接消息隐藏在另一组 event 内，从而确保无法看出谁在和谁交谈，也无法看出消息何时在用户之间传递。这在很大程度上解决了元数据泄露问题；虽然仍然可以看到某个用户正在接收 gift-wrapped event，但无法知道它们来自谁，也无法知道 gift-wrap 外层 event 内包含什么类型的 event。这提供了一定程度的可否认性/可抵赖性，但没有解决 forward secrecy 或 post compromise security。也就是说，如果用户的私钥（或两个用户之间用于加密消息的计算得出的共享会话密钥）被攻破，攻击者将完全访问这些用户之间过去和未来发送的所有 DM。

此外，[NIP-04](04.md) 和 [NIP-17](17.md) 都没有尝试解决群组消息问题。

### 为什么这很重要？

没有适当的 E2EE，Nostr 就不能作为安全消息客户端的协议使用。虽然 Signal 等客户端在 E2EE 方面做得很好，但它们仍然依赖中心化服务器，因此可能被强大的行为者（即国家级行为者）关闭。Nostr 的目标不仅是防止中心化实体审查你和你的通信，还要防止国家级行为者从一开始就阻止这类服务存在。通过用去中心化 relay 取代中心化服务器，我们使中心化行为者几乎不可能完全阻止个体用户之间的通信。

### 本 NIP 的目标

1. 私密且机密的 DM 和群组消息
   1. **私密**意味着观察者无法判断 Alice 和 Bob 正在互相交谈，或 Alice 是某个特定群组的一员。这必然要求保护元数据。
   2. **机密**意味着会话内容只能由预期接收者查看。
2. Forward secrecy 和 Post-compromise security
   1. **Forward secrecy** 意味着即使密钥材料泄露，过去的加密内容仍然保持加密。
   2. **Post compromise security** 意味着密钥材料泄露不会让攻击者无限期地继续读取未来消息。
3. 能够高效扩展到大型群组。
4. 允许在单个会话/群组中使用多个设备/客户端。

### 为什么选择 MLS？

该方案改编 Message Layer Security（MLS）协议以用于 Nostr。你可以把 MLS 看作 Signal Protocol 的演进。不过，它显著提高了大型群组消息中加密操作的可扩展性（线性 -> 对数），为适应联邦式环境而构建，并且还允许随着时间推移平滑更新 ciphersuite 和版本。此外，它非常灵活，对发送的消息内容保持无关。

解释 MLS 协议本身超出了本 NIP 的范围，但你可以在它的[架构概览](https://www.ietf.org/archive/id/draft-ietf-mls-architecture-13.html)或 [RFC](https://www.rfc-editor.org/rfc/rfc9420) 中了解更多。MLS 正在 IETF 下成为互联网标准，因此协议本身已经经过了非常充分的审查和研究。这也意味着随着 MLS 被更多采用，未来存在跨网络消息互操作性的潜力。

## 核心 MLS 概念

来自 [MLS Architectural Overview](https://www.ietf.org/archive/id/draft-ietf-mls-architecture-13.html)：

> MLS 为客户端提供了一种组建群组的方式，客户端可以在这些群组内安全通信。例如，一组用户可以使用手机或笔记本电脑上的客户端加入一个群组并相互通信。群组可以小到两个客户端（例如简单的点对点消息），也可以大到数十万客户端。属于某个群组的客户端就是该群组的成员。随着群组成员关系、群组属性或成员属性发生变化，群组会从一个 epoch 推进到另一个 epoch，其加密状态也随之演进。
>
> 群组被表示为一棵树，成员则表示为树的叶子。该树用于高效地向成员子集加密。每个成员都有一种称为 LeafNode 对象的状态，其中保存客户端的身份、凭证和能力。

MLS 协议的工作是管理并演进群组的加密状态。这包括管理群组成员关系、群组的加密状态（ratchet tree、密钥，以及消息的加密/解密/认证），并管理群组随时间的演进。

### 群组

群组由其第一个成员创建，该成员随后邀请一个或多个其他成员。群组随时间以称为 `Epochs` 的区块演进。新的 epoch 通过一个或多个 `Proposal` 消息提出，然后通过 `Commit` 消息提交。

### 客户端

用户加入群组时所使用的设备/客户端对（例如 iOS 上的 Primal 或 Web 上的 Coracle）在树中表示为一个 `LeafNode`。在这方面，`Client` 和 `Member` 这两个术语可以互换。无法在多个 `Clients` 之间共享群组状态。如果用户从 2 台不同设备加入一个群组，它们的状态是分离的，并会作为该群组的 2 个独立成员被追踪。

### 消息

群组内发送多种不同类型的消息。其中一些是控制消息，用于随时间更新群组状态。这些包括 `Welcome`、`Proposal` 和 `Commit` 消息。另一些是群组成员之间发送的实际消息。这些包括 `Application` 消息。

MLS 中的消息是“framed”的。也就是说，它们被包裹在一种数据结构中，该结构包含发送者、epoch、epoch 内的消息索引以及消息内容等信息。这种 framing 使得即使消息乱序到达，也可以正确认证和解密消息。

MLS 对所发送消息的“内容”保持无关。这是 MLS 的一个关键特性，使 MLS 可用于广泛的应用。

MLS 也对发送消息所使用的传输协议保持无关。显然，对我们来说，我们将使用 websockets、Nostr event 和 relay。

## 本 NIP 的重点

本 NIP 关注如何使用 Nostr 执行 MLS 协议所需的 Authentication Service 和 Delivery Service 功能。大多数客户端会选择使用 MLS 实现来处理密钥、ratcheting、群组状态管理以及 MLS 协议本身的其他方面。[OpenMLS](https://github.com/openmls/openmls) 是目前最活跃开发的 MLS 实现库。

本 NIP 规定以下内容：

1. Nostr 客户端应如何[创建 MLS 群组](#creating-groups)的标准化方式。
2. Nostr 客户端应用来表示群组中 Nostr 用户的 MLS [`Credential`](#mls-credentials) 的必需格式。
3. 发布到 relay 的 [KeyPackage Event](#keypackage-event-and-signing-keys) 结构，它允许以异步方式将 Nostr 用户加入群组。
4. 发布到 relay 的 [Group Event](#group-events) 结构，它表示群组状态的演进以及群组中发送的消息内容。

## 安全考虑

这是对本 NIP 在各种场景中的安全权衡和考虑因素的简明概览。本 NIP 力求完整保持 MLS 的安全保证。

### Forward Secrecy 和 Post-compromise Security

- 按照 MLS 规范，密钥一旦用于加密或解密消息就会被删除。这通常由 MLS 实现库本身处理，但客户端需要注意确保不要在绝对必要时间之外存储 secrets（尤其是 [exporter secret](#group-events)）。
- 本 NIP 保持 MLS 的 forward secrecy 和 post-compromise security 保证。你可以在 MLS Architectural Overview 的 [Forward Secrecy and Post-compromise Security](https://www.ietf.org/archive/id/draft-ietf-mls-architecture-15.html#name-forward-and-post-compromise) 小节中阅读更多内容。

### 各类密钥泄露

- 本 NIP 在 MLS 消息协议的任何方面都不依赖用户的 Nostr 身份密钥。用户 Nostr 身份密钥被攻破不会让攻击者访问任何基于 MLS 的群组中的过去或未来消息。
- 有关 MLS 密钥泄露的完整讨论，请参见 [MLS Architectural Overview](https://www.ietf.org/archive/id/draft-ietf-mls-architecture-15.html#name-endpoint-compromise) 的 Endpoint Compromise 小节。

### 元数据

- 发布到 relay 的唯一群组专用元数据是 Nostr group ID 值。该值用于在 Group Message Event（`kind: 445`）的 `h` 标签中标识群组。这些 event 以 ephemeral 方式发布，并且该 Nostr group ID 值可由群组管理员在群组生命周期内更新。这是一种权衡：它确保群组参与者和群组大小被混淆，同时仍然可以高效地将群组消息扇出给所有参与者。该 event 的 content 字段是一个以两种不同方式（使用 NIP-44 和 MLS）使用 MLS 群组状态/密钥加密的值。只有拥有最新群组状态的群组成员才能解密并读取这些消息。
- 用户的 key package event 可以被使用一次或多次来加入群组。这里存在一种固有权衡：复用 key package（初始签名密钥）带来一定程度的风险，但只要用户在加入群组后立即轮换其签名密钥，该风险就会被缓解。这一步也会提高整个群组的 forward secrecy。

### 设备被攻破

实现本 NIP 的客户端应采取一切预防措施，确保数据以安全方式存储在设备上，并在设备被攻破时防止不必要的访问（例如静态加密、生物认证等）。也就是说，完整的设备攻破应被视为灾难性事件；被攻破设备曾参与的任何群组都应视为已被攻破，直到群组能够移除该成员并更新群组状态。一些建议：

- 客户端应支持并鼓励自毁消息（确保完整会话历史不会永远保留在设备上）。
- 客户端应定期建议群组管理员移除不活跃用户。
- 客户端应定期建议（或自动）轮换用户在各个群组中的签名密钥。
- 客户端应使用一个不属于群组状态或用户 Nostr 身份密钥的 secret 值来加密设备上的群组状态和密钥。
- 客户端应在可能时使用 secure enclave 存储。

有关 MLS 安全考虑的完整讨论，请参见 [MLS RFC](https://www.rfc-editor.org/rfc/rfc9420.html#name-security-considerations) 的 Security Considerations 小节。

<a id="creating-groups"></a>

## 创建群组

MLS 群组使用一个随机的 32 字节 ID 值创建，该值实际上是永久的。该 ID 应被视为群组私有信息，并且 MUST NOT 以任何形式发布到 relay。

客户端还必须确保它们在创建群组时使用的 ciphersuite、capabilities 和 extensions 与它们希望邀请进群组的用户所公开声明的内容兼容。它们可以通过用户发布的 KeyPackage Event 检查这些信息。

创建新群组时，MUST 使用以下 MLS extension。

- [`required_capabilities`](https://docs.rs/openmls/latest/openmls/extensions/struct.RequiredCapabilitiesExtension.html)
- [`ratchet_tree`](https://docs.rs/openmls/latest/openmls/extensions/struct.RatchetTreeExtension.html)
- [`nostr_group_data`](https://github.com/rust-nostr/nostr/blob/master/mls/nostr-mls/src/extension.rs)

并且强烈建议使用以下 MLS extension（更多内容见[这里](#keypackage-event-and-signing-keys)）：
- [`last_resort`](https://docs.rs/openmls/latest/openmls/extensions/struct.LastResortExtension.html)

对 MLS 群组的更改通过先创建一个或多个 `Proposal` event，然后在 `Commit` event 中提交一组 proposal 来生效。这些是 MLS event，而不是 Nostr event。不过，为了使群组状态正确演进，Commit event（表示一组特定 proposal，例如向群组添加新用户）必须发布到 relay，以便其他群组成员看到。更多信息见 [Group Messages](#group-events)。

<a id="mls-credentials"></a>

## MLS Credential

MLS 中的 `Credential` 是对用户身份的断言，并与签名密钥绑定。为 MLS 构造 `Credentials` 时，客户端 MUST 使用 `BasicCredential` 类型，并将 `identity` 值设置为用户 Nostr 身份密钥的 32 字节十六进制编码公钥。客户端 MUST NOT 允许用户更改 identity 字段，并且 MUST 校验所有 `Proposal` 消息没有试图更改群组中任何 credential 的 identity 字段。

`Credential` 还具有关联的签名密钥。用户的初始签名密钥包含在 KeyPackage event 中。该签名密钥 MUST 不同于用户的 Nostr 身份密钥。该签名密钥 SHOULD 随时间轮换，以提供改进的 post-compromise security。

## Nostr 群组数据扩展

如上所述，`nostr_group_data` extension 是一个必需的 MLS extension，用于以加密安全且可证明的方式将 Nostr 专用数据与 MLS 群组关联起来。创建新群组时，MUST 将该 extension 作为 required capability 包含进去。

该 extension 存储有关群组的以下数据：

- `nostr_group_id`：群组的 32 字节 ID。这是一个不同于 MLS 所用 group ID 的值，并且 CAN 随时间更改。该值是在发送 group message event 时用于 `h` 标签的 group ID 值。
- `name`：群组名称。
- `description`：群组的简短描述。
- `admin_pubkeys`：群组管理员的十六进制编码公钥数组。MLS 协议本身没有群组管理员的概念。客户端在对群组数据（该 extension 中的任何内容）进行任何更改之前，或在更改群组成员关系（添加/移除成员）之前，或更新群组本身任何其他方面（例如 ciphersuite 等）之前，MUST 检查 `admin_pubkeys` 列表。注意，群组的所有成员都可以针对自己 credential 的更改（例如更新自己的签名密钥）发送 `Proposal` 和 `Commits` 消息。
- `relays`：群组用于发布和接收消息的 Nostr relay URL 数组。

所有这些值都可以随时间使用 MLS `Proposal` 和 `Commit` event（由群组管理员）更新。

<a id="keypackage-event-and-signing-keys"></a>

## KeyPackage Event 和签名密钥

每个希望通过基于 MLS 的消息被联系到的用户 MUST 首先发布至少一个 KeyPackage event。KeyPackage Event 用于认证用户，并创建以异步方式将成员加入群组所需的 `Credential`。用户可以发布多个具有不同参数的 KeyPackage Event（例如支持不同 ciphersuite 或 MLS extension）。KeyPackage 包含一个签名密钥，用于在群组内签署 MLS 消息。该签名密钥 MUST NOT 与用户的 Nostr 身份密钥相同。

KeyPackage 复用 SHOULD 被最小化。不过，在正常 MLS 使用中，KeyPackage 会在加入群组时被消费。为了减少多个群组邀请使用同一个 KeyPackage 时的竞争条件，Nostr 客户端 SHOULD 使用 "Last resort" KeyPackage。这要求在 KeyPackage 的 capabilities 上包含 `last_resort` extension（与 Group 相同）。

重要的是，客户端应在用户通过 last resort key package 加入群组后立即轮换其签名密钥，以改进 post-compromise security。签名密钥（KeyPackage Event 中包含的公钥）用于在群组内签名。因此，实现本 NIP 的客户端 MUST 确保它们保留对自己所属每个群组中签名密钥私钥材料的访问权限。

在大多数情况下，假定实现本 NIP 的客户端会管理 KeyPackage Event 的创建和轮换。

### KeyPackage Event 示例

```json
  {
    "id": <id>,
    "kind": 443,
    "created_at": <unix timestamp in seconds>,
    "pubkey": <main identity pubkey>,
    "content": "",
    "tags": [
        ["mls_protocol_version", "1.0"],
        ["ciphersuite", <MLS CipherSuite ID value e.g. "0x0001">],
        ["extensions", <An array of MLS Extension ID values e.g. "0x0001, 0x0002">],
        ["client", <client name>, <handler event id>, <optional relay url>],
        ["relays", <array of relay urls>],
        ["-"]
    ],
    "sig": <signed with main identity key>
}
```

- `content` 是 MLS 中序列化 `KeyPackageBundle` 的十六进制编码。
- `mls_protocol_version` 标签是必需的，并且 MUST 是所使用 MLS 协议版本的版本号。目前这是 `1.0`。
- `ciphersuite` 标签是该 KeyPackage Event 支持的 MLS ciphersuite 值。[阅读更多关于 MLS ciphersuite 的内容](https://www.rfc-editor.org/rfc/rfc9420.html#name-mls-cipher-suites)。
- `extensions` 标签是该 KeyPackage Event 支持的 MLS extension ID 数组。[阅读更多关于 MLS extension 的内容](https://www.rfc-editor.org/rfc/rfc9420.html#name-extensions)。
- （可选）`client` 标签帮助其他客户端在收到群组邀请但无法访问签名密钥时管理用户体验。
- `relays` 标签标识客户端将尝试发布该 KeyPackage event 的每个 relay。这允许稍后删除 KeyPackage Event。
- （可选）`-` 标签可用于确保 KeyPackage Event 只由其已认证作者发布。更多内容见 [NIP-70](70.md)

### 删除 KeyPackage Event

每当客户端成功处理给定 KeyPackage Event 的群组请求 event 时，客户端 SHOULD 在所有列出的 relay 上删除该 KeyPackage Event。客户端 MAY 同时创建一个新的 KeyPackage Event。

如果客户端无法处理 Welcome 消息（例如因为签名密钥是在另一个客户端上生成的），客户端 MUST NOT 删除该 KeyPackage Event，并且 SHOULD 向用户显示人类可理解的错误。

### 轮换签名密钥

客户端 MUST 定期轮换用户在其所属每个群组中的签名密钥。签名密钥轮换得越频繁，post-compromise security 越强。这种轮换通过 `Proposal` 和 `Commit` event 完成，并通过 Group Event 广播给群组。[阅读更多关于 MLS 固有 forward secrecy 和 post-compromise security 的内容](https://www.rfc-editor.org/rfc/rfc9420.html#name-forward-secrecy-and-post-co)。

### KeyPackage Relay 列表事件

`kind: 10051` event 表示用户将把自己的 KeyPackage Event 发布到哪些 relay。该 event MUST 包含带有 relay URI 的 relay 标签列表。这些 relay SHOULD 可由用户希望能够联系自己的任何人读取。

```json
{
  "kind": 10051,
  "tags": [
    ["relay", "wss://inbox.nostr.wine"],
    ["relay", "wss://myrelay.nostr1.com"],
  ],
  "content": "",
  //...other fields
}
```

### Welcome Event（欢迎事件）

当通过 MLS `Commit` 消息将新用户加入群组时，向群组发送 `Commit` 消息的成员负责向被添加进群组的用户发送 Welcome Event。该 Welcome Event 作为 [NIP-59](59.md) gift-wrapped event 发送给用户。Welcome Event 向新成员提供加入群组并开始发送消息所需的上下文。

创建 Welcome Event 的客户端 SHOULD 等到收到 relay 确认其包含 `Commit` 的 Group Event 已被接收后，再发布 Welcome Event。

```json
{
   "id": <id>,
   "kind": 444,
   "created_at": <unix timestamp in seconds>,
   "pubkey": <nostr identity pubkey of sender>,
   "content": <serialized Welcome object>,
   "tags": [
      ["e", <ID of the KeyPackage Event used to add the user to the group>],
      ["relays", <array of relay urls>],
   ],
   "sig": <NOT SIGNED>
}
```

- `content` 字段是必需的，并且是包含 MLS `Welcome` 对象的序列化 MLSMessage 对象。
- `e` 标签是必需的，并且是用于将用户添加到群组的 KeyPackage Event 的 ID。
- `relays` 标签是必需的，并且是客户端应查询 Group Event 的 relay 列表。

Welcome Event 随后会按 [NIP-59](59.md) 中详述的方式被 sealed 和 gift-wrapped 后发布。与所有被 sealed 和 gift-wrapped 的 event 一样，`kind: 444` event MUST 永远不被签名。这确保如果它们曾经泄露，也无法发布到 relay。

#### 大型群组

对于超过约 150 名参与者的群组，welcome 消息会变得大于 Nostr 允许的最大 event 大小。MLS 协议目前正在开展工作，以支持不需要将完整 Ratchet Tree 状态发送给新成员的“light”客户端 welcome。本小节将更新关于如何处理大型群组的建议。

<a id="group-events"></a>

## Group Event（群组事件）

Group Event 是群组内发送的所有消息。这包括所有随时间更新共享群组状态的“control”event（`Proposal`、`Commit`），以及群组成员之间发送的消息（`Application` messages）。

Group Event 使用 ephemeral Nostr keypair 发布，以混淆群组参与者数量和身份。客户端 MUST 为它们发布的每个 Group Event 使用新的 Nostr keypair。

```json
{
   "id": <id>,
   "kind": 445,
   "created_at": <unix timestamp in seconds>,
   "pubkey": <ephemeral sender pubkey>,
   "content": <NIP-44 encrypted serialized MLSMessage object>,
   "tags": [
      ["h", <group id>]
   ],
   "sig": <signed with ephemeral sender key>
}
```
- `content` 字段是一个 [tls-style](https://www.rfc-editor.org/rfc/rfc9420.html#name-the-message-mls-media-type) 序列化的 [`MLSMessage`](https://www.rfc-editor.org/rfc/rfc9420.html#section-6-4) 对象，随后按照 [NIP-44](44.md) 加密。不过，与使用发送者和接收者密钥派生 `conversation_key` 不同，NIP-44 加密使用从 MLS [`exporter_secret`](https://www.rfc-editor.org/rfc/rfc9420.html#section-8.5) 生成的 Nostr keypair 来计算 `conversation_key` 值。本质上，你使用十六进制编码的 `exporter_secret` 值作为私钥（用作发送者密钥），计算该私钥对应的公钥（用作接收者密钥），然后继续使用标准 NIP-44 方案来加密和解密消息。
- `exporter_secret` 值应以 32 字节长度生成，并标记为 `nostr`。该 `exporter_secret` 值会在群组每个新 epoch 上轮换。客户端每次处理有效的 `Commit` 消息时都应生成一个新的 32 字节值。
- `pubkey` 是 ephemeral sender 的十六进制编码公钥。
- `h` 标签是 nostr group ID 值（来自 Nostr Group Data Extension）。

### Application Message（应用消息）

Application message 是成员在群组内发送的消息。它们包含在 `MLSMessage` 对象中。这些消息的格式应为适当 kind 的未签名 Nostr event。对于普通 DM 或群组消息，客户端 SHOULD 使用 `kind: 9` 聊天消息 event。如果用户对消息做出反应，则会是 `kind: 7` event，依此类推。

这意味着一旦 application message 被解密并反序列化，客户端就可以存储这些 event，并像处理任何其他 Nostr event 一样对待它们，从而有效地创建群组活动的私有 Nostr feed，并利用 Nostr 的所有特性。

这些内部未签名 Nostr event MUST 在 `pubkey` 字段中使用成员的 Nostr 身份密钥，并且客户端 MUST 检查发送消息成员的身份是否与内部 Nostr event 的 pubkey 匹配。

这些 Nostr event MUST 保持**未签名**，以确保如果它们泄露到 relay，也不会被公开发布。这些 Nostr event MUST NOT 包含任何 "h" 标签或其他会标识其所属群组的标签。

### `Commit` 消息竞争条件

MLS 协议能够抵御几乎所有消息乱序到达的情况。不过，`Commit` 消息的顺序对于群组状态从一个 epoch 正确前进到下一个 epoch 很重要。鉴于 Nostr 作为去中心化网络的性质，客户端有可能接收到 2 个或更多都试图同时更新到新 epoch 的 `Commit` 消息。

发送 commit 消息的客户端 MUST 等到从至少一个 relay 收到其包含 `Commit` 的 Group Message Event 已被接收的确认后，才将该 commit 应用到自己的群组状态。

如果客户端接收到 2 个或更多试图更改同一 epoch 的 `Commit` 消息，它 MUST 只应用所收到的其中一个 `Commit` 消息，选择规则如下：

1. 使用 kind `445` event 上的 `created_at` 时间戳。`created_at` 值最低的 `Commit` 是要应用的消息。其他 `Commit` 消息被丢弃。
2. 如果两个或更多 `Commit` 消息的 `created_at` 时间戳相同，则 `id` 字段值最低的 `Commit` 消息是要应用的消息。

客户端 SHOULD 在短时间内保留先前的群组状态，以便从分叉的群组状态中恢复。
