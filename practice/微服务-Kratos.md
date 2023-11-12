# Kratos实现RPC微服务

## 项目结构

kratos cli 用于从模板创建项目，维护依赖包版本等。

Kratos使用Protobuf进行API定义。需要操作系统安装protoc工具，生成服务器和客户端协议解析和传输相关文件。

wire是google提供的依赖注入工具。它是一个代码生成器。我们只需要在一个特殊的go文件中告诉wire类型之间的依赖关系，它会自动帮我们生成代码，帮助我们创建指定类型的对象，并组装它的依赖。

Kratos定义了统一的注册接口，通过实现服务注册与发现Registrar和Discovery，您可以很轻松地将Kratos接入到您的注册中心中。

#### 实践

kratos cli 创建新项目。并修改代码中的hello硬编码。

编写proto文件，并生成服务器和客户端的pb文件。
