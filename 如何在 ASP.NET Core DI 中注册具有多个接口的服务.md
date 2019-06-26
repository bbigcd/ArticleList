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

