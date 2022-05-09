## Modules

模块是**相关联的包**作为**一个单元**被一起**版本化**后的组合。

每个模块都有着明确的依赖需求，并且能够创建可复现的构建。一个仓库里可以有多个模块，一个模块里面可以有多个包。

对于go modules，原则上所创建的目录不应该放在GOPATH之中。

### 命令

#### go mod 命令

|      命令       |               作用               |
| :-------------: | :------------------------------: |
|   go mod init   |         生成 go.mod 文件         |
| go mod download | 下载 go.mod 文件中指明的所有依赖 |
|   go mod tidy   |          整理现有的依赖          |
|  go mod graph   |    以图形化查看现有的依赖结构    |
|   go mod edit   |         编辑 go.mod 文件         |
|  go mod vendor  | 导出项目所有的依赖到 vendor 目录 |
|  go mod verify  |     校验一个模块是否被篡改过     |
|   go mod why    |     查看为什么需要依赖某模块     |

#### go get

|      命令       |                             作用                             |
| :-------------: | :----------------------------------------------------------: |
|     go get      | 拉取依赖，会进行指定性拉去（更新），并不会更新所依赖的其他的模块 |
|    go get -u    | 更新现有的依赖，会强制更新它所依赖的其他全部模块，不包括自身 |
| go get -u ./... |  更新所有直接依赖和间接依赖的模块版本，包括单元测试中用到的  |

##### go get 选择具体版本

|              命令               |                         作用                         |
| :-----------------------------: | :--------------------------------------------------: |
| go get golang.org/x/text@latest |       拉取最新的版本。若存在 tag，则优先使用。       |
| go get golang.org/x/text@master |           拉取 master 分支的最新 commit。            |
| go get golang.org/x/text@v0.3.2 |            拉取 tag 为 v0.3.2 的 commit。            |
| go get golang.org/x/text@342b2e | 拉取 hash 为 342b2e 的 commit，最终会被转换为 v0.3.2 |

go get 在没有指定任何版本的情况下，它的版本选择规则是怎样的，也就是为什么 go get 来取的是 v0.0.0，它什么时候会拉取正常带版本号的 tags 呢。实际上这需要区分两种情况：

1. 所拉取的模块有发布 tags：
   - 如果只有单个模块，那么就取主版本号最大的那个 tag。
   - 如果有多个模块，则推算相应的模块路径，取主版本号最大的那个 tag（子模块的 tag 的模块路径会有前缀要求）
2. 所拉取的模块没有发布过 tags：
   - 默认取主分支最新一次 commit 的 commit hash。

### 不同版本号的导入路径

Go modules在主版本号为 v0 和 v1 的情况下，省略版本号，而在主版本号为 v2 及以上则需要明确指定出主版本号，否则会出现冲突。

|  tag   |         模块导入路径         |
| :----: | :--------------------------: |
| v0.0.0 |  github.com/eddycjy/mquote   |
| v1.0.0 |  github.com/eddycjy/mquote   |
| v2.0.0 | github.com/eddycjy/mquote/v2 |

### Go modules 的语义化版本控制

我们不断地在 Go modules 的使用中提到版本号，其实质上被称为“语义化版本”，假设我们的版本号是 v1.2.3，其版本格式为“主版本号.次版本号.修订号”，其中每项的规则如下：

1. 主版本号：当你做了不兼容的API的修改后，修改主版本号。
2. 次版本号：当你做了向下兼容的新的功能，修改次版本号。
3. 修订号：当你做了向下兼容的问题修正，修改修订号。

假设你是先行版本号或特殊情况，可以将版本信息追加到“主版本号.次版本号.修订号”的后面，作为延伸，比如 ：v1.2.3-pre：其中 pre 就是先行版本。

### go.mod

go.mod文件中主要有四个部分组成：

- module

  用来声明当前module，如果当前版本大于v1的话，还需要在尾部显示的声明`/vN`.

  ​	module /path/to/your/mod/v2

  ​	module github.com/Xuanwo/go-mod-intro/v2

