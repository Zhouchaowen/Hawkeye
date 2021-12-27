# Gin 0.x

- NoRoute

```go
func (engine *Engine) NoRoute(handlers ...HandlerFunc)
	func (engine *Engine) rebuild404Handlers() 
		func (group *RouterGroup) combineHandlers(handlers []HandlerFunc) []HandlerFunc
```



- Use

```go
func (engine *Engine) Use(middlewares ...HandlerFunc)
	func (group *RouterGroup) Use(middlewares ...HandlerFunc) 
	func (engine *Engine) rebuild404Handlers()
		func (group *RouterGroup) combineHandlers(handlers []HandlerFunc) []HandlerFunc
	func (engine *Engine) rebuild405Handlers()
		func (group *RouterGroup) combineHandlers(handlers []HandlerFunc) []HandlerFunc
```

```go
// 流程就是个栈流程
-      -  先进后出，中间出错就直接从当前HandlerFunc 一次弹栈
 -    -
  -  -
   -
```



- Logger

```go
func Logger() HandlerFunc
	func LoggerWithFile(out io.Writer) HandlerFunc
```



- AbortWithStatus

```go
func (c *Context) AbortWithStatus(code int)
	func (c *Context) Abort()
```



- Fail

```go
func (c *Context) Fail(code int, err error)
	func (c *Context) Error(err error, meta interface{})
		func (c *Context) ErrorTyped(err error, typ int, meta interface{})
	func (c *Context) AbortWithStatus(code int)
		func (c *Context) Abort()
```



- RecoveryWithFile

```go
func RecoveryWithFile(out io.Writer) HandlerFunc
	func stack(skip int) []byte
	func (c *Context) AbortWithStatus(code int)
		func (c *Context) Abort()
	func (c *Context) Next()
```



- **responseWriter** 

```go
responseWriter struct {
  http.ResponseWriter
  size   int
  status int
}


ResponseWriter interface {
  http.ResponseWriter
  http.Hijacker
  http.Flusher
  http.CloseNotifier

  Status() int
  Size() int
  Written() bool
  WriteHeaderNow()
}
```



- **Group**

```go
func New() *Engine
func (group *RouterGroup) Group(relativePath string, handlers ...HandlerFunc) *RouterGroup
	func (group *RouterGroup) combineHandlers(handlers []HandlerFunc) []HandlerFunc
	func (group *RouterGroup) calculateAbsolutePath(relativePath string)
		func joinPaths(absolutePath, relativePath string) string
			func lastChar(str string) uint8
```

```go
func New() *Engine {
	engine := &Engine{
		RouterGroup: RouterGroup{
			Handlers:     nil,
			absolutePath: "/",
		},
		RedirectTrailingSlash:  true,
		RedirectFixedPath:      true,
		HandleMethodNotAllowed: true,
		trees: make(map[string]*node),
	}
	engine.RouterGroup.engine = engine
	engine.pool.New = func() interface{} {
		return engine.allocateContext()
	}
	return engine
}

// Engine 套 RouterGroup
// RouterGroup 保存最初 Engine

// RouterGroup 套 RouterGroup 继承上一次的 absolutePath
type RouterGroup struct {
	Handlers     []HandlerFunc
	absolutePath string
	engine       *Engine
}
```



- GET

```go
func (group *RouterGroup) POST(relativePath string, handlers ...HandlerFunc)
	func (group *RouterGroup) Handle(httpMethod, relativePath string, handlers []HandlerFunc)
		// 计算路径
		func (group *RouterGroup) calculateAbsolutePath(relativePath string) string
			func joinPaths(absolutePath, relativePath string) string
		// 合并handlerFunc
		func (group *RouterGroup) combineHandlers(handlers []HandlerFunc) []HandlerFunc
		// 设置认证级别
		func debugRoute(httpMethod, absolutePath string, handlers []HandlerFunc)
		func (engine *Engine) handle(method, path string, handlers []HandlerFunc)
			func (n *node) addRoute(path string, handlers []HandlerFunc)
```



