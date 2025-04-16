# wsgrpc - Golang GRPC over WebSocket

[English Version](./README_en.md)

wsgrpc 的主要目的是允许 Go gRPC 客户端以websocket与服务端通信。

原因：
1. 许多CDN和反向代理并不支持 HTTP/2（gRPC 使用 HTTP/2）回源或支持不佳（例如Nezha监控前段时间的 Cloudflare 大范围掉线）
2. 浏览器应用程序无法使用 grpc-go，因为 go-grpc 原生运行所需的底层网络不可用。（例如，dial tcp: Protocol not available 错误……）

此库支持 websocket 用作 grpc.Conn 来处理grpc。

让你的 grpc 也可以轻松支持cdn/反向代理

## 使用示例

```go
// 假设我们有这样一个 main 文件使用上述库：
package main

import (
	"context"
	"log"
	"net/http"

	"github.com/gin-gonic/gin"
	"google.golang.org/grpc"

	"github.com/VaalaCat/wsgrpc"
	"github.com/gorilla/websocket"
)
```

### 服务端实例

```go

// 
func main() {
	// 创建 WebSocket Listener，缓冲队列大小为 100，地址和网络标识可自定义
	listener := wsgrpc.NewWSListener("ws-listener", "ws", 100)

	// 在单独的 goroutine 中启动 gRPC Server
	go func() {
		grpcServer := grpc.NewServer()
		// 在此注册你的 gRPC 服务…
		if err := grpcServer.Serve(listener); err != nil {
			log.Fatalf("gRPC server error: %v", err)
		}
	}()

	// 使用 Gin 创建 HTTP 服务器，并在某个路径下提供 WebSocket 功能
	router := gin.Default()

	// 创建一个简单的 upgrader 实例；可根据需要自定义 CheckOrigin 等选项
	upgrader := &websocket.Upgrader{
		CheckOrigin: func(r *http.Request) bool { return true },
	}

	// 注册 WebSocket 处理 handler，路径可自定义，例如 /ws
	router.GET("/ws", wsgrpc.GinWSHandler(listener, upgrader))

	// 启动 HTTP 服务
	if err := router.Run(":8080"); err != nil {
		log.Fatalf("HTTP server error: %v", err)
	}

	// 示例中，当 HTTP 请求升级为 WebSocket 后，会将连接推入 listener，
	// gRPC Server 的 Accept 就会获取到该 net.Conn 连接，实现 gRPC 请求的代理。
}
```

### 客户端示例

```go
func main() {
	// 定义 websocket 服务器地址和 header（如果有需要）
	wsURL := "ws://127.0.0.1:8080/ws" // 示例地址
	header := http.Header{}

	// 创建 websocket dialer
	dialer := wsgrpc.WebsocketDialer(wsURL, header)

	// 使用 grpc.WithContextDialer 配置 GRPC Dial
	conn, err := grpc.DialContext(context.Background(), "ignored",
		grpc.WithContextDialer(dialer),
		grpc.WithInsecure(), // 示例中禁用 TLS，生产环境建议使用安全连接
	)
	if err != nil {
		log.Fatalf("failed to dial: %v", err)
	}
	defer conn.Close()

	// 接下来可使用 conn 创建 GRPC 客户端进行调用
}
```