- require

  这是最为常用的部分，在mod之后可以写任意有效的、能指向一个引用的字符串，比如Tag、Branch、Commit 或者是使用`last` 来表示引用最新的 commit。如果对应的引用刚好是一个 Tag 的话，这个字符串会被重写为对应的 tag；如果不是的话，这个字符串会被规范化为形如 `v2.0.0-20180128182452-d3ae77c26ac8` 这样的字符串。我们后面会发现这个字符串与底层的 mod 存储形式是对应的。

  ​	require /your/mod tag/branch/commit

  ​	require github.com/google/go-github/v24 v24.0.1

  ​	require gopkg.in/urfave/cli.v2 v2.0.0-20180128182452-d3ae77c26ac8

- replace

  `replace` 这边的花样比较多，主要是两种，一个是与 `require` 类似，可以指向另外一个repo，另一种是可以指向本地的一个目录。加了 `require` 的话，go 在编译的时候就会使用对应的项目代码来替换。需要注意的是这个只作用于当前模块的构建，其他模块的 replace 对它不生效。同理，它的 replace 对其它模块也不生效。

  需要额外注意的是，如果引用一个本地路径的话，那这个目录下必须要有 `go.mod` 文件，这个路径可以是绝对路径，也可以是相对路径。

  ​	replace original_name => real_name/branch/commit

  ​	replace origin_name => local_path

  ​	

  ​	replace test.dev/common => git.example.com/bravo/common.git v0.0.0-20190520075948-958a278528f8

  ​	replace test.dev/common => ../../another-project/common-go

  ​	replace github.com/qiniu/x => github.com/Xuanwo/qiniu_x v0.0.0-200190416044656-4dd63e731f37

- exclude

  这个用的比较少，主要是为了能在构建的时候排除特定的版本，跟 `replace` 一样，只能作用于当前模块的构建。

  ​	exclude /your/mod tag/branch/commit

## 项目结构

