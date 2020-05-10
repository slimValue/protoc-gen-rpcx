
## 安装

- 首先我们需要 `protoc` 编译器: [protobuf](https://github.com/protocolbuffers/protobuf)。一般平台上都有编译好的二进制程序，直接下载加载PATH路径里即可。或者你也可以源代码安装。

`protoc`负责将proto文件编译成不同编程语言的代码，一般通过插件的方式实现。

- 编译rpcx插件:

Google提供了go和grpc的`protoc`插件，但是依照官方的解释，不准备将其做成库的方式，所以我们要实现自己的插件的话，需要一点点小技巧。

首先下载`github.com/golang/protobuf`,切换到`v1.3.5`分支。 `v1.4.0`以上的分支是基于protobuf APIV2实现的，目前还在开发之中，所以我们还是采用常用的稳定的版本。

然后把本项目下的`link_rpcx.go`文件和`rpcx`文件夹复制到`protoc-gen-go`文件夹下。

最后我们编译`protoc-gen-go`插件，安装的PATH里的文件夹下。
这个插件除了将proto文件编译成protobuf的Go文件，还包含`grpc`插件和`rpcx`插件。

具体操作步骤可以参考下面的操作：

```sh
export GO111MODULE=off

go get github.com/golang/protobuf/{proto,protoc-gen-go}
go get github.com/rpcxio/protoc-gen-rpcx

export GOPATH="$(go env GOPATH)"

export GIT_TAG="v1.3.5" 
git -C $GOPATH/src/github.com/golang/protobuf checkout $GIT_TAG

cd $GOPATH/src/github.com/golang/protobuf/protoc-gen-go &&  cp -r $GOPATH/src/github.com/rpcxio/protoc-gen-rpcx/{link_rpcx.go, rpcx} .
go install github.com/golang/protobuf/protoc-gen-go

export PATH=$PATH:$GOPATH/bin
```

如果你到达了这一步，恭喜你，包含rpcx插件的protoc-gen-go你就编译安装成功了，按照下面的命令你就可以将proto中定义的service编译成rpcx的服务和客户端代码了:
```sh
protoc -I.:${GOPATH}/src  --go_out=plugins=rpcx:. *.proto
```

## 例子

- proto文件

最简单的一个打招呼的rpc服务。

```proto
syntax = "proto3";

option go_package = "helloword";

package helloworld;

// The greeting service definition.
service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1;
}

// The response message containing the greetings
message HelloReply {
  string message = 1;
}
```

- 使用protoc编译器编译出Go代码

```sh
protoc --go_out=plugins=rpcx:. helloworld.proto
```

上述命令生成了 `helloworld.pb.go` 文件， 它包含各种struct的定义， 还有服务端的一个骨架， 以及客户端的代码。

- 服务端代码

服务端的代码只是一个骨架，很显然你要实现你的逻辑。比如这个打招呼的例子， 客户端传入一个名称，你可以返回一个`hello <name>`的字符串。

它还提供了一个简单启动服务的方法，你可以在此基础上实现服务端的代码，注册很多的服务，配置注册中心和其它插件等等。

```go
package main

import (
	context "context"
	"fmt"

	helloworld "github.com/golang/protobuf/protoc-gen-go/testdata/rpcx"
	server "github.com/smallnest/rpcx/server"
)

func main() {
	s := server.NewServer()
	s.RegisterName("Greeter", new(GreeterImpl), "")
	err := s.Serve("tcp", ":8972")
	if err != nil {
		panic(err)
	}
}

type GreeterImpl struct{}

// SayHello is server rpc method as defined
func (s *GreeterImpl) SayHello(ctx context.Context, args *helloworld.HelloRequest, reply *helloworld.HelloReply) (err error) {
	*reply = helloworld.HelloReply{
		Message: fmt.Sprintf("hello %s!", args.Name),
	}
	return nil
}
```

- 客户端代码

客户端生成的代码更友好，它包装了`XClient`对象，提供了符合人工美学的方法调用格式(请求参数作为方法参数，返回结果作为方法的返回值)。并且提供了客户端的配置方式。

你也可以扩展客户端的配置，提供注册中心、路由算法，失败模式、重试、熔断等服务治理的设置。　


```go
package main

import (
	"context"
	"fmt"

	helloworld "github.com/golang/protobuf/protoc-gen-go/testdata/rpcx"
)

func main() {
	xclient := helloworld.NewXClientForGreeter("127.0.0.1:8972")
	client := helloworld.NewGreeterClient(xclient)

	args := &helloworld.HelloRequest{
		Name: "rpcx",
	}

	reply, err := client.SayHello(context.Background(), args)
	if err != nil {
		panic(err)
	}

	fmt.Println("reply: ", reply.Message)
}

```