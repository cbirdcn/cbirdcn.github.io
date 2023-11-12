# kratos-blog新项目全纪录

kratos官方说法：
不提供根据模板变量方式创建项目的途径：
https://github.com/go-kratos/kratos/issues/1631
需要自己创建模板，然后用kratos实现自己的框架。
那就只好把默认的helloworld中需要修改的变量整理出来，初始化一个干净的项目。

## 创建golang容器

拉取镜像
> docker pull golang

映射workspace，方便在宿主机编码，容器内运行
> docker run -itd -p 8888:8000 -p 9999:9000 --name golang -v /local/path/workspace:/workspace --workdir /workspace golang

映射后，在宿主机GOLAND修改代码，容器内运行，宿主机访问。

开启GO111MODULE
> go env -w GO111MODULE=on

更新下载protobuf并安装kratos/v2
> apt-get update && apt-get -y install protobuf-compiler
> export GOPROXY=https://goproxy.io,direct
> go install github.com/go-kratos/kratos/cmd/kratos/v2@latest && kratos upgrade

## 创建项目

```
docker exec -it golang /bin/sh
cd /workspace
kratos add blog
```

- 提示：安装tree工具，可以检查目录树
- 初始化：由于kratos默认使用helloworld模板，里面除了main、third_party外，都包含greeter内容，需要置换为自己的service名。
- 官方建议：一个微服务只创建一个service

## 替换proto

proto相关改成blog（项目名），service、server相关改成blog。proto可以直接添加新文件后删除helloworld。

比如blog.proto内容如下：
```
syntax = "proto3";

package api.blog.v1;

import "google/api/annotations.proto";

option go_package = "blog/api/blog/v1;v1";
option java_multiple_files = true;
option java_package = "api.blog.v1";

// service名称不能带Service，防止自动生成两个Service结尾的实现结构体
service Blog {
	rpc CreateBlog (CreateBlogRequest) returns (CreateBlogReply) {
//		option (google.api.http) = {
//			post: "/v1/blog/"
//			body: "*"
//		};
	};
	rpc UpdateBlog (UpdateBlogRequest) returns (UpdateBlogReply){
//		option (google.api.http) = {
//			put: "/v1/blog/{id}"
//			body: "*"
//		};
	};
	rpc DeleteBlog (DeleteBlogRequest) returns (DeleteBlogReply){
//		option (google.api.http) = {
//			delete: "/v1/blog/{id}"
//		};
	};
	rpc GetBlog (GetBlogRequest) returns (GetBlogReply){
//		option (google.api.http) = {
//			get: "/v1/blog/{id}"
//		};
	};
	rpc ListBlog (ListBlogRequest) returns (ListBlogReply) {
		option (google.api.http) = {
			get: "/v1/blog/"
		};
	};
}

message CreateBlogRequest {}
message CreateBlogReply {}

message UpdateBlogRequest {}
message UpdateBlogReply {}

message DeleteBlogRequest {}
message DeleteBlogReply {}

message GetBlogRequest {}
message GetBlogReply {}

message ListBlogRequest {}
message ListBlogReply {}
```


```
kratos proto add api/blog/v1/blog.proto
kratos proto client api/blog/v1/blog.proto
kratos proto server api/blog/v1/blog.proto -t internal/service
```

平移error_reason.proto到blog/v1中。greeter改成blog后，删掉整个helloworld。
最后根据新的error_reason.proto生成新的pb.go

生成所有proto相关文件

```
go generate ./...
```

如果有wire错误提示需要get：
```
go get -d github.com/google/wire/cmd/wire@v0.5.0
```

api最终结构：


```
./api
`-- blog
    `-- v1
        |-- blog.pb.go
        |-- blog.proto
        |-- blog_grpc.pb.go
        |-- error_reason.pb.go
        |-- error_reason.proto
        `-- error_reason_errors.pb.go
```

- 注意 http 代码只会在 proto 文件中声明了 http 时才会生成

可以在proto文件中添加：

```
import "google/api/annotations.proto";

service Blog {
    ...
	rpc ListBlog (ListBlogRequest) returns (ListBlogReply) {
		option (google.api.http) = {
			get: "/init_http"
		};
	};
}
```
然后调用kratos proto client或go generate就可以生成blog_http.pb.go
在ListBlog()中添加option http的原因是，这样可以避免在service/blog.go中添加新的实现方法，可以直接用自动生成的ListBlog()


## 替换internal

更改biz/greeter.go为blog.go:

```
type Blog struct {
}

