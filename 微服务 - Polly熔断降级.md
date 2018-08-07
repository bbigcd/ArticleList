

### 一、什么是熔断降级

熔断就是“保险丝”。当出现某些状况时，切断服务，从而防止应用程序不断地尝试执 行可能会失败的操作给系统造成“雪崩”，或者大量的超时等待导致系统卡死。 

降级的目的是当某个服务提供者发生故障的时候，向调用方返回一个错误响应或者替代 响应。举例子：调用联通接口服务器发送短信失败之后，改用移动短信服务器发送，如果移 动短信服务器也失败，则改用电信短信服务器，如果还失败，则返回“失败”响应；在从推 荐商品服务器加载数据的时候，如果失败，则改用从缓存中加载，如果缓存中也加载失败， 则返回一些本地替代数据。 

### 二、Polly 简介

.Net Core 中有一个被.Net 基金会认可的库 Polly，可以用来简化熔断降级的处理。主要 功能：重试（Retry）；断路器（Circuit-breaker）；超时检测（Timeout）；缓存（Cache）； 降级（FallBack）； 

官网：`https://github.com/App-vNext/Polly `

介绍文章：`https://www.cnblogs.com/CreateMyself/p/7589397.html`

`Install-Package Polly -Version 6.0.1 `

Polly 的策略由“故障”和“动作”两部分组成，“故障”包括异常、超时、返回值错 误等情况，“动作”包括

* FallBack（降级）
* 重试（Retry）
* 熔断（Circuit-breaker）等。

策略用来执行可能会有有故障的业务代码，当业务代码出现“故障”中情况的时候就执 行“动作”。

由于实际业务代码中故障情况很难重现出来，所以 Polly 这一些都是用一些无意义的代 码模拟出来。 

Polly 也支持请求缓存“数据不变化则不重复自行代码”，但是和新版本兼容不好，而 且功能局限性很大，因此这里不讲。 

由于调试器存在，看不清楚 Polly 的执行过程，因此本节都用【开始执行（不调试）】 

### 三、Polly 简单使用

使用Policy的静态方法创建ISyncPolicy实现类对象，创建方法既有同步方法也有异步方法，根 据自己的需要选择。下面先演示同步的，异步的用法类似。 

举例：当发生ArgumentException异常的时候，执行Fallback代码。 

```c#
Policy policy = Policy
.Handle<ArgumentException>() //故障
.Fallback(() =>//动作
{
Console.WriteLine("执行出错");
});
policy.Execute(() => {//在策略中执行业务代码
    //这里是可能会产生问题的业务系统代码
	Console.WriteLine("开始任务");
	throw new ArgumentException("Hello world!");
	Console.WriteLine("完成任务");
});
Console.ReadKey();
```

如果没有被Handle处理的异常，则会导致未处理异常被抛出。

还可以用Fallback的其他重载获取异常信息： 

```c#
Policy policy = Policy
.Handle<ArgumentException>() //故障
.Fallback(() =>//动作
{
	Console.WriteLine("执行出错");
},ex=> {
	Console.WriteLine(ex);
});
policy.Execute(() => {
	Console.WriteLine("开始任务");
	throw new ArgumentException("Hello world!");
	Console.WriteLine("完成任务");
});
```

如果Execute中的代码是带返回值的，那么只要使用带泛型的Policy类即可： 

```c#
Policy<string> policy = Policy<string>
.Handle<Exception>() //故障
.Fallback(() =>//动作
{
	Console.WriteLine("执行出错");
	return "降级的值";
});
string value = policy.Execute(() => {
	Console.WriteLine("开始任务");
	throw new Exception("Hello world!");
	Console.WriteLine("完成任务");
	return "正常的值";
});
Console.WriteLine("返回值："+value);
```

FallBack的重载方法也非常多，有的异常可以直接提供降级后的值。 

* 异常中还可以通过lambda表达式对异常判断“满足***条件的异常我才处理”，简单看看试试 重载即可。还可以多个Or处理各种不同的异常。 
* 还可以用HandleResult等判断返回值进行故障判断等，我感觉没太大必要。 