- processAccounts

```go
Accounts map[string]string
authPair struct {
  Value string
  User  string
}

func processAccounts(accounts Accounts) authPairs
	func authorizationHeader(user, password string) string
```



- BasicAuth 认证中间件

```go
func BasicAuth(accounts Accounts) HandlerFunc
  func BasicAuthForRealm(accounts Accounts, realm string) HandlerFunc
    func processAccounts(accounts Accounts) authPairs
      func authorizationHeader(user, password string) string
    func (a authPairs) searchCredential(auth string) (string, bool)
```



- **CleanPath**

```go
func CleanPath(p string) string 
```



- **ServeHTTP**

  - 拿到Context
  - 设置Context的参数，writer，req
  - 通过请求方式拿，找到给定HTTP方法的树的根
    - 获取参数 handlers, params, tsr := root.getValue(path, context.Params)
      - 调用 HandlerFunc
    - 获取不到 handlers
      - 尝试走重定向(修正url，去除多余 / . 或者 ../ ./  大小写转换)
      - 尝试不成功，就走 自定义NoRaute，NoMethod或serveError(context, 404, default404Body)
  - 找不到给定HTTP方法的树的根
    - 走默认func serveError(c *Context, code int, defaultMessage []byte)
    - 或走自定义 func (engine *Engine) NoMethod(handlers ...HandlerFunc)

  - 返回，将Context放回pool **(就是想复用一个engine，避免多次创建)**

```go
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) 
	func (engine *Engine) serveHTTPRequest(context *Context)
		func (n *node) getValue(path string, po Params) (h []HandlerFunc, p Params, tsr bool)
		if handlers != nil {context.Next()}
	
		if engine.HandleMethodNotAllowed 
			func serveError(c *Context, code int, defaultMessage []byte)
	engine.pool.Put(context)
```



- Copy 

```go
func (c *Context) Copy() *Context {
	var cp Context = *c
	cp.writermem.ResponseWriter = nil
	cp.Writer = &cp.writermem
	cp.index = AbortIndex
	cp.handlers = nil
	return &cp
}
```



- paramValue,formValue,postFormValue

```go
func (c *Context) paramValue(key string) (string, bool)
	func (ps Params) ByName(name string) string

func (c *Context) formValue(key string) (string, bool)
func (c *Context) postFormValue(key string) (string, bool)
	func (r *Request) ParseForm() error 
```



- Bind

```go
func (c *Context) Bind(obj interface{}) bool
	func Default(method, contentType string) Binding
	func (c *Context) BindWith(obj interface{}, b binding.Binding) bool
		type Binding interface {
      Name() string
      Bind(*http.Request, interface{}) error
    }
			func (v *Validator) ValidateStruct(s interface{}) *StructValidationErrors 
```





- Engine 就是一个gin框架
  - 包含RouterGroup，根路由的url和HandlerFunc
  - 包含所有allNoRoute，allNoMethod，没有路由或方法的处理策略(HandlerFunc)
  - 包含trees，快速查找路由的结构

```go
// Represents the web framework, it wraps the blazing fast httprouter multiplexer and a list of global middlewares.
	Engine struct {
		RouterGroup
		HTMLRender  render.Render
		pool        sync.Pool
		allNoRoute  []HandlerFunc
		allNoMethod []HandlerFunc
		noRoute     []HandlerFunc
		noMethod    []HandlerFunc
		trees       map[string]*node

		// Enables automatic redirection if the current route can't be matched but a
		// handler for the path with (without) the trailing slash exists.
		// For example if /foo/ is requested but a route only exists for /foo, the
		// client is redirected to /foo with http status code 301 for GET requests
		// and 307 for all other request methods.
		RedirectTrailingSlash bool

		// If enabled, the router tries to fix the current request path, if no
		// handle is registered for it.
		// First superfluous path elements like ../ or // are removed.
		// Afterwards the router does a case-insensitive lookup of the cleaned path.
		// If a handle can be found for this route, the router makes a redirection
		// to the corrected path with status code 301 for GET requests and 307 for
		// all other request methods.
		// For example /FOO and /..//Foo could be redirected to /foo.
		// RedirectTrailingSlash is independent of this option.
		RedirectFixedPath bool

		// If enabled, the router checks if another method is allowed for the
		// current route, if the current request can not be routed.
		// If this is the case, the request is answered with 'Method Not Allowed'
		// and HTTP status code 405.
		// If no other Method is allowed, the request is delegated to the NotFound
		// handler.
		HandleMethodNotAllowed bool
	}
```