type BlogRepo interface {
}

type BlogUsecase struct {
	repo BlogRepo
	log  *log.Helper
}

func NewBlogUsecase(repo BlogRepo, logger log.Logger) *BlogUsecase {
	return &BlogUsecase{repo: repo, log: log.NewHelper(logger)}
}
```

更改biz/biz.go为blog.go:

```
var ProviderSet = wire.NewSet(NewBlogUsecase)
```


更改conf/conf.proto

```
option go_package = "blog/internal/conf;conf";
```
重新编译生成conf.pb.go

更改data/greeter.go为blog.go

```
type blogRepo struct {
	data *Data
	log  *log.Helper
}

// NewBlogRepo .
func NewBlogRepo(data *Data, logger log.Logger) biz.BlogRepo {
	return &blogRepo{
		data: data,
		log:  log.NewHelper(logger),
	}
}
```

data/data.go

```
var ProviderSet = wire.NewSet(NewData, NewBlogRepo)
```

service/service.go


```
var ProviderSet = wire.NewSet(NewBlogService)
```

service/blog.go

- 注意：工具不会修改已经生成的service文件，并且生成的service文件中，struct type是服务名+Service的形式，比如BlogService

```
package service

import (
	pb "blog/api/blog/v1"
	"blog/internal/biz"
	"context"
	"github.com/go-kratos/kratos/v2/log"
)

type BlogService struct {
	pb.UnimplementedBlogServer
	uc  *biz.BlogUsecase
	log *log.Helper
}

func NewBlogService(uc *biz.BlogUsecase, logger log.Logger) *BlogService {
	return &BlogService{uc: uc, log: log.NewHelper(logger)}
}

func (s *BlogService) CreateBlog(ctx context.Context, req *pb.CreateBlogRequest) (*pb.CreateBlogReply, error) {
	return &pb.CreateBlogReply{}, nil
}
func (s *BlogService) UpdateBlog(ctx context.Context, req *pb.UpdateBlogRequest) (*pb.UpdateBlogReply, error) {
	return &pb.UpdateBlogReply{}, nil
}
func (s *BlogService) DeleteBlog(ctx context.Context, req *pb.DeleteBlogRequest) (*pb.DeleteBlogReply, error) {
	return &pb.DeleteBlogReply{}, nil
}
func (s *BlogService) GetBlog(ctx context.Context, req *pb.GetBlogRequest) (*pb.GetBlogReply, error) {
	return &pb.GetBlogReply{}, nil
}
func (s *BlogService) ListBlog(ctx context.Context, req *pb.ListBlogRequest) (*pb.ListBlogReply, error) {
	s.log.WithContext(ctx).Infof("ListBlog ...")
	return &pb.ListBlogReply{}, nil
}
```

删除service/greeter.go

修改server/grpc.go：

```
package server

import (
	pb "blog/api/blog/v1"
	"blog/internal/conf"
	"blog/internal/service"
	"github.com/go-kratos/kratos/v2/log"
	"github.com/go-kratos/kratos/v2/middleware/recovery"
	"github.com/go-kratos/kratos/v2/transport/grpc"
)

// NewGRPCServer new a gRPC server.
func NewGRPCServer(c *conf.Server, blog *service.BlogService, logger log.Logger) *grpc.Server {
	var opts = []grpc.ServerOption{
		grpc.Middleware(
			recovery.Recovery(),
		),
	}
	if c.Grpc.Network != "" {
		opts = append(opts, grpc.Network(c.Grpc.Network))
	}
	if c.Grpc.Addr != "" {
		opts = append(opts, grpc.Address(c.Grpc.Addr))
	}
	if c.Grpc.Timeout != nil {
		opts = append(opts, grpc.Timeout(c.Grpc.Timeout.AsDuration()))
	}
	srv := grpc.NewServer(opts...)
	pb.RegisterBlogServer(srv, blog)
	return srv
}
```

server/http.go：
- 注意必须在proto中声明http服务，并且生成http客户端解析文件后才能支持RegisterBlogHTTPServer()方法

```
package server

import (
	pb "blog/api/blog/v1"
	"blog/internal/conf"
	"blog/internal/service"
	"github.com/go-kratos/kratos/v2/log"
	"github.com/go-kratos/kratos/v2/middleware/recovery"
	"github.com/go-kratos/kratos/v2/transport/http"
)

