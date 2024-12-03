# 数据传输与分片机制

---

在 `Q2P` 系统中，数据传输是通过 UDP 协议实现的，由于 UDP 数据包的大小限制，`Q2P` 必须支持将大数据分片成多个小包进行传输。同时，系统还需要处理数据传输的超时、重试以及数据包的完整性验证。本文将详细介绍数据的分片与传输机制。

## 数据分片与重组

### 分片机制

由于 UDP 协议对每个数据包的大小有最大限制通常为 65506 字节，而路由器的包大小设置一般小于1500字节，加上还有包头的数据
为了安全性和速度的综合考虑，将Q2P的包大小定义为512字节，去掉头信息，所以每个body的大小是484
当数据超过484时，`Q2P` 会将数据拆分为多个Body放到一个数据包中。每个数据包都会带上本次传输的HASH，数据的总长度（length），和一个序号（SYN），以便接收方处理时不会和其他传输串包，并能够按正确的顺序将其重新组装。


## 数据传输与分片的核心代码

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
    // hash: 失败的 transimission 的HASH
    // syns: 丢失的包的SYN列表，如果长度为0 表示超时
    // 用户自行处理是否重传 还是补包
    // 如要补包，请调用 *Peer_T.TransportAPacket
    // 用户其他操作...
}
```

## 小结

`Q2P` 的数据传输机制通过分片、超时控制和重试机制确保了数据的可靠性和完整性。在实际应用中，系统能够自动处理丢包、超时等网络问题，保证大数据量的高效传输。数据传输和分片的设计使得 `Q2P` 能够在面对不稳定的网络环境时仍然能够稳定运行。

---

## 跳转

继续阅读了解更多关于报文结构与协议的细节：[**报文结构与协议**](06_message_structure_and_protocol.md)。
