[原文](https://itnext.io/how-i-pass-around-shared-resources-databases-configuration-etc-within-golang-projects-b27af4d8e8a)
在[上一篇](https://itnext.io/structuring-a-production-grade-rest-api-in-golang-c0229b3feedc)文章，我们探索了可以被应用于任何 web 应用程序的整体架构，为 todo app 构建了一个静态 JSON API，在这个文章中，我们介绍每个 web 应用的重要部分。

在大多数应用中，通常需要传递共享资源。这些共享资源可能是配置，数据库连接等。

接下来我将在代码中展示。

首先，我们来看一下上一文章离开的地方。我们有一个有路由和相应处理程序的项目。让我们从配置开始。在大多数项目中，需要从外部环境(配置文件，环境变量等)获得数据。这些数据通常包括像数据库连接字符串，数据库名和其他配置数据。

# 应用程序配置

我喜欢使用名为 [viper](https://github.com/spf13/viper) 的工具来处理我的配置。因为它已经出现很长时间了，并且它使用第三方包来免除我重写我自己的json/toml/yaml 的反序列化逻辑(虽然写起来并不困难)，但我也获得了一些额外的东西：

* 为不在配置文件中的字段设置默认值
* 从 JSON，TOML，YAML，HCL 和 Java 属性配置文件读取
* 实时监视和重读配置文件(可选)
* 从环境变量读取
* 从远端配置文件系统(etcd 或 Consul)，并监视修改
* 从命令行读取
* 从 buffer 缓冲区读取
* 设置显示的值

我们来写一些代码：

首先，我们创建一个放置初始化配置文件和第三方连接的包。即 `internal/config`。我们将配置文件包放置到 `internal`中，来阻止其他包使用这个配置包。

*Internal package is used to make specific packages unimportable.*

在这个配置包中，我们创建一个解析配置文件和返回一个包含该数据的结构体

```go
type Constants struct {
  PORT string
  Mongo struct {
    URL string
    DBName string
  }
}

func initViper() (Constants, error ){
  viper.SetConfigName("todo.config") // 没有 .TOML 或 .YAML 扩展的配置文件名
  viper.AddConfigPath(".") //从根目录搜索配置文件
  err := viper.ReadInConfig() // 找到并读取配置文件
  if err != nil {             // 处理读取配置文件的错误
    return Constants{}, err
  }
  viper.SetDefault("PORT", "8080")
  
  var constants Constants
  err = viper.Unmarshal(&constants)
  return constants, err
}
```

这个代码的作用相当简单：

* `viper.SetConfigName("todo.config")`使 viper 期待一个名为 `todo.config` 的配置文件，通常以 `.toml` 或 `.yaml` 为扩展名。
* `viper.AddConfigPath(".")` 使 viper 从当前/main 目录搜索配置文件。但是你多个拥有不同路径 `viper.AddConfigPath("xxx")` 的实例，以使 viper 可以从它们中搜索配置文件。
* `viper.SetDefault` 设置 PORT 的默认值为 8080，即使配置文件为空，或 PORT 并没有在配置文件中定义.
* `viper.Unmarshal` 将从配置文件中读取的数据解组为常量结构的最重要工作。

我们的配置文件看起来像：

```toml
PORT=3000
[Mongo]
	URL = "<content goes here>"
	DBName = "<content goes here>"
```

接下来， 我们会创建一个数据库连接实例，并将它放置到一个配置结构体中，它将被传递给我们的 app，需要数据库实例的模块，或其他配置数据。

```go
type Config struct {
  Constants
  Database *mgo.Database
}

// New is used to generate a configuration instance which will be passed around the codebase
func New() (*Config, error) {
  config := Config{}
  constants, err := initViper()
  config.Constants = constants
  if err != nil {
  	return &config, err
  }
  dbSession, err := mgo.Dial(config.Constants.Mongo.URL)
  if err != nil {
  	return &config, err
  }
  config.Database = dbSession.DB(config.Constants.Mongo.DBName)
  return &config, err
}
```

需要注意的是：

* 我们将 `initViper` 返回的值 constants 存储在 `config` ß实例中(该实例将被传递到应用程序中)。
* 我们在 mongodb 连接中使用这些常数(DB URL 和 DBName)
* 我们将数据库连接存储在 `config` 实例中
* 返回一个指向 `config` 实例的指针(我们将传递一个指向该配置的指针)

# 我们在应用的其余部分中使用该配置

```go
func Routes( configuration *config.Config) *chi.Mux {
  ...
  router.Route("/v1", func(r chi.Router){
    r.Mount("/api/todo", todo.New(configuration).Routes())
  })
  return router
}

func main(){
  configuration, err := config.New()
  if err != nil {
    log.Panicln("Configuration error", err)
  }
  router := Routes(configuration)
  ...
  ...
  log.Println("Serving application at PORT :" + configuration.Constants.PORT)
  log.Fatal(http.ListenAndServe(":"+configuration.Constants.PORT, router))
}
```

注意的是：

* 我们在主函数中调用 `config.New()` ，获得一个配置实例(实际上是个指针)
* 我们使用这个配置来访问配置文件的 PORT 地址
* 我们调整位于 `main.go` 中的 Routes 函数接收一个 configuration 作为它的参数

# 最重要的是

我们在 `todo` 特性包中现在有了 `New()` 函数。我们将 configuration 传递给 New()，它返回一个允许我们调用 Routes 方法的类型。

` r.Mount("/api/todo", todo.New(configuration).Routes())`

现在实际的代码是：

```go
type Config struct {
  *config.Config
}

func New(configuration *config.Config) *Config {
  return &Config{configuration}
}

func (config *Config) Routes() *chi.Mux {
  router := chi.NewRouter()
  router.Get("/{todoID}", config.GetATodo)
  router.Delete("/{todoID}", config.DeleteTodo)
  router.Post("/", config.CreateTodo)
  router.Get("/", config.GetAllTodos)
  return router
}
...
func (config *Config) GetATodo(w http.ResponseWriter, r *http.Request){
  todoID := chi.URLParam(r,"todoID")
  todos := Todo{
    Slug: todoID,
    Title: "Hello world",
    Body: "Heloo world from planet earth",
  }
  render.JSON(w, r, todos) // A chi router helper for serializing and returning json
}
```

注意的是：

* 在 Go 中，我们不能在导入类型中定义方法，所以我们在本地定义的 config 类型中嵌入一个导入类型。
* 现在每一个处理函数都是 config 类型的方法，所以现在它们可以访问位于配置文件中的数据，以及满足其持久性需要的数据库连接。
* **理解在些示例中我们使用的是 mongoDB 数据库，这个连接可以是任何源，redis 数据库，度量引擎的连接，以及第三方服务者的连接等等**。

在上面的例子中， 我们制作了 Config 结构体的 Handlers 方法，但这并不是必须的。我们来研究另一种替代方法。与使用结构体的方法不同的是，我们使用了闭包(是的，闭包也可以在 golang 中使用)

# 使用闭包替换结构体方法

```go
func Routes(configuration *config.Config) *chi.Mux{
  ...
  router.Route("/v1", func(r chi.Router){
    r.Mount("/api/todo", todo.Routes(configuration))
  })
  return router
}

func main(){
  configuration, err := config.New()
  if err != nil {
    log.Panicln("Configuration error", err)
  }
  router := Routes(configuration)
  ...
  ...
  log.Println("Serving application at PORT :" + configuration.Constants.PORT)
  log.Fatal(http.ListenAndServe(":" + configuration.Constants.PORT, router))
}
```

以防你有点儿困惑，在上面片断中最重要的一行是:

```go
r.Mount("/api/todo", todo.Routes(configuration))
```

我们做的就是调整我们之前安装 routes 的方法。因此，我们没有使用New函数进行初始化，而是将配置直接传递给Routes函数(不再是方法)。

接下来，调整我们的 Routes 方法来接受这个 configuration，它等价于**调整我们的处理程序成为闭包来接受configuration**，然后它返回一个处理请求的 handler(该handler也可以访问 configuration 参数，由于闭包的魔法)。

```go
func Routes(configuration *config.Config) *chi.Mux {
  router := chi.NewRouter()
  rotuer.Get("/{todoID}", GetATodo(configuration))
  router.Delete("/{todoID}", DeleteTodo(configuration))
  router.Post("/", CreateTodo(configuration))
  router.Get("/", GetAllTodos(configuration))
  return router
}
...
func GetATodo(configuration *config.Config){
  return func(w http.ResponseWriter, r *http.Request){
    todoID := chi.URLParam(r, "todoID")
    todos := Todo{
      Slug: todoID,
      Title: "Hello world from PORT: " + configuration.Constants.PORT,
      Body: "Heloo world from planet earth",
    }
    render.JSON(w, r, todos)
  }
}
```

注意的是：

* 较小的代码量。使用闭包，代码感觉更紧凑（我认为很优雅）

  ```go
  router.Get("/{todoID}", GetATodo(configuration))
  ```

  

* 闭包可以减少一点儿不自然，崮为你可能会遇到以下情况

  ```go
  functionA(argA)(argB)(argC)
  ```

该示例是使用闭包的合适的代码。functionA 接受 argA，并返回一个接受 argB 的函数，同样地它返回一个接受 argC 的函数，它可以返回任何内容(甚至是另一个函数)

但是我仍然认为闭包是用于代码编写的强大工具，在代码可读性和可维护性方面，在收益大于危害时使用它们。

代码 [github](https://github.com/tonyalaribe/todoapi/tree/master/withmongodb?source=post_page-----b27af4d8e8a----------------------)