// NewHTTPServer new a HTTP server.
func NewHTTPServer(c *conf.Server, blog *service.BlogService, logger log.Logger) *http.Server {
	var opts = []http.ServerOption{
		http.Middleware(
			recovery.Recovery(),
		),
	}
	if c.Http.Network != "" {
		opts = append(opts, http.Network(c.Http.Network))
	}
	if c.Http.Addr != "" {
		opts = append(opts, http.Address(c.Http.Addr))
	}
	if c.Http.Timeout != nil {
		opts = append(opts, http.Timeout(c.Http.Timeout.AsDuration()))
	}
	srv := http.NewServer(opts...)
	pb.RegisterBlogHTTPServer(srv, blog)
	return srv
}

```

最后，自动更新wire_gen.go。仍然通过：

```
go generate ./...
```
此时cmd/blog/wire_gen.go生成：

```
// initApp init kratos application.
func initApp(confServer *conf.Server, confData *conf.Data, logger log.Logger) (*kratos.App, func(), error) {
	dataData, cleanup, err := data.NewData(confData, logger)
	if err != nil {
		return nil, nil, err
	}
	blogRepo := data.NewBlogRepo(dataData, logger)
	blogUsecase := biz.NewBlogUsecase(blogRepo, logger)
	blogService := service.NewBlogService(blogUsecase, logger)
	httpServer := server.NewHTTPServer(confServer, blogService, logger)
	grpcServer := server.NewGRPCServer(confServer, blogService, logger)
	app := newApp(logger, httpServer, grpcServer)
	return app, func() {
		cleanup()
	}, nil
}

```
初始化完毕。



## 测试


```
# kratos run
INFO msg=[gRPC] server listening on: [::]:9000
INFO msg=[HTTP] server listening on: [::]:8000
```

- http

容器端口映射8888:8000，服务器配置在configs/config.yaml。

如果遇到容器内访问宿主机或其他容器的db、redis的情况，可以把ip改写。建议创建mysql容器，映射3306:3306，这样当前项目ip改成host.docker.internal就可以做到三方同端口访问了。
```
http://127.0.0.1:8888/init_http
```
访问显示：

```
INFO ts=2021-11-11T09:34:32Z caller=helper.go:73 service.id=2f25c2335d31 service.name= service.version= trace_id= span_id= =CreateBlog ...
```

- rpc客户端

配置端口9000，对外端口9999

可以用BloomRPC自行测试


# 编写新代码

## 修改proto

blog.proto

添加输入参数校验

```
// the validate rules:
// https://github.com/envoyproxy/protoc-gen-validate
import "validate/validate.proto";
```

service名称改成带Service，因为proto不允许声明重复，所以如果message blog和service blog同时声明，就会错误。

```
service BlogService {
    
}
```


http服务应该遵循REST规则：

PUT 和POST方法的区别是,PUT方法是幂等的：连续调用一次或者多次的效果相同（无副作用）。连续调用同一个POST可能会带来额外的影响，比如多次提交订单。

proto http rule:
[link](https://cloud.google.com/endpoints/docs/grpc-service-config/reference/rpc/google.api#google.api.Http)

```
	rpc CreateBlog (CreateBlogRequest) returns (CreateBlogReply) {
		option (google.api.http) = {
			post: "/v1/blog/"
			body: "*"
		};
	};
	rpc UpdateBlog (UpdateBlogRequest) returns (UpdateBlogReply){
		option (google.api.http) = {
			put: "/v1/blog/{id}"
			body: "*"
		};
	};
	rpc DeleteBlog (DeleteBlogRequest) returns (DeleteBlogReply){
		option (google.api.http) = {
			delete: "/v1/blog/{id}"
		};
	};
	rpc GetBlog (GetBlogRequest) returns (GetBlogReply){
		option (google.api.http) = {
			get: "/v1/blog/{id}"
		};
	};
	rpc ListBlog (ListBlogRequest) returns (ListBlogReply) {
		option (google.api.http) = {
			post: "/v1/blog/"
		};
	};
```
另外，可以使用additional_bindings选项为一个 RPC 定义多个 HTTP 方法。比如：

```
    option (google.api.http) = {
      get: "/v1/messages/{message_id}"
      additional_bindings {
        get: "/v1/users/{user_id}/messages/{message_id}"
      }
    };
```


