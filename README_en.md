# wsgrpc - Golang GRPC over WebSocket

[中文文档](./README.md)

The primary purpose of wsgrpc is to allow Go gRPC clients to communicate with the server via WebSocket.

Reasons:
1. Many CDNs and reverse proxies do not support HTTP/2 (gRPC uses HTTP/2) when proxying or the support is poor (for example, Cloudflare experienced widespread outages during a period of Nezha monitoring).
2. Browser applications cannot use grpc-go because the underlying network required for native operation is not available. (For example, the error "dial tcp: Protocol not available"...)

This library supports using WebSocket as a grpc.Conn to handle gRPC.

Let your gRPC easily support CDNs/reverse proxies.

## Example Usage

```go
// Assume we have a main file using the above library:
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

### Server Example

```go
// 
func main() {
	// Create a WebSocket Listener with a buffered queue size of 100; the address and network type can be customized.
	listener := wsgrpc.NewWSListener("ws-listener", "ws", 100)

	// Start the gRPC Server in a separate goroutine
	go func() {
		grpcServer := grpc.NewServer()
		// Register your gRPC services here...
		if err := grpcServer.Serve(listener); err != nil {
			log.Fatalf("gRPC server error: %v", err)
		}
	}()

	// Create an HTTP server using Gin and provide WebSocket functionality on a specific path
	router := gin.Default()

	// Create a simple upgrader instance; options such as CheckOrigin can be customized as needed.
	upgrader := &websocket.Upgrader{
		CheckOrigin: func(r *http.Request) bool { return true },
	}

	// Register the WebSocket handler at a customizable path, e.g., /ws
	router.GET("/ws", wsgrpc.GinWSHandler(listener, upgrader))

	// Start the HTTP service
	if err := router.Run(":8080"); err != nil {
		log.Fatalf("HTTP server error: %v", err)
	}

	// In this example, once an HTTP request is upgraded to WebSocket, the connection is pushed into the listener.
	// The gRPC Server's Accept will receive the net.Conn connection, enabling the proxy of gRPC requests.
}
```

### Client Example

```go
func main() {
	// Define the WebSocket server URL and header (if needed)
	wsURL := "ws://127.0.0.1:8080/ws" // Example URL
	header := http.Header{}

	// Create a WebSocket dialer
	dialer := wsgrpc.WebsocketDialer(wsURL, header)

	// Configure GRPC Dial using grpc.WithContextDialer
	conn, err := grpc.DialContext(context.Background(), "ignored",
		grpc.WithContextDialer(dialer),
		grpc.WithInsecure(), // In this example, TLS is disabled. It is recommended to use secure connections in production.
	)
	if err != nil {
		log.Fatalf("failed to dial: %v", err)
	}
	defer conn.Close()

	// You can now create a GRPC client using conn for further calls.
}
```