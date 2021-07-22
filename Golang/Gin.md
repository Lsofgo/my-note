[toc]

```shell
go get github.com/gin-gonic/gin
```

## 简单使用

```go
func main() {
	g := gin.Default()
	g.GET("/", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{
			"msg": "index page",
		})
	})
	_ = g.Run("127.0.0.1:8000")
}
```

## Gin 渲染

### 静态文件处理

```go
func main() {
	g := gin.Default()
	g.Static("/static", "./static")
	g.LoadHTMLGlob("templates/**/*")					// 指定template路径, ** 所有目录, * 所有文件
    // g.LoadHTMLFiles("templates/users/index.html")	// 指定template文件
	//......
}
```



### HTML 渲染

> templates/users/index.html

- define...end
  - HTML中有这个标记, define的路径, 就是 c.HTML 中调用文件的路径
  - HTML中没有这个标记, 加载template时, 会使用文件名作为 调用路径

```html
{{define "users/index.html"}}
<!DOCTYPE html>
<html lang="zh_CN">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
{{.username}}
</body>
</html>
{{end}}
```

> main.go

```go
g.GET("/", func(c *gin.Context) {
    c.HTML(http.StatusOK, "users/index.html", gin.H{"username": "alec"})
})
```

### 自定义模板函数

> templates/index.tmpl

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
    <div>{{ . | safe }}</div>
</body>
</html>
```

> main.go

```go
func main() {
	g := gin.Default()
    
    // 自定义一个safe模板函数
	g.SetFuncMap(template.FuncMap{
		"safe": func(str string) template.HTML {
			return template.HTML(str)
		},
	})
    
    g.LoadHTMLGlob("templates/index.tmpl")
	g.GET("/", func(c *gin.Context) {
		c.HTML(http.StatusOK, "index.tmpl", "<h1>hello template</h1>")
	})
	_ = g.Run("127.0.0.1:8000")
}
```

### 模板嵌套(略)

### JSON渲染

```go
// 使用map渲染, gin.H 是 map[string]interface{}
g.GET("/", func(c *gin.Context) {
    c.JSON(200, gin.H{
        "name": "alec",
        "age": 18,
    })
})

// 使用结构体渲染, 结构体字段首字母必须大写
type stu struct {
    Name string `json:"user"`
    Age  int
}
g.GET("/", func(c *gin.Context) {
    c.JSON(200, stu{"alec", 22})
})
```

### XML渲染

和 JSON 渲染类似, 调用 XML 方法; 页面返回为 XML 格式内容

```go
// 使用map渲染
g.GET("/", func(c *gin.Context) {
    c.XML(200, gin.H{"user": "alec", "age": 23})
})

// 使用结构体渲染
type stu struct {
    Name string `json:"user"`
    Age  int
}
g.GET("/", func(c *gin.Context) {
    c.XML(200, stu{"alec", 22})
})
```

### YAML渲染

和 JSON 渲染类似

```go
g.GET("/", func(c *gin.Context) {
    c.YAML(200, gin.H{"user": "alec", "age": 23})
})
```

## 获取参数

### 获取 queryString 参数

http://127.0.0.1:8000/api/users/query?user=alec

```go
g.GET("/api/users/query", func(c *gin.Context) {
    user := c.Query("user")
    // url中不包含参数, 就使用默认值
    username := c.DefaultQuery("username", "bob")
    c.JSON(200, gin.H{"user": user, "username": username})
})
```

### 获取 path 参数

http://127.0.0.1:8000/api/users/search/alec/18

```go
g.GET("/api/users/search/:user/:age", func(c *gin.Context) {
    user := c.Param("user")
    age := c.Param("age")

    c.JSON(200, gin.H{"user": user, "age": age})
})
```

### 获取 form 参数

```go
g.POST("/api/users/add", func(c *gin.Context) {
    user := c.PostForm("user")
    password := c.PostForm("password")
    c.JSON(200, gin.H{"user": user, "password": password})
})
```

```http
POST http://localhost:8000/api/users/add
Content-Type: application/x-www-form-urlencoded

user=alec&password=111111
```

### 获取 json 参数

```go
g.POST("/api/users/modify", func(c *gin.Context) {
    data, _ := c.GetRawData()	// data 是 []byte, 需要解析成 map
    var m map[string]interface{}
    _ = json.Unmarshal(data, &m)
    c.JSON(200, m)
})
```

```http
POST http://127.0.0.1:8000/api/users/modify
Content-Type: application/json

{
  "user": "alec",
  "password": "111111"
}
```

### 参数绑定

利用 `ShouldBind` 方法, 可以自动把 QueryString, Form, JSON, XML, YAML 中的数据绑定到结构体中

```go
type user struct {
	Username string `form:"username" json:"username" binding:"required"`
	Password string `form:"password" json:"password" binding:"required"`
}

func bindHandler(c *gin.Context) {
	var u user
	if err := c.ShouldBind(&u); err == nil {
		c.JSON(200, u)
	}
}

func main() {
	g := gin.Default()
	g.GET("/api/loginQueryString", bindHandler)
	g.POST("/api/loginJSON", bindHandler)
    g.POST("/api/loginForm", bindHandler)
	_ = g.Run("127.0.0.1:8000")
}
```

```http
###
GET http://127.0.0.1:8000/api/loginQueryString?username=alec&password=111111
Accept: application/json

###
POST http://127.0.0.1:8000/api/loginJSON
Content-Type: application/json

{
  "username": "alec",
  "password": "111112"
}

###
POST http://127.0.0.1:8000/api/loginForm
Content-Type: application/x-www-form-urlencoded

