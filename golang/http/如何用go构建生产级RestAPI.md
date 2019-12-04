[原文](https://itnext.io/structuring-a-production-grade-rest-api-in-golang-c0229b3feedc)

有个神话--使用golang写API，不像其他语言那样简单且惯用。实际上， 我遇到了很多REST API 代码库，它们使用了如此多的抽象使代码变得复杂混乱，最终有损可读性和可维护性。

在本系列中， 我们将逐步讲解如何构建生产级 todo 列表 api ，它将从必要性，如代码结构和路由，开始有机地增长，然后增加 mongo db 和 badger 数据存储层，然后是身份验证层。

在该系列我们使用 chi 路由。

为什么是 Chi?而不是标准库，或 Gin，或 router-x?

其实没关系，无论您使用哪种路由，在该系列讨论的概念都适用。但是有一些检查清单使我认为 Chi-regard 比大多数备选方案更优越：

* 和 `net/http` 100%兼容 -- 在生态系统中使用任何http或中间件包也兼容 `net/http`。
* 专为模块化/组合化的APIs设计 -- 中间件，内联中间件，路由分组和子路由安装。
* 没有外部依赖 -- Go1.7 + stdlib + net/http
* 很强大 -- 在 Pressly，CloudFlare，HeroKu，99Designs 等许多其他公司生产中
* 轻量级 -- 对于 chi 路由，cloc 约为 1000 LOC
* 真的很快

我最喜欢的是，您为其他 `net/http` 兼容的路由编写的旧的 http 处理程序和中间件也可以工作

# 我们开始吧

首先我们创建 main.go

```go
package main

import (
	"log"
  "net/http"
  
  "github.com/go-chi/chi"
  "github.com/go-chi/chi/middleware"
  "github.com/go-chi/render"
  "github.com/tonyalaribe/todoapi/basestructure/features/todo"
)

func Routes() *chi.Mux {
  router := chi.NewRouter()
  router.Use(
    render.SetContentType(render.ContentTypeJSON), // Set content-Type headers as application/json
    middleware.Logger, // Log API request calls
    middleware.DefaultCompress, // Compress results, mostly gzipping assets and json
    middleware.RedirectSlashes,  // Redirect slashes to no slash URL versions
    middleware.Recoverer,  // Recover from panics without crashing server
  )
  
  router.Route("/v1", func(r chi.Router){
    r.Mount("/api/todo", todo.Routes())
  })
  return router
}

func main(){
  router := Routes()
  
  walkFunc := func(method string, route string, handler http.Handler, middlewares ...func(http.Handler) http.Handler) error{
    log.Printf("%s %s\n", method, route) // Walk and print out all routes
    return nil
  }
  if err != chi.Walk(router, walkFunc); err != nil  {
    log.Panicf("Logging err: %s\n", err.Error()) // panic if there is an error
  }
  
  log.Fatal(http.ListenAndServe(":8080", router)) // Note, the port is usually gotten from the environment.
}
```

# 上面代码中一些最佳实践的亮点

1. 在独立的包中对路由进行逻辑分组，并安装这些路由：

   ```go
   r.Mount("/api/todo", todo.Routes())
   ```

2. API的版本，可以不用中断旧的客户端升级api

   ```go
   router.Route("/v1", ...)
   ```

3. 使用中间件扩展，许多被大量路由使用的代码都可以变成可链接的中间件。例如，身价认证，设计响应头，压缩，请求日志，限速等等。

chi 路由有一个名为 `walk` 的方法，该方法接收：

* 路由
* 回调

回调被每个在 `router` 中定义的路由 `route` 调用，接收四个参数：

1. 为 route 定义的方法
2. 实际的路由字符串
3. 处理程序（函数），用于处理给定路由的请求
4. 为给定路由定义的中间件列表(中间件是一个简单的函数，它在处理函数handler被调用之前调用，所以它们被用来预处理请求，身份认证等等)

在我的例子中，我简单地 walk 路由并打印所有定义的路由。这帮助我一目了然地查看所有可接触到的路由

# 接下来我们创建一个 todo 包，它实际上包含我们的 todo 逻辑

```go
package todo

import (
	"net/http"
  
  "github.com/go-chi/chi"
  "github.com/go-chi/render"
)

type Todo struct {
  Slug  string `json:"slug"`
  Title string `json:"title"`
  Body  string `json:"body"`
}

func Routes() *chi.Mux {
  router := chi.NewRouter()
  router.Get("/{todoID}", GetATodo)
  router.Delete("/{todoID}", DeleteTodo)
  router.Post("/", CreateTodo)
  router.Get("/", GetAllTodos)
  return router
}

func GetATodo(w http.ResponseWriter, r *http.Request) {
  todoID := chi.URLParam(r, "todoID")
  todos := Todo{
    Slug: todoID,
    Title: "Hello world",
    Body: "Heloo world from planet earth",
  }
  render.JSON(w, r, todos) // A chi router helper for serializing and returning json
}

func DeleteTodo( w http.ResponseWriter, r *http.Request){
  response := make(map[string]string)
  response["message"] = "Deleted TODO successfully"
  render.JSON(w, r, response) // Return some demo response
}

func CreateTodo(w http.ResponseWriter, r *http.Request){
  response := make(map[string]string)
  response["message"] = "Created TODO successfully"
  render.JSON(w, r, response) // Return some demo response
}

func GetAllTodos(w http.ResponseWriter, r *http.Request){
  todos := []Todo{
    {
      Slug:  "slug",
      Title: "Hello world",
      Body:  "Heloo world from planet earth",
    },
  }
  render.JSON(w, r, todos) // A chi router helper for serializing and returning json
}
```

# 注意事项

1. todo 包有一个返回所有路由的函数。它们被安装在 main.go 中。实际上，我通常在名为 *routes.go* 的文件中添加这些路由，所以很容易在包中找到
2. 有函数签名 `func (w http.ResponseWriter, r *http.Request)` 的处理程序，意味着如果你在标准库中使用 net/http ，你要写的处理程序没有不同的形式
3. render.JSON 的使用，仅仅是 `encoding.json` 的封装，它自动转义 JSON 响应中的所有 html，并将 content type 设置为 application/json