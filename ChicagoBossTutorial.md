Chicago Boss Hello world 教程。
---

### 一、引言

如果你想保守一个秘密，交给瑞典人好了。诞生于20年前的斯德哥尔摩，Erlang 几乎是当下最高级的开源服务器平台，但是看起来没什么人知道这件事情。Erlang 可以处理数以十万计的并发连接；也可以秒生出百万级的并发进程；服务器代码再生产环境下也可以热更新，服务不会被打断；服务器几乎不会宕机，错误完美处理。

毫无理由拒绝啊，这个世界为什么不完全用Erlang来构造呢？呃，Erlang 是一门函数式语言，这意味着你得使用递归而不是熟悉的"for"和"while"循环来实现所有的语法。 与主流脚本语言不一样的是，Erlang没有内置字典或者 hash map 在语法中。 要是想实现一个可用的服务器的话，你还得学点额外的东西，就是OTP。在我看来，正是这些既有设定将Erlang留在了斯坎蒂尼维亚。

然而，Chicago Boss 改变了这一切。它让那些希望用美好语言书写可靠网站的黑客们更加容易接受 Erlang。Boss 使用代码自动生成绕过了历史遗留问题，hash-map 窘境。同事有照顾到了 OTP 平台，由此，你可以专心在构造你想要的网站特性。至于函数式编程的问题，在日常的 web 开发中，递归很少用到。我猜测 99% 的服务端应用代码都可以直接读写数据库，所以在构造网站这种项目中，程序员们是不会怀念他的 "do/while" 循环的。

如果你是一个有经验的 web 程序员，你可能会喜欢 CB 提供的所有便利：一个支持数据库连接高级ORM，分片，缓存；可编译 Erlang 字节码的高速模板；自动重编译以及浏览器内报错；简洁的重载与重定向指令；完整的发送接收邮件功能；内置的消息队列；功能测试框架；以及史无前例的数据 Model 监测系统。

最后，通过整合 Erlang 平台以及自身的创新，Chicago boss 让网站轻松开发，愉悦部署。相比于 Rails 应用，Boss应用所需的开发时间相当甚至更短，同时稳健性更好，内存管理完善。得益于完全异步网络的实现，诸如聊天之类的应用变得易于构建，要知道再次之前只能通过基于 callback 的框架（像 Nginx, Node.js, Twisted, 或者 Perlbal）。

上文的关于进步的重要性描述绝对不是过誉，CB帮助了小团队通过很小的开销开发处理基于数据库高交互，高流量的网站。尽管CB不能告诉你到哪儿去找这么多用户访问你的产品，这个说明剩下的部分将对于处理请求以及满足用户期望所要做的事情详加阐述。

### 二、框架

Chicago Boss 只是 Erlang web 应用的一个编译链和运行依赖库。当然，我们会致力于运行依赖库的说明，这样你能明白request处理的基本流程，编译器听起来实在不是一件容易的事情。

一个 Chicago Boss 服务器能部署一个或这个更多的 CB 应用。 一个 CB 应用是 Controller、 View 、 Model 以及CB 服务器配置基于一个基础URL的路由的集合。你可以部署一个博客应用在 /blog 下，一个维基应用在 /wiki 下， 以及主站应用放在 / 下。 应用可以重定向到其他应用，并且与其它应用共享数据 Model 以及消息队列。 这样的逻辑对于模块重用，相互通信很有用。但是考虑到安全因素，你可能要将不被信任的应用放到一个独立的服务器里面去。

一个 CB 应用包含多个服务块：

    Web Controller： 获取 HTTP request 信息，以及对其进行处理。
    Web  View ：渲染从Controller获取的数据，给出呈现。
     Model ：提供数据库的抽象层。
    Email Controller和 View 。
    初始化脚本。
    测试脚本。
    一个路由配置文件。

呃...其实不只这么多！CB 服务器还开启了一系列可用部署应用使用的服务：

    url 路由 （BossRouter）
    对话存储层 (BossSession)
    数据库连接和缓存层 (BossDB and BossCache)
    消息队列 (Boss MQ)
    Model 事件系统 (BossMQ)
    Email 服务器 (BossMail)

