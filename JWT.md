### JWT 算法

### 一、JWT简介

内部 Restful 接口可以“我家大门常打开”，但是如果要给 app 等使用的接口，则需要 做权限校验，不能谁都随便调用。 

Restful 接口不是 web 网站，App 中很难直接处理 SessionId，而且 Cookie 有跨域访问的 限制，所以一般不能直接用后端 Web 框架内置的 Session 机制。但是可以用类似 Session 的 机制，用户登录之后返回一个类似 SessionId 的东西，服务器端把 SessionId 和用户的信息对 应关系保存到 Redis 等地方，客户端把 SessionId 保存起来，以后每次请求的时候都带着这个 SessionId。 

用类似 Session 这种机制的坏处：需要集中的 Session 机制服务器；不可以在 nginx、CDN 等静态文件处理服务器上校验权限；每次都要根据 SessionId 去 Redis 服务器获取用户信息， 效率低； 

JWT（Json Web Token）是现在流行的一种对 Restful 接口进行验证的机制的基础。JWT 的 特点：把用户信息放到一个 JWT 字符串中，用户信息部分是明文的，再加上一部分签名区 域，签名部分是服务器对于“明文部分+秘钥”加密的，这个加密信息只有服务器端才能解 析。用户端只是存储、转发这个 JWT 字符串。如果客户端篡改了明文部分，那么服务器端 解密时候会报错。 

JWT 由三块组成，可以把用户名、用户 Id 等保存到 Payload 部分 ：

![](https://upload-images.jianshu.io/upload_images/1366611-4f38fe9639ce7a6c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

注意Payload和Header部分都是Base64编码，可以轻松的Base64解码回来。因此Payload 部分约等于是明文的，因此不能在 Payload 中保存不能让别人看到的机密信息。虽然说 Payload 部分约等于是明文的，但是不用担心 Payload 被篡改，因为 Signature 部分是根据 header+payload+secretKey 进行加密算出来的，如果 Payload 被篡改，就可以根据 Signature 解密时候校验。 



用 JWT 做权限验证的好处：无状态，更有利于分布式系统，不需要集中的 Session 机制 服务器；可以在 nginx、CDN 等静态文件处理服务器上校验权限；获取用户信息直接从 JWT 中就可以读取，效率高； 



### 二、.NET 中使用 JWT 算法

* 1）加密

  Install-Packgae JWT

  ```c#
  var payload = new Dictionary<string, object>{
       { "UserId", 123 },
       { "UserName", "admin" }
  };
  var secret = "GQDstcKsx0NHjPOuXOYg5MbeJ1XT0uFiwDVvVBrk";//不要泄露
  IJwtAlgorithm algorithm = new HMACSHA256Algorithm();
  IJsonSerializer serializer = new JsonNetSerializer();
  IBase64UrlEncoder urlEncoder = new JwtBase64UrlEncoder();
  IJwtEncoder encoder = new JwtEncoder(algorithm, serializer, urlEncoder);
  var token = encoder.Encode(payload, secret);
  Console.WriteLine(token);
  ```

* 2 ) 解密

  ```c#
  var token =
  "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJVc2VySWQiOjEyMywiVXNlck5hbWUiOiJhZG1pbiJ9.Qjw1epD5P6p4Yy2yju3-fkq28PddznqRj3ESfALQy_U";
  var secret = "GQDstcKsx0NHjPOuXOYg5MbeJ1XT0uFiwDVvVBrk";
  try
  {
      IJsonSerializer serializer = new JsonNetSerializer();
      IDateTimeProvider provider = new UtcDateTimeProvider();
      IJwtValidator validator = new JwtValidator(serializer, provider);
      IBase64UrlEncoder urlEncoder = new JwtBase64UrlEncoder();
      IJwtDecoder decoder = new JwtDecoder(serializer, validator, urlEncoder);
      var json = decoder.Decode(token, secret, verify: true);
      Console.WriteLine(json);
  }
  catch (FormatException)
  {
      Console.WriteLine("Token format invalid");
  }
  catch (TokenExpiredException)
  {
      Console.WriteLine("Token has expired");
  }
  catch (SignatureVerificationException)
  {
      Console.WriteLine("Token has invalid signature");
  }
  ```

  试着篡改一下 Payload 部分。 

* 3 ) 过期时间

  在 payload 中增加一个名字为 exp 的值，值为过期时间和 1970/1/1 00:00：00 相差的秒数 

  ```c#
  double exp = (DateTime.UtcNow.AddSeconds(10) - new DateTime(1970, 1, 1)).TotalSeconds; 
  ```

* 4）不用密钥解析数据 payload

  因为 payload 部分是明文的，所以在不知道秘钥的时候也可以用 Decode、DecodeToObject 等不 需要秘钥的方法把 payload 部分解析出来。 

  ```c#
  var token = "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJVc2VySWQiOjEyMywiVXNlck5hbWUiOiJhZG1pbiJ9.Qjw1epD5P6p4Yy2yju3-fkq28PddznqRj3ESfALQy_U";
  try
  {
      IJsonSerializer serializer = new JsonNetSerializer();
      IDateTimeProvider provider = new UtcDateTimeProvider();
      IJwtValidator validator = new JwtValidator(serializer, provider);
      IBase64UrlEncoder urlEncoder = new JwtBase64UrlEncoder();
      IJwtDecoder decoder = new JwtDecoder(serializer, validator, urlEncoder);
      var json = decoder.Decode(token);
      Console.WriteLine(json);
  }
  catch (FormatException)
  {
      Console.WriteLine("Token format invalid");
  }
  catch (TokenExpiredException)
  {
      Console.WriteLine("Token has expired");
  }
  ```

  