# 1.Gin基础

## 入门Demo

```go

func main() {
	r := gin.Default() // 获取默认配置gin
	r.GET("/ping", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "pong",
		})
	})
	r.Run() // listen and serve on 0.0.0.0:8080 (for windows "localhost:8080")
}

// curl --location --request GET '127.0.0.1:8080/ping'
```

## 请求方法

```go
func main() {
	// 使用默认中间件创建一个gin路由器
	// logger and recovery (crash-free) 中间件
	router := gin.Default()

	router.GET("/some", func(c *gin.Context) {
		c.JSON(200,gin.H{
			"method":"GET",
		})
	})
	router.POST("/some", func(c *gin.Context) {
		c.JSON(200,gin.H{
			"method":"POST",
		})
	})
	router.PUT("/some", func(c *gin.Context) {
		c.JSON(200,gin.H{
			"method":"PUT",
		})
	})
	router.DELETE("/some", func(c *gin.Context) {
		c.JSON(200,gin.H{
			"method":"DELETE",
		})
	})
	router.PATCH("/some", func(c *gin.Context) {
		c.JSON(200,gin.H{
			"method":"PATCH",
		})
	})
	router.HEAD("/some", func(c *gin.Context) {
		c.JSON(200,gin.H{
			"method":"HEAD",
		})
	})
	router.OPTIONS("/some", func(c *gin.Context) {
		c.JSON(200,gin.H{
			"method":"OPTIONS",
		})
	})

	// 默认启动的是 8080端口，也可以自己定义启动端口
	router.Run()
	// router.Run(":3000") for a hard coded port
}

// curl --location --request GET '127.0.0.1:8080/some'
// curl --location --request POST '127.0.0.1:8080/some'
// curl --location --request PUT '127.0.0.1:8080/some'
// curl --location --request DELETE '127.0.0.1:8080/some'
// curl --location --request PATCH '127.0.0.1:8080/some'
// curl --location --request HEAD '127.0.0.1:8080/some'
// curl --location --request OPTIONS '127.0.0.1:8080/some'
```

## 参数绑定

### 路径参数

```go
func main() {
	router := gin.Default()

	// This handler will match /user/john but will not match /user/ or /user
	router.GET("/user/:name", func(c *gin.Context) {
		name := c.Param("name")
		c.String(http.StatusOK, "Hello %s", name)
	})

	// However, this one will match /user/john/ and also /user/john/send
	// If no other routers match /user/john, it will redirect to /user/john/
	router.GET("/user/:name/*action", func(c *gin.Context) {
		name := c.Param("name")
		action := c.Param("action")
		message := name + " is " + action
		c.String(http.StatusOK, message)
	})

	// For each matched request Context will hold the route definition
	router.POST("/user/:name/*action", func(c *gin.Context) {
		b := c.FullPath() == "/user/:name/*action" // true
		c.String(http.StatusOK, "%t", b)
	})

	// This handler will add a new router for /user/groups.
	// Exact routes are resolved before param routes, regardless of the order they were defined.
	// Routes starting with /user/groups are never interpreted as /user/:name/... routes
	router.GET("/user/groups", func(c *gin.Context) {
		c.String(http.StatusOK, "The available groups are [...]")
	})

	router.Run(":8080")
}

// curl --location --request GET '127.0.0.1:8080/user/zcw'
// curl --location --request GET '127.0.0.1:8080/user/zcw/bac'
// curl --location --request POST '127.0.0.1:8080/user/zcw/bac'
```

### Get参数

```go
func main() {
	router := gin.Default()

	// Query string parameters are parsed using the existing underlying request object.
	// The request responds to a url matching:  /welcome?firstname=Jane&lastname=Doe
	router.GET("/welcome", func(c *gin.Context) {
		firstname := c.DefaultQuery("firstname", "Guest")
		lastname := c.Query("lastname") // shortcut for c.Request.URL.Query().Get("lastname")

		c.String(http.StatusOK, "Hello %s %s", firstname, lastname)
	})
	router.Run(":8080")
}

// curl --location --request GET '127.0.0.1:8080/welcome?firstname=Jane&lastname=Doe'
```

### Post参数

```go
func main() {
	router := gin.Default()

	router.POST("/form_post", func(c *gin.Context) {
		message := c.PostForm("message")
		nick := c.DefaultPostForm("nick", "anonymous")

		c.JSON(200, gin.H{
			"status":  "posted",
			"message": message,
			"nick":    nick,
		})
	})
	router.Run(":8080")
}

/* 
curl --location --request POST '127.0.0.1:8080/form_post' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'message=123' \
--data-urlencode 'nick=zcw' 
*/
```

### Get+Post参数

```go
func main() {
	router := gin.Default()

	router.POST("/post", func(c *gin.Context) {

		id := c.Query("id")
		page := c.DefaultQuery("page", "0")
		name := c.PostForm("name")
		message := c.PostForm("message")

		c.JSON(200, gin.H{
			"id":      id,
			"page":    page,
			"message": message,
			"nick":    name,
		})
	})
	router.Run(":8080")
}
/*
curl --location --request POST '127.0.0.1:8080/post?id=1&page=10' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'message=123' \
--data-urlencode 'nick=zcw'
*/
```

- 参数校验
- 文件上传
- 响应格式
- Router分组
- 中间件
- 认证

# 2.项目

- 登录
- 增删改查
- 配置文件
- Redis对接
- 开源项目阅读

# 3.走进源码

- net/http
- gin全局设计
- 实现细节



