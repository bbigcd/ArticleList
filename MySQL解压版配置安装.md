官网下载地址

`https://downloads.mysql.com/archives/community/`

选择对应的操作系统，依据自己系统的版本选择32位或64位版本

将下载好的安装包解压到指定的路径中

打开解压后的文件，进入bin文件夹，bin文件中主要是mysql的执行命令的功能

准备配置文件

* 配置文件

  ```mysql
  [mysql]
  # 设置mysql客户端默认字符集
  default-character-set=utf8
  
  [mysqld]
  #设置3306端口
  port = 3306
  
  # 设置mysql的安装目录
  basedir=D:\database\mysql
  
  # 设置mysql数据库的数据的存放目录
  datadir=D:\database\mysql\data
  
  # 允许最大连接数
  max_connections=200
  
  # 服务端使用的字符集默认为8比特编码的latin1字符集
  character-set-server=utf8
  
  # 创建新表时将使用的默认存储引擎
  default-storage-engine=INNODB
  ```


* 初始化数据库并指定用户为mysql

  ```mysql
  mysqld --initialize --user=mysql --console
  ```

  生成一个密码：需记住

* 使用密码连接数据库

  ```mysql
  mysql -uroot -p
  ```

  按照提示，输入密码

* 重置密码:

  ```mysql
  set password for root@localhost = password('123'); （注意分号）
  ```

* 配置成服务启动模式

  ```mysql
  mysqld --install MySQL
  ```

  