对于严谨的 web 开发人员，一台“服务器”应该包含了多个 Erlang 节点连接集群。在这种环境下，集群中的所有节点遵从主节点（配置文件中定义）调度共享服务，诸如数据库缓存，消息队列以及邮件等。一般的节点可以处理常规的 HTTP 流量，也可以处理其他任务。在这些配置文件中，你需要一个代理服务器来分流请求，比如 Nginx。Erlang VM 是多线程的，一个 Erlang 节点应该对应一台物理机器。一台机器运行多个 Erlang 节点可能是资源的浪费（译者：说实话，我没读懂这话的意思）。

HTTP 请求最先被 Mochiweb 或者 Misultin 解析（具体哪个取决于你的服务器配置）。如果成功解析，会递交一个 SimpleBridge 请求对象给 CB 服务器，然后 Boss 开始工作。

首先，BossRoute 解析请求 URL 并决定那个应用应该处理这个请求。URL 通常映射一个 Controller 下的一个动作。一个 Controller 可以包含一系列的动作，这意味着一可以用 "create", "edit" 以及 "delete" ；每个动作都对应 Controller 模块下的一个同名方法。

在任意一个动作被执行前， Boss 会检查执行者这个动作在当前 Controller 下是否需要额外的特殊认证。这个认证功能拥有完全的数据库和对话信息的访问权限，如果认证返回失败的话可以做一个重定向（例如，跳转到登录页面）。

如果认证通过的话，动作会被执行；HTTP 方法，URL 的记号，以及认证方法的返回值都会被这个动作处理。因为 Controller 是参数化的模块，  Controller 动作也将 SimpleBridge 请求对象以及当前可获取的session ID 作为模块参数。

 Controller 动作可以做一些列的事情来处理一个请求，诸如从数据库读取信息，等待消息队列的信息，或者发送一封邮件。动作的返回值决定了 CB 服务器如何回复请求。通常情况下，会返回一个填充好的 HTML 模板，当然JSON数据也是可以生成响应的，重定向，错误，调用其他动作都可以实现。

在 Controller 动作返回后，下一步就是生成相应。一般而言，响应由与 Controller 动作同名的 ErlyDTL 模板生成的。（ErlyDTL 是一个非常快，完全 Erlang 实现的 Django 模板语言。）模板也有数据库访问权限，所以生成 HTML 过程中也可能有数据库访问。由此， Controller 不必预取所有相关的数据。 例如， Controller 可能检索一条博客消息，然后模板可以自己检索相关信息，诸如博客作者的雇员的二表哥的女儿的猫的信息，或者你想要的随便什么东西。

但是，生成的响应不会马上回复到客户端， Controller 可以定义特殊的提交处理函数来处理输出或者缓存、记录日志什么的。这个提交处理过程可能会对响应增加时间戳或者压缩响应数据。

在提交处理之后，客户端会收到相应，当然也很有可能再发个请求。

处理邮件的流程也是类似，但是更加简单。每个应用只有一个 email  Controller ，而且返回值是忽略的。email 没有路由文件；收件人的名字往往和 Controller 动作名对应起来的，发送邮件给 "post@domain.com" 将会调用 "post" 动作。这个 Controller 动作将会处理邮件地址以及一个解析过的邮件内容，也可以按想法处理它们(可能在数据库打一个版本标签)。如果安全是一个问题，认证功能可以在收邮件调用句柄方法前校验发送者的邮件地址和 IP 地址。

框架应该已经说够了，我们开始干活儿吧！

### 三、开始
为了即将开始的 Chicago Boss 开发之旅，你得有台装好 erlang 的电脑，版本么，最好是 R13A 或者更新的。机器还得配上终端，网页浏览器，以及你熟悉的编辑器。

#### 1. Hello, world

通过以下链接下载 Chicago Boss 0.7.0 的源码：

第一件要做的事情就是打开安装包，编译源码，然后创建一个新的工程。

