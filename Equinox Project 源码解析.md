# Equinox Project 源码解析V1.4版本

因原项目没有 releases 版本，所以 fork 了一份 V1.4 的版本代码

<https://github.com/bbigcd/EquinoxProject>

### 项目目录结构

| 工程名称                            | 描述                                                         |
| :---------------------------------- | :----------------------------------------------------------- |
| Equinox.Application                 | Equinox的应用层，使用了 [AutoMapper](<https://github.com/AutoMapper/AutoMapper>) 模型转换工具 |
| Equinox.Domain                      | Equinox的领域层，使用CQRS架构                                |
| Equinox.Domain.Core                 | Equinox的领域核心层，领域层的基建                            |
| Equinox.Infra.CrossCutting.Bus      | Equinox的事件总线层                                          |
| Equinox.Infra.CrossCutting.Identity |                                                              |
| Equinox.Infra.CrossCutting.IoC      | Equinox的IoC控制反转容器                                     |
| Equinox.Infra.Data                  | Equinox的数据仓储层                                          |
| Equinox.Services.Api                | Equinox的Api服务层                                           |
| Equinox.UI.Web                      | Equinox的UI层                                                |

客户管理

账号管理