### 四、重试处理

```c#
Policy policy = Policy
.Handle<Exception>()
.RetryForever();
policy.Execute(() => {
	Console.WriteLine("开始任务");
	if (DateTime.Now.Second % 10 != 0)
	{
		throw new Exception("出错");
	}
	Console.WriteLine("完成任务");
});

```

* RetryForever()是一直重试直到成功 
* Retry()是重试最多一次
* Retry(n) 是重试最多n次
* WaitAndRetry()可以实现“如果出错等待100ms再试还不行再等150ms秒。。。。”，重载方法很 多，不再一一介绍。还有WaitAndRetryForever。 

### 五、短路保护 Circuit Breaker 

出现N次连续错误，则把“熔断器”（保险丝）熔断，等待一段时间，等待这段时间内如果再Execute 则直接抛出BrokenCircuitException异常，根本不会再去尝试调用业务代码。等待时间过去之后，再 执行Execute的时候如果又错了（一次就够了），那么继续熔断一段时间，否则就恢复正常。 

这样就避免一个服务已经不可用了，还是使劲的请求给系统造成更大压力。 

```c#
Policy policy = Policy
 .Handle<Exception>()
 .CircuitBreaker(6,TimeSpan.FromSeconds(5));//连续出错6次之后熔断5秒(不会再去尝试执行业务代码）。
while(true)
{
	Console.WriteLine("开始Execute");
	try
	{
		policy.Execute(() => {
			Console.WriteLine("开始任务");
			throw new Exception("出错");
			Console.WriteLine("完成任务");
    	});
	}
	catch(Exception ex)
	{
		Console.WriteLine("execute出错"+ex);
	}
	Thread.Sleep(500);
} 
```

其计数的范围是policy对象，所以如果想整个服务器全局对于一段代码做短路保护，则需要共用 一个policy对象。 

`https://andrewlock.net/when-you-use-the-polly-circuit-breaker-make-sure-you-share-your-policy-instances-2/`

### 六、策略封装

可以把多个ISyncPolicy合并到一起执行:

```c#
policy3 = policy1.Wrap(policy2); 
```

执行policy3就会把policy1、policy2封装到一起执行

```c#
policy9 = Policy.Wrap(policy1, policy2, policy3, policy4, policy5);
```

把更多一起封装。

### 七、超时处理

这些处理不能简单的链式调用，要用到Wrap。例如下面实现“出现异常则重试三次，如果还出错 就FallBack”这样是不行的 

```c#
Policy policy = Policy
.Handle<Exception>()
.Retry(3)
.Fallback(()=> { Console.WriteLine("执行出错"); });//这样不行
policy.Execute(() => {
	Console.WriteLine("开始任务");
	throw new ArgumentException("Hello world!");
	Console.WriteLine("完成任务");
});
```

注意Wrap是有包裹顺序的，内层的故障如果没有被处理则会抛出到外层。 

下面代码实现了“出现异常则重试三次，如果还出错就FallBack

```c#
Policy policyRetry = Policy.Handle<Exception>()
.Retry(3);
Policy policyFallback = Policy.Handle<Exception>()
.Fallback(()=> {
	Console.WriteLine("降级");
});
//Wrap：包裹。policyRetry在里面，policyFallback裹在外面。
//如果里面出现了故障，则把故障抛出来给外面
Policy policy = policyFallback.Wrap(policyRetry);
policy.Execute(()=> {
	Console.WriteLine("开始任务");
	if (DateTime.Now.Second % 10 != 0)
	{
		throw new Exception("出错");
	}
	Console.WriteLine("完成任务");
});
```

Timeout是定义超时故障。 

```c#
Policy policy = Policy.Timeout(3, TimeoutStrategy.Pessimistic);// 创建一个3秒钟（注意单位）的超时策略。
```

Timeout生成的Policy要和其他Policy一起Wrap使用。 

超时策略一般不能直接用，而是和其他封装到一起用： 

