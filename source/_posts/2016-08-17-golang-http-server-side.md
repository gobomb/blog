---
layout: post
title: "go http 服务器编程"
excerpt: "net/http 封装了 golang http 编程的很多功能，我们只需要用很少的代码就能实现很常用的功能。"
categories: blog
tags: [golang, http, net, server]
comments: true
share: true
---


## 1. 初识

http 是典型的 C/S 架构，客户端向服务端发送请求（request），服务端做出应答（response）。

golang 的标准库 `net/http` 提供了 http 编程有关的接口，封装了内部TCP连接和报文解析的复杂琐碎的细节，使用者只需要和 `http.request` 和 `http.ResponseWriter` 两个对象交互就行。也就是说，我们只要写一个 handler，请求会通过参数传递进来，而它要做的就是根据请求的数据做处理，把结果写到 Response 中。废话不多说，来看看 hello world 程序有多简单吧！

```go
package main

import (
	"io"
	"net/http"
)

type helloHandler struct{}

func (h *helloHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	w.Write([]byte("Hello, world!"))
}

func main() {
	http.Handle("/", &helloHandler{})
	http.ListenAndServe(":12345", nil)
}
```

运行 `go run hello_server.go`，我们的服务器就会监听在本地的 `12345` 端口，对所有的请求都会返回 `hello, world!`：

