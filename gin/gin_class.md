# 1.Gin基础

## 入门Demo

```go
import "github.com/gin-gonic/gin"

// Ch1
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
// Ch2
// https://www.runoob.com/http/http-methods.html
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

## 响应格式

```go
// Ch2-1
func main() {
	r := gin.Default()

	// gin.H is a shortcut for map[string]interface{}
	r.GET("/someJSON", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{"message": "hey", "status": http.StatusOK})
	})

	r.GET("/moreJSON", func(c *gin.Context) {
		// You also can use a struct
		var msg struct {
			Name    string `json:"user"`
			Message string
			Number  int
		}
		msg.Name = "Lena"
		msg.Message = "hey"
		msg.Number = 123
		// Note that msg.Name becomes "user" in the JSON
		// Will output  :   {"user": "Lena", "Message": "hey", "Number": 123}
		c.JSON(http.StatusOK, msg)
	})

	r.GET("/someXML", func(c *gin.Context) {
		c.XML(http.StatusOK, gin.H{"message": "hey", "status": http.StatusOK})
	})

	r.GET("/someYAML", func(c *gin.Context) {
		c.YAML(http.StatusOK, gin.H{"message": "hey", "status": http.StatusOK})
	})

	r.GET("/someProtoBuf", func(c *gin.Context) {
		reps := []int64{int64(1), int64(2)}
		label := "test"
		// The specific definition of protobuf is written in the testdata/protoexample file.
		data := &protoexample.Test{
			Label: &label,
			Reps:  reps,
		}
		// Note that data becomes binary data in the response
		// Will output protoexample.Test protobuf serialized data
		c.ProtoBuf(http.StatusOK, data)
	})

	// Listen and serve on 0.0.0.0:8080
	r.Run(":8080")
}
```

## 参数绑定

### 路径参数

```go
// Ch3
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

### Bind Uri

```go
// Ch3-1

type Person struct {
	ID string `uri:"id" binding:"required,uuid"`
	Name string `uri:"name" binding:"required"`
}

func main() {
	route := gin.Default()
	route.GET("/:name/:id", func(c *gin.Context) {
		var person Person
		if err := c.ShouldBindUri(&person); err != nil {
			c.JSON(400, gin.H{"msg": err.Error()})
			return
		}
		c.JSON(200, gin.H{"name": person.Name, "uuid": person.ID})
	})
	route.Run(":8080")
}

// curl --location --request GET '127.0.0.1:8080/thinkerou/987fbc97-4bed-5078-9f07-9141ba07c9f3'
```

### Get参数

```go
// Ch4
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
// Ch5
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
// Ch6
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

## Struct绑定

```go
// Ch6-1

// Login Binding from JSON
// form，json，xml，header， 
type Login struct {
	User     string `form:"user" json:"user" binding:"required"`
	Password string `form:"password" json:"password" binding:"required"`
}

func main() {
	router := gin.Default()

	// Example for binding JSON ({"user": "manu", "password": "123"})
	router.POST("/loginJSON", func(c *gin.Context) {
		var json Login
		if err := c.ShouldBindJSON(&json); err != nil {
			c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
			return
		}

		if json.User != "manu" || json.Password != "123" {
			c.JSON(http.StatusUnauthorized, gin.H{"status": "unauthorized"})
			return
		}

		c.JSON(http.StatusOK, gin.H{"status": "you are logged in"})
	})


	// Example for binding a HTML form (user=manu&password=123)
	router.POST("/loginForm", func(c *gin.Context) {
		var form Login
		// This will infer what binder to use depending on the content-type header.
		if err := c.ShouldBind(&form); err != nil {
			c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
			return
		}

		if form.User != "manu" || form.Password != "123" {
			c.JSON(http.StatusUnauthorized, gin.H{"status": "unauthorized"})
			return
		}

		c.JSON(http.StatusOK, gin.H{"status": "you are logged in"})
	})
    
    // function only binds the query params and not the post data. (user=manu&password=123)
	router.POST("/loginQuery", func(c *gin.Context) {
		var form Login
		// This will infer what binder to use depending on the content-type header.
		if err := c.ShouldBindQuery(&form); err != nil {
			c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
			return
		}

		if form.User != "manu" || form.Password != "123" {
			c.JSON(http.StatusUnauthorized, gin.H{"status": "unauthorized"})
			return
		}

		c.JSON(http.StatusOK, gin.H{"status": "you are logged in"})
	})

	// Listen and serve on 0.0.0.0:8080
	router.Run(":8080")
}