```c#
Policy policy = Policy
.Handle<Exception>() //定义所处理的故障
.Fallback(() =>
{
	Console.WriteLine("执行出错");
});
policy = policy.Wrap(Policy.Timeout(2, TimeoutStrategy.Pessimistic));
policy.Execute(()=> {
	Console.WriteLine("开始任务");
	Thread.Sleep(5000);
	Console.WriteLine("完成任务");
});
```

上面的代码就是如果执行超过2秒钟，则直接Fallback。 

这个的用途：请求网络接口，避免接口长期没有响应造成系统卡死 。

### 八、 Polly 的异步用法 

所有方法都用Async方法即可，Handle由于只是定义异常，所以不需要异常方法： 带返回值的例子： 

```c#
Policy<byte[]> policy = Policy<byte[]>
.Handle<Exception>()
.FallbackAsync(async c => {
	Console.WriteLine("执行出错");
	return new byte[0];
},async r=> {
	Console.WriteLine(r.Exception);
});
policy = policy.WrapAsync(Policy.TimeoutAsync(20, TimeoutStrategy.Pessimistic,
async(context, timespan, task) =>
{
	Console.WriteLine("timeout");
}));
var bytes = await policy.ExecuteAsync(async () => {
	Console.WriteLine("开始任务");
	HttpClient httpClient = new HttpClient();
	var result = await
		httpClient.GetByteArrayAsync(
        "http://static.rupeng.com/upload/chatimage/20183/07EB793A4C247A654B31B4D14EC64BCA.png");
	Console.WriteLine("完成任务");
	return result;
});
Console.WriteLine("bytes长度"+bytes.Length);
```

没返回值的例子 

```c#
Policy policy = Policy
.Handle<Exception>()
.FallbackAsync(async c => {
	Console.WriteLine("执行出错");
},async ex=> {//对于没有返回值的，这个参数直接是异常
	Console.WriteLine(ex);
});
policy = policy.WrapAsync(Policy.TimeoutAsync(3, TimeoutStrategy.Pessimistic,
async(context, timespan, task) =>
{
	Console.WriteLine("timeout");
}));
await policy.ExecuteAsync(async () => {
	Console.WriteLine("开始任务");
await Task.Delay(5000);//注意不能用Thread.Sleep(5000);
	Console.WriteLine("完成任务");
});
```

### 九、 AOP 框架基础 

要求懂的知识：AOP、Filter、反射（Attribute）。 

如果直接使用 Polly，那么就会造成业务代码中混杂大量的业务无关代码。我们使用 AOP （如果不了解 AOP，请自行参考网上资料）的方式封装一个简单的框架，模仿 Spring cloud 中的 Hystrix。 

需要先引入一个支持.Net Core 的 AOP，目前我发现的最好的.Net Core 下的 AOP 框架是 AspectCore（国产，动态织入），其他要不就是不支持.Net Core，要不就是不支持对异步方法进行 拦截。MVC Filter

GitHub：`https://github.com/dotnetcore/AspectCore-Framework`

`Install-Package AspectCore.Core -Version 0.5.0`

这里只介绍和我们相关的用法： 

* 1、编写拦截器 CustomInterceptorAttribute 一般继承自 AbstractInterceptorAttribute 

  ```c#
  public class CustomInterceptorAttribute : AbstractInterceptorAttribute
  {
   	//每个被拦截的方法中执行
  	public async override Task Invoke(AspectContext context, AspectDelegate next)
  	{
  		try
  		{
  			Console.WriteLine("Before service call");
  			await next(context);//执行被拦截的方法
  		}
  		catch (Exception)
  		{
  			Console.WriteLine("Service threw an exception!");
  			throw;
  		}
  		finally
  		{		
  			Console.WriteLine("After service call");
  		}
  	}
  }
  ```

  ```c#
  AspectContext 的属性的含义：
  Implementation 实际动态创建的 Person 子类的对象。
  ImplementationMethod 就是 Person 子类的 Say 方法
  Parameters 方法的参数值。
  Proxy==Implementation：当前场景下
  ProxyMethod==ImplementationMethod：当前场景下
  ReturnValue 返回值
  ServiceMethod 是 Person 的 Say 方法
  ```

