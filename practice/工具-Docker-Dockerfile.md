# 工具-Docker-Dockerfile

Dockerfile 是一个用来构建自定义镜像的文本文件，文本内容包含了一条条构建镜像所需的指令和说明。

dockerhub以及各种开源官方提供的镜像往往是基于最简基础镜像的，这些基础镜像一般是alpine、debian等。

在实际项目中，需要对这些官方镜像添加诸如编译好的项目可执行文件、linux工具(如vi)、网络联通环境、端口暴露、文件映射等，甚至可能还包括多阶段编译。

这一般就需要结合Dockerfile+docker-compose，得到最合适的自定义镜像。然后用这一个镜像到处生成容器即可。

## Dockerfile

### 整体格式

```Dockerfile
# Comment 表示注释
INSTRUCTION arguments  # INSTRUCTION表示指令（不止一条，跨行用`\`）
```

### FROM-指定基础镜像

第一条指令必须是FROM指令，且在同一个Dockerfile中创建多个镜像时，可以使用多个FROM指令。

```Dockerfile
FROM <image> 
FROM <image>:<tag> 
```
其中tag是可选项，默认值为latest。

如果不以任何镜像为基础，那么写法为：FROM scratch。

生产举例

```Dockerfile
FROM golang:alpine as builder
...

FROM alpine:latest
...
COPY --from=0 src_path ./dst_path
```

```Dockerfile
FROM golang:1.18-alpine as step1
...

FROM alpine:latest
...
COPY --from=step1 src ./dst
```

### RUN-运行指定的命令

包含两种语法格式

```Dockerfile
# shell格式：就像在命令行中输入的Shell脚本命令一样。
RUN <command> 

# exec格式：就像是函数调用的格式。
RUN ["executable", "param1", "param2"] 
```

第一种后边直接跟shell命令。在linux操作系统上默认使用`/bin/sh -c`将字符串作为指令执行。当然`/bin/sh`一般是`/bin/bash`的软链，所以用后者也行。

第二种相当于传入参数。可将executable理解成为可执行文件，后面就是两个参数。

举例

```Dockerfile
RUN /bin/bash -c 'source $HOME/.bashrc; echo $HOME'

RUN ["/bin/bash", "-c", "echo hello"] 
```

> 注意：
> 多行命令不要写多个RUN，原因是Dockerfile中每一个指令都会建立一个镜像层用于缓存。多少个RUN就构建了多少层镜像，会造成镜像的臃肿、多层，不仅仅增加了构件部署的时间，还容易出错（缓存的读写失效和删除是不稳定的）。
> RUN书写时可以用`&&`连接多个命令，换行符是`\`

生产举例
```Dockerfile
RUN go env -w GO111MODULE=on \
    && go env -w GOPROXY=https://goproxy.cn,direct \
    && go env -w CGO_ENABLED=0 \
    && go env \
    && go mod tidy \
    && go build -o server .
```

### CMD-容器启动时要运行的命令

三种语法格式

## 主要参考

[Docker Dockerfile指令大全](https://zhuanlan.zhihu.com/p/419175543)

[官方Dockerfile reference](https://docs.docker.com/engine/reference/builder/)