/*
curl --location --request POST '127.0.0.1:8080/loginJSON' \
--header 'Content-Type: application/json' \
--data-raw '{
    "user":"manu",
    "password":"123"
}'

curl --location --request POST '127.0.0.1:8080/loginForm' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'user=manu' \
--data-urlencode 'password=123'

curl --location --request POST '127.0.0.1:8080/loginQuery?user=manu&password=123'
*/
```

```go
// Ch6-2

func main() {
	router := gin.Default()

	// Example for binding JSON ({"user": "manu", "password": "123"})
	router.POST("/test", func(c *gin.Context) {
		var fakeForm myForm
		c.ShouldBind(&fakeForm)
		c.JSON(200, gin.H{"color": fakeForm.Colors})
	})

	// Listen and serve on 0.0.0.0:8080
	router.Run(":8080")
}

/*
curl --location --request POST '127.0.0.1:8080/test' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'colors[]=red' \
--data-urlencode 'colors[]=green' \
--data-urlencode 'colors[]=blue'
*/
```



## 参数校验

```go
// Ch6-3

type Booking struct {
	// 包含绑定和验证的数据,bookabledate就是自定义的验证函数
	CheckIn time.Time `form:"check_in" binding:"required,bookabledate" time_format:"2006-01-02"`
	// gtfield=CheckIn只对数字或时间有效，参考官网链接
	// https://pkg.go.dev/github.com/go-playground/validator/v10#section-readme
	CheckOut time.Time `form:"check_out" binding:"required,gtfield=CheckIn,bookabledate" time_format:"2006-01-02"`
}

// 自定义验证器
var bookableDate validator.Func = func(fl validator.FieldLevel) bool {
	if date, ok := fl.Field().Interface().(time.Time); ok {
		today := time.Now()
		if today.After(date) {  // 输入的日期必须大于今天的日期，否则验证失败
			return false
		}
	}
	return true
}

func main() {
	router := gin.Default()

	// 注册验证器
	validate, ok := binding.Validator.Engine().(*validator.Validate)
	if ok {
		validate.RegisterValidation("bookabledate", bookableDate)
	}
	// http://127.0.0.1:8085/bookable?check_in=2022-01-07&check_out=2022-01-08
	// https://frankhitman.github.io/zh-CN/gin-validator/
	router.GET("/bookable", getBookable)
	router.Run()
}

func getBookable(context *gin.Context) {
	var book Booking
	if err := context.ShouldBindWith(&book, binding.Query); err == nil {
		context.JSON(200, gin.H{"message": "book date is valid"})
	} else {
		context.JSON(http.StatusBadRequest, gin.H{"err": err.Error()})
	}
}