* 2、编写需要被代理拦截的类 

  在要被拦截的方法上标注 CustomInterceptorAttribute 。类需要是 public 类，方法需要是虚 方法，支持异步方法，因为动态代理是动态生成被代理的类的动态子类实现的。 

  ```c#
  public class Person
  {
  	[CustomInterceptor]
  	public virtual void Say(string msg)
  	{
  		Console.WriteLine("service calling..."+msg);
  	}
  }
  ```

* 3、通过 AspectCore 创建代理对象 

  ```c#
  ProxyGeneratorBuilder proxyGeneratorBuilder = new ProxyGeneratorBuilder();
  using (IProxyGenerator proxyGenerator = proxyGeneratorBuilder.Build())
  {
  	Person p = proxyGenerator.CreateClassProxy<Person>();
  	p.Say("rupeng.com");
  }
  Console.ReadKey();
  ```

  注意 p 指向的对象是 AspectCore 生成的 Person 的动态子类的对象，直接 new Person 是无法被 拦截的。 

### 十、创建简单的熔断降级框架

要达到的目标是： 

```c#
public class Person
{
    [HystrixCommand("HelloFallBackAsync")]
	public virtual async Task<string> HelloAsync(string name)
	{
		Console.WriteLine("hello"+name);
		return "ok";
	}
	public async Task<string> HelloFallBackAsync(string name)
	{
		Console.WriteLine("执行失败"+name);
		return "fail";
	}
}
```

参与降级的方法参数要一样。 

当 HelloAsync 执行出错的时候执行 HelloFallBackAsync 方法。 

* 1、编写 HystrixCommandAttribute

  ```c#
  using AspectCore.DynamicProxy;
  using System;
  using System.Threading.Tasks;
  namespace hystrixtest1
  {
      [AttributeUsage(AttributeTargets.Method)]
      public class HystrixCommandAttribute : AbstractInterceptorAttribute
      {
  		public HystrixCommandAttribute(string fallBackMethod)
  		{
  			this.FallBackMethod = fallBackMethod;
  		}
  		public string FallBackMethod { get; set; }
  		public override async Task Invoke(AspectContext context, AspectDelegate next)
  		{
  			try
  			{
  				await next(context);//执行被拦截的方法
  			}
  			catch (Exception ex)
  			{
  				//context.ServiceMethod 被拦截的方法。context.ServiceMethod.DeclaringType被拦截方法所在的类
  				//context.Implementation 实际执行的对象 p
  				//context.Parameters 方法参数值
  				//如果执行失败，则执行 FallBackMethod
  				var fallBackMethod = context.ServiceMethod.DeclaringType.GetMethod(this.FallBackMethod);
  				Object fallBackResult = fallBackMethod.Invoke(context.Implementation,context.Parameters);
  				context.ReturnValue = fallBackResult;
  			}
  		}
      }
  }
  ```

  

* 2、编写类

  ```c#
  public class Person//需要 public 类
  {
  	[HystrixCommand(nameof(HelloFallBackAsync))]
  	public virtual async Task<string> HelloAsync(string name)//需要是虚方法
  	{
  		Console.WriteLine("hello"+name);
  		String s = null;
  		// s.ToString();
  		return "ok";
  	}
  	public async Task<string> HelloFallBackAsync(string name)
  	{
  		Console.WriteLine("执行失败"+name);
  		return "fail";
  	}
  	[HystrixCommand(nameof(AddFall))]
  	public virtual int Add(int i,int j)
  	{
  		String s = null;
  		// s.ToArray();
  		return i + j;
  	}
  	public int AddFall(int i, int j)
  	{
  		return 0;
  	}
  }
  ```

