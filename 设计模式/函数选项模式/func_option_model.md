## 函数式选项模式

Go语言实现函数式选项模式（Functional Options Pattern）的简单例子
函数式选项模式是一种在Go语言中用于简化配置和构造对象的方式，它通过传递一系列配置函数来设置对象的属性，而不是直接传递多个参数。

## Optional模式在TCC中的应用
解决问题：在设计一个函数时，当存在配置参数较多，同时参数可选时，函数式选项模式是一个很好的选择，它既有为不熟悉的调用者准备好的默认配置，还有为需要定制的调用者提供自由修改配置的能力，且支持未来灵活扩展属性。

## 示例

```go
package func_options_model

import (
	"time"
)

type Server struct {
	addr    string
	port    int
	timeout time.Duration
}

// options 定义函数类型 设置 Server 配置属性
type Option func(*Server)

// WithAddr 设置服务器地址
func WithAddr(addr string) Option {
	return func(s *Server) {
		s.addr = addr
	}
}

// WithPort 设置服务器端口
func WithPort(port int) Option {
	return func(s *Server) {
		s.port = port
	}
}

// WithTimeout 设置服务器超时时间
func WithTimeout(timeout time.Duration) Option {
	return func(s *Server) {
		s.timeout = timeout
	}
}

func NewServer(opts ...Option) *Server {
	server := &Server{
		addr:    "localhost",
		port:    8080,
		timeout: 5 * time.Second,
	}
	for _, opt := range opts {
		opt(server)
	}
	return server
}
```

## 单测
```go
package func_options_model

import (
	"fmt"
	"testing"
	"time"
)

func TestFuncOptionsModel(t *testing.T) {
	// 创建服务器对象
	server := NewServer(WithAddr("IP_ADDRESS"), WithPort(8080), WithTimeout(5*time.Second))
	fmt.Printf("Server address: %s\n", server.addr)
	fmt.Printf("Server port: %d\n", server.port)
	fmt.Printf("Server timeout: %s\n", server.timeout)
}

/*
Running tool: /opt/homebrew/bin/go test -timeout 30s -run ^TestFuncOptionsModel$ chain_of_duty_model/func_options_model

=== RUN   TestFuncOptionsModel
Server address: IP_ADDRESS
Server port: 8080
Server timeout: 5s
--- PASS: TestFuncOptionsModel (0.00s)
PASS
ok      chain_of_duty_model/func_options_model  0.247s

*/

```