// check_in=2022-01-11&check_out=2022-01-12 (输入的日期必须大于今天的日期，否则验证失败)
// curl --location --request GET 'http://127.0.0.1:8080/bookable?check_in=2022-01-11&check_out=2022-01-12'
```

## 文件上传

```go
// Ch7
func main() {
	router := gin.Default()
	// Set a lower memory limit for multipart forms (default is 32 MiB)
	router.MaxMultipartMemory = 8 << 20  // 8 MiB
	router.POST("/upload", func(c *gin.Context) {
		// Single file
		file, _ := c.FormFile("Filename")
		log.Println(file.Filename)

		// Upload the file to specific dst.
		c.SaveUploadedFile(file, "./"+file.Filename)

		c.String(http.StatusOK, fmt.Sprintf("'%s' uploaded!", file.Filename))
	})
	router.Run(":8080")
}
/*
curl --location --request POST '127.0.0.1:8080/upload' \
--form 'Filename=@"/Users/Pictures/psc.jpg"'
*/
```

## Router分组

```go
// Ch8
func main() {
	router := gin.Default()

	// Simple group: v1
	v1 := router.Group("/v1")
	{
		v1.POST("/login", func(c *gin.Context) {
			c.String(http.StatusOK, "v1 login")
		})
		v1.POST("/submit", func(c *gin.Context) {
			c.String(http.StatusOK, "v1 submit")
		})
		v1.POST("/read", func(c *gin.Context) {
			c.String(http.StatusOK, "v1 read")
		})
	}

	// Simple group: v2
	v2 := router.Group("/v2")
	{
		v2.POST("/login", func(c *gin.Context) {
			c.String(http.StatusOK, "v2 login")
		})
		v2.POST("/submit", func(c *gin.Context) {
			c.String(http.StatusOK, "v2 submit")
		})
		v2.POST("/read", func(c *gin.Context) {
			c.String(http.StatusOK, "v2 read")
		})
	}

	router.Run(":8080")
}
// curl --location --request POST '127.0.0.1:8080/v1/login'
// curl --location --request POST '127.0.0.1:8080/v2/login'
```

## 中间件

```go
// Ch9
func main() {
	// Creates a router without any middleware by default
	r := gin.New()

	// Global middleware
	// Logger middleware will write the logs to gin.DefaultWriter even if you set with GIN_MODE=release.
	// By default gin.DefaultWriter = os.Stdout
	r.Use(gin.Logger())

	// Recovery middleware recovers from any panics and writes a 500 if there was one.
	r.Use(gin.Recovery())

	r.Use(func(c *gin.Context) {
		fmt.Println("I am middleware")
		c.Set("flag","I am middleware")
	})

	//r.Use(func(c *gin.Context) {
	//	fmt.Println("I am middleware")
	//	c.Next()	// 错误示例
	//	c.Set("flag","I am middleware")
	//})

	//r.Use(func(c *gin.Context) {
	//	fmt.Println("begin")
	//	c.Next()
	//	fmt.Println("end")
	//})

	// Per route middleware, you can add as many as you desire.
	r.GET("/middleware", func(c *gin.Context) {
		c.String(http.StatusOK, c.Keys["flag"].(string))
	})

	//r.Use(func(c *gin.Context) {
	//	fmt.Println("I am middleware")
	//})

	// Listen and serve on 0.0.0.0:8080
	r.Run(":8080")
}

// curl --location --request GET '127.0.0.1:8080/middleware'
-     -
 -   -
   -
```

## 写日志到文件

```go
// Ch10
func main() {
	// Disable Console Color, you don't need console color when writing the logs to file.
	gin.DisableConsoleColor()

	// Logging to a file.
	f, _ := os.Create("gin.log")
	gin.DefaultWriter = io.MultiWriter(f)

	// Use the following code if you need to write the logs to file and console at the same time.
	// gin.DefaultWriter = io.MultiWriter(f, os.Stdout)

	router := gin.Default()
	router.GET("/ping", func(c *gin.Context) {
		c.String(200, "pong")
	})

	router.Run(":8080")
}
```

## BasicAuth认证

```go
// simulate some private data
var secrets = gin.H{
	"foo":    gin.H{"email": "foo@bar.com", "phone": "123433"},
	"austin": gin.H{"email": "austin@example.com", "phone": "666"},
	"lena":   gin.H{"email": "lena@guapa.com", "phone": "523443"},
}

func main() {
	r := gin.Default()

	// Group using gin.BasicAuth() middleware
	// gin.Accounts is a shortcut for map[string]string
	authorized := r.Group("/admin", gin.BasicAuth(gin.Accounts{
		"foo":    "bar",
		"austin": "1234",
		"lena":   "hello2",
		"manu":   "4321",
	}))

	// /admin/secrets endpoint
	// hit "localhost:8080/admin/secrets
	authorized.GET("/secrets", func(c *gin.Context) {
		// get user, it was set by the BasicAuth middleware
		user := c.MustGet(gin.AuthUserKey).(string)
		if secret, ok := secrets[user]; ok {
			c.JSON(http.StatusOK, gin.H{"user": user, "secret": secret})
		} else {
			c.JSON(http.StatusOK, gin.H{"user": user, "secret": "NO SECRET :("})
		}
	})

	// Listen and serve on 0.0.0.0:8080
	r.Run(":8080")
}
```

https://github.com/topics/epoll?l=go

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