* 3、创建代理对象

  ```c#
  ProxyGeneratorBuilder proxyGeneratorBuilder = new ProxyGeneratorBuilder();
  using (IProxyGenerator proxyGenerator = proxyGeneratorBuilder.Build())
  {
  	Person p = proxyGenerator.CreateClassProxy<Person>();
  	Console.WriteLine(p.HelloAsync("yzk").Result);
  	Console.WriteLine(p.Add(1, 2));
  }
  ```

  上面的代码还支持多次降级，方法上标注[HystrixCommand]并且 virtual 即可：

  ```c#
  public class Person//需要 public 类
  {
  	[HystrixCommand(nameof(Hello1FallBackAsync))]
  	public virtual async Task<string> HelloAsync(string name)//需要是虚方法
  	{
  		Console.WriteLine("hello" + name);
  		String s = null;
  		s.ToString();
  		return "ok";
  	}
  	[HystrixCommand(nameof(Hello2FallBackAsync))]
  	public virtual async Task<string> Hello1FallBackAsync(string name)
  	{
  		Console.WriteLine("Hello 降级 1" + name);
  		String s = null;
  		s.ToString();
  		return "fail_1";
  	}
  	public virtual async Task<string> Hello2FallBackAsync(string name)
  	{
  		Console.WriteLine("Hello 降级 2" + name);
  		return "fail_2";
  	}
  	[HystrixCommand(nameof(AddFall))]
  	public virtual int Add(int i, int j)
  	{
  		String s = null;
  		s.ToString();
  		return i + j;
  	}
  	public int AddFall(int i, int j)
  	{
  		return 0;
  	}
  }
  ```

### 十一、 细化框架 

上面明白了原理，然后直接展示写好的更复杂的 HystrixCommandAttribute，讲解代码，不现场敲了。

这是我会持续维护的开源项目

github 最新地址 `https://github.com/yangzhongke/RuPeng.HystrixCore`

Nuget 地址：`https://www.nuget.org/packages/RuPeng.HystrixCore`

* 重试：MaxRetryTimes 表示最多重试几次，如果为 0 则不重试，RetryIntervalMilliseconds 表示重试间隔的毫秒数；
* 熔断：EnableCircuitBreaker 是否启用熔断，ExceptionsAllowedBeforeBreaking 表示熔断前出现允许错误几次，MillisecondsOfBreak 表示熔断多长时间（毫秒）；
* 超时：TimeOutMilliseconds 执行超过多少毫秒则认为超时（0 表示不检测超时）
* 缓存：CacheTTLMilliseconds 缓存多少毫秒（0 表示不缓存），用“类名+方法名+所有参数值ToString 拼接”做缓存 Key（唯一的要求就是参数的类型 ToString 对于不同对象一定要不一样）。

由于 CircuitBreaker 要求同一段代码必须共享同一个 Policy 对象。一般我们熔断控制是针对一个 方 法 ， 一 个 方 法 无 论 是 通 过 几 个 Person 对 象 调 用 ， 无 论 是 谁 调 用 ， 只 要 全 局 出 现ExceptionsAllowedBeforeBreaking 次错误，就会熔断，这是我框架的实现，你如果认为不合理，你自己改去。

非常抱歉，因为我把.Net 中的 Attribute 和 Java 中类似技术的行为弄混了，在.Net 中每次获取Attribute 获取的都是新的实例。我上课讲错了，我讲的是“每次获取 Attribute 获取的都是相同实例”。下面的代码是我修改后的代码，通过 ConcurrentDictionary 来保证一个方法对应一个 Policy实例。

`Install-Package Microsoft.Extensions.Caching.Memory `

