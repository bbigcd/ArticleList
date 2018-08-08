### Dapper框架

#### [Dapper](<https://stackexchange.github.io/Dapper/> ) 是什么？

* ORM概念

  ORM：全称`Object Relational Mapping` ，中文意思是 `对象关系映射`

  wiki中的解释：

  > **对象关系映射**（英语：**Object Relational Mapping**，简称**ORM**，或**O/RM**，或**O/R mapping**），是一种[程序设计](https://zh.wikipedia.org/wiki/%E7%A8%8B%E5%BC%8F%E8%A8%AD%E8%A8%88)技术，用于实现[面向对象](https://zh.wikipedia.org/wiki/%E7%89%A9%E4%BB%B6%E5%B0%8E%E5%90%91)编程语言里不同[类型系统](https://zh.wikipedia.org/wiki/%E9%A1%9E%E5%9E%8B%E7%B3%BB%E7%B5%B1)的数据之间的转换。从效果上说，它其实是创建了一个可在编程语言里使用的“虚拟[对象数据库](https://zh.wikipedia.org/wiki/%E7%89%A9%E4%BB%B6%E8%B3%87%E6%96%99%E5%BA%AB)”。 

* Dapper就是.NET下的一种ORM框架，在.NET中的常用ORM框架有：

  - [SqlSugar](https://github.com/sunkaixuan/SqlSugar) (国内)
  - [Dos.ORM](https://github.com/itdos/Dos.ORM) (国内)
  - [Chloe](http://www.52chloe.com/) (国内)
  - [StackExchange/Dapper](https://github.com/StackExchange/Dapper) (国外)
  - [Entity Framework (EF)](https://github.com/aspnet/EntityFramework6) (国外)
  - [NHibernate](http://nhibernate.info/) (国外)
  - [ServiceStack/ServiceStack.OrmLite](https://github.com/ServiceStack/ServiceStack.OrmLite) (国外)
  - [linq2db](https://github.com/linq2db/linq2db) (国外)
  - [Massive](https://github.com/FransBouma/Massive) (国外)
  - [PetaPoco](https://github.com/CollaboratingPlatypus/PetaPoco) (国外)

* Dapper的优点

  * 轻量；
  * 速度快。Dapper的速度接近与IDataReader，取列表的数据超过了DataTable；
  * 支持多种数据库。Dapper可以在所有Ado.net Providers下工作，包括sqlite, sqlce, firebird, oracle, MySQL, PostgreSQL and SQL Server；
  * 可以映射一对一，一对多，多对多等多种关系；
  * 性能高。通过Emit反射IDataReader的序列队列，来快速的得到和产生对象，性能不错；
  * 支持FrameWork2.0，3.0，3.5，4.0，4.5；
  * Dapper语法十分简单。并且无须迁就数据库的设计。

#### 安装

新建一个控制台程序

使用NuGet安装

* Package Manager

  > Install-Package Dapper

* .NET CLI

  > dotnet add package Dapper 

这里数据库演示使用MySQL，需安装MySql.Data包，同样使用NuGet安装

- Package Manager

  > Install-Package MySql.Data 

- .NET CLI

  > dotnet add package MySql.Data 

注意，如果是安装最新的版本，则包的依赖框架会比较新，导致安装失败

解决方式为，更改控制台的.NET版本到符合的依赖版本，或者安装较低的MySql.Data版本

#### 使用

* 新建实体类

  ```c#
  public class User
  {
      public int Id { get; set; }//自增主键
      public string Name { get; set; }
      public int Age { get; set; }
      public override string ToString()
      {
          return "{ Id：" + Id + ", Name: " + Name + ", Age: " + Age + "}";
      }
  }
  ```

* 增

  ```c#
  var user = new User();
  user.Name = "bbigcd";
  user.Age = 24;
  using (var conn = new MySqlConnection(connectionString))
  {
      string sqlCommandText = @"insert into user(Name,Age)values(@Name,@Age)";
      try
      {
          int result = conn.Execute(sqlCommandText, user);
          Console.WriteLine(result);
      }
      catch(Exception ex)
      {
          Console.WriteLine(ex.ToString());
      }
  }
  ```

* 删

  ```c#
  using (var conn = new MySqlConnection(connectionString))
  {
      User user = new User();
      user.Id = 1;
      string sqlCommandText = @"delete from user where Id = {=Id}";
      try
      {
          int result = conn.Execute(sqlCommandText, user);
          Console.WriteLine(result);
      }
      catch (Exception ex)
      {
          Console.WriteLine(ex.ToString());
      }
  }
  ```

* 改

  ```c#
  using (var conn = new MySqlConnection(connectionString))
  {
      User user = new User();
      user.Id = 1;
      user.Age = 80;
      string sqlCommandText = @"update user set Age={=Age} where Id = {=Id}";
      try
      {
          int result = conn.Execute(sqlCommandText, user);
          Console.WriteLine(result);
      }
      catch (Exception ex)
      {
          Console.WriteLine(ex.ToString());
      }
  }
  ```

* 查

  ```c#
  using (var conn = new MySqlConnection(connectionString))
  {
      string sqlCommandText = @"select * from user";
      var users = conn.Query<User>(sqlCommandText);
      foreach (var item in users)
      {
          Console.WriteLine(item);
      }
  };
  ```

* 事务

  ```c#
  using (var conn = new MySqlConnection(connectionString))
  {
      conn.Open();// 需要打开，否则失败
      IDbTransaction trans = conn.BeginTransaction();
      string sqlCommandText1 = @"update user set name='www.lanhusoft.com' where id=@id";
      string sqlCommandText2 = @"delete from user where id=@id";
      int row = conn.Execute(sqlCommandText1, new { id = 3 }, trans);
      row += conn.Execute(sqlCommandText2, new { id = 4 }, trans);
      trans.Commit();
      conn.Close();
  }
  ```

#### 释义

以查询的例子解释：

1. 创建MySql.Data.MySqlClient.MySqlConnection类，使用using的方式，方便执行完操作后内存被及时回收；

2. 创建sql语句，查询user表中的数据；

3. MySqlConnection的Query方法为Dapper框架中对MySqlConnection类的拓展，以拓展的方式，直接查询出数据，并映射到User这个实体类上，以很简洁的方式实现了对象关系映射（ORM）；

   Query有多个重载方法，并且支持异步方式，以下是上述的查询中使用到的方法：

   ```c#
   public static IEnumerable<T> Query<T>(this IDbConnection cnn,
    string sql,
    object param = null,
    IDbTransaction transaction = null,
    bool buffered = true,
    int? commandTimeout = null,
    CommandType? commandType = null);
   ```