cb_tutorial 是新项目的名字。你可以随便给你的项目安排一个什么名字， 但是必须得是小写字母开头，然后只包含小写字母，数字以及下划线。

第二件事情是进入这个新项目的目录中。

额，随便看看（哥会等你的）。你可能会看到下面的一个组件：

    boss.config - 这个是配置文件，你可以通过它设置数据库连接，起缓存，或者定义监听端口等等很多有用的事情。但是在这个教程里面，我们只会用一个内存数据库，监听8001端口，毛线都不会缓存。
    
    rebar, rebar.config - 这个是构建工具以及构建配置文件。你暂时不会需要对这个有所修改。
    
    cb_tutorial.app.src - 这个是用来生成 .app 文件的，.app 文件可以帮助 OTP 组件管理依赖以及生产系统热升级。在这个文件里写上系统描述以及版本号，呃，别用 0.0.1。
    
    ebin/ -这是 .app 文件以及编译后的模块的主目录。开发中这个文件没什特别的作用，但部署生产系统的时候，你需要使用 make 编译系统。有问题的话，可以用 make clean 处理。
    
    init-dev.sh - 启动一个开发服务器的脚本，这个会自动帮你重新编译所有的代码，优雅打印错误信息，另外会送你一个交互的 console。 你可以在 console 里面敲入 q(). 来停止开发服务器。
    
    init-sh - 通过./init.sh start 可以开始一个生产服务器，生产服务器进程都是后台运行的额。 执行 ./init.sh stop 可以停止生产服务器。重启的话就是 ./init.sh reload。
    
    priv/ - 就是放一些静态文件 css啊，js啊，图片啊巴拉巴拉，priv 听起来高大上，其实也就这样。
    
    src/ - 这个文件夹将会放你的源代码。点进去，你会看到：
    
         controller/ -  Controller 。暂时为空，我们写代码一般考虑从这儿开始。
    
         model/ -  Model 之用来做对象关系映射（ORM）的。这里的文件会用一个特殊的 Model 编译器编译，所以不要放任何库模块在这儿。
    
         lib/ - 一般库模块都放这儿。在开发中，这里可能每个 http request 都会重新编译。所以，如果你用一个大号的三方库的话，最好别放在这儿，然后通过启动脚本调用路径好了。
    
         view/ - 模板。你会需要为每个 Controller 创建子目录。别忘了 view/lib/, 这里你可以放一些用于客户标记过滤的复用模块和代码。
    
         log/ - 错误日志。CB 会维护一个一直指向最新日志文件的连接（log/boss_error-LATEST.log）
    
         include/ - 头文件，一般会提供库模块的定义，你也可以把应用作用域的宏定义放在这儿。

好了，看完了吧，我们开启一个开发服务器吧。

    ./init-dev.sh

屏幕会飞过相当多的消息，它们大都长这个样子：

    =PROGRESS REPORT==== 27-Nov-2011::18:54:17 ===
                       supervisor: {local,sasl_safe_sup}
                            started: [{pid,<0.40.0>},
                                         {name,alarm_handler},
                                         {mfargs,{alarm_handler,start_link,[]}},
                                         {restart_type,permanent},
                                         {shutdown,2000},
                                         {child_type,worker}]

就目前而言，我不知道这是个么子意思，但是我认定这些应该是无害的。不管怎样，这些消息过去之后，你会看到一个 Erlang shell 提示：

    (cb_tutorial@blackstone)1>

这个是一个服务器的shell。 cb_tutorial， 不仅仅是项目的名字，也是开发服务器节点的名字。blackstone 是我的计算机的名字；你的计算机会看到你的计算机的名字。

你可以在 shell 里面直接输入 Erlang 语句；你可以完全获取到服务器的状态，同时也加载了代码库。别忘了每个指令的句号就行。q(). 可以帮你停掉服务器。

现在我们已经准备好了 hello world 应用。在项目目录下，创建一个文件：src/controller/cb_tutorial_greeting_controller.erl（随便用一个你喜欢的编辑器，vim, emacs 都成）。这个文件是 cb_tutorial 应用的一个 greeting  Controller 。（一般 Controller 的模块都是这样的形式：<应用面>_< Controller 名>_controller.）

