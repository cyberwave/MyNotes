Go语言将 `makefile` 中的变量写入到程序中
```makefile
all:
	@echo "current compile app ${BIAN_APP_NAME}..."
	go build -ldflags "-X 'git.jd.com/baas/bian/common.AppName=${BIAN_APP_NAME}' " #-o ${BIAN_APP_NAME}
	@echo "app ${BIAN_APP_NAME} compile success!"
clean:
	go clean

.PHONY: all clean ${BIAN_APP_NAME}
```
程序中需要在相关地方定义该变量！！**经尝试，该变量不能定义到 `main.go` ，测试时 `main.go` 是一个单一的文件，未测试将 `main.go` 放到 `main` 包中是否起作用。**

上面 `Makefile` 中的变量，可以使用环境变量环境。

```makefile
export FLASK_ENV=dev
export FLASK_DEBUG=1

dev:
    @echo $(FLASK_ENV)
    @echo $(FLASK_DEBUG)
```

在不同 `target` 下设置不同的环境变量，如下：

```makefile
dev:export FLASK_ENV=dev
dev:export FLASK_DEBUG=1
dev:
    @echo $(FLASK_ENV)
    @echo $(FLASK_DEBUG)

prod:export FLASK_ENV=prod
prod:export FLASK_DEBUG=0
prod:
    @echo $(FLASK_ENV)
    @echo $(FLASK_DEBUG)
```

这样再执行`make dev`和`make prod`时，不同的target下的环境变量就不会干扰了：

```shell
$ make dev
dev
1
$ make prod
prod
0
```

上面文件中的 `@` 在 `Makefile` 中，打印不会显示 `@` 后面的命令本身。

# 使用 patsubpt 注入环境变量

**`subst`语法: **

`$(subst from, to, text)`

对文本执行替换。每次在 `from` 中出现的 `text`，都将被 `to` 替换。下列函数调用

```shell
$(subst ee, EE, feet on the street)
```

的结果为：`fEEt on the strEEt`

**`patsubst`语法: **

`$(patsubst pattern, replacement, text)`

在 `text` 中找到匹配 `pattern` 的以`空格`分隔的单词，使用 `replacement` 替换它。这里，pattern可以包括通配符“%”，表示任意长度的字串。如果replacement中也包含“%”，那么，replacement中的这个“%”将是pattern中的那个“%”所代表的字串。（可以用“\”来转义，以“%”来表示真实含义的“%”字符）函数返回被替换过后的字符串。

**注意：** 注意这里的空格分隔符，它包括空格，tab，换行。如果 text中出现空格，会出现 `/usr/local/go/pkg/tool/darwin_amd64/link: -X flag requires argument of the form importpath.name=value` 的错误

##  使用示例

1. 文档目录

    ```shell
    src
    | -- example
        | -- main.go
        | -- version
            | -- version.go
    ```

    

```go
// main.go
package main
import(
	"example/version"
)
func main(){
  version.ShowInfo()
}
```

```go
package version
import (
	"fmt"
  "runtime"
)
var (
	BuildVersion string
	BuildTime string
	BuildName string
	CommitID string
	CodeBranch string
)
func ShowAppVersion(){
	fmt.Printf("build name:\t%s\n", BuildName)
	fmt.Printf("build version:\t%s\n", BuildVersion)
	fmt.Printf("build time:\t%s\n", BuildTime)
	fmt.Printf("commit id:\t%s\n", CommitID)
	fmt.Printf("code branch:\t%s\n", CodeBranch)
	fmt.Printf("go version:\t%s\n", runtime.Version())
}
```

刚开始使用的Makefile 文件如下：

```makefile
BUILD_VERSION := 5.0
BUILD_NAME := example
BUILD_TIME := $(shell date "+[%F][%T]")
#BUILD_TIME := $(shell date "+%F")
TARGET_DIR := ./
COMMIT_SHA1 := $(shell git rev-parse  HEAD)
GIT_TAG := $(shell git describe --all)
all:
	@echo "current compile app ..."
	go build -ldflags \
    " \
    -X 'git.jd.com/baas/bian/version.BuildVersion=${BUILD_VERSION}' \
    -X 'git.jd.com/baas/bian/version.BuildTime=${BUILD_TIME}' \
    -X 'git.jd.com/baas/bian/version.BuildName=${BUILD_NAME}' \
    -X 'git.jd.com/baas/bian/version.CommitID=${COMMIT_SHA1}' \
    -X 'git.jd.com/baas/bian/version.CodeBranch=${GIT_TAG}' \
    " \
    -o ${BUILD_NAME}
	@echo "app compile success!"
	clean:
		go clean
		rm $(BUILD_NAME) -f
	.PHONY: all clean $(BUILD_NAME)
```

使用 `patsubst` 的Makefile:

```makefile	
BUILD_VERSION := 5.0
BUILD_NAME := example
BUILD_TIME := $(shell date "+[%F][%T]")
#BUILD_TIME := $(shell date "+%F")
TARGET_DIR := ./
COMMIT_SHA1 := $(shell git rev-parse  HEAD)
GIT_TAG := $(shell git describe --all)

METADATA_VAR = BuildVersion=$(BUILD_VERSION)
METADATA_VAR += BuildTime=${BUILD_TIME}
METADATA_VAR += BuildName=$(BUILD_NAME)
METADATA_VAR += CommitID=$(COMMIT_SHA1)
METADATA_VAR += CodeBranch=$(GIT_TAG)
PKGNAME = example
GO_LDFLAGS = $(patsubst %, -X $(PKGNAME)/version.%,$(METADATA_VAR))
all:
	@echo "current compile app ..."
	GOOS=linux GOARCH=amd64 go build -ldflags "$(GO_LDFLAGS)" -o $(BUILD_NAME)
	@echo "app compile success!"
clean:
	go clean
	rm $(BUILD_NAME} -f

.PHONY: all clean ${BUILD_NAME}
```



**参考：**

1. [在Makefile中设置环境变量](https://segmentfault.com/a/1190000008535305)
2. [跟我一起写Makefile:使用变量](https://wiki.ubuntu.org.cn/跟我一起写Makefile:使用变量#.E6.A8.A1.E5.BC.8F.E5.8F.98.E9.87.8F)
3. [跟我一起写 Makefile](https://blog.csdn.net/haoel/article/details/2886)
4. [8.2 Functions for String Substitution and Analysis](https://www.gnu.org/software/make/manual/html_node/Text-Functions.html)

FROM golang:1.10.2-alpine3.7 as builder
WORKDIR /go/src/git.jd.com/baas/bian
COPY . .
ARG appname
RUN echo "https://mirror.tuna.tsinghua.edu.cn/alpine/v3.7/main" > /etc/apk/repositories && \
    echo "https://mirror.tuna.tsinghua.edu.cn/alpine/v3.7/community" >> /etc/apk/repositories && \
    apk --update add make tzdata && \
    make && echo "appname=${appname}"

FROM alpine:3.7
WORKDIR /usr/local/bin

COPY --from=builder /go/src/git.jd.com/baas/bian/bian .
COPY --from=builder /go/src/git.jd.com/baas/bian/resource ./
COPY --from=builder /go/src/git.jd.com/baas/bian/addOn ./
COPY --from=builder /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
RUN chmod +x /usr/local/bin/addOn
RUN echo "Asia/Shanghai" > /etc/timezone

ARG appname
ENV BIAN_APP_FUNCTION=${appname}
EXPOSE 8080
CMD ["${appname}"]