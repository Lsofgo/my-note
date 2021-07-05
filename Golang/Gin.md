[toc]

```shell
go get github.com/gin-gonic/gin
```

## Gin 简单使用

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
	g.LoadHTMLGlob("templates/**/*")					// 指定template路径
    // g.LoadHTMLFiles("templates/users/index.html")	// 指定template文件
	//......
}
```



### HTML 渲染

> templates/users/index.html

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
// 使用结构体渲染

```