- Context
  - 上下文，可以在所有HandlerFunc中传输，封装了http.ResponseWriter，http.Request方便操作读写
  - 保存请求参数Params
  - 存储当前状态用于的HandlerFunc
- **就是对读写请求的封装，解析请求参数，终止运行HandlerFunc，保存调用链中间错误信息，封装一下便捷操作，存储一些每个HandlerFunc运行必要的参数**

```go
// Context is the most important part of gin. It allows us to pass variables between middleware,
// manage the flow, validate the JSON of a request and render a JSON response for example.
type Context struct {
	context.Context
	writermem responseWriter
	Request   *http.Request
	Writer    ResponseWriter

	Params   Params
	handlers []HandlerFunc
	index    int8

	Engine   *Engine
	Keys     map[string]interface{}
	Errors   errorMsgs
	Accepted []string
}
```



# Gin 1.3

- Auth
  - 注册Auth中间件 BasicAuth
  - 注册正常逻辑
  - 请求带上：Authorization的header头
  - Realm 

```go
func TestBasicAuthSucceed(t *testing.T) {
	accounts := Accounts{"admin": "password"}
	router := New()
	router.Use(BasicAuth(accounts))
	router.GET("/login", func(c *Context) {
		c.String(http.StatusOK, c.MustGet(AuthUserKey).(string))
	})

	w := httptest.NewRecorder()
	req, _ := http.NewRequest("GET", "/login", nil)
	req.Header.Set("Authorization", authorizationHeader("admin", "password"))
	router.ServeHTTP(w, req)

	assert.Equal(t, http.StatusOK, w.Code)
	assert.Equal(t, "admin", w.Body.String())
}

```

## binding

- binding定义，各种类型的抽象

```go
type Binding interface {
	Name() string
	Bind(*http.Request, interface{}) error
}
```

- 验证器的抽象

```go
type StructValidator interface {
  .....
}
```

- 绑定器实体

```go
var (
	JSON          = jsonBinding{}
	XML           = xmlBinding{}
	Form          = formBinding{}
	Query         = queryBinding{}
	FormPost      = formPostBinding{}
	FormMultipart = formMultipartBinding{}
	ProtoBuf      = protobufBinding{}
	MsgPack       = msgpackBinding{}
)
```

## Render

- Render提供不同类型的返回：
  - application/json; charset=utf-8
  - application/javascript; charset=utf-8
  - application/json
  - text/html; charset=utf-8
  - .......

```go
// Render interface is to be implemented by JSON, XML, HTML, YAML and so on.
type Render interface {
	// Render writes data with custom ContentType.
	Render(http.ResponseWriter) error
	// WriteContentType writes custom ContentType.
	WriteContentType(w http.ResponseWriter)
}
```

```go
var (
	_ Render     = JSON{}
	_ Render     = IndentedJSON{}
	_ Render     = SecureJSON{}
	_ Render     = JsonpJSON{}
	_ Render     = XML{}
	_ Render     = String{}
	_ Render     = Redirect{}
	_ Render     = Data{}
	_ Render     = HTML{}
	_ HTMLRender = HTMLDebug{}
	_ HTMLRender = HTMLProduction{}
	_ Render     = YAML{}
	_ Render     = MsgPack{}
	_ Render     = Reader{}
	_ Render     = AsciiJSON{}
	_ Render     = ProtoBuf{}
)
```