你可以键入（粘贴也不是不可以）下面的代码段：

    -module(cb_tutorial_greeting_controller, [Req]).
    -compile(export_all).
    hello('GET', []) ->
         {output, "Hello, world!"}.

现在你可以在你的浏览器里面键入: http://localhost:8001/greeting/hello. 你应该能看到一个熟悉的信息。

我们已经在一个 Controller 里面写了一个简单的方法，叫做 "hello". 当然你也可以在这个 Controller 里面写点其他方法。每个方法会有自己的 url，格式一般这样；/< Controller 名>/<方法名>。 如果 url 包含了其他使用 "/" 分割出来的记号，这些信息会被解析成一个列表。这个列表作为方法的第二个参数存在，第一个参数你可能已经猜到了，对，就是 HTTP method ，包括（'GET', 'POST', 'PUT', 以及 'DELETE'）。

 Controller 模块都是参数化的，由 -module 的参数队列（[Req]）指定的。尽管 erlang 是一个函数式语言，参数化模块还是添加了一点 OO（面向对象，Object Oriented） 的味道，由此我们不需要对于每个模块的所有方法都传递相同值。在 CB controller 中，每个 function 都可以访问到 Req 参数， 而 Req 参数包含当前请求很多有意义的信息。 对此我们在看参数化模块的时候会有很多的认识。

Controller action 支持多返回值，最简单的就是 {output, Value}，原生的 HTML 也是支持的。 我们可以使用{json, Values} 返回 JSON： 

    hello('GET', []) ->
        {json, [{greeting, "Hello, world!"}]}.

我已经将字符串转换成 proplist 了，也就是一列 key-value tuple。Proplist 在 erlang 中用的相当普遍了，所以，我也希望你能习惯它们。json 返回值会告诉 CB 将 proplist 转换成实实在在的 JSON 字符串，这样 客户端才能够处理。刷新一下你的浏览器，你会看到结果。

如果需要用 Chicago Boss 创建 JSON APIs, json 返回值无疑非常便利。当然，如果我们想生成 HTML，使用模板也是非常方便的。我们需要稍微修改一下 controller 的代码才可以使用模板，然后创建一个真实的模板 文件。首先，选择 controller 文件使用 ok 替换 json:

    hello('GET', []) ->
        {ok, [{greeting, "Hello, world!"}]}.

如果 template 存在的化，变量列表将会被传递到相关的 template。额，暂时还是没有的，所以我们创建一个。打开一个新的文件 src/view/greeting/hello.html （你需要创建一个 greeting 子文件夹），然后输入或者粘贴下文代码：

    <b>{{ greeting }}</b>

这是一个Django template 语言片段，这在 erlang web开发中也是一种通用语言。它很简单，对于没有 erlang 经验的开发者也很易于上手。刷下浏览器，你会看到粗体的 greeting。

额，你可以试试下面这句，能够加强 greeting，

    <b>{{ greeting|upper }}</b>

这里，我们使用乐一个 filter，如果你刷新了浏览器的话，filter 功能显而易见。 Filter 提供了一个聚合的格式化功能， 它们也可以和 Unix 命令一样 连接到一块儿：把管道符号（"|"）放在 filter值间就行。其他的 filter 你可以自己玩玩看。其中包括，长度，分块，标题，字数等。

总的来说，我们可以看到 controller 可以返回三种类型的 tuples:

    - {output, Value} - 返回原生的HTML 
    - {json, Values} - 返回格式化的JSON数据
    - {ok, Values} - 传递参数值到相关的 template中

额，这个很有趣把。我们可以对世界说 hello，但是世界对我们说 hello 时，我们怎么办呢？绝大多数 web 应用需要存储和处理数据。接触完 View 和 Control 后，我们开始聊聊 Model。

#### 2. 数据库

