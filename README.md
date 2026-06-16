## 1、拓扑

![](https://gblog-1258458597.cos.ap-chengdu.myqcloud.com/images/image-20250208001433721.png)

## 2、技术栈

- `Nexus`
- `Sonar`
- `Harbor`
- `Redis`
- `MapStruct`
- `MybatisPlus`
- `Mysql 8.0.33`
- `Docker 27.4.0`
- `Spring Boot 3.2.1`
- `Kubernetes v1.31.4`
- `Spring Cloud 2023.0.1`
- `Nacos、Sentinel、Seata`
- `Jdk 21.0.5 2024-10-15 LTS`
- `Spring Cloud Alibaba 2023.0.1.0`
- `GitLab Community Edition v16.9.9`

## 3、服务说明

|            服务            | 端口  |                             说明                             |
| :------------------------: | :---: | :----------------------------------------------------------: |
|       `hiseas-cloud`       |   -   |    maven父项目，统一管理版本依赖，私服地址，仓库地址等。     |
|      `hiseas-common`       |   -   | 通用模块，定义所有项目公共内容。包括：枚举、常量、工具类等。 |
|     `hiseas-sa-token`      |   -   | [Sa-Token](https://sa-token.cc/)相关配置类，以SpringBoot自动装配方式加载。 |
|    `hiseas-center-iam`     | 15000 |                          微服务网关                          |
|  `hiseas-center-gateway`   | 8431  |  基于[Sa-Token](https://sa-token.cc/)实现的认证、鉴权中心。  |
| `hiseas-mgmgt-application` | 10000 |    非业务聚合服务，负责调用其他原子微服务，进行业务组装。    |
|   `hiseas-center-course`   | 9000  |             课程原子微服务，针对课程模块的CRUD。             |
|    `hiseas-center-user`    | 8000  |             用户原子微服务，针对用户模块的CRUD。             |

## 4、部分截图

![](https://gblog-1258458597.cos.ap-chengdu.myqcloud.com/images/image-20250207232147376.png)

![](https://gblog-1258458597.cos.ap-chengdu.myqcloud.com/images/image-20250207233044187.png)

![](https://gblog-1258458597.cos.ap-chengdu.myqcloud.com/images/image-20250207234320196.png)

![](https://gblog-1258458597.cos.ap-chengdu.myqcloud.com/images/image-20250208142544639.png)

![](https://gblog-1258458597.cos.ap-chengdu.myqcloud.com/images/image-20250208160327106.png)

## 5、使用

[参考文章](https://zhengxiang.cc/blog/?tag=Kubernetes)