引用 [github](https://github.com/golang-standards/project-layout/blob/master/README_zh.md)

随着项目的增长，请记住保持代码结构良好非常重要，否则你最终会得到一个凌乱的代码，这其中就包含大量隐藏的依赖项和全局状态。当有更多的人参与这个项目时，你将需要更多的结构。这时候，介绍一种管理包/库的通用方法就很重要。克隆存储库，保留你需要的内容，删除其他所有的内容！仅仅因为它在那里并不意味着你必须全部使用它。这些模式都没有在每个项目中使用。甚至 vendor 模式也不是通用的。

Go 1.14 Go Modules 终于可以投入生产了。除非你有特定的理由不适用它们，否则使用 Go Modules。如果你使用，就无需担心 `$GOPATH` 以及项目放置的位置。存储库中的 `go.mod` 文件基本假定你的项目托管在 Github 上，但这不是必须的。模块路径可以是任何地方，尽管第一个模块路径组件的名称中应该有一个点（当前版本的 Go 不再强制使用该模块，但是如果使用稍旧的版本，如果没有 mod 文件构建失败的话，不要惊讶）。如果你想知道更多信息，请参阅 Issues [`37554`](https://github.com/golang/go/issues/37554) 和 [`32819`](https://github.com/golang/go/issues/32819) 。

此项目布局时通用的，并且不会尝试加强一个特定的 Go 包结构。

### Go 目录

#### /cmd

本项目的主干。

每个应用程序的目录名应该与你想要的可执行文件的名称匹配（例如，`/cmd/myapp`）。

不要在这个目录中放置太多的代码。如果你认为代码可以导入并在其他项目中使用，那么应该位于 `/pkg/` 目录中。如果代码不是可重用的，或者你不希望其他人重用它，请将该代码放到 `/internal` 目录中。你会惊讶于别人怎么做，所以要明确你的意图！

通常有一个小的 `main` 函数，从 `/internal` 和 `/pkg` 目录导入和调用代码，除此之外没有别的东西。

有关示例，请参阅 [`/cmd` ](https://github.com/moby/moby/tree/master/cmd/dockerd)目录。

#### /internal

私用应用程序和库代码。这是你不希望其他人在其应用程序或库中导入的代码。请注意，这个布局模式是由 Go 编译器本身执行的。请参阅Go 1.4 [`release notes`](https://golang.org/doc/go1.4#internalpackages) 。注意，你并不局限于顶级 `internal` 目录。在项目树的任何级别上都可以有多个内部目录。

你可以选择向 internal 包中添加一些额外的结构，以分隔共享和非共享的内部代码。这不是必须的（特别是对于较小的项目），但是最好有有可视化的线索来显示预期的包的用途。你的实际应用程序代码可以放在 `/intrenal/app` 目录下（例如 `/internal/app/myapp` ），这些应用程序共享的代码可以放在 `/internal/pkg` 目录下（例如 `/internal/pkg/myprivlib`）。

#### /pkg

外部应用程序可以使用的库代码（例如 `/pkg/mypubliclib`）。其他项目会导入这些库，希望它们能正常工作，所以在这里放东西之前要三思。注意，`internal` 目录是确保私有包不可导入的更好方法，因为它是由 Go 强制执行的。`/pkg` 目录仍然是一种很好的方式，可以显示地表示该目录的代码对于其他人来说是安全使用的好方法。由 Travis Jeffery 撰写的 [`I'll take pkg over internal`](https://travisjeffery.com/b/2019/11/i-ll-take-pkg-over-internal/) 博客文章提供了 `pkg` 和 `internal` 目录的一个很好的概述，以及什么时候使用它们是有意义的。

当根目录包含大量非 Go 组件和目录时，这也是一种将 Go 代码分组到一个位置的方法，这使得运行各种 Go 工具变得更加容易（正如在这些演讲中提到的那样: 来自 GopherCon EU 2018 的 [`Best Practices for Industrial Programming`](https://www.youtube.com/watch?v=PTE4VJIdHPg) , [GopherCon 2018: Kat Zien - How Do You Structure Your Go Apps](https://www.youtube.com/watch?v=oL6JBUk6tj0) 和 [GoLab 2018 - Massimiliano Pippi - Project layout patterns in Go](https://www.youtube.com/watch?v=3gQa1LWwuzk) ）。

如果你想查看哪个流行的 Go 存储库使用此项目布局模式，请查看 `/pkg` 目录。这是一种常见的布局模式，但并不是所有人都接受它，一些 Go 社区的人也不推荐它。

如果你的应用程序项目真的很小，并且额外的嵌套并不能增加多少价值（除非你真的想要），那就不要使用它。当它变得足够大时，你的根目录会变得非常繁琐时（尤其是当你有很多非 Go 应用组件时），请考虑以下。

#### /vendor

应用程序依赖项（手动管理或使用你喜欢的依赖管理工具，如新的内置 [`Go Modules`](https://github.com/golang/go/wiki/Modules) 功能)。`go mod vendor` 命令将为你创建 `/vendor` 目录。请注意，如果未使用默认情况下处于启用状态的 Go 1.14，则可能需要在 `go build` 命令中添加 `-mod=vendor` 标志。

如果你正在构建一个库，那么不要提交你的应用程序的依赖项。

注意，自从 1.13 以后，Go 还启用了模块代理功能（默认使用 [`https://proxy.golang.org`](https://proxy.golang.org) 作为它们的模块代理服务器）。在[`here`](https://blog.golang.org/module-mirror-launch) 阅读更多关于它的信息，看看它是否符合你的所有要求和约束。如果需要，那么你根本不需要 vendor 目录。

国内模块代理功能默认是被墙的，七牛云有维护专门的 [`模块代理`](https://github.com/goproxy/goproxy.cn/blob/master/README.zh-CN.md) 。

### 服务应用程序目录

#### /api

OpenAPI/Swagger规范，JSON模式文件，协议定义文件。

有关示例，请参见 [`/api`](https://github.com/golang-standards/project-layout/blob/master/api/README.md) 目录。

### Web应用程序目录

#### /web

特定于 Web 应用程序的组件：静态 Web 资产，服务器端模板和 SPAs。

### 通用应用目录

#### /configs

配置文件模板或默认配置。

将你的 confd 或 consul-template 模板文件放在这里。

#### /init

System init（systemd， upstart， sysv）和 process manager/supervisor（runit， supervisor）配置。

#### /scripts

执行各种构建、安装、分析等操作的脚本。

这些脚本使得根级别的 Makefile 变得小而简单（例如， 

[`https://github.com/hashicorp/terraform/blob/master/Makefile`](https://github.com/hashicorp/terraform/blob/master/Makefile) )。

有关示例，请参见  [`/scripts`](https://github.com/golang-standards/project-layout/blob/master/scripts/README.md) 目录。

#### /build

打包和持续集成。

将你的云（AMI）、容器（Docker）、操作系统（deb、rpm、pkg）包配置和脚本放在 `/build/package` 目录下。

将你的 CI（travis、circle、drone）配置和脚本放在 `/build/ci`目录中。请注意，有些 CI 工具（例如 Travis CI）对配置文件的位置非常挑剔。尝试将配置文件放在 `/build/ci` 目录中，将它们连接到 CI 工具期望它们的位置（如果可能的话）。

#### /deployments(部署)

laaS、PaaS、系统和容器编排部署配置和模板（docker-compose、kubernetes/helm、mesos、terraform、bosh）。注意，在一些存储库中（特别是使用 kubernetes 部署的应用程序），这个目录被称为 `/deploy`

#### /test

额外的外部测试应用程序和测试数据。你可以随时根据需求构造 `/test` 目录。对于较大的项目，有一个数据子目录是有意义的。例如，你可以使用 `/test/data` 或 `/test/testdata`（如果你需要忽略项目中的内容）。请注意，Go 还会忽略以 “.” 和 “_” 开头的目录或文件，因此在如何命名测试数据目录方面有更大的灵活性。

有关示例，请参见 [`/test`](https://github.com/golang-standards/project-layout/blob/master/test/README.md) 目录。

### 其他目录

#### /doc

设计和用户文档（出了 godoc 生成的文档之外）。

有关示例，请参阅 [`/docs`](https://github.com/golang-standards/project-layout/blob/master/docs/README.md) 目录。

#### /tools

这个项目支持的工具。注意，这些工具可以从 `/pkg` 和 `/internal` 目录导入代码。

有关示例，请参见 [`/tools`](https://github.com/golang-standards/project-layout/blob/master/tools/README.md) 目录。

#### /examples

你的应用程序和 / 或公共库的示例。

有关示例，请参见 [`/examples`](https://github.com/golang-standards/project-layout/blob/master/examples/README.md) 目录。

#### /third_party

外部辅助工具，分叉代码和其他第三方工具（例如 Swagger UI）。

#### /githooks

Git hooks。

#### /assets

与存储库一起使用的其他资产（图像、徽标等）。

#### /website

如果你不使用 Github 页面，则在这里防止项目的网站数据。

有关示例，请参见 [`/website`](https://github.com/golang-standards/project-layout/blob/master/website/README.md) 目录。

### 你不应该拥有的目录

#### /src

有些 Go 项目确实有一个 src 文件夹，但这通常发生在开发人员有着 Java 背景，在那里它是一种常见的模式。如果可以的话，尽量不要采用这种 Java 模式。你真的不希望你的 Go 代码 或 Go 项目看起来像 Java。

不要将项目级别的 src 目录与 Go 用于其工作空间的 src 目录(如 [`How to Write Go Code`](https://golang.org/doc/code.html) 中所述)混淆。`$GOPATH` 环境变量指向你的（当前）工作空间（默认情况下，它指向非 windows 系统上的 `$HOME/go`）。这个工作空间包括顶层 `/pkg` 、`/bin` 和  `/src` 目录。你的实际项目最终是 `/src` 下的一个子目录，因此，如果你的项目中有 `/src` 目录，那么项目路径将是这样的： `/some/path/to/workspace/src/your_project/src/your_code.go`。注意，在 Go 1.11中，可以将项目放在 `GOPATH` 之外，但这并不意味着使用这种布局模式是一个好主意。

## context

### 什么是context

context 主要是用来在 goroutine 之间传递上下文信息，包括：取消信号、超时时间、截止时间、k-v 等。

context.Context 类型的值可以协调多个 goroutine 中的代码执行 “取消” 操作，并且可以存储键值对。最重要它是并发安全的。

### 为什么有 context

有时候，由于请求者的放弃或者超过了超时时间，多个协同工作的 goroutine 需要快速退出，然后由系统回收资源。

context 包就是为了解决上面所说的这些问题而开发的：在一组 goroutine 之间传递共享的值、取消信息、deadline...

用简练一些的话来说，在 Go 里，我们不能直接杀死 goroutine，goroutine 的关闭一般会用 channel + select 方式来控制。但是在某些场景下，例如处理一个请求衍生了很多 goroutine，这些 goroutine 之间是互相关联的：需要共享一些全局变量、有共同的 deadline 等，而且可以同时被关闭。这时候在使用 channel + select 就会比较麻烦，这时可以通过 context 来实现。

一句话：context 用来解决 goroutine 之间退出通知、元数据传递的功能

