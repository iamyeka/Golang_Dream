## 前言

> 嗨，我小asong又回来了。托了两周没有更新，最近比较忙，再加上自己懒，所以嘛，嗯嗯，你们懂的。不过我今天的带来的分享，绝对干货，在实际项目中开发也是需要用到的，所以为了能够讲明白，我特意写了一个样例，仅供参考。本文会围绕样例进行展开学习，已上传[github](https://github.com/asong2020/Golang_Dream/tree/master/wire_cron_example)，可自行下载。好了，不说废话了，知道你们迫不及待了，我们直接开始吧！！！



###  wire

#### 依赖注入

在介绍wire之前，我们先来了解一下什么是依赖注入。使用过Spring的同学对这个应该不会陌生。其中控制反转（IOC）最常见的方式就叫做依赖注入。将依赖的类作为行参放入依赖中的类就成为依赖注入。这么说可能你们不太懂。用一句大白话来说，一个实例化的对象，本来我接受各种参数来构造一个对象，现在只接受一个参数，对对象的依赖是注入进来的，和它的构造方式解耦了。构造他这个控制操作也交给了第三方，即控制反转。举个例子：go中是没有类的概念的，以结构体的形式体现。假设我们现在船类，有个浆类，我们现在想要设置该船有12个浆，那我们可以写出如下代码：

```go
package main

import (
	"fmt"
)

type ship struct {
	pulp *pulp
}
func NewShip(pulp *pulp) *ship{
	return &ship{
		pulp: pulp,
	}
}
type pulp struct {
	count int
}

func Newpulp(count int) *pulp{
	return &pulp{
		count: count,
	}
}

func main(){
	p:= Newpulp(12)
	s := NewShip(p)
	fmt.Println(s.pulp.count)
}


```

相信你们一眼就看出问题了，每当需求变动时，我们都要重新创建一个对象来指定船桨，这样的代码不易维护，我们变通一下。

```go
package main

import (
	"fmt"
)

type ship struct {
	pulp *pulp
}
func NewShip(pulp *pulp) *ship{
	return &ship{
		pulp: pulp,
	}
}
type pulp struct {
	count int
}

func Newpulp() *pulp{
	return &pulp{
	}
}

func (c *pulp)set(count int)  {
	c.count = count
}

func (c *pulp)get() int {
	return c.count
}

func main(){
	p:= Newpulp()
	s := NewShip(p)
	s.pulp.set(12)
	fmt.Println(s.pulp.get())
}

```

这个代码的好处就在于代码松耦合，易维护，还易测试。如果我们现在更换需求了，需要20个船桨，直接`s.pulp.set(20)`就可以了。



#### wire的使用

`wire`有两个基础概念，`Provider`（构造器）和`Injector`（注入器）。`Provider`实际上就是创建函数，大家意会一下。我们上面`InitializeCron`就是`Injector`。每个注入器实际上就是一个对象的创建和初始化函数。在这个函数中，我们只需要告诉`wire`要创建什么类型的对象，这个类型的依赖，`wire`工具会为我们生成一个函数完成对象的创建和初始化工作。

拉了这么长，就是为了引出wire，上面的代码虽然是实现了依赖注入，这是在代码量少，结构不复杂的情况下，我们自己来实现依赖是没有问题的，当结构之间的关系变得非常复杂的时候，这时候手动创建依赖，然后将他们组装起来就会变的异常繁琐，并且很容出错。所以wire的作用就来了。在使用之前我们先来安装一下wire。

```go
$ go get github.com/google/wire/cmd/wire
```

执行该命令会在`$GOPATH/bin`中生成一个可执行程序`wire`，这个就是代码生成器。别忘了吧`$GOPATH/bin`加入系统环境变量`$PATH`中。

先根据上面的简单例子，我们先来看看wire怎么用。我们先创建一个wire文件，文件内容如下：

```go
//+build wireinject

package main

import (
	"github.com/google/wire"
)

type Ship struct {
	Pulp *Pulp
}
func NewShip(pulp *Pulp) *Ship {
	return &Ship{
		pulp: pulp,
	}
}
type Pulp struct {
	Count int
}

func NewPulp() *Pulp {
	return &Pulp{
	}
}

func (c *Pulp)set(count int)  {
	c.count = count
}

func (c *Pulp)get() int {
	return c.count
}

func InitShip() *Ship {
	wire.Build(
		NewPulp,
		NewShip,
		)
	return &Ship{}
}

func main(){

}
```

其中`InitShip`这个函数的返回值就是我们需要创建的对象类型，`wire`只需要知道类型，返回什么不重要。在函数中我们调用了`wire.Build()`将创建`ship`所依赖的的类型构造器传进去。这样我们就编写好了，现在我们需要到控制台执行`wire`。

```go
$ wire
wire: asong.cloud/Golang_Dream/wire_cron_example/ship: wrote /Users/asong/go/src/asong.cloud/Golang_Dream/wire_cron_example/ship/wire_gen.go
```

我们看到生成了wire_gen.go这个文件：

```go
// Code generated by Wire. DO NOT EDIT.

//go:generate wire
//+build !wireinject

package main

// Injectors from mian.go:

func InitShip() *Ship {
	pulp := NewPulp()
	ship := NewShip(pulp)
	return ship
}

// mian.go:

type Ship struct {
	pulp *Pulp
}

func NewShip(pulp *Pulp) *Ship {
	return &Ship{
		pulp: pulp,
	}
}

type Pulp struct {
	count int
}

func NewPulp() *Pulp {
	return &Pulp{}
}

func (c *Pulp) set(count int) {
	c.count = count
}

func (c *Pulp) get() int {
	return c.count
}

func main() {

}

```

可以看出来，生成的这个文件根据刚才定义的生成了`InitShip（）`这个函数，依赖绑定关系也都实现了，我们直接调用这个函数，就可以了，省去了大量代码自己去实现依赖绑定关系。

**注意：**如果你是第一次使用wire，那么你一定会遇到一个问题，生成的代码和原来的代码会出现冲突，因为都定义相同的函数`func InitShip() *Ship`，所以这里需要在原文件中首行添加`//+build wireinject`，并且还要与包名有空行，这样就解决了冲突。

上面的例子还算是简单，下面我们来看一个比较多一点的例子，我们在日常web后台开发时，代码都是有分层的，比较熟悉的有`dao`、`service`、`controller`、`model`等等。其实`dao`、`service`、`controller`，是有调用先后顺序的。`controller`调用`service`层，`service`层调用`dao`层，这就形成了依赖关系，我们在实际开发中，通过分层依赖注入的方式，更加层次分明，且代码是易于维护的。所以，我写了一个样例，让我们来学习一下怎么使用。这个采用`cron`定时任务代替`controller`来代替`controller`，`cron`定时任务我会在后文进行讲解。

```go
//+build wireinject

package wire

import (
	"github.com/google/wire"

	"asong.cloud/Golang_Dream/wire_cron_example/config"
	"asong.cloud/Golang_Dream/wire_cron_example/cron"
	"asong.cloud/Golang_Dream/wire_cron_example/cron/task"
	"asong.cloud/Golang_Dream/wire_cron_example/dao"
	"asong.cloud/Golang_Dream/wire_cron_example/service"
)

func InitializeCron(mysql *config.Mysql)  *cron.Cron{
	wire.Build(
		dao.NewClientDB,
		dao.NewUserDB,
		service.NewUserService,
		task.NewScanner,
		cron.NewCron,
		)
	return &cron.Cron{}
}
```

我们来看看这段代码，`dao.NewClientDB`即创建一个`*sql.DB`对象，依赖于`mysql`的配置文件，`dao.NewUserDB`即创建一个`*UserDB`对象，他依赖于`*sql.DB`，`service.NewUserService`即创建一个`UserService`对象，依赖于`*UserDB`对象，`task.NewScanner`创建一个`*Scanner`对象，他依赖于`*UserService`对象，`cron。NewCron`创建一个`*Cron`对象，他依赖于`*Scanner`对象，其实这里是层层绑定关系，一层调一层，层次分明，且易于代码维护。

好啦，基本使用就介绍到这里，我们接下来我们学习一下`cron`。



### cron

#### 基础学习

我们在日常开发或运维中，经常遇到一些周期性执行的任务或需求，例如：每一段时间执行一个脚本，每个月执行一个操作。linux给我们提供了一个便捷的方式—— crontab定时任务；crontab就是一个自定义定时器，我们可以利用 crontab 命令在固定的间隔时间执行指定的系统指令或 shell script 脚本。而这个时间间隔的写法与我们平常用到的cron 表达式相似。作用都是通过利用字符或命令去设置定时周期性地执行一些操作.

知道了基本概念，我们就来介绍一下cron表达式。常用的cron规范格式有两种：一种是“标准”cron格式，由`cron linux`系统程序使用，还有一种是`Quartz Scheduler`使用cron格式。这两种的差别就在一个是支持`seconds`字段的，一个是不支持的，不过差距不是很大，我们接下来的讲解都带上`seconds`这个字段，没有影响的。

**cron** 表达式是一个字符串，该字符串由 `6` 个空格分为 `7` 个域，每一个域代表一个时间含义。 格式如下：

```
[秒] [分] [时] [日] [月] [周] [年]
```

[年]的部分通常是可以省略的，实际上由前六部分组成。

关于各部分的定义，我们以一个表格的形式呈现：

| 域   | 是否必填 | 值以及范围      | 通配符        |
| ---- | -------- | --------------- | ------------- |
| 秒   | 是       | 0-59            | , - * /       |
| 分   | 是       | 0-59            | , - * /       |
| 时   | 是       | 0-23            | , - * /       |
| 日   | 是       | 1-31            | , - * ? / L W |
| 月   | 是       | 1-12 或 JAN-DEC | , - * /       |
| 周   | 是       | 1-7 或 SUN-SAT  | , - * ? / L # |
| 年   | 否       | 1970-2099       | , - * /       |

看这个值的范围，还是很好理解的，最难理解的是通配符，我们着重来讲一下通配符。

- `,`  这里指的是在两个以上的时间点中都执行，如果我们在 “分” 这个域中定义为 `5,10,15` ，则表示分别在第5分，第10分 第15分执行该定时任务。


- `-`  这个比较好理解就是指定在某个域的连续范围，如果我们在 “时” 这个域中定义 `6-12`，则表示在6到12点之间每小时都触发一次，用 `,` 表示 `6,7,8,9,10,11,12`

- `*`  表示所有值，可解读为 “每”。 如果在“日”这个域中设置 `*`,表示每一天都会触发。

- `?`  表示不指定值。使用的场景为不需要关心当前设置这个字段的值。例如:要在每月的8号触发一个操作，但不关心是周几，我们可以这么设置 `0 0 0 8 * ?`

- `/`  在某个域上周期性触发，该符号将其所在域中的表达式分为两个部分，其中第一部分是起始值，除了秒以外都会降低一个单位，比如 在 “秒” 上定义 `5/10` 表示从 第 5 秒开始 每 10 秒执行一次，而在 “分” 上则表示从 第 5 秒开始 每 10 分钟执行一次。

- `L`  表示英文中的**LAST** 的意思，只能在 “日”和“周”中使用。在“日”中设置，表示当月的最后一天(依据当前月份，如果是二月还会依据是否是润年), 在“周”上表示周六，相当于”7”或”SAT”。如果在”L”前加上数字，则表示该数据的最后一个。例如在“周”上设置”7L”这样的格式,则表示“本月最后一个周六”

- `W`  表示离指定日期的最近那个工作日(周一至周五)触发，只能在 “日” 中使用且只能用在具体的数字之后。若在“日”上置”15W”，表示离每月15号最近的那个工作日触发。假如15号正好是周六，则找最近的周五(14号)触发, 如果15号是周未，则找最近的下周一(16号)触发.如果15号正好在工作日(周一至周五)，则就在该天触发。如果是 “1W” 就只能往本月的下一个最近的工作日推不能跨月往上一个月推。

- `#`  表示每月的第几个周几，只能作用于 “周” 上。例如 ”2#3” 表示在每月的第三个周二。



学习了通配符，下面我们来看几个例子：

- 每天10点执行一次：`0 0 10 * * *`
- 每隔10分钟执行一次：`0 */10 * * *`
- 每月1号凌晨3点执行一次：`0 0 3 1 * ?`
- 每月最后一天23点30分执行一次：`0 30 23 L * ?`
- 每周周六凌晨3点实行一次：`0 0 3 ? * L`
- 在30分、50分执行一次：`0 30，50 * * * ?`



#### go中使用cron

前面我们学习了基础，现在我们想要在go项目中使用定时任务，我们该怎么做呢？`github`上有一个星星比较高的一个`cron`库，我们可以使用`robfig/cron`这个库开发我们的定时任务。

学习之前，我们先来安装一下`cron`

```shell
$ go get -u github.com/robfig/cron/v3
```

这是目前比较稳定的版本，现在这个版本是采用标准规范的，默认是不带`seconds`，如果想要带上字段，我们需要创建cron对象是去指定。一会展示。我们先来看一个简单的使用：

```go
package main

import (
  "fmt"
  "time"

  "github.com/robfig/cron/v3"
)

func main() {
  c := cron.New()

  c.AddFunc("@every 1s", func() {
    fmt.Println("task start in 1 seconds")
  })

  c.Start()
  select{}
}
```

这里我们使用`cron.New`创建一个cron对象，用于管理定时任务。调用`cron`对象的`AddFunc()`方法向管理器中添加定时任务。`AddFunc()`接受两个参数，参数 1 以字符串形式指定触发时间规则，参数 2 是一个无参的函数，每次触发时调用。`@every 1s`表示每秒触发一次，`@every`后加一个时间间隔，表示每隔多长时间触发一次。例如`@every 1h`表示每小时触发一次，`@every 1m2s`表示每隔 1 分 2 秒触发一次。`time.ParseDuration()`支持的格式都可以用在这里。调用`c.Start()`启动定时循环。

注意一点，因为`c.Start()`启动一个新的 goroutine 做循环检测，我们在代码最后加了一行`select{}`防止主 goroutine 退出。



上面我们定义时间时，使用的是`cron`预定义的时间规则，那我们就学习一下他都有哪些预定义的一些时间规则：

- `@yearly`：也可以写作`@annually`，表示每年第一天的 0 点。等价于`0 0 1 1 *`；
- `@monthly`：表示每月第一天的 0 点。等价于`0 0 1 * *`；
- `@weekly`：表示每周第一天的 0 点，注意第一天为周日，即周六结束，周日开始的那个 0 点。等价于`0 0 * * 0`；
- `@daily`：也可以写作`@midnight`，表示每天 0 点。等价于`0 0 * * *`；
- `@hourly`：表示每小时的开始。等价于`0 * * * *`。

`cron`也是支持固定时间间隔的，格式如下：

```go
@every <duration>
```

含义为每隔`duration`触发一次。`<duration>`会调用`time.ParseDuration()`函数解析，所以`ParseDuration`支持的格式都可以。



#### 项目使用

因为我自己写的项目是通过实现job接口来加入定时任务，所以下面我们再来介绍一下Job接口的使用，除了直接将无参函数作为回调外，`cron`还支持`job`接口：

```
type Job interface{
	Run()
}
```

我们需要实现这个接口，这里我就以我写的例子来做演示吧，我现在这个定时任务是周期扫DB表中的数据，实现任务如下：

```go
package task

import (
	"fmt"

	"asong.cloud/Golang_Dream/wire_cron_example/service"
)

type Scanner struct {
	lastID uint64
	user *service.UserService
}

const  (
	ScannerSize = 10
)

func NewScanner(user *service.UserService)  *Scanner{
	return &Scanner{
		user: user,
	}
}

func (s *Scanner)Run()  {
	err := s.scannerDB()
	if err != nil{
		fmt.Errorf(err.Error())
	}
}

func (s *Scanner)scannerDB()  error{
	s.reset()
	flag := false
	for {
		users,err:=s.user.MGet(s.lastID,ScannerSize)
		if err != nil{
			return err
		}
		if len(users) < ScannerSize{
			flag = true
		}
		s.lastID = users[len(users) - 1].ID
		for k,v := range users{
			fmt.Println(k,v)
		}
		if flag{
			return nil
		}
	}
}

func (s *Scanner)reset()  {
	s.lastID = 0
}
```

上面是实现`Run`方法的部分，之后我们还需要调用`cron`对象的AddJob方法将`Scanner`对象添加到定时管理器中。

```go
package cron

import (
	"github.com/robfig/cron/v3"

	"asong.cloud/Golang_Dream/wire_cron_example/cron/task"
)

type Cron struct {
	Scanner *task.Scanner
	Schedule *cron.Cron
}

func NewCron(scanner *task.Scanner) *Cron {
	return &Cron{
		Scanner: scanner,
		Schedule: cron.New(),
	}
}

func (s *Cron)Start()  error{
	_,err := s.Schedule.AddJob("*/1 * * * *",s.Scanner)
	if err != nil{
		return err
	}
	s.Schedule.Start()
	return nil
}
```

实际上`AddFunc()`方法内部也调用了`AddJob()`方法。首先，`cron`基于`func()`类型定义一个新的类型`FuncJob`：

```go
// cron.go
type FuncJob func()
```

然后让`FuncJob`实现`Job`接口：

```go
// cron.go
func (f FuncJob) Run() {
  f()
}
```

在`AddFunc()`方法中，将传入的回调转为`FuncJob`类型，然后调用`AddJob()`方法：

```go
func (c *Cron) AddFunc(spec string, cmd func()) (EntryID, error) {
  return c.AddJob(spec, FuncJob(cmd))
}
```



好啦，基本的使用到这里我们就讲解完了，最后再补充一下最后一个知识点，也就时间规范的问题，默认`v3`版本是不带`seconds`字段的，要想使用需要这样使用

```go
cron.New(cron.WithSeconds())
```

创建对象的传入这个参数就可以了。

好啦。我想要讲解的完事了，代码就不运行了，已上传github，可自行下载学习：https://github.com/asong2020/Golang_Dream/tree/master/wire_cron_example



## 总结

今天的文章就到这里了，这一篇总结的并不全，只是达到入门的一个效果，想要继续深入，还需要各位小伙伴自行看文档学习呦。学会看官方文档，才能进步更多的呦。就比如时间规范这里，如果不看文档，我就不会知道现在使用的时间规范是什么的，所以还是要养成看文档的好习惯。打个预告，下一期是go-elastic的教程，有需要的小伙伴可以关注一下。

**结尾给大家发一个小福利吧，最近我在看[微服务架构设计模式]这一本书，讲的很好，自己也收集了一本PDF，有需要的小伙可以到自行下载。获取方式：关注公众号：[Golang梦工厂]，后台回复：[微服务]，即可获取。**

**我翻译了一份GIN中文文档，会定期进行维护，有需要的小伙伴后台回复[gin]即可下载。**

**我是asong，一名普普通通的程序猿，让我一起慢慢变强吧。欢迎各位的关注，我们下期见~~~**

![](https://song-oss.oss-cn-beijing.aliyuncs.com/wx/qrcode_for_gh_efed4775ba73_258.jpg)

推荐往期文章：

- [听说你还不会jwt和swagger-饭我都不吃了带着实践项目我就来了](https://mp.weixin.qq.com/s/z-PGZE84STccvfkf8ehTgA)
- [掌握这些Go语言特性，你的水平将提高N个档次(二)](https://mp.weixin.qq.com/s/7yyo83SzgQbEB7QWGY7k-w)
- [go实现多人聊天室，在这里你想聊什么都可以的啦！！！](https://mp.weixin.qq.com/s/H7F85CncQNdnPsjvGiemtg)
- [grpc实践-学会grpc就是这么简单](https://mp.weixin.qq.com/s/mOkihZEO7uwEAnnRKGdkLA)
- [go标准库rpc实践](https://mp.weixin.qq.com/s/d0xKVe_Cq1WsUGZxIlU8mw)
- [2020最新Gin框架中文文档 asong又捡起来了英语，用心翻译](https://mp.weixin.qq.com/s/vx8A6EEO2mgEMteUZNzkDg)
- [基于gin的几种热加载方式](https://mp.weixin.qq.com/s/CZvjXp3dimU-2hZlvsLfsw)
- [boss: 这小子还不会使用validator库进行数据校验，开了～～～](https://mp.weixin.qq.com/s?__biz=MzIzMDU0MTA3Nw==&mid=2247483829&idx=1&sn=d7cf4f46ea038a68e74a4bf00bbf64a9&scene=19&token=1606435091&lang=zh_CN#wechat_redirect)