### Ocelot源码阅读

地址：`https://github.com/ThreeMammals/Ocelot`

源码文件目录树

| 文件名称              | 描述     | 作用     |
| ---------- | -------- | -------- |
| Authentication        | 身份验证 | 身份验证 |
| Authorisation         | 授权     | 身份验证身份验证身份验证身份验证身份验证 |
| Cache                 | 缓存     |          |
| Claims                |          |          |
| Configuration         | 配置     |          |
| DependencyInjection   | 依赖注入 |          |
| DownstreamRouteFinder | 下游路径查找器 |          |
| DownstreamUrlCreator  | 下游资源创建器 |          |
| Errors                | 错误 |          |
| Headers               | 头部 |          |
| Infrastructure        | 基础设施 |          |
| LoadBalancer          | 负载均衡 |          |
| Logging               | 日志 |          |
| Middleware            | 中间件 |          |
| QueryStrings          | 查询字符串 |          |
| Raft                  |          |          |
| RateLimit             | 速度限制 |          |
| Request               | 请求 |          |
| Requester             | 请求器 |          |
| RequestId             | 请求Id |          |
| Responder             | 响应器 |          |
| Responses             | 响应 |          |
| ServiceDiscovery      | 服务发现 |          |
| Values                | 值 |          |
| WebSockets            | WebSocket |          |

DependencyInjection 目录下，在 ServiceCollectionExtensions 的方法中，是依赖注入的入口

```c#
public static IOcelotBuilder AddOcelot(this IServiceCollection services)
{
    var service = services.First(x => x.ServiceType == typeof(IConfiguration));
    var configuration = (IConfiguration)service.ImplementationInstance;
    return new OcelotBuilder(services, configuration);
}

public static IOcelotBuilder AddOcelot(this IServiceCollection services, IConfiguration configuration)
{
    return new OcelotBuilder(services, configuration);
}
```

上述方法中，创建了一个 OcelotBuilder 类对象，将 OcelotBuilder 注入到 services 中，并提供了配置文件。

OcelotBuilder 的构造方法：

```c#
public OcelotBuilder(IServiceCollection services, IConfiguration configurationRoot)
{
	//···
}

```

OcelotMiddlewareExtensions

