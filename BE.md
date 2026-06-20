> __Warning__  `unrecommended`: 只实现过一次，且不清楚是否有效，需要审查

NIP-BE
======

Nostr BLE 通信协议
---------------------------------

`draft` `unrecommended` `optional`

本 NIP 指定 Nostr app 如何使用 BLE 相互通信和同步。BLE 协议遵循 client-server 模式，因此本 NIP 以类似方式模拟 WS 结构，但针对其限制做了一些适配。

## 设备公告
设备使用以下内容公告自己：
- Service UUID：`0000180f-0000-1000-8000-00805f9b34fb`
- Data：ByteArray 格式的 Device UUID

## GATT 服务
设备暴露一个 Nordic UART Service，具有以下 characteristic：

1. 写入 Characteristic
   - UUID：`87654321-0000-1000-8000-00805f9b34fb`
   - Properties：Write

2. 读取 Characteristic
   - UUID：`12345678-0000-1000-8000-00805f9b34fb`
   - Properties：Notify, Read

## 角色分配

当一个设备最初发现另一个设备正在公告该服务时，它会读取该服务的数据以获取 device UUID，并与自己公告的 device UUID 比较。在此通信中，ID 最高的设备将扮演 GATT Server（Relay）角色，另一个设备被视为 GATT Client（Client），并继续建立连接。

对于用途要求单一角色的设备，其 device UUID 始终为：

- GATT Server：`FFFFFFFF-FFFF-FFFF-FFFF-FFFFFFFFFFFF`
- GATT Client：`00000000-0000-0000-0000-000000000000`

## 消息

所有消息都将遵循 [NIP-01](/01.md) 消息结构。对于给定消息，会对消息应用压缩流（DEFLATE）以生成字节数组。根据 BLE 版本，字节数组可能对单条消息来说过大（BLE 4.2 为 20-23 字节，BLE > 4.2 为 256 字节）。在这种情况下，此字节数组会被拆分为任意数量的 batch，结构如下：

```
[batch index (first 2 bytes)][batch n][is last batch (last byte)]
```
接收所有 batch 后，另一台设备即可将它们合并并解压。为确保可靠性，一次只会读取/写入 1 条消息。MTU 可以预先协商。消息最大大小为 64KB；更大的消息将被拒绝。

## 示例

此示例实现一个函数，用于将字节数组拆分并压缩为 chunk；以及另一个函数，用于按顺序合并并解压它们，以获得初始结果：

```kotlin
fun splitInChunks(message: ByteArray): Array<ByteArray> {
   val chunkSize = 500 // define the chunk size
   var byteArray = compressByteArray(message)
   val numChunks = (byteArray.size + chunkSize - 1) / chunkSize // calculate the number of chunks
   var chunkIndex = 0
   val chunks = Array(numChunks) { ByteArray(0) }

   for (i in 0 until numChunks) {
         val start = i * chunkSize
         val end = minOf((i + 1) * chunkSize, byteArray.size)
         val chunk = byteArray.copyOfRange(start, end)

         // add chunk index to the first 2 bytes and last chunk flag to the last byte
         val chunkWithIndex = ByteArray(chunk.size + 2)
         chunkWithIndex[0] = chunkIndex.toByte() // chunk index
         chunk.copyInto(chunkWithIndex, 1)
         chunkWithIndex[chunkWithIndex.size - 1] = numChunks.toByte()

         // store the chunk in the array
         chunks[i] = chunkWithIndex

         chunkIndex++
   }

   return chunks
}

fun joinChunks(chunks: Array<ByteArray>): ByteArray {
   val sortedChunks = chunks.sortedBy { it[0] }
   var reassembledByteArray = ByteArray(0)
   for (chunk in sortedChunks) {
         val chunkData = chunk.copyOfRange(1, chunk.size - 1)
         reassembledByteArray = reassembledByteArray.copyOf(reassembledByteArray.size + chunkData.size)
         chunkData.copyInto(reassembledByteArray, reassembledByteArray.size - chunkData.size)
   }

   return decompressByteArray(reassembledByteArray)
}

```

## 工作流

### Client 到 relay

- client 想发送给 relay 的任何消息都将是 write message。
- client 从 relay 收到的任何消息都将是 read message。

### Relay 到 client

relay 应通过 Read Characteristic 的 Notify action 通知 client 有任何匹配订阅 filter 的新 event。之后，client 可以继续从 relay 读取消息。

### 设备同步

鉴于 BLE 的性质，两个设备之间的直接连接预计可能极其间歇，中间间隔数小时甚至数天。因此，按照 [NIP-77](./77.md) 定义同步流程并针对技术限制进行适配非常关键。

两个设备成功连接并建立 Client-Server 角色后，设备将使用半双工通信间歇地发送和接收消息。

#### 半双工同步

两个设备连接后，Client 立即通过发送第一条消息开始工作流。

1. Client - 写入 ["NEG-OPEN"](/77.md#initial-message-client-to-relay) 消息。
2. Server - 发送 `write-success`。
3. Client - 发送 `read-message`。
4. Server - 以 ["NEG-MSG"](./77.md#subsequent-messages-bidirectional) 消息响应。
5. Client -
   1. 如果 Client 有 Server 缺少的消息，则写入一个 `EVENT`。
   2. 如果 Client 没有 Server 缺少的任何消息，则写入 `EOSE`。在这种情况下，只要 Server 声称还有更多 note 给 Client，后续发往 Server 的消息将为空。
6. Server - 发送 `write-success`。
7. Client - 发送 `read-message`。
8. Server -
   1. 如果 Server 有 Client 缺少的消息，则以一个 `EVENT` 响应。
   2. 如果 Client 没有 Server 缺少的任何消息，则以 `EOSE` 响应。在这种情况下，后续发给 Client 的响应将为空。
9. 如果 Client 检测到设备尚未同步，则跳转到第 5 步。
10. 两个设备都检测到两端不再有缺失 event 后，工作流将在此处暂停。

#### 半双工事件传播

当两个设备已连接并同步时，可能其中一个设备从另一个已连接 peer 收到新消息。设备 MUST 跟踪在连接期间已向 peer 发送过哪些 note。如果新收到的 event 被检测为某个已连接且已同步 peer 缺失：

1. 如果 peer 是 Server：
   1. Client - 写入该 `EVENT`。
   2. Server - 发送 `write-success`。
2. 如果 peer 是 Client：
   1. Server - 向 Client 发送一个空通知。
   2. Client - 发送 `read-message`。
   3. Server - 以该 `EVENT` 响应。
