
# Lotus项目源代码导读--Lotus Daemon进程启动框架

lotus是Lotus项目中的全节点命令行二进制。启动全节点的命令是：
```bash
lotus daemon
```
daemon进程的功能是全节点的完整功能，概述一下主要包括如下
* 通过P2P网络发现其他节点，并进行通信
* 从其它全节点获取区块数据，并对每一次区块的消息及区块打包信息进行校验并保存到本地存储
* 为其它全节点提供区块同步的服务
* 钱包管理服务，包括转帐、消息签名等
* RPC API服务

本文就daemon进程的启动过程进行分析

## Command Action

lotus的命令行采用[https://gopkg.in/urfave/cli.v2](https://gopkg.in/urfave/cli.v2)，以下是daemon命令的Action代码逻辑：

```golang
var DaemonCmd = &cli.Command{
 Name:  "daemon",
 Usage: "Start a lotus daemon process",
 Flags: []cli.Flag{
 ...
 },
 Action: func(cctx *cli.Context) error {
   ...
  ...
  // 创建FsRepo,即lotus的Repo
  r, err := repo.NewFS(cctx.String("repo"))

  // 初始化Repo,例如创建初始文件等
  if err := r.Init(repo.FullNode); err != nil && err != repo.ErrRepoExists {
   return xerrors.Errorf("repo init error: %w", err)
  }

  // 获取参数文件
  if err := paramfetch.GetParams(build.ParametersJson(), 0); err != nil {
   return xerrors.Errorf("fetching proof parameters: %w", err)
  }


  // 如果是Genesis启动方式,则加载Genesis文件
  genesis := node.Options()
  if len(genBytes) > 0 {
   genesis = node.Override(new(modules.Genesis), modules.LoadGenesis(genBytes))
  }

  ...
  var api api.FullNode
  // 创建全节点, 从第二个参数起都是Option
  stop, err := node.New(ctx,
   node.FullAPI(&api),
   // 上线, 多数初始化逻辑在此实现
   node.Online(),
   // Repo相关的初始化
   node.Repo(r),

   genesis,

   node.ApplyIf(func(s *node.Settings) bool { return cctx.IsSet("api") },
    node.Override(node.SetApiEndpointKey, func(lr repo.LockedRepo) error {
     apima, err := multiaddr.NewMultiaddr("/ip4/127.0.0.1/tcp/" +
      cctx.String("api"))
     if err != nil {
      return err
     }
     return lr.SetAPIEndpoint(apima)
    })),
   node.ApplyIf(func(s *node.Settings) bool { return !cctx.Bool("bootstrap") },
    node.Unset(node.RunPeerMgrKey),
    node.Unset(new(*peermgr.PeerMgr)),
   ),
  )
  ...
  // 根据repo内的配置创建APIEndpoint, 即网络监听对象
  endpoint, err := r.APIEndpoint()
  ...
  // 启动RPC服务
  return serveRPC(api, stop, endpoint)
 },
```


## 启动框架逻辑

上述代码虽省略了一些关键代码，但粗略一看代码逻辑很少啊，肯定是有大量的逻辑藏在哪些语句中！是的，就是node.Online这句！在进入node.Online之前，我们先看一下node.New是什么东西：

```golang
func New(ctx context.Context, opts ...Option) (StopFunc, error) {
	settings := Settings{
		modules:  map[interface{}]fx.Option{},
		invokes:  make([]fx.Option, _nInvokes),
		nodeType: repo.FullNode,
	}

	// apply module options in the right order
	if err := Options(Options(defaults()...), Options(opts...))(&settings); err != nil {
		return nil, xerrors.Errorf("applying node options failed: %w", err)
	}

	// gather constructors for fx.Options
	ctors := make([]fx.Option, 0, len(settings.modules))
	for _, opt := range settings.modules {
		ctors = append(ctors, opt)
	}

	// fill holes in invokes for use in fx.Options
	for i, opt := range settings.invokes {
		if opt == nil {
			settings.invokes[i] = fx.Options()
		}
	}

	app := fx.New(
		fx.Options(ctors...),
		fx.Options(settings.invokes...),

		fx.NopLogger,
	)

	// TODO: we probably should have a 'firewall' for Closing signal
	//  on this context, and implement closing logic through lifecycles
	//  correctly
	if err := app.Start(ctx); err != nil {
		// comment fx.NopLogger few lines above for easier debugging
		return nil, xerrors.Errorf("starting node: %w", err)
	}

	return app.Stop, nil
}

```
估计多数人看到以上代码有点蒙了，又是Settings，又是Options（复数）和Option(单数)，而且更费解的是有的Option/Options带了fx包名，有的则没有带fx包名。好的，我们先看一眼Settings的完整代码：

```golang
type Settings struct {
	// modules is a map of constructors for DI
	//
	// In most cases the index will be a reflect. Type of element returned by
	// the constructor, but for some 'constructors' it's hard to specify what's
	// the return type should be (or the constructor returns fx group)
	modules map[interface{}]fx.Option

	// invokes are separate from modules as they can't be referenced by return
	// type, and must be applied in correct order
	invokes []fx.Option

	nodeType repo.RepoType

	Online bool // Online option applied
	Config bool // Config option applied
}

```
里面又是Option! 看来一定得了解这个Option是什么了，注意这是带fx包名的：

```golang
// An Option configures an App using the functional options paradigm
// popularized by Rob Pike. If you're unfamiliar with this style, see
// https://commandcenter.blogspot.com/2014/01/self-referential-functions-and-design.html.
type Option interface {
	apply(*App)
}
type optionFunc func(*App)

func (f optionFunc) apply(app *App) { f(app) }
```
它是在第三方包中定义，代码文件为"go.uber.org/fx@v1.9.0/app.go"。
Option定义是很超级简单，但仍是不容易理解。原作者已经估计到多数人的感受，在注释中加了一条链接：[https://commandcenter.blogspot.com/2014/01/self-referential-functions-and-design.html](https://commandcenter.blogspot.com/2014/01/self-referential-functions-and-design.html)，有兴趣者自行去详细了解一下。

但Option只是[go\.uber\.org\/fx]这个包中很小一部分，fx包是uber公司开源的Golang的依赖注入框架。lotus的代码也是采用该框架，所以对这个框架的理解是非常有必要的。

依赖注入(Dependency Injection)是一种设计模式，最出名的应用是Java的Spring框架，不了解的读取自行搜索一下。关于这个包的使用方法可以参考[https://pkg.go.dev/go.uber.org/fx](https://pkg.go.dev/go.uber.org/fx)

这里回头继续介绍Option, 这里先做一个简要的解释：Option可以理解成是影响App行为的一个选项，如果想让选项生效就执行apply方法，而App则可理解为整个程序进程，而Options也就是多个Option的组合，可以想象到Options里所有Option应该是按顺序执行的，否则可能会导致结果不是我们所预期的。

fx.Option是fx包内定义的Option，是一个接口（见以上代码），而Option(不带fx.前缀)是lotus内部定义的：

```golang
type Option func(*Settings) error
```
继续看一下Option相关的几个函数：
```golang
func ApplyIf(check func(s *Settings) bool, opts ...Option) Option {
	return func(s *Settings) error {
		if check(s) {
			return Options(opts...)(s)
		}
		return nil
	}
}

func If(b bool, opts ...Option) Option {
	return ApplyIf(func(s *Settings) bool {
		return b
	}, opts...)
}

```
以上是根据判断条件应用Option，回头看一下前面daemon中的node.New语句的代码，多个option可组合成Options(实际是一个函数)的参数，返回类型又是Option，而且某些Option还可以用ApplyIf指定条件应用。感觉象是。。。另一种语言？这个大胆的猜想是正确的，这就是领域描述语言(DSL)的至简版本，有兴趣的读者可以上网搜索了解一下概念。

到此，为什么lotus在fx.Option基础上又定义了一个Option，原因就比较清楚了：fx.Option的功能比较简单，缺少类似象ApplyIf这样的逻辑，所以Option是lotus里对fx.Option的功能增强。另外fx.Option对应是的App(即fx.App)，但在lotus中使用的是Settings结构。

接着，再看一下方法：
```golang
// Override option changes constructor for a given type
func Override(typ, constructor interface{}) Option {
	return func(s *Settings) error {
		if i, ok := typ.(invoke); ok {
			s.invokes[i] = fx.Invoke(constructor)
			return nil
		}

		if c, ok := typ.(special); ok {
			s.modules[c] = fx.Provide(constructor)
			return nil
		}
		ctor := as(constructor, typ)
		rt := reflect.TypeOf(typ).Elem()

		s.modules[rt] = fx.Provide(ctor)
		return nil
	}
}

```
以上也有些费解，这里解释一下用途即可：Override函数输入参数是两个参数：typ, constructor，typ就是类型，constructor就是构造函数了。输出是一个Option，本质就是一个func。func里面做的事情是将typ和constructor的对应关系保存起来。那为什么叫Override呢，原因是typ类型的对象本身默认用New即可构造，现在默认的构造行为需要被重载。

至此，大家可以感觉到这些与DI有关，这里我们大致这样理解就可以了：如果需要定义一个新的类型对象（例如一个接口），我们就定义好构造函数（第一个输出参数必须是该类型），然后将类型和构造函数作为输入参数去调用Override即可，后续至于什么时候去生成这个类型的对象，就是DI容器去操心的事情。

现在回过头再去理解前面的node.New的代码逻辑，应该就比较好理解一些了，这里我就仅大致介绍一下：开始是将所有的Option进行apply，然后将settings.modules里保存的构造方法传递给fx的DI框架，再调用fx里的app.Start。


## 后续

前面讲了lotus启动过程的框架逻辑，后续将继续讲解lotus启动过程中与业务逻辑相关的部分。
