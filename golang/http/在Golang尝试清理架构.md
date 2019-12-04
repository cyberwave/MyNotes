在阅读鲍伯大叔的 [清理架构概念](https://clevercoder.net/2018/09/08/clean-architecture-summary-review/) 之后，我尝试使用 Golang 实现它。这和我在我们公司使用的架构类似，**[Kurio - App Berita Indonesia](https://kurio.co.id/)**，但是在结构上有一点不同。概念是一样的，只是结构文件夹不一样。

你可以在[这儿](https://github.com/bxcodec/go-clean-arch)寻找一个示例项目，一个 CRUD 管理示例的文章

![图片1](https://miro.medium.com/max/1440/1*CyteJRpIHC-DFE23UtlZfQ.png)

* 免责声明：

  您可以用自己的或具有相同功能的第三方替换此处的任何内容。

# Basic

众所周知，设计整洁架构前的约束是：

1. 独立的框架。架构不依赖现有的某些功能丰富的软件库。这允许你使用这些框架作为工具，而不是将你的系统塞入他们有限的约束中。
2. 可测试的。业务规则可以在没有UI，数据库，Web服务器或任何其他外部元素的情况下测试。
3. 独立于UI。UI可以很容易地被更改，而不用系统的其他部分。Web UI可以被控制台UI替换，例如，不修改业务规则。
4. 独立于数据库。您可以使用Mongo，BigTable，CouchDB 或其他替换Oracle 或 SQL Server。你的业务规则未绑定数据库。
5. 独立于任何外部机构。事实上，你的业务规则对外界一无所知。

更多内容，[点击这里](https://8thlight.com/blog/uncle-bob/2012/08/13/the-clean-architecture.html)

所以，基于这些约束，每层必须是独立的并可测试的。

如果鲍伯大叔的架构有四层：

* 实体（Entities）
* 用例（Usecase）
* 控制层（Controller）
* 框架 & 驱动（Framework & Driver）

在我的项目中，也有四层：

* 模板（Models）
* 仓库（Repository）
* 用例（Usecase）
* 投递（Delivery）

# Models

和 Entities 一样，将在所有层使用。该层，将存储任何对象的结构体和它的方法。例如：文章(Article)，学生(Student)，书籍(Book)。结构体示例：

```go
// Article.go
import "time"

type Article struct {
	ID        int64     `json:"id"`
	Title     string    `json:"title"`
	Content   string    `json:"content"`
	UpdatedAt time.Time `json:"updated_at"`
	CreatedAt time.Time `json:"created_at"`
}
```

任何实体，或模型都将存储于此。

# Repository

仓库将存储任何数据库处理程序，查询，或创建/插入任何数据库将存储于此。这一层将 仅对数据库扮演 CRUD角色。这里没有业务处理。仅有数据库的普通函数。

这一层也有责任选择应用程序中使用何种数据库，可以是MySQL，MongoDB，MariaDB，Postgresql等，将在这里决定。

如果使用 ORM，这一层将控制输入，直接将它们给 ORM 服务。

如果调用微服务，将在这里处理。创建到其他服务的 HTTP 请求，并清理数据。该层必须完全充当仓库。处理所有数据输入 - 输出没有特定的逻辑发生。

该仓库层取决于连接到的DB，或其他微服务(如果存在)。

# Usecase

该层将扮演业务流程处理程序。任何流程都在这儿被处理。该层将决定使用哪个仓库层。同时有责任提供数据以交付使用。处理数据以进行计算或完成任何事情。

Usecase 层将接收 Delivery 层已经清理过的的任何输入，然后输入的处理可能存储到数据库中，或从数据库读取数据等等。

该层将依赖取决于 Repository 层。

# Delivery

该层将扮演主持人。决定数据应该如何呈现。可能是 REST API 或 HTML 文件，或 gRPC，任何交付类型。该层也接受用户的输入。清理数据并将其发送给用例层。

对于我的样例工程，我使用 REST API 作为交付(delivery)方法。客户端将通过网络调用资源端点，交付层会获得输入或请求，并将其发送到用例层。

该层会取决于用例层。

# 层间的通信

除了 Models 外，每层将通过接口通信。例如，用例层需要仓库层，所以它们如何通信？仓库层将提供一个接口以供通信。

仓库接口示例

```go
package repository

import models "github.com/bxcodec/go-clean-arch/article"

type ArticleRepository interface {
	Fetch(cursor string, num int64) ([]*models.Article, error)
	GetByID(id int64) (*models.Article, error)
	GetByTitle(title string) (*models.Article, error)
	Update(article *models.Article) (*models.Article, error)
	Store(a *models.Article) (int64, error)
	Delete(id int64) (bool, error)
}
```

用例层（Usecase layer）将使用这个接口与仓库层通信，仓库层**必须**实现这些接口，以便用例可以使用它

Usecase 接口示例

```go
package usecase

import (
	"github.com/bxcodec/go-clean-arch/article"
)

type ArticleUsecase interface {
	Fetch(cursor string, num int64) ([]*article.Article, string, error)
	GetByID(id int64) (*article.Article, error)
	Update(ar *article.Article) (*article.Article, error)
	GetByTitle(title string) (*article.Article, error)
	Store(*article.Article) (*article.Article, error)
	Delete(id int64) (bool, error)
}
```

和用例层一样，Delivery 层将使用该接口，Usecase 层**必须**实现这些接口

# 测试每层

从所周知，干净意味着独立。每层都是可测试的，甚至其他层还不存在。

* Models 层

  该层仅测试任何结构体中声明的函数/方法是否存在。可以很容易地测试并与独立于其他层。

* Repository

  要测试这层，更好的方法是进行集成测试。但是您也可以为每个测试做模拟我使用 github.com/DATA-DOG/go-sqlmock 作为我模拟 mysql 查询流程的辅助。

* Usecase

  由于该层取决于 Repository 层，意味着该层需要 Repository 进行测试。所以我们必须制作一个 Repository 的模型，使用 mockery 进行根据，根据之前定义的合同接口。

* Delivery

  和 Usecase 一样，由于该层取决于 Usecase 层，意味着我们需要 Usecase 进行测试。并且 Usecase 层也必须使用 mockery 模拟，基于之前定义的接口。

为了模拟，我使用 vektra 的  golang mockery，可以在 [https://github.com/vektra/mockery](https://github.com/vektra/mockery) 查看 

# Repository 测试

为了测试该层，就像我之前所说的，我使用 sql-mock 来模拟我的查询流程。你可以使用像我在 github.com/DATA-DOG/go-sqlmock 使用的，或其他拥有类似的函数的测试用例。

```go
func TestGetByID(t *testing.T) {
 db, mock, err := sqlmock.New() 
 if err != nil { 
    t.Fatalf(“an error ‘%s’ was not expected when opening a stub  
        database connection”, err) 
  } 
 defer db.Close() 
 rows := sqlmock.NewRows([]string{
        “id”, “title”, “content”, “updated_at”, “created_at”}).   
        AddRow(1, “title 1”, “Content 1”, time.Now(), time.Now()) 
 query := “SELECT id,title,content,updated_at, created_at FROM 
          article WHERE ID = \\?” 
 mock.ExpectQuery(query).WillReturnRows(rows) 
 a := articleRepo.NewMysqlArticleRepository(db) 
 num := int64(1) 
 anArticle, err := a.GetByID(num) 
 assert.NoError(t, err) 
 assert.NotNil(t, anArticle)
}
```

# Usecase 测试

Usecase 层测试样例，取决于 Repository 层。

```go
package usecase_test

import (
	"errors"
	"strconv"
	"testing"

	"github.com/bxcodec/faker"
	models "github.com/bxcodec/go-clean-arch/article"
	"github.com/bxcodec/go-clean-arch/article/repository/mocks"
	ucase "github.com/bxcodec/go-clean-arch/article/usecase"
	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/mock"
)

func TestFetch(t *testing.T) {
	mockArticleRepo := new(mocks.ArticleRepository)
	var mockArticle models.Article
	err := faker.FakeData(&mockArticle)
	assert.NoError(t, err)

	mockListArtilce := make([]*models.Article, 0)
	mockListArtilce = append(mockListArtilce, &mockArticle)
	mockArticleRepo.On("Fetch", mock.AnythingOfType("string"), mock.AnythingOfType("int64")).Return(mockListArtilce, nil)
	u := ucase.NewArticleUsecase(mockArticleRepo)
	num := int64(1)
	cursor := "12"
	list, nextCursor, err := u.Fetch(cursor, num)
	cursorExpected := strconv.Itoa(int(mockArticle.ID))
	assert.Equal(t, cursorExpected, nextCursor)
	assert.NotEmpty(t, nextCursor)
	assert.NoError(t, err)
	assert.Len(t, list, len(mockListArtilce))

	mockArticleRepo.AssertCalled(t, "Fetch", mock.AnythingOfType("string"), mock.AnythingOfType("int64"))

}
```

Mockery 为我生成仓库层的模拟样例。所以我不需要事先完成 Repository 层。甚至我的 Repository 层还未实现，我也可以先完成我的 Usecase 测试。

# Delivery 测试

Delivery 测试将取决于你如何交付数据。如果使用 REST API，我们可以使用 httptest，一个 golang 为 http 测试内置的包。

由于它取决于 Usecase，所以我们需要 Usecase 的一个模拟。和 Repository 一样，为了交付测试，我使用了 Mockery 来模拟我的用例。

```go
func TestGetByID(t *testing.T) {
 var mockArticle models.Article 
 err := faker.FakeData(&mockArticle) 
 assert.NoError(t, err) 
 mockUCase := new(mocks.ArticleUsecase) 
 num := int(mockArticle.ID) 
 mockUCase.On(“GetByID”, int64(num)).Return(&mockArticle, nil) 
 e := echo.New() 
 req, err := http.NewRequest(echo.GET, “/article/” +  
             strconv.Itoa(int(num)), strings.NewReader(“”)) 
 assert.NoError(t, err) 
 rec := httptest.NewRecorder() 
 c := e.NewContext(req, rec) 
 c.SetPath(“article/:id”) 
 c.SetParamNames(“id”) 
 c.SetParamValues(strconv.Itoa(num)) 
 handler:= articleHttp.ArticleHandler{
            AUsecase: mockUCase,
            Helper: httpHelper.HttpHelper{}
 } 
 handler.GetByID(c) 
 assert.Equal(t, http.StatusOK, rec.Code) 
 mockUCase.AssertCalled(t, “GetByID”, int64(num))
}
```

# 最终输出和合并

在完成所有的层并通过测试后，你应该在根项目的 main.go 中合并到一个系统中。这里你将定义并创建环境需要的所有需要，并将它们合并成一个。

我的 main.go 例子：

```go
package main

import (
	"database/sql"
	"fmt"
	"net/url"

	httpDeliver "github.com/bxcodec/go-clean-arch/article/delivery/http"
	articleRepo "github.com/bxcodec/go-clean-arch/article/repository/mysql"
	articleUcase "github.com/bxcodec/go-clean-arch/article/usecase"
	cfg "github.com/bxcodec/go-clean-arch/config/env"
	"github.com/bxcodec/go-clean-arch/config/middleware"
	_ "github.com/go-sql-driver/mysql"
	"github.com/labstack/echo"
)

var config cfg.Config

func init() {
	config = cfg.NewViperConfig()

	if config.GetBool(`debug`) {
		fmt.Println("Service RUN on DEBUG mode")
	}

}

func main() {

	dbHost := config.GetString(`database.host`)
	dbPort := config.GetString(`database.port`)
	dbUser := config.GetString(`database.user`)
	dbPass := config.GetString(`database.pass`)
	dbName := config.GetString(`database.name`)
	connection := fmt.Sprintf("%s:%s@tcp(%s:%s)/%s", dbUser, dbPass, dbHost, dbPort, dbName)
	val := url.Values{}
	val.Add("parseTime", "1")
	val.Add("loc", "Asia/Jakarta")
	dsn := fmt.Sprintf("%s?%s", connection, val.Encode())
	dbConn, err := sql.Open(`mysql`, dsn)
	if err != nil && config.GetBool("debug") {
		fmt.Println(err)
	}
	defer dbConn.Close()
	e := echo.New()
	middL := middleware.InitMiddleware()
	e.Use(middL.CORS)

	ar := articleRepo.NewMysqlArticleRepository(dbConn)
	au := articleUcase.NewArticleUsecase(ar)

	httpDeliver.NewArticleHttpHandler(e, au)

	e.Start(config.GetString("server.address"))
}
```

可以看到，每层和他的依赖合并为了一个。

# 结论 ：

* 简而言之，如果合并到一个图表中，可以看到如下图表：

  ![图片2-图表](https://miro.medium.com/max/911/1*GQdkAd7IwIwOWW-WLG5ikQ.png)

* 这儿使用的每个库，你可以修改为你自己的。因为整理架构的重点是：无论你的库是什么，架构是整洁的，可测试的，也是独立的。

# 项目样例

项目样例放在 [https://github.com/bxcodec/go-clean-arch](https://github.com/bxcodec/go-clean-arch)

项目中使用的库：

* Glide: 包管理 
* 位于 github.com/DATA-DOG/go-sqlmock
* Testiy: 为了测试
* 为交互层的 Echo Labstack(Golang Web 框架)
* Viper: 环境变量配置

关于整理架构的更多阅读：

* 这篇文章的第二部分：https://hackernoon.com/trying-clean-architecture-on-golang-2-44d615bf8fdf
* https://8thlight.com/blog/uncle-bob/2012/08/13/the-clean-architecture.html
* http://manuel.kiessling.net/2012/09/28/applying-the-clean-architecture-to-go-applications/ Golang 整理架构的另一个版本

[email](iman.tumorang@gmail.com)