- 重定向

```go
// Render (Redirect) redirects the http request to new location and writes redirect response.
func (r Redirect) Render(w http.ResponseWriter) error {
	// todo(thinkerou): go1.6 not support StatusPermanentRedirect(308)
	// when we upgrade go version we can use http.StatusPermanentRedirect
	if (r.Code < 300 || r.Code > 308) && r.Code != 201 {
		panic(fmt.Sprintf("Cannot redirect with status code %d", r.Code))
	}
	http.Redirect(w, r.Request, r.Location, r.Code)
	return nil
}
```





## authPair

- 认证中间件

```go
func BasicAuth(accounts Accounts) HandlerFunc
	func BasicAuthForRealm(accounts Accounts, realm string) HandlerFunc 
```

- 编码

```go
func authorizationHeader(user, password string) string {
	base := user + ":" + password
	return "Basic " + base64.StdEncoding.EncodeToString([]byte(base))
}
```



## Context

- 在中间件之间传递变量，管理流，验证JSON的请求并呈现JSON例如响应

```go
type Context struct {
  // 封装 http.ResponseWriter
	writermem responseWriter
	Request   *http.Request
	Writer    ResponseWriter

  // 保存url参数的切片
	Params   Params
	handlers HandlersChain
	index    int8

	engine *Engine

	// Keys 专用于每个请求上下文的键/值对。
	Keys map[string]interface{}

	// Errors 是附加到所有使用此上下文的处理程序/中间件的错误列表。
	Errors errorMsgs

	// Accepted 定义了手动接受的内容协商格式列表。
	Accepted []string
}
```

- reset重置Context

```go
func (c *Context) reset() {
	c.Writer = &c.writermem
	c.Params = c.Params[0:0]
	c.handlers = nil
	c.index = -1
	c.Keys = nil
	c.Errors = c.Errors[0:0]
	c.Accepted = nil
}
```

- Copy 返回可以在请求范围之外安全使用的当前上下文的副本。 当必须将上下文传递给 goroutine 时，必须使用它

```go
func (c *Context) Copy() *Context {
	var cp = *c
	cp.writermem.ResponseWriter = nil
	cp.Writer = &cp.writermem
	cp.index = abortIndex
	cp.handlers = nil
	return &cp
}
```

- Next 应该只在中间件内部使用。 它在调用处理程序内执行链中的挂起处理程序。压栈弹栈过程。

```go
func (c *Context) Next() {
	c.index++
	for s := int8(len(c.handlers)); c.index < s; c.index++ {
		c.handlers[c.index](c)
	}
}

// 终止方法
func (c *Context) Abort() {
	c.index = abortIndex
}
```

- Query, PostForm, File, Multipart函数。

- Bind函数(解析参数到结构体等的函数)。



## gin

**框架核心**

```go
type Engine struct {
	RouterGroup

	// 启用自动重定向，如果请求 /foo/ 但只有 /foo 的路由存在，则客户端将重定向到 /foo
	RedirectTrailingSlash bool

	// 修复当前请求路径
	RedirectFixedPath bool

	// 检查当前路由是否允许使用另一种方法
	HandleMethodNotAllowed bool
	ForwardedByClientIP    bool

	// #726 #755 它将推送一些以“X-AppEngine ...”开头的标题，以便更好地与该 PaaS 集成。
	AppEngine bool

	// If enabled, the url.RawPath will be used to find parameters.
	UseRawPath bool

	// If true, the path value will be unescaped.
	// If UseRawPath is false (by default), the UnescapePathValues effectively is true,
	// as url.Path gonna be used, which is already unescaped.
	UnescapePathValues bool

	// Value of 'maxMemory' param that is given to http.Request's ParseMultipartForm
	// method call.
	MaxMultipartMemory int64

	delims           render.Delims
	secureJsonPrefix string
	HTMLRender       render.HTMLRender
	FuncMap          template.FuncMap
	allNoRoute       HandlersChain
	allNoMethod      HandlersChain
	noRoute          HandlersChain
	noMethod         HandlersChain
  // 提升性能，保存Context
	pool             sync.Pool
  // 存储和匹配url的结构
	trees            methodTrees
}
```

