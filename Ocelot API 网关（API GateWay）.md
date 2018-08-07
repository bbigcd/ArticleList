### Ocelot API 网关（API GateWay）

现有微服务的几点不足：

* 对于在微服务体系中、和 Consul 通讯的微服务来讲，使用服务名即可访问。但是对于手 机、web 端等外部访问者仍然需要和 N 多服务器交互，需要记忆他们的服务器地址、端 口号等。一旦内部发生修改，很麻烦，而且有时候内部服务器是不希望外界直接访问的。 
* 各个业务系统的人无法自由的维护自己负责的服务器； 
* 现有的微服务都是“我家大门常打开”，没有做权限校验。如果把权限校验代码写到每 个微服务上，那么开发工作量太大。 
*  很难做限流、收费等。 

ocelot 中文文档：`https://blog.csdn.net/sD7O95O/article/details/79623654`

资料：`http://www.csharpkit.com/apigateway.html`

官网：`https://github.com/ThreeMammals/Ocelot`

### 一、Ocelot 基本设置

Ocelot 就是一个提供了请求路由、安全验证等功能的 API 网关微服务。 

建一个空的 asp.net core 项目。 

`Install-Package Ocelot `

项目根目录下创建 configuration.json 

ReRoutes 下就是多个路由规则 

```json
{  
   "ReRoutes":[  
      {  
         "DownstreamPathTemplate":"/api/{url}",
         "DownstreamScheme":"http",
         "DownstreamHostAndPorts":[  
            {  
               "Host":"localhost",
               "Port":5001
            }
         ],
         "UpstreamPathTemplate":"/MsgService/{url}",
         "UpstreamHttpMethod":[  
            "Get",
            "Post"
         ]
      },
      {  
         "DownstreamPathTemplate":"/api/{url}",
         "DownstreamScheme":"http",
         "DownstreamHostAndPorts":[  
            {  
               "Host":"localhost",
               "Port":5003
            }
         ],
         "UpstreamPathTemplate":"/ProductService/{url}",
         "UpstreamHttpMethod":[  
            "Get",
            "Post"
         ]
      }
   ]
}
```

Program.cs的CreateWebHostBuilder中 

```c#
.UseUrls("http://127.0.0.1:8888")
.ConfigureAppConfiguration((hostingContext, builder) => {
	builder.AddJsonFile("configuration.json",false, true);
})
```

在ConfigureAppConfiguration 中AddJsonFile是解析json配置文件的方法。为了确保直接在bin 下直接dotnet运行的时候能找到配置文件，所以要在vs中把配置文件的【复制到输出目录】设置为【如 果如果较新则复制】。 

Startup.cs中 

通过构造函数注入一个private IConfiguration Configuration; 

```c#
public void ConfigureServices(IServiceCollection services)
{
    services.AddOcelot(Configuration);
}
public void Configure(IApplicationBuilder app, IHostingEnvironment env)
{
    if (env.IsDevelopment())
    {
        app.UseDeveloperExceptionPage();
    }
    app.UseOcelot().Wait();//不要忘了写Wait
}
```

这样当访问`http://127.0.0.1:8888/MsgService/Send?msg=aaa`的时候就会访问

`http://127.0.0.1:5002/api/email/Send?msg=aaa`

UpstreamHttpMethod表示对什么样的请求类型做转发。 

### 二、Ocelot + Consul

上面的配置还是把服务的ip地址写死了，Ocelot可以和Consul通讯，通过服务名字来配 置。 

只要改配置文件即可 

```json
{  
   "ReRoutes":[  
      {  
         "DownstreamPathTemplate":"/api/{url}",
         "DownstreamScheme":"http",
         "UpstreamPathTemplate":"/MsgService/{url}",
         "UpstreamHttpMethod":[  
            "Get",
            "Post"
         ],
         "ServiceName":"MsgService",
         "LoadBalancerOptions":{  
            "Type":"RoundRobin"
         },
         "UseServiceDiscovery":true
      }
   ],
   "GlobalConfiguration":{  
      "ServiceDiscoveryProvider":{  
         "Host":"localhost",
         "Port":8500
      }
   }
}
```

有多个服务就 ReRoutes 下面配置多组即可 访 问 http://localhost:8888/MsgService/SMS/Send_MI 即 可 ， 请 求 报 文 体 

```json
{
	phoneNum:"110",
	msg:"aaaaaaaaaaaaa"
} 
```

表示只要是/MsgService/开头的都会转给后端的服务名为" MsgService "的一台服务器，转发的路径"/api/{url}"。LoadBalancerOptions 中"LeastConnection"表示负载均衡算法是“选择当前最少连接数的服务器”，如果改为 RoundRobin 就是“轮询”。ServiceDiscoveryProvider是 Consul 服务器的配置。
Ocelot 因为是流量中枢，也是可以做集群的。

(*)也支持Eureka进行服务的注册、查找（`http://ocelot.readthedocs.io/en/latest/features/servicediscovery.html`），也支持访问 Service
Fabric 中的服务（`http://ocelot.readthedocs.io/en/latest/features/servicefabric.html`）。

### 三、Ocelot 其他功能简单介绍 

* 1、 限流：

  文档：`http://ocelot.readthedocs.io/en/latest/features/ratelimiting.html`

  需要和 Identity Server 一起使用，其他的限速是针对 clientId 限速，而不是针对 ip 限速。比如我调用微博的 api 开发了一个如鹏版新浪微博，我的 clientid 是 rpwb，然后限制了 1 秒钟只能调用 1000 次，那么所有用如鹏版微博这个 app 的所有用户加在一起，在一秒钟之内，不能累计超过 1000 次。目前开放式 api 的限流都是这个套路。

  如果要做针对 ip 的限速等，要自己在 Ocelot 前面架设 Nginx 来实现。

* 2、请求缓存
  `http://ocelot.readthedocs.io/en/latest/features/caching.html`
  只支持 get，只要 url 不变，就会缓存。

* 3、QOS (熔断器)

  `http://ocelot.readthedocs.io/en/latest/features/qualityofservice.html`



