# 服务器-Gin实践

## Gin介绍

### 简介

Gin是Golang编写的Web框架。得益于[julienschmidt/httprouter](https://github.com/julienschmidt/httprouter.git)，有良好的性能。

HttpRouter是一个轻量级的高性能HTTP请求路由器(也称为多路复用器或简称mux)。与Go的net/http包的默认mux相反，这个路由器支持路由模式中的变量，并与请求方法相匹配。它的可扩展性也更好。该路由器针对高性能和小内存占用进行了优化。即使有很长的路径和大量的路线，它也能很好地扩展。采用压缩动态树(根树)结构进行高效匹配。

Web应用的本质：
- 浏览器发送一个HTTP请求；
- 服务器收到请求，生成一个HTML文档；
- 服务器把HTML文档作为HTTP响应的Body发送给浏览器；
- 浏览器收到HTTP响应，从HTTP Body取出HTML文档并显示。

所以，最简单的Web应用就是先把HTML用文件保存好，用一个现成的HTTP服务器软件，接收用户请求，从文件中读取HTML，返回。Apache、Nginx、Lighttpd等这些常见的静态服务器就是干这件事情的。

如果要动态生成HTML，就需要把上述步骤自己来实现。不过，接受HTTP请求、解析HTTP请求、发送HTTP响应都是苦力活，如果我们自己来写这些底层代码，还没开始写动态HTML呢，就得花个把月去读HTTP规范。

正确的做法是底层代码由专门的服务器软件实现，所以，需要一个统一的接口，让我们专心编写Web业务。这个接口就是WSGI：Web Server Gateway Interface。

开发者根据接口定义，实现一个函数，响应HTTP请求就可以了。

## Gin实践

基本使用

```go
package main

import (
	"log"
	"net/http"

	"github.com/gin-gonic/gin"
)

// https://geektutu.com/post/quick-go-gin.html

func main() {
	r := gin.Default() // 生成一个应用程序实例

	// 声明一个路由，以及触发的函数
	// curl http://localhost:8081/
	r.GET("/", func(c *gin.Context) {
		// 函数内返回客户端想要的响应
		c.String(http.StatusOK, "hello world.")
	})

	// 路由方法有 GET, POST, PUT, PATCH, DELETE 和 OPTIONS，还有Any，可匹配以上任意类型的请求。

	// 带参数的URL。:name表示传入不同的 name。/user/:name/*role，* 代表可选。
	// curl http://localhost:8081/user/geektutu
	r.GET("/user/:name", func(c *gin.Context) {
		name := c.Param("name")
		c.String(http.StatusOK, "Hello %s", name)
	})

	// 获取Query参数
	// 匹配users?name=xxx&role=xxx，role可选
	// 注意query和param格式的不同
	// curl "http://localhost:8081/users?name=Tom&role=student"
	r.GET("/users", func(c *gin.Context) {
		name := c.Query("name")
		role := c.DefaultQuery("role", "teacher") // 为query设置默认值
		c.String(http.StatusOK, "%s is a %s", name, role)
	})

	// POST
	// curl http://localhost:8081/form  -X POST -d 'username=geektutu&password=1234'
	r.POST("/form", func(c *gin.Context) {
		username := c.PostForm("username")
		password := c.DefaultPostForm("password", "000000") // 可设置默认值

		// c.JSON()返回JSON消息
		// gin.H{} is a shortcut for map[string]interface{}
		// 注意，会被自动排序
		c.JSON(http.StatusOK, gin.H{
			"username": username,
			"password": password,
		})
	})

	// GET 和 POST 混合
	// curl "http://localhost:8081/posts?id=9876&page=7"  -X POST -d 'username=geektutu&password=1234'
	r.POST("/posts", func(c *gin.Context) {
		id := c.Query("id")
		page := c.DefaultQuery("page", "0")
		username := c.PostForm("username")
		password := c.DefaultPostForm("username", "000000") // 可设置默认值

		c.JSON(http.StatusOK, gin.H{
			"id":       id,
			"page":     page,
			"username": username,
			"password": password,
		})
	})

	// Map参数(字典参数)
	// curl -g "http://localhost:8081/post?ids[Jack]=001&ids[Tom]=002" -X POST -d 'names[a]=Sam&names[b]=David'
	// {"ids":{"Jack":"001","Tom":"002"},"names":{"a":"Sam","b":"David"}}
	r.POST("/post", func(c *gin.Context) {
		ids := c.QueryMap("ids")
		names := c.PostFormMap("names")

		c.JSON(http.StatusOK, gin.H{
			"ids":   ids,
			"names": names,
		})
	})

	// 重定向(Redirect)
	// https://pkg.go.dev/gopkg.in/gin-gonic/gin.v1#section-readme
	// curl -i http://localhost:8081/redirect
	// curl "http://localhost:8081/reindex"
	// Issuing a HTTP redirect is easy. Both internal and external locations are supported.
	r.GET("/redirect", func(c *gin.Context) {
		// http重定向
		c.Redirect(http.StatusMovedPermanently, "/reindex")
	})
	// Issuing a Router redirect, use HandleContext like below.
	r.GET("/reindex", func(c *gin.Context) {
		c.Request.URL.Path = "/" // 路由重定向，目标是"/"
		r.HandleContext(c)       // 执行调用
	})

	// 分组路由(Grouping Routes)
	// curl http://localhost:8081/v1/posts
	// curl http://localhost:8081/v2/posts
	defaultHandler := func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{
			"path": c.FullPath(),
		})
	}
	// group: v1
	v1 := r.Group("/v1")
	{
		v1.GET("/posts", defaultHandler)
		v1.GET("/series", defaultHandler)
	}
	// group: v2
	v2 := r.Group("/v2")
	{
		v2.GET("/posts", defaultHandler)
		v2.GET("/series", defaultHandler)
	}

	// 上传文件
	// 单个文件
	r.POST("/upload1", func(c *gin.Context) {
		file, _ := c.FormFile("file")
		// c.SaveUploadedFile(file, dst)
		c.String(http.StatusOK, "%s uploaded!", file.Filename)
	})
	// 多文件
	r.POST("/upload2", func(c *gin.Context) {
		// Multipart form
		form, _ := c.MultipartForm()
		files := form.File["upload[]"]

		for _, file := range files {
			log.Println(file.Filename)
			// c.SaveUploadedFile(file, dst)
		}
		c.String(http.StatusOK, "%d files uploaded!", len(files))
	})

	// HTML模板
	// curl http://localhost:8081/arr
	// Gin默认使用模板Go语言标准库的模板text/template和html/template
	// 标准库：https://pkg.go.dev/text/template
	// https://pkg.go.dev/html/template
	type student struct {
		Name string
		Age  int8
	}
	r.LoadHTMLGlob("templates/*") // LoadHTMLGlob loads HTML files identified by glob pattern and associates the result with HTML renderer.
	stu1 := &student{Name: "Geektutu", Age: 20}
	stu2 := &student{Name: "Jack", Age: 22}
	r.GET("/arr", func(c *gin.Context) {
		c.HTML(http.StatusOK, "arr.tmpl", gin.H{
			"title":     "Gin",
			"stuArr":    [2]*student{stu1, stu2},  // 写法1：[2]*student{stu1, stu2}表示长度=2的slice，slice内每个元素都是*student指针类型
			"stuValArr": [2]student{*stu1, *stu2}, // 写法2：[2]student{*stu1, *stu2}。两者模板写法一致，原因应该是模板编译过程中，自动把写法1的变量var编译成了*var，不管指针还是值最终都是拿到值的拷贝了
		})
	})

	r.Run(":8081") //让应用运行在一个本地服务器上，监听端口默认为8080
}
```

使用中间件
```go
// middleware.go
package main

import (
	"fmt"
    "time"
	"github.com/gin-gonic/gin"
)

// 使用教程：https://www.topgoer.com/gin%E6%A1%86%E6%9E%B6/gin%E4%B8%AD%E9%97%B4%E4%BB%B6/next%E6%96%B9%E6%B3%95.html
// 其他中间件：https://www.topgoer.com/gin%E6%A1%86%E6%9E%B6/gin%E4%B8%AD%E9%97%B4%E4%BB%B6/%E4%B8%AD%E9%97%B4%E4%BB%B6%E6%8E%A8%E8%8D%90.html

// 定义中间件
func MiddleWare() gin.HandlerFunc {
    return func(c *gin.Context) {
        t := time.Now()
        fmt.Println("前置中间件开始执行")
        // 设置变量到Context的key中，可以通过Get()取
        c.Set("request", "中间件")
        // 前置中间件结束，开始执行函数
		fmt.Println("前置中间件结束")
        c.Next()
        // 后置中间件开始（执行完handleFunc后要做的工作）
		fmt.Println("后置中间件开始")
        status := c.Writer.Status()
        fmt.Println("后置中间件执行完毕", status)
        t2 := time.Since(t)
        fmt.Println("time:", t2)
    }
}

func main() {
    // 1.创建路由
    // 默认使用了2个中间件Logger(), Recovery()
    r := gin.Default()
    // 注册全局中间件
    r.Use(MiddleWare())
    // {}为了代码规范
    {
        r.GET("/ce", func(c *gin.Context) {
            // 取值
            req, _ := c.Get("request")
            fmt.Println("函数执行，获取request:", req)
            // 页面接收
            c.JSON(200, gin.H{"request": req})
        })

    }
	// 打印：
	/*
	前置中间件开始执行
	前置中间件结束
	函数执行，获取request: 中间件
	后置中间件开始
	后置中间件执行完毕 200
	*/

	// 局部中间件
    r.GET("/ce2", MiddleWare(), func(c *gin.Context) {
        // 取值
        req, _ := c.Get("request")
        fmt.Println("request:", req)
        // 页面接收
        c.JSON(200, gin.H{"request": req})
    })
	// 输出：相当于两次经过同样的中间件
	/*
	前置中间件开始执行
	前置中间件结束
	前置中间件开始执行
	前置中间件结束
	request: 中间件
	后置中间件开始
	后置中间件执行完毕 200
	time: 39.598µs
	后置中间件开始
	后置中间件执行完毕 200
	time: 60.384µs
	*/

    r.Run(":8082")
}
```
以及模板文件templates/arr.tmpl
```html
<!-- templates/arr.tmpl -->
<html>
<body>
    <p>hello, {{.title}}</p>
    {{range $index, $ele := .stuArr }}
    <p>{{ $index }}: {{ $ele.Name }} is {{ $ele.Age }} years old</p>
    {{ end }}
    {{range $index, $ele := .stuValArr }}
    <p>{{ $index }}: {{ $ele.Name }} is {{ $ele.Age }} years old</p>
    {{ end }}
</body>
```