username=alec&password=111113
```



## 文件上传

### 单个文件上传

```go
```

### 多个文件上传

```go
```



## 重定向

### HTTP 重定向

重定向到 外部, 内部 地址

```go
// 被重定向的地址
g.GET("/test", func(c *gin.Context) {
    c.JSON(200, gin.H{"msg": "test page"})
})
// 重定向到外部地址, 页面URL是 "https://www.baidu.com"
g.GET("/api/test", func(c *gin.Context) {
    c.Redirect(301, "https://www.baidu.com")
})
// 重定向到内部地址, 页面URL是 "127.0.0.1:8000/test"
g.GET("/api/test2", func(c *gin.Context) {
    c.Redirect(301, "/test")
})
```

### 路由重定向

```go
g.GET("/test", func(c *gin.Context) {
    c.JSON(200, gin.H{"msg": "test page"})
})
// 重定向到/test, 页面URL是 "127.0.0.1:8000/api/test3", 内容是 /test 的内容
g.GET("/api/test3", func(c *gin.Context) {
    c.Request.URL.Path = "/test"
    g.HandleContext(c)
})
```



## 路由

### 普通路由

```
g.GET("/api/test", func(c *gin.Context) { c.JSON(200, gin.H{"msg": "test page"}) })
g.POST("/api/test2", func(c *gin.Context) { c.JSON(200, gin.H{"msg": "test page"}) })
g.PUT("/api/test3", func(c *gin.Context) { c.JSON(200, gin.H{"msg": "test page"}) })
```

Any 可以匹配所有的请求方法

```go
g.Any("/api/test4", func(c *gin.Context) { c.JSON(200, gin.H{"msg": "test page"}) })
```

没有定义的路由, 可以使用 NoRoute 处理, 默认返回404代码

> templates/404.html

```html
<!DOCTYPE html>
<html lang="zh_CN">
<head>
    <meta charset="UTF-8">
    <title>404</title>
</head>
<body>
<h1>404 Error Page</h1>
</body>
</html>
```

```go
g.LoadHTMLFiles("templates/404.html")
g.NoRoute(func(c *gin.Context) { c.HTML(http.StatusNotFound, "404.html", nil) })
```

### 路由组

`{}` 是为了结构看着清晰一些, 实际可以不加

```go
apiGroup := g.Group("/api")
{
    apiGroup.GET("/list", func(c *gin.Context) { c.String(200, c.Request.URL.Path) })
    apiGroup.POST("/add", func(c *gin.Context) { c.String(200, c.Request.URL.Path) })
    // 路由组的嵌套
    userGroup := apiGroup.Group("/user")
    {
        userGroup.GET("list", func(c *gin.Context) { c.String(200, c.Request.URL.Path) })
    }
}
```

## 中间件

Gin框架允许开发者在处理请求的过程中，加入用户自己的钩子（Hook）函数。

这个钩子函数就叫中间件，中间件适合处理一些公共的业务逻辑，比如登录认证、权限校验、数据分页、记录日志、耗时统计等。

### 定义中间件

```go
// 定义一个统计请求耗时的中间件
func requestCost() gin.HandlerFunc{
	return func(c *gin.Context) {
		start := time.Now()
		c.Set("name", "alec")	// 可以通过 c.Set 在请求上下文中设置值，后续的处理函数能够取到该值
		// 调用该请求的剩余处理程序
		c.Next()
		// 不调用该请求的剩余处理程序
		//c.Abort()
		cost := time.Since(start)
		log.Println(c.Request.URL.Path, "消耗时间:", cost)
	}
}
```

### 注册中间件

在gin框架中，我们可以为每个路由添加任意数量的中间件

#### 为全局路由注册

Next 会执行 time.Sleep, 并返回 web页面

Abort 会跳过 time.Sleep, 返回一个空页面

```go
g := gin.Default()
g.Use(requestCost())
g.GET("/", func(c *gin.Context) {
    time.Sleep(3 * time.Second)
    c.String(200, "index page")
})
```

#### 为单个路由注册

可以添加多个中间件

```go
g.GET("/", requestCost(),func(c *gin.Context) {
    time.Sleep(3 * time.Second)
    c.String(200, "index page")
})
```

#### 为路由组注册

有两种写法

```go
apiGroup := g.Group("/api", requestCost())
{
    apiGroup.GET("/list", func(c *gin.Context) {...})
}
```

```go
apiGroup := g.Group("/api")
apiGroup.Use(requestCost())
{
    apiGroup.GET("/list", func(c *gin.Context) {...})
}
```

#### 中间件注意事项

**gin默认中间件**

`gin.Default()`默认使用了`Logger`和`Recovery`中间件，其中：

- `Logger`中间件将日志写入`gin.DefaultWriter`，即使配置了`GIN_MODE=release`
- `Recovery`中间件会recover任何`panic`。如果有panic的话，会写入500响应码

如果不想使用上面两个默认的中间件，可以使用`gin.New()`新建一个没有任何默认中间件的路由

**gin中间件中使用goroutine**

当在中间件或`handler`中启动新的`goroutine`时，**不能使用**原始的上下文（c *gin.Context），必须使用其只读副本（`c.Copy()`)



## gin框架路由拆分与注册

把类似 InitApiRouter 的方法放到不同的包中就可以实现路由拆分注册

```go
func InitApiRouter(e *gin.Engine) {
	apiGroup := e.Group("/api")
	{
		apiGroup.GET("/test", func(c *gin.Context) { c.String(200, "test") })
	}
}

func main() {
	g := gin.Default()
	InitApiRouter(g)
	_ = g.Run("127.0.0.1:8000")
}
```