Chicago Boss 包含了一套特殊的查询语法，一个完整的ORM以及一系列的数据库驱动。我们通过一个简单的模型起步，看看到底能干些什么。在 src/model 下面创建一个 greeting.erl 的文件夹，然后输入以下代码，同样粘贴也行：
    
    -module(greeting, [Id, GreetingText]).
    -compile(export_all).

这个是 "greeting" 对象的模型。这是一个参数化的模块，然后里面有两个参数（Id 和 GreetingText）。所有的模型的第一个参数都是 Id, 接下来的参数你可以随意定义。这看起来像是一个传统的 Erlang 模块， 但是它包含了很多神奇的功能。是时候探寻一下 Chicago Boss ORM 隐藏的功能了，这货叫做 BossRecord。
刷新一下浏览器，（这样这个 model 文件会被编译然后加载），然后尝试一下下面的 shell 命令：

    > Greeting = greeting:new(id, "Hello, world!").
    {greeting,id,"Hello, world!"}

这个新功能返回一个新的 greeting 实例。 我把 id 作为第一个参数传入，是为了告诉 BossDB 必要的时候就生成一个新的 ID 。如果我想要一个指定的 ID， 比如我想包含我的球服的号码，这个可以自定义。
如你所见，这个 greeting 实例就是一个包含 模块 名字以及所有传入参数的 tuple。 当你调用一个参数化 模块 的function 的时候（额，别用 new()）,运行环境在执行 function 之前，会绑定 模块 的参数列表的。尽管这看起来有点像面向对象百年城，调用 greeting 实例就是使用其所罕有的参数值。当然如同 erlang 其他参数一样，变量单次赋值后无法变化，所以我们没必要考虑，锁，副作用或者其他面向对象的坏毛病。

我们还是看看 model 吧。试一下下面这段代码：

    > Greeting:greeting_text().
    "Hello, world!"

“该死，等会”，你可能会想，“我压根不记得我调用过 greeting_text/0 ， 事实上我记得我没有调用过任何 function，我老年痴呆了么？ 我在哪儿？”
额，别担心，你没问题（你坐在你电脑前）。在系统载入 model 模块 之前，Chicago Boss 编译器已经后台将一些方法附加到 模块s 上去了。这里有一些其他 function 你可以尝试一下。你现在可以去看看这些防范都是干什么的。

> Greeting:attributes().
[{id,id},{greeting_text,"Hello, world!"}]
> Greeting:attribute_names().
[id,greeting_text]

这个也相当有用，但是可能并不会如你所愿的工作：

> Greeting:set(greeting_text, "Good-bye, world!").
{greeting,id,"Good-bye, world!"}

Erlang 变量不可变， 所以调用 set/2 会返回一个新的 record 。之前的比纳凉并不会发生变化，试试这个，你会懂我的意思的：

> Greeting.
{greeting,id,"Hello, world!"}

看到了么？Greeting 变量依旧或者，充满希望。这些编译器生成的方法极大的方便了 Chicago Boss 的日常开发。没有它们，你会需要调用 proplists:get_value/2, proplists:delete/2, 以及一些列的冗余的方法。这些方法中最重要的是：

> Greeting:save().
{ok,{greeting,"greeting-1","Hello, world!"}}

这个很简单，没有保存的话，数据没法持久化，除了报错一切操作都会失去意义。试试吧，你将会实现第一个数据持久化，id 属性会被注入一个实时的认证字符串。Chicago Boss 生成的字符串 ID 会是 "model-number" 的形式（比如 "greeting-1"）.Chicago Boss 通过这一命名策略来保证 ID 的唯一性。（这也大大的降低了 API 设计的难度。）

你可能也会思考：怎么才能记住所有的由编译器添加到 BossRecord 的 function 。你能保密咩？有个秘密可以跟你说，但你连你的小伙伴也不能说。（尤其他（她）是一个 Rails 程序员）。打开一个浏览器，访问 http://localhost:8001/doc/greeting. 看一眼就关掉吧，你已经发现 Chicago Boss 的全自动文档了，上面每个模型所有的方法。在 Chicago Boss 社区，我们称这个特性叫 "/doc"，你可以自己研究一下。