```c#
using System;
using AspectCore.DynamicProxy;
using System.Threading.Tasks;
using Polly;
using System.Collections.Concurrent;
using System.Reflection;
namespace RuPeng.HystrixCore
{
    [AttributeUsage(AttributeTargets.Method)]
    public class HystrixCommandAttribute : AbstractInterceptorAttribute
    {
        /// <summary>
        /// 最多重试几次，如果为0则不重试
        /// </summary>
        public int MaxRetryTimes { get; set; } = 0;
        /// <summary>
        /// 重试间隔的毫秒数
        /// </summary>
        public int RetryIntervalMilliseconds { get; set; } = 100;
        /// <summary>
        /// 是否启用熔断
        /// </summary>
        public bool EnableCircuitBreaker { get; set; } = false;
        /// <summary>
        /// 熔断前出现允许错误几次
        /// </summary>
        public int ExceptionsAllowedBeforeBreaking { get; set; } = 3;
        /// <summary>
        /// 熔断多长时间（毫秒）
        /// </summary>
        public int MillisecondsOfBreak { get; set; } = 1000;
        /// <summary>
        /// 执行超过多少毫秒则认为超时（0表示不检测超时）
        /// </summary>
        public int TimeOutMilliseconds { get; set; } = 0;
        /// <summary>
        /// 缓存多少毫秒（0表示不缓存），用“类名+方法名+所有参数ToString拼接”做缓存Key
        /// </summary>
        public int CacheTTLMilliseconds { get; set; } = 0;
        private static ConcurrentDictionary<MethodInfo, Policy> policies = new ConcurrentDictionary<MethodInfo, Policy>();
        private static readonly Microsoft.Extensions.Caching.Memory.IMemoryCache memoryCache
       = new Microsoft.Extensions.Caching.Memory.MemoryCache(new Microsoft.Extensions.Caching.Memory.MemoryCacheOptions());

        /// <summary>
        /// 
        /// </summary>
        /// <param name="fallBackMethod">降级的方法名</param>
        public HystrixCommandAttribute(string fallBackMethod)
        {
            this.FallBackMethod = fallBackMethod;
        }
        public string FallBackMethod { get; set; }
        public override async Task Invoke(AspectContext context, AspectDelegate next)
        {
            //一个HystrixCommand中保持一个policy对象即可
            //其实主要是CircuitBreaker要求对于同一段代码要共享一个policy对象
            //根据反射原理，同一个方法的MethodInfo是同一个对象，但是对象上取出来的HystrixCommandAttribute
            //每次获取的都是不同的对象，因此以MethodInfo为Key保存到policies中，确保一个方法对应一个policy实例
            policies.TryGetValue(context.ServiceMethod, out Policy policy);
            lock (policies)//因为Invoke可能是并发调用，因此要确保policies赋值的线程安全
            {
                if (policy == null)
                {
                    policy = Policy.NoOpAsync();//创建一个空的Policy
                    if (EnableCircuitBreaker)
                    {
                        policy =
                       policy.WrapAsync(Policy.Handle<Exception>().CircuitBreakerAsync(ExceptionsAllowedBeforeBreaking,
                       TimeSpan.FromMilliseconds(MillisecondsOfBreak)));
                    }
                    if (TimeOutMilliseconds > 0)
                    {
                        policy = policy.WrapAsync(Policy.TimeoutAsync(() =>
                       TimeSpan.FromMilliseconds(TimeOutMilliseconds), Polly.Timeout.TimeoutStrategy.Pessimistic));
                    }
                    if (MaxRetryTimes > 0)
                    {
                        policy =
                       policy.WrapAsync(Policy.Handle<Exception>().WaitAndRetryAsync(MaxRetryTimes, i =>
                       TimeSpan.FromMilliseconds(RetryIntervalMilliseconds)));
                    }
                    Policy policyFallBack = Policy
                    .Handle<Exception>()
                    .FallbackAsync(async (ctx, t) =>
                    {
                        AspectContext aspectContext = (AspectContext)ctx["aspectContext"];
                        var fallBackMethod =
    context.ServiceMethod.DeclaringType.GetMethod(this.FallBackMethod);
                        Object fallBackResult = fallBackMethod.Invoke(context.Implementation,
    context.Parameters);
                        //不能如下这样，因为这是闭包相关，如果这样写第二次调用Invoke的时候context指向的
                        //还是第一次的对象，所以要通过Polly的上下文来传递AspectContext
                        //context.ReturnValue = fallBackResult;
                        aspectContext.ReturnValue = fallBackResult;
                    }, async (ex, t) => { });
                    policy = policyFallBack.WrapAsync(policy);
                    //放入
                    policies.TryAdd(context.ServiceMethod, policy);
                }
            }
            //把本地调用的AspectContext传递给Polly，主要给FallbackAsync中使用，避免闭包的坑
            Context pollyCtx = new Context();
            pollyCtx["aspectContext"] = context;
            //Install-Package Microsoft.Extensions.Caching.Memory
            if (CacheTTLMilliseconds > 0)
            {
                //用类名+方法名+参数的下划线连接起来作为缓存key
                string cacheKey = "HystrixMethodCacheManager_Key_" +
               context.ServiceMethod.DeclaringType
                + "." +
               context.ServiceMethod + string.Join("_", context.Parameters);
                //尝试去缓存中获取。如果找到了，则直接用缓存中的值做返回值
                if (memoryCache.TryGetValue(cacheKey, out var cacheValue))
                {
                    context.ReturnValue = cacheValue;
                }
                else
                {
                    //如果缓存中没有，则执行实际被拦截的方法
                    await policy.ExecuteAsync(ctx => next(context), pollyCtx);
                    //存入缓存中
                    using (var cacheEntry = memoryCache.CreateEntry(cacheKey))
                    {
                        cacheEntry.Value = context.ReturnValue;
                        cacheEntry.AbsoluteExpiration = DateTime.Now +
                        TimeSpan.FromMilliseconds(CacheTTLMilliseconds);
                    }
                }
            }
            else//如果没有启用缓存，就直接执行业务方法
            {
                await policy.ExecuteAsync(ctx => next(context), pollyCtx);
            }
        }
    }
}
```

