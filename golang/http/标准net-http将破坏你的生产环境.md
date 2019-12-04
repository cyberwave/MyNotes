另一个标题**net/http超时介绍**

原文发表在[blog.simon-frey.eu](https://blog.simon-frey.eu/go-as-in-golang-standard-net-http-config-will-break-your-production)

首先，正如你已经从标题中认识到，这篇博客文章是站在巨人的肩上。以下两个博客文章启发了我修改net/http超时，因为链接的博客文章某些时候已经过时了:

* [golang 的 net/http 超时完整指南](https://blog.cloudflare.com/the-complete-guide-to-golang-net-http-timeouts/)
* [不要使用go的默认http客户端](https://medium.com/@nate510/don-t-use-go-s-default-http-client-4804cb19f779)

阅读这篇文章后请访问他们，看看在这么短的时间内情况发生了什么变化。

# 为什么不应使用标准的net/http配置？

Go 的核心团队决定在标准的 net/http 客户端或服务器配置上不设置任何超时，这真是一个明知的决定，为什么？

为了不破坏东西！超时是一个高度个性化的设置， 在大多数情况下，超时时间过短将会导致您的应用程序出现无法解释的错误，而太长时间则不会(或在Go中则不会)。

想象下列使用 go 的 net/http 客户端不同用例：

1. 从web服务下载一个大文件（10GB）。平均（德国）互联网连接将花费大约5分钟。

   => 连接的超时应该大于5分钟，因为任何更短的时间将在下载文件的过程中(任何百分比)被取消而中断您的应用程序。

2. 通过大量并发连接访问REST API。每个连接至少需要几秒种。

   => 超时应该不走过10秒钟，因为任何更长的时间意味着，您将使该连接长时间保持打开状态，并使您的应用程序饿死，因为它只能具有X（取决于系统，配置和编码）打开的连接。因此，如果您访问的该 REST API 以某种方式破坏了该连接，使得该连接保持打开状态而没有向您发送所需的数据，则您希望阻止它这样做。

因此，应针对哪种情况优化标准库？相信我，你不会想要为全球上百万开发者做决定。

这是为什么我们必须设置超时，来让它适合我们的用例！

**所以不要使用标准的 go http 客户端/服务器！它将中断你的生产系统！**

# HTTP 连接中发生什么类型的超时？

我假设您对 [TCP](https://en.wikipedia.org/wiki/Transmission_Control_Protocol) 和 [HTTP](https://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol) 协议有基本的了解。（如果不是，维基是一个好的起点）

发生的超时主要有三种不同的类型：

* 在连接建立期间
* 在接收/发送头部信息的期间
* 在接收/发送正文的期间

正如您在介绍中的两个示例可能会预料到那样，我们必须关注的超时大部分是关于正文的超时。在大多数情况下，其他的超时通常较短且相似。（例如，只有一定数量的标题将被发送）我们仍然必须考虑和关注头部的超时，因为某些DOS攻击会与标头格式不符，永远不关闭头([SLOWLORIS DOS 攻击](https://www.slashroot.in/slowloris-http-dosdenial-serviceattack-and-prevention)),但是我们将在帖子的后续内容中介绍这一点。

# 您必须至少做这些：简单的路径

net/http 给你为完成的数据转移设置超时的能力(建立连接，头部，正文)。它不像后来的解决方案那样精细，但是它会帮助你阻止大多数明显的问题：

* 连接耗尽
* 格式不正确的标头攻击

**所以你应该至少在你使用的go net/http的客户端/服务器中使用这些超时！**

# 客户端

下列的客户端示例，给您提供了完成的 5 秒超时。

```go
c := &http.Client{
  Timeout: 5 * time.Second,
}
c.Get("https://blog.simon-frey.eu/")
```

如果连接一直打开着，它将被取消 `net/http: request canceled(CLient.Timeout exceeded  while reading...)`

所以这个超时适应于小文件，但不适用于下载大文件。在后面的文章中，我们将看到如何为正文设置可变的超时时间。

# 服务器

对于服务器，我们必须在简单设置设置两个超时：Read 和 Write。所以 `ReadTimeout` 定义了在客户端发送数据期间，你允许连接打开多长时间。同时 `WriteTimeout` 是在另一个方向。（是的，也可能是，您将数据发送到某个地方，而程序包从未得到接受的TCP-ACK，服务器将再次饿死）

```go
s := &http.Server{
  ReadTimeout: 1 * time.Second,
  WriteTimeout: 10 * time.Second,
  Addr: ":8080",
}
s.ListenAndServe()
```

该服务器将监听 `8080` 端口，拥有你想要的超时。

**对于很多的用例，这个简单的路径可能足够 了。但请继续阅读，看还有哪些可能**

# [客户端] 超时的深入配置

在我们开始之前，有一个事情需要注意是下面的不同：

* 为完整的请求（包括重定向）定义了简单路径超时（以上所述）
* 下列的超时为每个连接。由于他们是通过 `http.Transport`，因此它本身没有有关重定向的信息。因此，如果发生大量重定向，则超时是每个连接的时间加起来。您可以同时使用两者，以防止无限重定向

# 连接设置

下面的设置有两个参数，我设置了一个超时。它们的连接类型不同：

* `DialContext`: 定义未加密的 HTTP 连接设置的超时
* `TLSHandshakeTimeout`：关心将未加密的连接升级到加密的一个HTTPS的设置超时

在 2019 年的设置中，你应该始终尝试与加密的 HTTPS 站点进行通信，因此在极少数情况下，仅设置两个参数之一是有意义的。

```go
c := &http.Client{
    Transport: &http.Transport{
        DialContext:(&net.Dialer{
            Timeout:   3 * time.Second,
        }).DialContext,
        TLSHandshakeTimeout:   10 * time.Second,
    }
}
c.Get("https://blog.simon-frey.eu/")
```

通过设置这些参数，您可以定义连接的建立最长应持续多长时间。这会帮助你以最快的方法"检测"(实际上的检测，你做的不仅仅是这几行)关闭的主机。所以你不必在项目中等待主机，面是首先关闭主机。

# 响应头

现在我们已经建立(希望是HTTPS)了连接，所以我们必须接收我们获得的连接的元信息。这些地元信息存储在头部中。我们可以设置超时，我们希望主机多长时间响应我们。

这又是要定义的两个不同的超时：

* `ExpectContinueTimeout`：这配置了在发送有效载荷后在开始答案之后要等待多长时间（以标头开头的形式）
* `ResponseHeaderTimeout`：使用这个参数，您设置了头部的完整转移允许持续多长时间。

所以你希望在发送完成的请求 ExpectContinueTimeout` + `ResponseHeaderTimeout 后 ，获得拥有完成的头部信息

```go
c := &http.Client{
    Transport: &http.Transport{
        ExpectContinueTimeout: 4 * time.Second,
        ResponseHeaderTimeout: 10 * time.Second,
    },
}
c.Get("https://blog.simon-frey.eu/")
```

通过设置此参数，我们可以定义接受服务器响应所需的时间，因此也可以定义内部操作所需的时间。

想象下面的场景：*您访问一个API，它可以改变你发送给它的图像的大小。所以你上传图像，通常它花费大约 1 秒种来调整图像的大小，然后它开始返回给你的服务。但是可能该 API 由于任何原因而崩溃掉，然后花费 60 秒调整图像的大小。正是你现在定义的超时，你可以在几秒钟之内中止，并告诉您的客户 API xyz 已经关闭并且您正在和服务提供者联系...胜过花哨的图像编辑器加载了一段时间并且不显示任何状态信息，这全都归因于错误，这 甚至不是您的错！*

# 正文（Body）

根据定义，正文的超时是最难的，因为这是响应的一部分，它将在大小上变化最大，进而需要时间进行转移数据。

我们将介绍两种帮助您定义正文超时的方法：

* 静态超时，在一定的时间后将终止传输
* 可变超时，在一定时间内没有传输任何数据后，该超时会终止

## 静态超时

**在示例代码中我们抛弃所有的错误。您不应该这样做！**

```go
c := &http.Client{}
resp, _ := c.GET("https://blog.simon-frey.eu")
defer resp.Body.Close()

time.AfterFunc(5*time.Second, func(){
  resp.Body.Close()
})
bodyBytes, _ := ioutil.ReadAll(resp.Body)
```

在代码示例中， 我们设置了一个 `timer`，它在完成之后执行 `resp.Body.Close()`。在这个命令下我们关闭 body ，`ioutil.ReadAll` 将抛出 `read on closed response body` 的错误。

## 可变超时

```go
c := &http.Client{}
resp, _ := c.GET(https://blog.simon-frey.eu")
defer resp.Body.Close()

timer := time.AfterFunc(5*time.Second, func() {
    resp.Body.Close()
})  
                 
bodyBytes := make([]byte, 0)
for {
    //We reset the timer, for the variable time
    timer.Reset(1 * time.Second)

    _, err = io.CopyN(bytes.NewBuffer(bodyBytes), resp.Body, 256)
    if err == io.EOF {
        // This is not an error in the common sense
        // io.EOF tells us, that we did read the complete body
            break
    } else if err != nil {
        //You should do error handling here
        break
    }
}
```

这里的区别是，我们有一个无限循环，它遍历整个body并将数据拷贝出来。有两个选项可以离开此循环：

* 从 `io.CopyN` 得到一个 `io.EOF` 文件错误，这意味着我们已经读完该body 且没有触发超时限制
* 我们得到其他错误，如果错误是 `read on closed response body`，超时会被触发。

该解决方案有效，因为 `io.CopyN` 是阻塞的。所以如果body中没有足够的字节(我们的例子是256个字节)要读取，它将等待。如果在这期间超时触发，我们将停止执行。

# 我的默认配置

再次说明：**这是我自己对超时的看法，你应该根据你的项目需求来适配它们！**我不会在每个项目中都使用完全相同的设置！

```go
c := &http.Client{
    Transport: &http.Transport{
        DialContext:(&net.Dialer{
            Timeout:   10 * time.Second,
            KeepAlive: 10 * time.Second,
        }).DialContext,
        TLSHandshakeTimeout:   10 * time.Second,
           
        ExpectContinueTimeout: 4 * time.Second,
        ResponseHeaderTimeout: 3 * time.Second,
        
        // Prevent endless redirects
        Timeout: 10 * time.Minute,
    },
}
```

## [服务器]超时的深入配置

由于 `http.Server` 没有特定的拨号超时，因此我们将直接开始头信息的超时

## Headers

对于 请求头，我们有一个确定的超时：`ReadHeaderTimeout`，它代表读取完整个请求头(客户端发送的)需要的时间。所以如果客户端花费过长的时间发送headers，连接将超时。这个超时在对抗类似 [SLOWLORIS](https://www.slashroot.in/slowloris-http-dosdenial-serviceattack-and-prevention) 的攻击尤其重要，由于header永远不会得到关闭标志，而连接相应的也一直保持打开状态。

```go
s := &http.Server{
  ReadHeaderTimeout: 20 * time.Second,
}
s.ListenAndServe()
```

正如您可能已经认识到，只有一个ReadHeaderTimeout，因为对于向客户端发送数据，go在 header 和 body 之间的超时没有一定的区别。

## Body

这里我们必须区分请求（客户端发送到服务端）和响应正文。

### Response body

对于响应 body ，关于超时只有一个静态解决方案：

```shell
s := &http.Server{
	WriteTimeout: 20 * time.Second,
}
s.ListenAndServe()
```

只要连接是打开的，我们就无法区分数据是否正确发送或客户端是否在此处做假。但是，正如我们所知的有效负载数据，很容易在这里根据我们过去有关服务器的信息设置超时时间。所以如果你是一个文件服务器，这个超时应该比 API 服务器的长。出于测试的目的，你可以设置不超时来跟踪一个 '正常’ 的请求需要花费多长时间。增加百分之几的变化，然后就可以了。

### Request body

**注意：如果你设置了 WriteTimeout，它也会对请求超时产生影响。这是因为 `WriteTimeout` 的定义。当请求的 header 被读取时开始。** *所以如果读取请求 body 花费 5 秒而你写超时是 4 秒，它也会关闭请求正文的读取！*

对于请求头，对应的也有两个可能的解决方案：

* 我们可以通过 `http.Client` 配置设置静态超时
* 为此，我们必须构建自己的代码变通办法来解决超时问题（因为目前尚不支持）

**Static**

对于静态超时，我们可以使用已经在 easy path 中使用的 `ReadTimeout` 参数：

```go
s := &http.Server{
  ReadTimeout: 20 * time.Second,
}
s.ListenAndServe()
```

**可变**

对于可变超时，我们需要在 handlers 级进行工作。**不要设置 `ReadTimeout`，因为静态超时会干扰可变超时。你也一定不要设置 `WriteTimeout` ，由于它从请求头结束开始计时，也将干扰可变头部**

我们必须为服务器定义人们自己的处理程序（handler），在我们的例子中，我们称它为 `timeoutHandler`。该处理程序除了使用循环从正文读取数据，如果没有数据要发送则超时外，不做任何事情 。

```go
type timeoutHandler struct{}
func (h timeoutHandler) ServeHTTP(w http.ResponseWriter, r *http.Request){
	defer r.Body.Close()
	timer := time.AfterFunc(5*time.Second, func() {
		r.Body.Close()
	})
	bodyBytes := make([]byte, 0)
	for {
		//We reset the timer, for the variable time
		timer.Reset(1 * time.Second)
        
		_, err := io.CopyN(bytes.NewBuffer(bodyBytes), r.Body, 256)
		if err == io.EOF {
			// This is not an error in the common sense
			// io.EOF tells us, that we did read the complete body
			break
		} else if err != nil {
			//You should do error handling here
			break
		}
	}
}
func main() {
	h := timeoutHandler{}
	s := &http.Server{
		ReadHeaderTimeout:20 *time.Second,
		Handler:h,
		Addr:":8080",
	}
	s.ListenAndServe()
}
```

它和我们在客户端所做的非常类似。你已经在每个分开的处理程序中定义了该循环。所以你或许考虑为之构建一个函数，所以你不需要为之重复造轮子。

# 我的默认配置

**这是我关于超时的想法，你应该根据你项目的需求适配它们**

```go
s := &http.Server{
	ReadHeaderTimeout:20 *time.Second,
	ReadTimeout: 1 * time.Minute,
    WriteTimeout: 2 * time.Minute,
}
```



[RSS Feed](https://blog.simon-frey.eu/feed) — This work is licensed under [Creative Commons Attribution 4.0 International License](https://medium.com/@simonfrey/ http://creativecommons.org/licenses/by/4.0/)

# 资料参考

https://golang.org/pkg/net/http/

https://medium.com/@nate510/don-t-use-go-s-default-http-client-4804cb19f779

https://blog.cloudflare.com/exposing-go-on-the-internet/

Gopher Image (CC BY-SA 3.0): [Wikimedia](https://commons.wikimedia.org/wiki/File:Gophercolor.jpg)