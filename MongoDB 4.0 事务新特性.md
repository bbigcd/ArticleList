### 安装MongoDB 4.0

下载地址: `https://www.mongodb.com/download-center?jmp=nav#community`

根据自己的操作系统选择对应的版本，本文在 windows 64位环境中做演示。

下载64位的msi安装包，msi安装包可以自动生成服务，无需手动将 MongoDB 配置成服务。

### 开启副本集

由于MongoDB 4.0的事务新特性只能在副本集模式下使用，所以需要对数据库的配置文件进行修改，以副本集的模式开启数据库。

以msi安装包情况，找到数据库目录中的 bin 文件夹中的 `mongod.cfg`配置文件，修改配置成：

```yaml
# mongod.conf

# for documentation of all options, see:
#   http://docs.mongodb.org/manual/reference/configuration-options/

# Where and how to store data.
storage:
  dbPath: F:\Mongodb4.0\data
  journal:
    enabled: true
#  engine:
#  mmapv1:
#  wiredTiger:

# where to write logging data.
systemLog:
  destination: file
  logAppend: true
  path:  F:\Mongodb4.0\log\mongod.log

# network interfaces
net:
  port: 27017
  bindIp: 127.0.0.1


#processManagement:

#security:

#operationProfiling:

replication:
  replSetName: "foo"
#sharding:

## Enterprise-Only Options:

#auditLog:

#snmp:

```

较原配置文件需要修改的地方为：

```yaml
replication:
  replSetName: "foo"
```

保存修改后的配置文件，重启数据库后，MongoDB即以副本集模式启动。

### Robo - 3T 客户端

下载地址: `https://robomongo.org/`

### 创建唯一索引



### 事务特性

使用NuGet包管理控制器安装MongoDB C# 驱动，版本需为2.7.0以上（文章当前最新版本）

如果是VS IDE中使用：

> Install-Package MongoDB.Driver -Version 2.7.0 

DotNet中使用:

> dotnet add package MongoDB.Driver --version 2.7.0 

创建一个控制台应用程序，在Main中添加如下代码

```c#
static void Main(string[] args)
{
    var client = new MongoClient("mongodb://localhost:27017/?replicaSet=foo");
    var database = client.GetDatabase("foo");
    var collection = database.GetCollection<BsonDocument>("test");
    var collection1 = database.GetCollection<BsonDocument>("test1");

    Console.WriteLine("开始事务");
    using (var session = client.StartSession())
    {
        session.StartTransaction();

        var d1 = new BsonDocument("Name", "Jack1");
        var d2 = new BsonDocument("Name", "Jack1");

        collection1.InsertOne(session, d2);//test1 collection
        collection.InsertOne(session, d1);//test collection

        session.CommitTransaction();//提交事务
        // session.AbortTransaction();//中止事务
    };
    Console.WriteLine("结束事务");
}
```

注意点：

在事务的seesion中，必需使用含有 `IClientSessionHandle `参数的API，例如：

```c#
collection.InsertOne(session, d1);
```

InsertOne 中的 session 即 `IClientSessionHandle `

假如将该插入方式换成如下：

```c#
session.Client.GetDatabase("foo").GetCollection<BsonDocument>("test1").InsertOne(new BsonDocument("Name", "Jack1"));
```

那么无论事务成功还是失败亦或者回滚事务，在数据库可以插入的情况下，该次操作都可以被执行成功，即失去了事务的控制。

异步的事务提交方式如下：

