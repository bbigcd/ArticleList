### 【译】如何在 ASP.NET Core DI 中注册具有多个接口的服务

[【原文】如何在 ASP.NET Core DI 中注册具有多个接口的服务](https://andrewlock.net/how-to-register-a-service-with-multiple-interfaces-for-in-asp-net-core-di/)

在本文中，我将介绍如何在 ASP.NET Core 中，将一个包含多个公共接口的具体类注册到 *Microsoft.Extensions.DependencyInjection* 容器中。使用这种方法，你将能够使用它实现的任何接口检索具体的类。例如，如果你有下面的类：

```c#
public class MyTestClass: ISomeInterface, ISomethingElse { }
```

接下来，你将会去注入 `ISomeInterface`  和 `ISomethingElse` 二者的其中之一，接下来，你会获取到相同的`MyTestClass` 实例。

有一点是很重要的，注册 `MyTestClass`  时须以特定的方式以避免生存周期问题。比如一个单例中有两个实例！

在本文中，我简要概述了 ASP.NET Core 中的 DI 容器和一些第三方容器相比的局限性。然后，我将介绍对一个具体类型的多接口请求“转发”的概念，以及如何在 ASP.NET Core 中的 DI 容器实现它。

> **TL;DR** *ASP.NET Core DI 容器原生并不支持多个服务实现的注册（有时被称为"转发"）. 相应的，你必须手动将服务的解析委托给工厂函数，例如：* *services.AddSingleton<IFoo>(x=> x.GetRequiredService<Foo>())*

#### ASP.NET Core 中的依赖注入

ASP.NET Core 的关键特性之一就是其使用了依赖注入（DI）。ASP.NET Core 框架本身是围绕 “标准容器” 这一个抽象概念设计的，它允许自身使用一些简单的容器，同时还允许你插入功能更丰富的第三方容器。

> *“标准容器”的想法并不是没有争议的 - 我建议你去阅读 [Mark Seemann](http://blog.ploeh.dk/2014/05/19/conforming-container/) 这篇关于标准容器作为反模式的文章，或者这篇来自 [SimpleInjector](https://simpleinjector.org/blog/2016/06/whats-wrong-with-the-asp-net-core-di-abstraction/) 团队关于 ASP.NET Core DI 容器的特殊性。*

为了使第三方容器实现起来更简单，“标准容器”被设计得更简单，它公开的 API 数量非常有限。对于给定的服务（例如： `IFOO` )，你可以定义一个具体的类去实现它（例如: `Foo`）,以及它的生命周期 （例如： 单例）。在这个基础上，可以直接提供服务实例，也可以提供工厂方法，但是这种方法会更复杂。

相比之下，.NET 的第三方 DI 容器通常提供更高级的注册 API. 例如， 多数 DI 容器为配置公开一个 ["scan" API](https://autofaccn.readthedocs.io/en/latest/register/scanning.html) , 你可以通过它搜索程序集中的所有类型。然后，把它们添加到你的 DI 容器中。下面是一个 `Autofac` 框架的例子：

```c#
var dataAccess = Assembly.GetExecutingAssembly();

builder.RegisterAssemblyTypes(dataAccess) // 查找程序集中的所有类型
       .Where(t => t.Name.EndsWith("Repository")) // 过滤类型
       .AsImplementedInterfaces()  // 注册服务及其所有公共接口
       .SingleInstance(); // 将服务注册为单例
```

在这个例子中， `Autofac` 会找到程序集中那些所有名字以 `"Repository"` 结尾的具体类，并且根据他们实现的所有公共接口在容器中注册。例如，给定的下列类和接口：

```c#
public interface IStubRepository {}
public interface ICachingRepository {}

public class StubRepository : IStubRepository {}
public class MyRepository : ICachingRepository {}
```

前面的 `Autofac` 代码相当于在 ASP.NET Core 容器的 `Startup.ConfigureServices` 中手动注册这两个类及其各自的接口：

```c#
services.AddSingleton<IStubRepository, StubRepository>();
services.AddSingleton<ICachingRepository, MyRepository>();
```

不过，如果一个类实现了多个接口，会发生什么呢？

#### 将单个实现注册为多个服务

实现多个接口的类很常见，例如:

```c#
public interface IBar {}
public interface IFoo {}

public class Foo : IFoo, IBar {}
```

让我们编写一个快速测试，看看如果我们使用 ASP.NET Core DI 容器对这两个接口注册该类会发生什么:

```c#
[Fact]
public void WhenRegisteredAsSeparateSingleton_InstancesAreNotTheSame()
{
    var services = new ServiceCollection();

    services.AddSingleton<IFoo, Foo>();
    services.AddSingleton<IBar, Foo>();

    var provider = services.BuildServiceProvider();

    var foo1 = provider.GetService<IFoo>(); // An instance of Foo
    var foo2 = provider.GetService<IBar>(); // An instance of Foo

    Assert.Same(foo1, foo2); // FAILS
}
```

我们将 `Foo` 注册为 `IFoo` 和 `IBar` 二者的单例，但是结果可能并不是你预计的。我们实际上有两个 `Foo` “单例”实例，每个都是以服务的方式被注册。

#### 转发服务请求

