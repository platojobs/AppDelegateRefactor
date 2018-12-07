# AppDelegateRefactor
#### 重构 AppDelegate

>`AppDelegate`连接应用程序和系统，通常被认为是每个iOS项目的核心。随着开发的迭代升级，不断增加新的功能和职责，它的代码量也不断增长，最终导致了`Massive AppDelegate`（臃肿的`AppDelegate`）。
>在复杂`App Delegate`里修改任何东西的成本都是很高的，因为它将会影响你的整个APP，一不留神就吃`bug`。毫无疑问，保持`AppDelegate`的简洁和清晰对于健康的iOS架构来说是至关重要的。本文我们将使用多种方法来重构，使之简洁、可重用和可测试。

-------

`AppDelegate`是应用程序的根对象。它确保应用程序与系统以及其他应用程序正确的交互。AppDelegate理通常会承担很多职责，这使得很难进行更改，扩展和测试。

通过调研几十个最受欢迎的开源iOS应用程序，我把`AppDelegate`常见的业务代码列出如下。我相信我的读者也写过这样的代码，或者维护或者支持这种类似混乱的项目。

+ 初始化许多第三方库（如分享，日志，第三方登陆，支付）

+ 初始化数据存储系统

+ 管理`UserDefaults`：设置首先启动标志，保存和加载数据

+ 管理通知：请求权限，存储令牌，处理自定义操作，将通知传播到应用程序的其余部分

+ 配置`UIAppearance`

+ 管理`App Badge Counter`

+ 管理后台任务

+ 管理UI堆栈配置：选择初始视图控制器，执行根视图控制器转换

+ 日志埋点统计数据分析

+ 管理设备方向

+ 更新位置信息

这些臃肿的代码是反模式的，像屎一样难于维护。显然支持扩展和测试这样的类非常复杂且容易出错。例如，查看`Telegram`的`AppDelegate`的源代码会激起我的恐惧。`Massive App Delegates`与我们经常谈的`Massive View Controller`的症状非常类似。


##### 命令模式（`Command Pattern`）
>是一种数据驱动的设计模式，它属于行为型模式。请求以命令的形式包裹在对象中，并传给调用对象。调用对象寻找可以处理该命令的合适的对象，并把该命令传给相应的对象，该对象执行命令。因此命令的调用者无需关心命令做了什么以及响应者是谁。

可以为`APPdelegate`的每一个职责定义一个命令，这个命令的名字有他们自己指定:
```java

protocol Command {
func execute()
}
struct InitializeThirdPartiesCommand:Command {
func execute (){
// Third parties are init here
}
}

struct InitializeViewCotrollerCommand:Command {
let keyWindow: UIWindow

func execute (){
//pick root viewController here
keyWindow.rootViewController=UIViewController()
}
}

struct InitializeAppearanceCommand:Command {

func execute (){
//setup UIAppearance
}
}

struct RegisterRemoteNotificationsCommand:Command {

func execute (){
//Register remote notifications here
}
}
```

-----

然后我们定义`StartupCommandsBuilder`来封装如何创建命令的详细信息。APPdelegate调用这个builder去初始化命令并执行这些命令:
```swift

final class StartupCommandsBuilder{

private var window: UIWindow!

func setKeyWindow(_ window:UIWindow) -> StartupCommandsBuilder {
self.window=window
return self
}

func build() -> [Command] {
return[InitializeThirdPartiesCommand(),
InitializeViewCotrollerCommand(keyWindow: window),
InitializeAppearanceCommand(),
RegisterRemoteNotificationsCommand()
]
}

}


```

最后调用在`didFinishLaunchingWithOptions`中初始化就可以了：
```swift
func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
// Override point for customization after application launch.
StartupCommandsBuilder()
.setKeyWindow(window!)
.build()
.forEach { $0.execute()}
return true
}

```
