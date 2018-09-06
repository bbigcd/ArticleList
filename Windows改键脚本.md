### Windows改键脚本

windows系统下，通过修改注册表的方式修改键位，将下方两个命令分别复制到txt文件中，然后修改文件后缀名为reg，双击执行，重启系统即可完成配置。

LeftCtrl2Caps

```
Windows Registry Editor Version 5.00

[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Keyboard Layout]
"Scancode Map"=hex:00,00,00,00,00,00,00,00,03,00,00,00,3a,00,1d,00,1d,00,3a,00,00,00,00,00
```

Rollback

Caps2LeftCtrl

```
Windows Registry Editor Version 5.00

[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Keyboard Layout]
"Scancode Map"=hex:00,00,00,00,00,00,00,00,03,00,00,00,1d,00,3a,00,3a,00,1d,00,00,00,00,00
```

[原理](http://www.cnblogs.com/xiaobaibuhei/p/3629133.html)