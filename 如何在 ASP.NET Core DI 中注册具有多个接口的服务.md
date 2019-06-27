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

针对多个服务注册实现的一般模式是常见的。大多数第三方 DI 容器框架都实现了这种概念。例如:

* [Autofac](https://autofac.org/) 将该种模式当作默认的模式 - 上面的测试代码已经证明了这点
* [Windsor](http://www.castleproject.org/projects/windsor/) 中有一个“转发类型”的概念，它允许你“转发”多个服务到一个单例实现中
* [StructureMap](http://structuremap.github.io/) ([now sunsetted](https://jeremydmiller.com/2018/01/29/sunsetting-structuremap/)) 也同样有“转发”类型的类似概念。据我所知，它的继承者 [Lamar](https://jasperfx.github.io/lamar/) 并没有提供类似的概念，不过，我可能是错的。

鉴于这个需求非常普遍，而 ASP.NET Core DI 容器中没有，这看起来很奇怪。这个问题2年前被提出来[(David Fowler)](https://github.com/aspnet/DependencyInjection/issues/360) ，不过已经被关闭了。幸运的是，有一个概念上简单，但不是很优雅的解决方案。

1. ##### 提供一个服务实例(仅限单例)

   最简单的方法就是在注册你的服务时提供一个 `Foo` 实例。每个已注册的服务会在请求时返回你提供的确切实例，确保只有一个单例实例。

   ```c#
   [Fact]
   public void WhenRegisteredAsInstance_InstancesAreTheSame()
   {
       var foo = new Foo(); // The singleton instance
       var services = new ServiceCollection();
   
       services.AddSingleton<IFoo>(foo);
       services.AddSingleton<IBar>(foo);
   
       var provider = services.BuildServiceProvider();
   
       var foo1 = provider.GetService<IFoo>();
       var foo2 = provider.GetService<IBar>();
   
       Assert.Same(foo1, foo); // PASSES;
       Assert.Same(foo2, foo); // PASSES;
   }
   ```

   这里有一个非常不好的地方（警告）- 你必须在配置的时候能够实例化 `Foo` 对象，并且你必须知道并且提供所有关于它的依赖。在某些情况下它是有用的，但这种方式不是很灵活。

   另外，你只能用这种方法来注册单例类。如果你希望 `Foo` 是各自请求范围内的单例实例（Scoped模式），你很不走运，你可能须要下面的技术。

2. ##### 使用工厂方法实现转发

   如果我们分解我们的需求，那么替代的解决方案可能是:

   * 我们希望已注册的服务（`Foo`）有特殊的生命周期（例如：Singleton or Scoped）
   * 当 `IFoo` 被请求，返回一个 `Foo` 实例
   * 当 `IBar` 被请求，也返回一个 `Foo` 实例

   根据这三条规则，我们可以再写一个测试：

   ```c#
   [Fact]
   public void WhenRegisteredAsForwardedSingleton_InstancesAreTheSame()
   {
       var services = new ServiceCollection();
   
       services.AddSingleton<Foo>(); // We must explicitly register Foo
       services.AddSingleton<IFoo>(x => x.GetRequiredService<Foo>()); // Forward requests to Foo
       services.AddSingleton<IBar>(x => x.GetRequiredService<Foo>()); // Forward requests to Foo
   
       var provider = services.BuildServiceProvider();
   
       var foo1 = provider.GetService<Foo>(); // An instance of Foo
       var foo2 = provider.GetService<IFoo>(); // An instance of Foo
       var foo3 = provider.GetService<IBar>(); // An instance of Foo
   
       Assert.Same(foo1, foo2); // PASSES
       Assert.Same(foo1, foo3); // PASSES
   }
   ```

   为了“转发”对具体类型接口的请求，你必须做两件事:
   
   * 使用 `services.AddSingleton<Foo>()` 显示的注册具体类型
   * 通过提供工厂函数，将接口请求委托给具体的类型：`services.AddSingleton<IFoo>(x => x.GetRequiredService<Foo>())`
   
   通过这种方法，你会得到一个真正的 `Foo` 单例实例，无论你请求的是哪种实现的服务。
   
   > *这种提供“转发”类型的方式被记录在原始的 [issue](https://github.com/aspnet/DependencyInjection/issues/360) 中，顺带了一个警告 - 它不是很有效。通常最好尽可能避免“服务定位器样式” `GetService()` 调用。不过，我觉得在这种情况下，这绝对是更好的做法。*

#### 总结

在这篇文章中，我总结了如果你使用 ASP.NET Core DI 服务去注册一个以多服务模式的具体类型会发生什么。特别的，我展示了如何使用单例对象的多个副本，这可能会导致一些小bug。为了解决这个问题，你可以在注册的时候提供一个实例，或者使用工厂方法代理服务的解析。尽管使用工厂方法不是非常高效的，但是却是通常情况下的最好方法。