- Use 添加中间件

```go
func (engine *Engine) Use(middleware ...HandlerFunc) IRoutes {
	engine.RouterGroup.Use(middleware...)
	engine.rebuild404Handlers()
	engine.rebuild405Handlers()
	return engine
}
```

- addRoute 添加Handler到Tree结构

```go
func (engine *Engine) addRoute(method, path string, handlers HandlersChain)
```

- Run 启动http服务

```go
func (engine *Engine) Run(addr ...string) (err error) {
	defer func() { debugPrintError(err) }()

	address := resolveAddress(addr)
	debugPrint("Listening and serving HTTP on %s\n", address)
	err = http.ListenAndServe(address, engine)
	return
}
```

- **ServeHTTP** 实现httpserver接口，接收处理请求。
  - 拿到Context
  - 设置Context的参数，writer，req
  - 重置 Params，handlers，index等参数
  - 通过请求方式拿，找到给定HTTP方法的树的根 ServeHTTP
    - 循环遍历获取对应method的root
    - 获取参数 handlers, params, tsr := root.getValue(path, context.Params)
      - 通过Next循环调用查找到的 HandlerFunc
    - 获取不到 handlers
      - 尝试走重定向(修正url，去除多余 / . 或者 ../ ./  大小写转换)
      - 尝试不成功，就走 自定义NoRaute，NoMethod或serveError(context, 404, default404Body)
  - 找不到Http对应method的根
    - 走默认func serveError(c *Context, code int, defaultMessage []byte)
    - 或走自定义 func (engine *Engine) NoMethod(handlers ...HandlerFunc)
  - 返回，将Context放回pool **(就是想复用一个engine，避免多次创建)**

```go
func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	c := engine.pool.Get().(*Context)
	c.writermem.reset(w)
	c.Request = req
	c.reset()

	engine.handleHTTPRequest(c)

	engine.pool.Put(c)
}
```



## RouterGroup

- 每组路由struct定义

```go
// RouterGroup is used internally to configure router, a RouterGroup is associated with
// a prefix and an array of handlers (middleware).
type RouterGroup struct {
	Handlers HandlersChain
	basePath string
	engine   *Engine
	root     bool
}
```

- Use

```go
// Use adds middleware to the group, see example code in github.
func (group *RouterGroup) Use(middleware ...HandlerFunc) IRoutes {
	group.Handlers = append(group.Handlers, middleware...)
	return group.returnObj()
}
```

- Group

```go
// Group creates a new router group. You should add all the routes that have common middlwares or the same path prefix.
// For example, all the routes that use a common middlware for authorization could be grouped.
func (group *RouterGroup) Group(relativePath string, handlers ...HandlerFunc) *RouterGroup {
	return &RouterGroup{
		Handlers: group.combineHandlers(handlers),
		basePath: group.calculateAbsolutePath(relativePath),
		engine:   group.engine,
	}
}
```



# GIN1.7

企业级：模块依赖，服务发现，配置管理



- httprouter

- chi
  - Router
  - Middleware
  - Context
- gin
- echo
- fiber
  - fasthttp
- beego
  - Router
  - Filter(middleware)
  - Context
  - Task
  - Orm,httpclient,Cache,validator,config,swagger,template



微服务框架组件

Config：配置管理组件

Logger：遵守第三方日志收集规范的日志组件。必须统一

Metrics：使框架能够与Prometheus 等监控系统集成的metrics组件

Tracing：遵守OpenTelemetry的tracing组件

Registry：服务发现组件

MQ：可以切换不同队列实现的mq组件

依赖注入：wire, dig 等组件



goframe



Go-Micro

go-zero

yoyogo

Dubbo-go

Kratos











