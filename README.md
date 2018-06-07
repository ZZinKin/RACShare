# ReactiveCocoa 是什么
ReactiveCocoa是一个FRP（函数响应式编程）的思想在Objective-C中的实现框架，目前在美团的项目中被广泛使用。函数响应式编程最初是由微软面向数据流的框架` Reactive Extensions `发展出来的一种编程范式。

这种编程范式很适合前端页面和客户端的开发，Angular、Vue、React这些常见的前端框架都采用了这种思想。

不少人都接触过这个框架，都被它的编程方法给吓到了，demo也没有显示出它有多优越，于是敬而远之。确实，使用RAC来编程跟我们熟悉的命令式编程是两种完全不同的编程范式。比如以下代码：
```c
b = 3;
a = b + 5; // 8 
b = 4; // a 仍为 8
```
如果是响应式编程，则是以下效果(伪代码）：
```
b <~ 3;
a <~ b + 5;
b = 4; // a 相应地变化，为9
```

适应了这种编程范式，我们在代码设计的时候，只要能画出数据流图，就很容易编写出清晰、维护性强、容易测试的代码。

# 核心元素
RACSignal是整个框架的核心，我们可以把UI交互事件、通知、代理等都用信号表示，通过过滤、组合，reduce等操作得到我们最终想要的事件/数据信号，并让UI响应这些数据的变化。

## RACSignal
RACSignal其实是一个信号源，Signal会给它的订阅者（Subscriber）发送一连串的事件，一个Signal可比作流水线中的一段管线，负责决定管线传输什么样的数据。Subscriber是Signal的订阅者，我们将Subscriber比作管线上的工人，它在拿到数据后对其进行加工处理。数据经过加工后要么进入下一条管线继续处理，要么直接被当做成品使用。

在实际使用中，Subscriber对象是被隐藏了的，其实我们使用signal的时候，subscribeNext这个Block成为了订阅者的一个属性。

当信号完成发送的时候，RACDisposable会负责把相应得资源回收。

## RACCommand
RACCommand提供executionSignals、 executing、 error等一连串公开的信号，方便外界对action执行过程与执行结果进行观察。executionSignals是signal of signals，如果外部直接订阅executionSignals，得到的输出是当前执行的信号，而不是执行信号输出的数据，所以一般会配合flatten或switchToLatest使用。 errors，RACCommand的错误不是通过sendError来实现的，而是通过errors属性传递出来的。 executing，表示该command当前是否正在执行。它常用于监听按钮点击、网络请求等。

使用时，我们通常会去生成一个RACCommand对象，并传入一个返回signal对象的block。每次RACCommand execute 执行操作时，都会通过传入的这个signal block生成一个执行信号E (1)，并将该信号添加到RACCommand内部信号数组activeExecutionSignals中 (2)，同时将信号E由冷信号转成热信号(3)，最后订阅该热信号(4)并将其返回(5)。

```objc
- (RACSignal *)execute:(id)input 
{ 
    RACSignal *signal = self.signalBlock(input); //（1）

    RACMulticastConnection *connection = [[signal 
    subscribeOn:RACScheduler.mainThreadScheduler]
    multicast:[RACReplaySubject subject]]; // (3)

    @weakify(self);
    [self addActiveExecutionSignal:connection.signal]; // (2)

    [connection.signal subscribeError:^(NSError *error) {
        @strongify(self);
        [self removeActiveExecutionSignal:connection.signal];
    } completed:^{
        @strongify(self);
        [self removeActiveExecutionSignal:connection.signal];
    }];

    [connection connect]; // (4)

    return [connection.signal]; // (5)
}
```
RACCommand是RAC很重要的组成部分，通常用来表示某个action的执行。


# 几个例子

1. 响应式的例子，绑定控件状态：假设我们要实现一种情况，只有密码输入框的内容和确认框的内容一致的时候，才能使注册按钮启用。
```objc
RAC(registerButton, enabled) = [RACSignal
	combineLatest:@[ [_passwordTextField rac_newTextChannel], [_passwordConfirmTextField rac_newTextChannel] ]
	reduce:^(NSString *password, NSString *passwordConfirm) {
		return @([passwordConfirm isEqualToString:password]);
	}];
```

2. 绑定一个按钮点击事件（MVVM有用）。

```objc

self.loginCommand = [[RACCommand alloc] initWithSignalBlock:^(id sender) {
	return [client logIn];
}];


[self.loginCommand.executionSignals subscribeNext:^(RACSignal *loginSignal) {
	[loginSignal subscribeCompleted:^{
		NSLog(@"成功登录");
	}];
}];

self.loginButton.rac_command = self.loginCommand;

```

3. 合并两个并行的网络请求处理：
```
[[RACSignal
	merge:@[ [client fetchUserRepos], [client fetchOrgRepos] ]]
	subscribeCompleted:^{
		NSLog(@"他们都完成了!");
	}];
```

4.  简化KVO、Notification的处理

```objc
[RACObserve(scrolView, contentOffset) subscribeNext:^(id x) {
     NSLog(@"%@", x);
}];
```

```
 [[[NSNotificationCenter defaultCenter] rac_addObserverForName:@"postData" object:nil] subscribeNext:^(NSNotification *notification) {
     NSLog(@"%@", notification.name);
     NSLog(@"%@", notification.object);
 }];
```

这几个例子看起来不是很酷，但要明白一点，这里所有的异步事件，包括网络请求、UI响应，都可以转化成`RACSignal`对象来处理，而这些`RACSignal`可以任意组合，转化成最终的数据流来处理。比如有一些网络请求需要验证token有效才能进行，这时候就是两个网络请求，而ReactiveCocoa为我们构造的编程模型大大简化了这种编程难度，只要flattenMap一个token信号，再返回响应的请求信号就可以达到了。
# 函数式编程

FP有个很重要的概念是和我们的主题相关的，那就是纯函数。

纯函数就是返回值只由输入值决定、而且没有可见副作用的函数或者表达式。这和数学中的函数是一样的，比如：

f(x) = 5x + 1

这个函数在调用的过程中除了返回值以外的没有任何对外界的影响，除了入参x以外也不受任何其他外界因素的影响。
# 响应式的对象间交互
这里先来说一下响应式，响应式和函数式往往会混为一谈，但实际上两者只是互相利用的关系，并不是同一个概念。响应式对应于命令式的差别就在于主动性的差别，举一个日常生活的例子会更容易理解：

```
假设你现在是主人，你肚子饿了。你需要让你的仆人们给你做饭，端过来给你吃。
```

在命令式思路的方案下，若要完成这个任务，你需要这么做：

```
1. 对仆人中负责买菜的人吩咐你去买菜带给你
2. 你拿着仆人买来的菜，给到负责做菜的仆人，吩咐他去做菜
3. 厨师做好菜端给你，你就可以吃了
```

在响应式思路的方案下，若要完成这个任务，你需要这么做：

```
你喊一声：我肚子饿了！

1. 负责买菜的仆人听到你喊的这一声，就去买菜了，买好菜就放到仓库里，然后喊一声:"我买好菜了！"
2. 负责做菜的仆人听到买菜的仆人喊的这一声，就去仓库里拿东西做菜，做好了端给你。
```


# 封装一个网络请求。
假设我写了一个RequestManager，接口是这样的
```objc
- (NSURLSessionDataTask *)requestWithAction:(NSString *)action
                               appendingURL:(NSString *)appendingURL
                                 parameters:(NSDictionary *)params
                     shouldDetectDictionary:(BOOL)shouldDetectDictionary
                                   callback:(RequestCallback)callback;
```
可以写成RAC的扩展：
```objc
RACSignal *request = [RACSignal createSignal:^RACDisposable * _Nullable(id<RACSubscriber>  _Nonnull subscriber) {
        
       NSURLSessionDataTask *task = [self.requestManager requestWithAction:action appendingURL:appendingURL parameters:params callback:^(BOOL success, id data, NSError *error) {
            if (success) {
                SomeModel *model = [SomeModel new];
                NSDictionary *responseData = data[@"datas"];
                [model mj_setKeyValues:responseData];
                [subscriber sendNext:model];
                [subscriber sendCompleted];
            } else {
                [subscriber sendError:error];
            }
        }];
        return [RACDisposable disposableWithBlock:^{
            [task cancel];
        }];
    }];
```

这里利用`createSignal`的方法创建一个冷信号，当外部有subscribeNext调用它的时候，信号会启动。有时候，我们需要在两个地方订阅这个信号的变化，这时候要把**冷信号**转换成**热信号**来处理，不然这个网络请求会执行两次。

# 区分冷信号和热信号

先说结论：除了`RACSubject`以及其子类信号，都是冷信号。比如我们用`createSignal`创建的都是属于`RACDynamicSignal`，是冷信号，在`subscribeNext`的时候会执行创建的方法，每次`subscribe`操作都会创建一个新的信号。

- 热信号是主动的，即使你没有订阅事件，它仍然会时刻推送。

- 热信号可以有多个订阅者，是一对多，信号可以与订阅者共享信息。

详见这个例子：https://tech.meituan.com/talk-about-reactivecocoas-cold-signal-and-hot-signal-part-1.html

当我们使用自己创建的信号的时候（通常是网络请求），很容易忽略这个问题，出现多次订阅冷信号，即发出了多次网络请求的情况。

## 热信号本质
所有的热信号都属于一个类：`RACSubject`。感兴趣的朋友都知道，`RACSubject`是继承`RACSignal`的，也就是说它有Signal的特性，而且它还能给自己发送数据。所以它既是订阅者，又是发送者。

根据RAC框架的描述，我们得出以下结论：
- Subject是“可变”的。
- Subject是非RAC到RAC的一个桥梁。
- Subject可以附加行为，例如RACReplaySubject具备为未来订阅者缓冲事件的能力。

## 把冷信号转化成热信号
1. 自己使用RACSubject来订阅这个信号。
2. RACMulticastConnection 来广播，背后原理也是同1。
3. signal里面的multicast 方法，其实也是返回RACMulticastConnection。
4. publish， 基于上面方法的封装。
5. replay 方法会把RACReplaySubject当成RACMulticastConnection的RACSubject传递进去，初始化好了RACMulticastConnection，再自动调用connect方法，返回的信号就是转换好的热信号，即RACMulticastConnection里面的RACSubject信号。

这里必须是RACReplaySubject，因为在replay方法里面先connect了。如果用RACSubject，那信号在connect之后就会通过RACSubject把原信号发送给各个订阅者了。用RACReplaySubject把信号保存起来，即使replay方法里面先connect，订阅者后订阅也是可以拿到之前的信号值的。
5. replayLast 和 replay的实现基本一样，唯一的不同就是传入的RACReplaySubject的Capacity是1，意味着只能保存最新的值。所以使用replayLast，订阅之后就只能拿到原信号最新的值。
6. replayLazily 的实现也和 replayLast、replay实现很相似。只不过把connect放到了defer的操作里面去了。
7. defer 单词的字面意思是延迟的。也和这个函数实现的效果是一致的。只有当defer返回的新信号被订阅的时候，才会执行入参block( )闭包。订阅者会订阅这个block( )闭包的返回值RACSignal。
8. then的操作也是延迟，只不过它是把block( )闭包延迟到原信号发送complete之后。通过then信号变化得到的新的信号，在原信号发送值的期间的时间内，都不会发送任何值，因为ignoreValues了，一旦原信号sendComplete之后，就紧接着block( )闭包产生的信号。

回到replayLazily操作上来，作用同样是把冷信号转换成热信号，只不过sourceSignal是在返回的新信号第一次被订阅的时候才被订阅。原因就是defer延迟了block( )闭包的执行了。

# 信号操作

RACSignal有很多操作，这里简单介绍一些常用的：
- concat 前面一个信号完成之后，执行下一个信号。串行操作。
- zipWith 压缩信号，可以把一组信号压缩成一个信号，当所有信号都执行了next之后，subscribeNext会有数据。
- merge 跟上面一样，也是reduce成一个信号的操作，但是这组信号的任意一个信号有数据的时候，merge之后的信号都会执行next。
- flattenMap 基于bind的一个封装，其实大部分操作都是基于bind的封装，这个操作是把一个信号转换成一个信号。
- mapReplace/map 跟flattenMap一样的效果，但是它的输入是一个数据而不是一个信号。
- filter 过滤数据。
- take 当数据超过count数量的时候，signal会执行。
- ignore 忽略某些数据。
- throttle 延缓。比如搜索框的数据回调，如果每输入一个信息就执行一个搜索请求，那太频繁了，可以延迟个0.3秒执行这个信号的发送。
- delay 延迟信号的执行，跟延缓不一样，延缓是多次回调控制信号传递数据的的频率。而延迟相当于gcd的dispatch_after。
- repeat 重复执行，直到信号完成，结合throttle在写网络请求重试机制十分合适。
- combineLatestWith 把一组信号reduce成一个信号，每当一个信号有数据的时候，这个信号会执行，每次send都是一组数据（tuple），所有的信号的数据（包括历史数据）都在里面。这是跟merge操作不一样的地方。
- flatten 限制信号的并发数。

还有很多实用的信号操作，`详见 RACSignal+Operations.h`、 `RACSignal.h`。

这里有一篇很好的文章：
https://tech.meituan.com/ReactiveCocoaSignalFlow.html

# MVVM 
RAC特别适合用于MVVM设计模式，他们两个是相辅相成。如果MVVM不用RAC或者别的FRP框架， 利用Observer、Notification也可以做。
## MVVM模式的组成部分
- 模型
模型是指代表真实状态内容的领域模型（面向对象），或指代表内容的数据访问层（以数据为中心）。
- 视图
就像在MVC和MVP模式中一样，视图是用户在屏幕上看到的结构、布局和外观（UI）。
- 视图模型
视图模型是暴露公共属性和命令的视图的抽象。MVVM没有MVC模式的控制器，也没有MVP模式的presenter，有的是一个绑定器。在视图模型中，绑定器在视图和数据绑定器之间进行通信。
- 绑定器
声明性数据和命令绑定隐含在MVVM模式中。在Microsoft解决方案堆中，绑定器是一种名为XAML的标记语言。 绑定器使开发人员免于被迫编写样板式逻辑来同步视图模型和视图。在微软的堆之外实现时，声明性数据绑定技术的出现是实现该模式的一个关键因素

# 参考资料

1. 初识
https://github.com/ReactiveCocoa/ReactiveObjC

2. RAC的核心元素和信号流
https://tech.meituan.com/ReactiveCocoaSignalFlow.html

3. 区分冷信号和热信号

https://tech.meituan.com/tag/ReactiveCocoa

4. 实战

http://limboy.me/tech/2014/06/06/deep-into-reactivecocoa2.html

5. 封装网络请求:

http://limboy.me/tech/2014/01/05/ios-rest-client-implementation.html
