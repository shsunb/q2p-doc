# Q2P

`Q2P` 是一个基于 UDP 协议实现的点对点（P2P）网络通信库，旨在通过简单的网络事件驱动和数据传输协议，支持多个节点之间的连接和数据交换。该库采用 Go 语言编写，提供了一种简洁的方式来实现 P2P 网络中的设备连接、通信和数据传输。

`Q2P` 分为协议层和传输层：协议层、传输层

`Q` 代表 quick，因为其传输层的实现方式与特性与 QUIC 协议很相似

## 特性

- **网络事件驱动**：支持多种网络事件（如连接请求、数据传输等）处理。
- **点对点通信**：基于 UDP 协议进行高效的数据传输和节点间连接。
- **可靠的连接管理**：通过事件机制实现节点间的连接请求、连接确认等过程。
- **数据分片传输**：支持大数据量的分片传输，并在传输失败时进行重试。
- **传输超时控制**：支持传输过程中的超时检测和失败处理。

### 环境要求

- Go 1.22


## 安装
```bash
go get github.com/mosalut/q2p
```


### 克隆代码

```bash
git clone https://github.com/mosalut/q2p.git
cd q2p
```

### 安装依赖

```bash
go mod tidy
```

## 快速上手

### 配置

你可以通过以下命令行参数来配置并启动一个 P2P 节点：

- `-ip`：指定本地节点的 IP 地址，默认为 `0.0.0.0`。
- `-port`：指定本地节点的端口，默认为 `10000`。
- `-remote_host`：指定远程节点的地址（可选）。

### 运行
假使已知一个种子节点在127.0.0.1:10000启动

```
package main

import (
	"github.com/mosalut/q2p"
	"log"
)

// 自定义回调函数
func callback(data []byte) {
	fmt.Println(string(data))
}

func main() {
	seedAddrs = make(map[string]bool)
	seedAddrs[127.0.0.1:10000] = false // 将已知种子节点放入seedAddrs

        // 创建新节点
	peer := q2p.NewPeer("127.0.0.1", "10001", seedAddrs, []byte{0x0, 0x0}, callback)
	err := peer.Run()
	if err != nil {
		log.Fatal(err)
	}
}
```
通过运行q2p.NewPeer(ip, port, seedAddrs, networkID, callback)函数来运行一个节点）

参数ip、port是本节点要启动q2p网络主机所在的IP和端口组成的UDP地址
参数seedAddrs是一个map,其中每一个key都是一个种子节点的UDP地址
参数networkID是一个网络ID号，2个字节，这里使用0
callback是用于收到TRANSPORT事件时的用户自定义函数，callback函数的参数data，是被TRANSPORT事件发来并处理的


### 数据传输

数据传输通过 `Transport` 方法实现，该方法将数据分片并通过 UDP 进行发送，确保大数据量的可靠传输。如果数据传输失败，用户可以决定是否尝试重新传输。

### 传输超时和失败处理

在 `transmission.go` 中，`transmissionSending` 和 `transmissionReceiving` 函数处理了数据传输的发送和接收超时。如果在规定时间内没有收到完整的数据或发生错误，系统会记录传输失败，并触发相应的重试或失败处理机制。

### 示例代码

以下是一个简单的使用示例：

```go
package main

import (
	"fmt"
	"log"
	"net"
	"github.com/mosalut/q2p"
)

func main() {
	peer := q2p.NewPeer("127.0.0.1", 10000, nil, 0, func(data []byte) {
		fmt.Println("Received data:", string(data))
	})

	err := peer.Run()
	if err != nil {
		log.Fatal(err)
	}

	// 发送数据
	remoteAddr, _ := net.ResolveUDPAddr("udp", "127.0.0.1:10001")
	peer.Transport(remoteAddr, []byte("Hello, P2P Network"))
}
```

## 测试

该项目提供了一些简单的单元测试，涵盖了 P2P 节点的基本运行和数据传输功能。你可以通过以下命令运行测试：

```bash
go test -v -count 1 test.run TestQ2p . // 默认 host是127.0.0.1:10000

go test -v -count 1 test.run TestTransport -remote_host 127.0.0.1:10000 -port 10001
```

### 测试文件

- `q2p_test.go` 文件包含了用于测试节点连接、事件处理和数据传输的示例。

### 事件传输测试

`TestTransport` 测试函数模拟了数据传输的全过程，包括数据的发送和接收，验证了数据分片传输和超时控制机制。

## 贡献

欢迎提交 issues 和 pull requests，任何改进建议和功能需求都会被认真考虑。请确保代码符合 Go 语言的代码风格，并编写必要的单元测试。

## 许可证

此项目采用 **GNU General Public License v3.0** 许可证。

详细的许可证内容请参见 [`LICENSE`](LICENSE) 文件。