没必要、也不可能把所有 Polly 都封装到 Hystrix 中。框架不是万能的，不用过度框架，过度框 架带来的复杂度陡增，从人人喜欢变成人人恐惧。 

### 十二、 结合 asp.net core 依赖注入 

在 asp.net core 项目中，可以借助于 asp.net core 的依赖注入，简化代理类对象的注入，不用 再自己调用 ProxyGeneratorBuilder 进行代理类对象的注入了。 

`Install-Package AspectCore.Extensions.DependencyInjection `

修改 Startup.cs 的 ConfigureServices 方法，把返回值从 void 改为 IServiceProvider 

```c#
using AspectCore.Extensions.DependencyInjection;
public IServiceProvider ConfigureServices(IServiceCollection services)
{
    services.AddMvc();
    services.AddSingleton<Person>();
    return services.BuildAspectCoreServiceProvider();
}
```

其 中 services.AddSingleton(); 表 示 把 Person 注 入 。 BuildAspectCoreServiceProvider 是让 aspectcore 接管注入。 

在 Controller 中就可以通过构造函数进行依赖注入了： 

```c#
public class ValuesController : Controller
{
    private Person p;
    public ValuesController(Person p)
    {
        this.p = p;
    }
}
```

当然要通过反射扫描所有 Service 类，只要类中有标记了 CustomInterceptorAttribute 的方法 都算作服务实现类。为了避免一下子扫描所有类，所以 RegisterServices 还是手动指定从哪个程序集中加载。

```c#
public IServiceProvider ConfigureServices(IServiceCollection services)
{
    services.AddMvc();
    RegisterServices(this.GetType().Assembly, services);
    return services.BuildAspectCoreServiceProvider();
}
private static void RegisterServices(Assembly asm, IServiceCollection services)
{
    //遍历程序集中的所有 public 类型
    foreach (Type type in asm.GetExportedTypes())
    {
        //判断类中是否有标注了 CustomInterceptorAttribute 的方法
        bool hasCustomInterceptorAttr = type.GetMethods()
         .Any(m => m.GetCustomAttribute(typeof(CustomInterceptorAttribute)) != null);
        if (hasCustomInterceptorAttr)
        {
            services.AddSingleton(type);
        }
    }
}
```

RestTemplate+Hystrix+之前测试用的两个微服务，结合到一起组合演示一下。 

如鹏网《.Net微服务》-by杨中科 