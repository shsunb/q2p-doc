# 数据传输与分片机制

---

在 `Q2P` 系统中，数据传输是通过 UDP 协议实现的，由于 UDP 数据包的大小限制，`Q2P` 必须支持将大数据分片成多个小包进行传输。同时，系统还需要处理数据传输的超时、重试以及数据包的完整性验证。本文将详细介绍数据的分片与传输机制。

## 1. 数据分片与重组

### 分片机制

由于 UDP 协议对每个数据包的大小有最大限制（通常为 65507 字节，但考虑到头部信息等因素，实际可用的数据大小较小），因此当数据超过这个限制时，`Q2P` 会将数据拆分为多个小包（每个小包最大为 484 字节）。每个数据包都会带上一个序号（SYN），以便接收方能够按正确的顺序将其重新组装。

#### 分片过程

1. **数据分片**：当需要传输的数据超过最大包大小时，系统会将其拆分成多个片段。每个片段携带一个序号，确保接收方能够正确拼接。
2. **每片数据大小**：每个数据包的最大有效载荷为 484 字节，除去 UDP 和应用层的头部信息。超出的部分会被拆分成多个小包。
3. **数据包格式**：每个数据包除了包含原始数据外，还会包含一个标识字段（如 `NetworkID` 和 `Event`），以及其他控制信息（如包序号）。

#### 数据包示例：

- 数据包头部（固定部分）：包括 `NetworkID`、事件类型等。
- 数据包体（分片内容）：实际的数据内容，最多为 484 字节。
- 序号（SYN）：用于标识数据包的顺序，确保接收端能够正确组装数据。

### 重组机制

在接收端，`Q2P` 会根据数据包中的序号（SYN）将收到的片段重新组装成完整的数据。

#### 重组过程

1. **接收数据包**：接收端收到多个数据包后，会根据每个数据包中的序号将它们重新排序。
2. **数据验证**：接收方会验证每个数据包是否完整，是否按序号到达。如果有数据包丢失，则会通知数据发送方。
3. **数据合并**：所有片段被成功接收后，系统会将它们拼接成原始数据，并交给应用层处理。

## 2. 数据传输与重试机制

`Q2P` 采用了超时控制和丢包通知机制，确保数据能够可靠传输，即使在网络不稳定的情况下。

### 超时控制

数据传输过程中，每个数据包都会设置超时时间。如果在规定的时间内没有收到目标数据包，系统会触发超时机制，进行相应的重试。

1. **发送数据包时**，会设置一个超时时间（通常为 10 秒），如果在此时间内没有收到对方的确认，发送方会认为数据包丢失。
2. **超时后重试**，发送方会根据丢失的包重新发送数据，直到所有数据包被成功接收。

### 丢包处理

如果经过数据包数量检查，发现数据包传输失败，会触发 `TRANSPORT_FAILED` 事件，通知发送方重新发送失败的数据包。

1. **失败处理**：通过 `TRANSPORT_FAILED` 事件，系统会告知发送方哪些数据包未能成功传输，发送方可以根据该信息进行重新发送。
2. **超时通知**：如果传输过程中的某个数据包超时，系统会通知接收方重新请求该数据包。

## 3. 数据传输的核心逻辑

### 数据传输过程

1. **数据拆分**：当数据超过最大包大小时，数据会被拆分成多个包。每个包都包含一个序号（SYN）以保证数据的顺序。
2. **数据发送**：每个数据包会通过 UDP 协议发送到目标节点。
3. **数据接收**：接收端收到数据包后，按照序号对数据进行排序，并检查是否有丢包。
4. **丢包通知**：丢失数据包时，会通知发送方进行丢包处理。
5. **数据合并**：接收到所有数据包后，接收端会将这些数据包合并为原始数据并交给应用层。

## 4. 数据传输与分片的核心代码

在 `Q2P` 中，数据传输的实现主要集中在 `peer.go` 文件中的相关函数。以下是数据传输和分片的关键代码片段：

### 关键函数：`Transport()`

```go
func (peer *Peer_T) Transport(rAddr *net.UDPAddr, data []byte) error {
    // 数据分片处理
    length := len(data)
    packetNum := length / 484
    if length % 484 != 0 {
        packetNum++
    }

    for i := 0; i < packetNum; i++ {
        start := i * 484
        end := start + 484
        if length < end {
            end = length
        }
        
        // 分片数据包
        body := data[start:end]
        // 发送数据包
        _, err := peer.Conn.WriteToUDP(body, rAddr)
        if err != nil {
            return err
        }
    }

    return nil
}
```

### 关键函数：`transportFailed()`

```go
func (peer *Peer_T) transportFailed(rAddr *net.UDPAddr, hash []byte, syns []uint32) {
    // 发送 TRANSPORT_FAILED 消息
    // ...
}
```

## 小结

`Q2P` 的数据传输机制通过分片、超时控制和重试机制确保了数据的可靠性和完整性。在实际应用中，系统能够自动处理丢包、超时等网络问题，保证大数据量的高效传输。数据传输和分片的设计使得 `Q2P` 能够在面对不稳定的网络环境时仍然能够稳定运行。

---

## 跳转

继续阅读了解更多关于报文结构与协议的细节：[**报文结构与协议**](06_message_structure_and_protocol.md)。