![](http://ww4.sinaimg.cn/large/006tKfTcgw1f6uoaz8reuj30ne0hrq56.jpg)

正如上面程序展示的那样，我们只要实现的一个 Handler，它的[接口原型](https://golang.org/pkg/net/http/#Handler)是（也就是说只要实现了 `ServeHTTP` 方法的对象都可以作为 Handler）：

```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

然后，注册到对应的路由路径上就 OK 了。

`http.HandleFunc`接受两个参数：第一个参数是字符串表示的 url 路径，第二个参数是该 url 实际的处理对象。

`http.ListenAndServe` 监听在某个端口，启动服务，准备接受客户端的请求（第二个参数这里设置为 `nil`，这里也不要纠结什么意思，后面会有讲解）。每次客户端有请求的时候，把请求封装成 `http.Request`，调用对应的 handler 的 `ServeHTTP` 方法，然后把操作后的 `http.ResponseWriter` 解析，返回到客户端。

## 2. 封装

上面的代码没有什么问题，但是有一个不便：每次写 Handler 的时候，都要定义一个类型，然后编写对应的 `ServeHTTP` 方法，这个步骤对于所有 Handler 都是一样的。重复的工作总是可以抽象出来，`net/http` 也正这么做了，它提供了 `http.HandleFunc` 方法，允许直接把特定类型的函数作为 handler。上面的代码可以改成：

```go
package main

import (
	"io"
	"net/http"
)

func helloHandler(w http.ResponseWriter, req *http.Request) {
	io.WriteString(w, "hello, world!\n")
}

func main() {
	http.HandleFunc("/", helloHandler)
	http.ListenAndServe(":12345", nil)
}
```

其实，`HandleFunc` 只是一个适配器，

```go
// The HandlerFunc type is an adapter to allow the use of
// ordinary functions as HTTP handlers.  If f is a function
// with the appropriate signature, HandlerFunc(f) is a
// Handler object that calls f.
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```

自动给 `f` 函数添加了 `HandlerFunc` 这个壳，最终调用的还是 `ServerHTTP`，只不过会直接使用 `f(w, r)`。这样封装的好处是：使用者可以专注于业务逻辑的编写，省去了很多重复的代码处理逻辑。如果只是简单的 Handler，会直接使用函数；如果是需要传递更多信息或者有复杂的操作，会使用上部分的方法。

如果需要我们自己写的话，是这样的：

```go
package main

import (
	"io"
	"net/http"
)

func helloHandler(w http.ResponseWriter, req *http.Request) {
	io.WriteString(w, "hello, world!\n")
}

func main() {
    // 通过 HandlerFunc 把函数转换成 Handler 接口的实现对象
    hh := http.HandlerFunc(helloHandler)
	http.Handle("/", hh)
	http.ListenAndServe(":12345", nil)
}
```



## 3. 默认

大部分的服务器逻辑都需要使用者编写对应的 Handler，不过有些 Handler 使用频繁，因此 `net/http` 提供了它们的实现。比如负责文件 hosting 的 `FileServer`、负责 404 的`NotFoundHandler` 和 负责重定向的`RedirectHandler`。下面这个简单的例子，把当前目录所有文件 host 到服务端：

```go
package main

import (
	"net/http"
)

func main() {
	http.ListenAndServe(":12345", http.FileServer(http.Dir(".")))
}
```

强大吧！只要一行逻辑代码就能实现一个简单的静态文件服务器。从这里可以看出一件事：`http.ListenAndServe` 第二个参数就是一个 Handler 函数（请记住这一点，后面有些内容依赖于这个）。

运行这个程序，在浏览器中打开 `http://127.0.0.1:12345`，可以看到所有的文件，点击对应的文件还能看到它的内容。

![](http://ww3.sinaimg.cn/large/006tKfTcgw1f6uujkzgc9j30mc0e8gm5.jpg)

其他两个 Handler，这里就不再举例子了，读者可以自行参考文档。

## 4. 路由

虽然上面的代码已经工作，并且能实现很多功能，但是实际开发中，HTTP 接口会有许多的 URL 和对应的 Handler。这里就要讲 `net/http` 的另外一个重要的概念：`ServeMux`。`Mux` 是 `multiplexor` 的缩写，就是多路传输的意思（请求传过来，根据某种判断，分流到后端多个不同的地方）。`ServeMux ` 可以注册多了 URL 和 handler 的对应关系，并自动把请求转发到对应的 handler 进行处理。我们还是来看例子吧：

```go
package main

import (
	"io"
	"net/http"
)

func helloHandler(w http.ResponseWriter, r *http.Request) {
	io.WriteString(w, "Hello, world!\n")
}

func echoHandler(w http.ResponseWriter, r *http.Request) {
	io.WriteString(w, r.URL.Path)
}

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/hello", helloHandler)
	mux.HandleFunc("/", echoHandler)

	http.ListenAndServe(":12345", mux)
}
```

这个服务器的功能也很简单：如果在请求的 URL 是 `/hello`，就返回 `hello, world!`；否则就返回 URL 的路径，路径是从请求对象 `http.Requests` 中提取的。

![](http://ww3.sinaimg.cn/large/006tKfTcgw1f6vdun05zpj30ne0hrgoe.jpg)

这段代码和之前的代码有两点区别：

1. 通过 `NewServeMux` 生成了 `ServerMux` 结构，URL 和 handler 是通过它注册的
2. `http.ListenAndServe` 方法第二个参数变成了上面的 `mux` 变量

还记得我们之前说过，`http.ListenAndServe` 第二个参数应该是 Handler 类型的变量吗？这里为什么能传过来 `ServeMux`？嗯，估计你也猜到啦：`ServeMux` 也是是 `Handler` 接口的实现，也就是说它实现了 `ServeHTTP` 方法，我们来看一下：

```go
type ServeMux struct {
        // contains filtered or unexported fields
}

func NewServeMux() *ServeMux
func (mux *ServeMux) Handle(pattern string, handler Handler)
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request))
func (mux *ServeMux) Handler(r *Request) (h Handler, pattern string)
func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request)
```

哈！果然，这里的方法我们大都很熟悉，除了 `Handler()` 返回某个请求的 Handler。`Handle` 和 `HandleFunc` 这两个方法 `net/http` 也提供了，后面我们会说明它们之间的关系。而 `ServeHTTP` 就是 `ServeMux` 的核心处理逻辑：**根据传递过来的 Request，匹配之前注册的 URL 和处理函数，找到最匹配的项，进行处理。**可以说 `ServeMux` 是个特殊的 Handler，它负责路由和调用其他后端 Handler 的处理方法。

关于`ServeMux` ，有几点要说明：

- URL 分为两种，末尾是 `/`：表示一个子树，后面可以跟其他子路径； 末尾不是 `/`，表示一个叶子，固定的路径
- 以`/` 结尾的 URL 可以匹配它的任何子路径，比如 `/images` 会匹配 `/images/cute-cat.jpg`
- 它采用最长匹配原则，如果有多个匹配，一定采用匹配路径最长的那个进行处理
- 如果没有找到任何匹配项，会返回 404 错误
- `ServeMux` 也会识别和处理 `.` 和 `..`，正确转换成对应的 URL 地址

你可能会有疑问？我们之间为什么没有使用 `ServeMux` 就能实现路径功能？那是因为 `net/http` 在后台默认创建使用了 `DefaultServeMux`。

## 5. 深入

嗯，上面基本覆盖了编写 HTTP 服务端需要的所有内容。这部分就分析一下，它们的源码实现，加深理解，以后遇到疑惑也能通过源码来定位和解决。

### Server

首先来看 `http.ListenAndServe()`:

```go
func ListenAndServe(addr string, handler Handler) error {
	server := &Server{Addr: addr, Handler: handler}
	return server.ListenAndServe()
}
```

这个函数其实也是一层封装，创建了 `Server` 结构，并调用它的 `ListenAndServe` 方法，那我们就跟进去看看：

```go
// A Server defines parameters for running an HTTP server.
// The zero value for Server is a valid configuration.
type Server struct {
	Addr           string        // TCP address to listen on, ":http" if empty
	Handler        Handler       // handler to invoke, http.DefaultServeMux if nil
	......
}

// ListenAndServe listens on the TCP network address srv.Addr and then
// calls Serve to handle requests on incoming connections.  If
// srv.Addr is blank, ":http" is used.
func (srv *Server) ListenAndServe() error {
	addr := srv.Addr
	if addr == "" {
		addr = ":http"
	}
	ln, err := net.Listen("tcp", addr)
	if err != nil {
		return err
	}
	return srv.Serve(tcpKeepAliveListener{ln.(*net.TCPListener)})
}
```

`Server` 保存了运行 HTTP 服务需要的参数，调用 `net.Listen` 监听在对应的 tcp 端口，`tcpKeepAliveListener` 设置了 TCP 的 `KeepAlive` 功能，最后调用 `srv.Serve()`方法开始真正的循环逻辑。我们再跟进去看看 `Serve` 方法：

```go
// Serve accepts incoming connections on the Listener l, creating a
// new service goroutine for each.  The service goroutines read requests and
// then call srv.Handler to reply to them.
func (srv *Server) Serve(l net.Listener) error {
	defer l.Close()
	var tempDelay time.Duration // how long to sleep on accept failure
    // 循环逻辑，接受请求并处理
	for {
         // 有新的连接
		rw, e := l.Accept()
		if e != nil {
			if ne, ok := e.(net.Error); ok && ne.Temporary() {
				if tempDelay == 0 {
					tempDelay = 5 * time.Millisecond
				} else {
					tempDelay *= 2
				}
				if max := 1 * time.Second; tempDelay > max {
					tempDelay = max
				}
				srv.logf("http: Accept error: %v; retrying in %v", e, tempDelay)
				time.Sleep(tempDelay)
				continue
			}
			return e
		}
		tempDelay = 0
         // 创建 Conn 连接
		c, err := srv.newConn(rw)
		if err != nil {
			continue
		}
		c.setState(c.rwc, StateNew) // before Serve can return
         // 启动新的 goroutine 进行处理
		go c.serve()
	}
}
```

最上面的注释也说明了这个方法的主要功能：

- 接受 `Listener l` 传递过来的请求
- 为每个请求创建 goroutine 进行后台处理
- goroutine 会读取请求，调用 `srv.Handler`

```go
func (c *conn) serve() {
	origConn := c.rwc // copy it before it's set nil on Close or Hijack

  	...

	for {
		w, err := c.readRequest()
		if c.lr.N != c.server.initialLimitedReaderSize() {
			// If we read any bytes off the wire, we're active.
			c.setState(c.rwc, StateActive)
		}

         ...

		// HTTP cannot have multiple simultaneous active requests.[*]
		// Until the server replies to this request, it can't read another,
		// so we might as well run the handler in this goroutine.
		// [*] Not strictly true: HTTP pipelining.  We could let them all process
		// in parallel even if their responses need to be serialized.
		serverHandler{c.server}.ServeHTTP(w, w.req)

		w.finishRequest()
		if w.closeAfterReply {
			if w.requestBodyLimitHit {
				c.closeWriteAndWait()
			}
			break
		}
		c.setState(c.rwc, StateIdle)
	}
}
```

看到上面这段代码 `serverHandler{c.server}.ServeHTTP(w, w.req)`这一句了吗？它会调用最早传递给 `Server` 的 Handler 函数：

```go
func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
	handler := sh.srv.Handler
	if handler == nil {
		handler = DefaultServeMux
	}
	if req.RequestURI == "*" && req.Method == "OPTIONS" {
		handler = globalOptionsHandler{}
	}
	handler.ServeHTTP(rw, req)
}
```

哇！这里看到 `DefaultServeMux` 了吗？如果没有 handler 为空，就会使用它。`handler.ServeHTTP(rw, req)`，Handler 接口都要实现 `ServeHTTP` 这个方法，因为这里就要被调用啦。

也就是说，无论如何，最终都会用到 `ServeMux`，也就是负责 URL 路由的家伙。前面也已经说过，它的 `ServeHTTP` 方法就是根据请求的路径，把它转交给注册的 handler 进行处理。这次，我们就在源码层面一探究竟。

### ServeMux

我们已经知道，`ServeMux` 会以某种方式保存 URL 和 Handlers 的对应关系，下面我们就从代码层面来解开这个秘密：

```go
type ServeMux struct {
	mu    sync.RWMutex
    m     map[string]muxEntry  // 存放路由信息的字典！\(^o^)/
	hosts bool // whether any patterns contain hostnames
}

type muxEntry struct {
	explicit bool
	h        Handler
	pattern  string
}
```

没错，数据结构也比较直观，和我们想象的差不多，路由信息保存在字典中，接下来就看看几个重要的操作：路由信息是怎么注册的？`ServeHTTP` 方法到底是怎么做的？路由查找过程是怎样的？

```go
// Handle registers the handler for the given pattern.
// If a handler already exists for pattern, Handle panics.
func (mux *ServeMux) Handle(pattern string, handler Handler) {
	mux.mu.Lock()
	defer mux.mu.Unlock()

    // 边界情况处理
	if pattern == "" {
		panic("http: invalid pattern " + pattern)
	}
	if handler == nil {
		panic("http: nil handler")
	}
	if mux.m[pattern].explicit {
		panic("http: multiple registrations for " + pattern)
	}

    // 创建 `muxEntry` 并添加到路由字典中
	mux.m[pattern] = muxEntry{explicit: true, h: handler, pattern: pattern}

	if pattern[0] != '/' {
		mux.hosts = true
	}

    // 这是一个很有用的小技巧，如果注册了 `/tree/`， `serveMux` 会自动添加一个 `/tree` 的路径并重定向到 `/tree/`。当然这个 `/tree` 路径会被用户显示的路由信息覆盖。
	// Helpful behavior:
	// If pattern is /tree/, insert an implicit permanent redirect for /tree.
	// It can be overridden by an explicit registration.
	n := len(pattern)
	if n > 0 && pattern[n-1] == '/' && !mux.m[pattern[0:n-1]].explicit {
		// If pattern contains a host name, strip it and use remaining
		// path for redirect.
		path := pattern
		if pattern[0] != '/' {
			// In pattern, at least the last character is a '/', so
			// strings.Index can't be -1.
			path = pattern[strings.Index(pattern, "/"):]
		}
		mux.m[pattern[0:n-1]] = muxEntry{h: RedirectHandler(path, StatusMovedPermanently), pattern: pattern}
	}
}
```

路由注册没有什么特殊的地方，很简单，也符合我们的预期，注意最后一段代码对类似 `/tree` URL 重定向的处理。

```go
// ServeHTTP dispatches the request to the handler whose
// pattern most closely matches the request URL.
func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
	if r.RequestURI == "*" {
		if r.ProtoAtLeast(1, 1) {
			w.Header().Set("Connection", "close")
		}
		w.WriteHeader(StatusBadRequest)
		return
	}
	h, _ := mux.Handler(r)
	h.ServeHTTP(w, r)
}
```

好吧，`ServeHTTP` 也只是通过 `mux.Handler(r)` 找到请求对应的 handler，调用它的 `ServeHTTP` 方法，代码比较简单我们就显示了，它最终会调用 `mux.match()` 方法，我们来看一下它的实现：

```go
// Does path match pattern?
func pathMatch(pattern, path string) bool {
	if len(pattern) == 0 {
		// should not happen
		return false
	}
	n := len(pattern)
	if pattern[n-1] != '/' {
		return pattern == path
	}
    // 匹配的逻辑很简单，path 前面的字符和 pattern 一样就是匹配
	return len(path) >= n && path[0:n] == pattern
}

// Find a handler on a handler map given a path string
// Most-specific (longest) pattern wins
func (mux *ServeMux) match(path string) (h Handler, pattern string) {
	var n = 0
	for k, v := range mux.m {
		if !pathMatch(k, path) {
			continue
		}
         // 最长匹配的逻辑在这里
		if h == nil || len(k) > n {
			n = len(k)
			h = v.h
			pattern = v.pattern
		}
	}
	return
}
```

`match` 会遍历路由信息字典，找到所有匹配该路径最长的那个。路由部分的代码解释就到这里了，最后回答上面的一个问题：`http.HandleFunc` 和 `ServeMux.HandlerFunc` 是什么关系？

```go
// Handle registers the handler for the given pattern
// in the DefaultServeMux.
// The documentation for ServeMux explains how patterns are matched.
func Handle(pattern string, handler Handler) { DefaultServeMux.Handle(pattern, handler) }

// HandleFunc registers the handler function for the given pattern
// in the DefaultServeMux.
// The documentation for ServeMux explains how patterns are matched.
func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	DefaultServeMux.HandleFunc(pattern, handler)
}
```

 原来是直接通过 `DefaultServeMux` 调用对应的方法，到这里上面的一切都串起来了！

### Request

最后一部分，要讲讲 Handler 函数接受的两个参数：`http.Request` 和 `http.ResponseWriter`。

Request 就是封装好的客户端请求，包括 URL，method，header 等等所有信息，以及一些方便使用的方法：

```go
// A Request represents an HTTP request received by a server
// or to be sent by a client.
//
// The field semantics differ slightly between client and server
// usage. In addition to the notes on the fields below, see the
// documentation for Request.Write and RoundTripper.
type Request struct {
	// Method specifies the HTTP method (GET, POST, PUT, etc.).
	// For client requests an empty string means GET.
	Method string

	// URL specifies either the URI being requested (for server
	// requests) or the URL to access (for client requests).
	//
	// For server requests the URL is parsed from the URI
	// supplied on the Request-Line as stored in RequestURI.  For
	// most requests, fields other than Path and RawQuery will be
	// empty. (See RFC 2616, Section 5.1.2)
	//
	// For client requests, the URL's Host specifies the server to
	// connect to, while the Request's Host field optionally
	// specifies the Host header value to send in the HTTP
	// request.
	URL *url.URL

	// The protocol version for incoming requests.
	// Client requests always use HTTP/1.1.
	Proto      string // "HTTP/1.0"
	ProtoMajor int    // 1
	ProtoMinor int    // 0

	// A header maps request lines to their values.
	// If the header says
	//
	//	accept-encoding: gzip, deflate
	//	Accept-Language: en-us
	//	Connection: keep-alive
	//
	// then
	//
	//	Header = map[string][]string{
	//		"Accept-Encoding": {"gzip, deflate"},
	//		"Accept-Language": {"en-us"},
	//		"Connection": {"keep-alive"},
	//	}
	//
	// HTTP defines that header names are case-insensitive.
	// The request parser implements this by canonicalizing the
	// name, making the first character and any characters
	// following a hyphen uppercase and the rest lowercase.
	//
	// For client requests certain headers are automatically
	// added and may override values in Header.
	//
	// See the documentation for the Request.Write method.
	Header Header

	// Body is the request's body.
	//
	// For client requests a nil body means the request has no
	// body, such as a GET request. The HTTP Client's Transport
	// is responsible for calling the Close method.
	//
	// For server requests the Request Body is always non-nil
	// but will return EOF immediately when no body is present.
	// The Server will close the request body. The ServeHTTP
	// Handler does not need to.
	Body io.ReadCloser

	// ContentLength records the length of the associated content.
	// The value -1 indicates that the length is unknown.
	// Values >= 0 indicate that the given number of bytes may
	// be read from Body.
	// For client requests, a value of 0 means unknown if Body is not nil.
	ContentLength int64

	// TransferEncoding lists the transfer encodings from outermost to
	// innermost. An empty list denotes the "identity" encoding.
	// TransferEncoding can usually be ignored; chunked encoding is
	// automatically added and removed as necessary when sending and
	// receiving requests.
	TransferEncoding []string

	// Close indicates whether to close the connection after
	// replying to this request (for servers) or after sending
	// the request (for clients).
	Close bool

	// For server requests Host specifies the host on which the
	// URL is sought. Per RFC 2616, this is either the value of
	// the "Host" header or the host name given in the URL itself.
	// It may be of the form "host:port".
	//
	// For client requests Host optionally overrides the Host
	// header to send. If empty, the Request.Write method uses
	// the value of URL.Host.
	Host string

	// Form contains the parsed form data, including both the URL
	// field's query parameters and the POST or PUT form data.
	// This field is only available after ParseForm is called.
	// The HTTP client ignores Form and uses Body instead.
	Form url.Values

	// PostForm contains the parsed form data from POST or PUT
	// body parameters.
	// This field is only available after ParseForm is called.
	// The HTTP client ignores PostForm and uses Body instead.
	PostForm url.Values

	// MultipartForm is the parsed multipart form, including file uploads.
	// This field is only available after ParseMultipartForm is called.
	// The HTTP client ignores MultipartForm and uses Body instead.
	MultipartForm *multipart.Form

	...

	// RemoteAddr allows HTTP servers and other software to record
	// the network address that sent the request, usually for
	// logging. This field is not filled in by ReadRequest and
	// has no defined format. The HTTP server in this package
	// sets RemoteAddr to an "IP:port" address before invoking a
	// handler.
	// This field is ignored by the HTTP client.
	RemoteAddr string
    ...
}
```

Handler 需要知道关于请求的任何信息，都要从这个对象中获取，一般不会直接修改这个对象（除非你非常清楚自己在做什么）！

### ResponseWriter

ResponseWriter 是一个接口，定义了三个方法：

- `Header()`：返回一个 Header 对象，可以通过它的 `Set()` 方法设置头部，注意最终返回的头部信息可能和你写进去的不完全相同，因为后续处理还可能修改头部的值（比如设置 `Content-Length`、`Content-type` 等操作）
- `Write()`： 写 response 的主体部分，比如 `html`  或者 `json` 的内容就是放到这里的
- `WriteHeader()`：设置 status code，如果没有调用这个函数，默认设置为 `http.StatusOK`， 就是 `200` 状态码

```go
// A ResponseWriter interface is used by an HTTP handler to
// construct an HTTP response.
type ResponseWriter interface {
	// Header returns the header map that will be sent by WriteHeader.
	// Changing the header after a call to WriteHeader (or Write) has
	// no effect.
	Header() Header

	// Write writes the data to the connection as part of an HTTP reply.
	// If WriteHeader has not yet been called, Write calls WriteHeader(http.StatusOK)
	// before writing the data.  If the Header does not contain a
	// Content-Type line, Write adds a Content-Type set to the result of passing
	// the initial 512 bytes of written data to DetectContentType.
	Write([]byte) (int, error)

	// WriteHeader sends an HTTP response header with status code.
	// If WriteHeader is not called explicitly, the first call to Write
	// will trigger an implicit WriteHeader(http.StatusOK).
	// Thus explicit calls to WriteHeader are mainly used to
	// send error codes.
	WriteHeader(int)
}
```

实际上传递给 Handler 的对象是:

```go
// A response represents the server side of an HTTP response.
type response struct {
	conn          *conn
	req           *Request // request for this response
	wroteHeader   bool     // reply header has been (logically) written
	wroteContinue bool     // 100 Continue response was written

	w  *bufio.Writer // buffers output in chunks to chunkWriter
	cw chunkWriter
	sw *switchWriter // of the bufio.Writer, for return to putBufioWriter

	// handlerHeader is the Header that Handlers get access to,
	// which may be retained and mutated even after WriteHeader.
	// handlerHeader is copied into cw.header at WriteHeader
	// time, and privately mutated thereafter.
	handlerHeader Header
	...
	status        int   // status code passed to WriteHeader
    ...
}
```

它当然实现了上面提到的三个方法，具体代码就不放到这里了，感兴趣的可以自己去看。

## 6. 扩展

虽然 `net/http` 提供的各种功能已经满足基本需求了，但是很多时候还不够方便，比如：

- 不支持 URL 匹配，所有的路径必须完全匹配，不能捕获 URL 中的变量，不够灵活
- 不支持 HTTP 方法匹配
- 不支持扩展和嵌套，URL 处理都在都一个 `ServeMux` 变量中

虽然这些都可以自己手动去码，但实在很不方便。这部分看看有哪些三方的包，都提供了哪些额外的功能。

### [alice](https://github.com/justinas/alice)

alice 的功能很简单——把多个 handler 串联起来，有请求过来的时候，逐个通过这个 handler 进行处理。

```go
alice.New(Middleware1, Middleware2, Middleware3).Then(App)
```

### [Gorilla Mux](http://www.gorillatoolkit.org/pkg/mux)

Gorilla 提供了很多网络有关的组件， Mux 就是其中一个，负责 HTTP 的路由功能。这个组件弥补了上面提到的 `ServeMux` 的一些缺陷，支持的功能有：

- 更多的匹配类型：HTTP 方法、query 字段、URL host 等
- 支持正则表达式作为 URL path 的一部分，也支持变量提取功能
- 支持子路由，也就是路由的嵌套，`SubRouter` 可以实现路由信息的传递
- 并且和 `ServeMux` 完全兼容

```go
r := mux.NewRouter()
r.HandleFunc("/products/{key}", ProductHandler)
r.HandleFunc("/articles/{category}/", ArticlesCategoryHandler)
r.HandleFunc("/articles/{category}/{id:[0-9]+}", ArticleHandler)
```

### [httprouter](https://github.com/julienschmidt/httprouter)

httprouter 和 `mux` 一样，也是扩展了自带 `ServeMux` 功能的路由库。它的主要特点是速度快、内存使用少、可扩展性高（使用 radix tree 数据结构进行路由匹配，路由项很多的时候速度也很快）。

```go
package main

import (
    "fmt"
    "github.com/julienschmidt/httprouter"
    "net/http"
    "log"
)

func Index(w http.ResponseWriter, r *http.Request, _ httprouter.Params) {
    fmt.Fprint(w, "Welcome!\n")
}

func Hello(w http.ResponseWriter, r *http.Request, ps httprouter.Params) {
    fmt.Fprintf(w, "hello, %s!\n", ps.ByName("name"))
}

func main() {
    router := httprouter.New()
    router.GET("/", Index)
    router.GET("/hello/:name", Hello)

    log.Fatal(http.ListenAndServe(":8080", router))
}
```



### [negroni](https://github.com/urfave/negroni)

http middleware 库，支持嵌套的中间件，能够和其他路由库兼容。同时它也自带了不少 middleware 可以使用，比如`Recovery`、`Logger`、`Static`。

```go
router := mux.NewRouter()
router.HandleFunc("/", HomeHandler)

n := negroni.New(Middleware1, Middleware2)
// Or use a middleware with the Use() function
n.Use(Middleware3)
// router goes last
n.UseHandler(router)

http.ListenAndServe(":3001", n)
```

## 7. 参考

这篇文章参考了以下资料：

- [golang net/http 官方文档](https://golang.org/pkg/net/http/)
- [net/http 源码](https://golang.org/src/net/http/server.go)
- [A Recap of Request Handling in Go](http://www.alexedwards.net/blog/a-recap-of-request-handling)
- [Not Another Go/Golang net/http Tutorial](http://soryy.com/blog/2014/not-another-go-net-http-tutorial/)
- [the http handlerfunc wrapper technique in golang](https://medium.com/@matryer/the-http-handlerfunc-wrapper-technique-in-golang-c60bf76e6124#.esdqmzzah)
- [why do all golang url routers suck](https://husobee.github.io/golang/url-router/2015/06/15/why-do-all-golang-url-routers-suck.html)
