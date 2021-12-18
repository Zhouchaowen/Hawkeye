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





































