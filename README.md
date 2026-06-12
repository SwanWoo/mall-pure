# mall

<p>
  <img src="https://img.shields.io/badge/SpringBoot-3.5-brightgreen" alt="SpringBoot">
  <img src="https://img.shields.io/badge/JDK-17-green" alt="JDK">
  <img src="https://img.shields.io/badge/License-Apache%202.0-blue" alt="License">
  <a href="https://github.com/macrozheng/mall-swarm"><img src="https://img.shields.io/badge/Cloud版本-mall--swarm-brightgreen" alt="SpringCloud版本"></a>
  <a href="https://github.com/macrozheng/mall-admin-web"><img src="https://img.shields.io/badge/后台管理系统-mall--admin--web-green" alt="后台管理系统"></a>
  <a href="https://github.com/macrozheng/mall-app-web"><img src="https://img.shields.io/badge/前台商城-mall--app--web-green" alt="前台商城项目"></a>
</p>

## 友情提示

> 1. **微服务版本**：基于Spring Cloud Alibaba的项目：[mall-swarm](https://github.com/macrozheng/mall-swarm) 。
> 2. **分支说明**：`master`分支基于Spring Boot 3.5+JDK 17，`dev-v2`分支基于Spring Boot 2.7+JDK 8。
> 3. **最小启动**：如果只启动`mall-admin`模块，仅需安装MySQL、Redis即可。

## 前言

`mall`项目致力于打造一个完整的电商系统，采用现阶段主流技术实现。基于SpringBoot+MyBatis实现，采用Docker容器化部署，包含前台商城系统及后台管理系统。

## 项目介绍

`mall`项目是一套电商系统，包括前台商城系统及后台管理系统。前台商城系统包含首页门户、商品推荐、商品搜索、商品展示、购物车、订单流程、会员中心、客户服务、帮助中心等模块。后台管理系统包含商品管理、订单管理、会员管理、促销管理、运营管理、内容管理、统计报表、财务管理、权限管理、设置等模块。

### 项目演示

#### 后台管理系统

前端项目`mall-admin-web`地址：https://github.com/macrozheng/mall-admin-web

![后台管理系统功能演示](./document/resource/mall_admin_show.png)

#### 前台商城系统

前端项目`mall-app-web`地址：https://github.com/macrozheng/mall-app-web

![前台商城系统功能演示](./document/resource/re_mall_app_show.jpg)

### 组织结构

```
mall
├── mall-common -- 工具类及通用代码
├── mall-mbg -- MyBatisGenerator生成的数据库操作代码
├── mall-security -- SpringSecurity封装公用模块
├── mall-admin -- 后台商城管理系统接口
├── mall-search -- 基于Elasticsearch的商品搜索系统
├── mall-portal -- 前台商城系统接口
└── mall-demo -- 框架搭建时的测试代码
```

### 技术选型

#### 后端技术

| 技术                 | 说明                | 官网                                           |
| -------------------- | ------------------- | ---------------------------------------------- |
| SpringBoot           | Web应用开发框架      | https://spring.io/projects/spring-boot         |
| SpringSecurity       | 认证和授权框架      | https://spring.io/projects/spring-security     |
| MyBatis              | ORM框架             | http://www.mybatis.org/mybatis-3/zh/index.html |
| MyBatisGenerator     | 数据层代码生成器     | http://www.mybatis.org/generator/index.html    |
| Elasticsearch        | 搜索引擎            | https://github.com/elastic/elasticsearch       |
| RabbitMQ             | 消息队列            | https://www.rabbitmq.com/                      |
| Redis                | 内存数据存储         | https://redis.io/                              |
| MongoDB              | NoSql数据库         | https://www.mongodb.com                        |
| LogStash             | 日志收集工具        | https://github.com/elastic/logstash            |
| Kibana               | 日志可视化查看工具  | https://github.com/elastic/kibana              |
| Nginx                | 静态资源服务器      | https://www.nginx.com/                         |
| Docker               | 应用容器引擎        | https://www.docker.com                         |
| Druid                | 数据库连接池        | https://github.com/alibaba/druid               |
| MinIO                | 对象存储            | https://github.com/minio/minio                 |
| JWT                  | JWT登录支持         | https://github.com/jwtk/jjwt                   |
| Lombok               | Java语言增强库      | https://github.com/rzwitserloot/lombok         |
| Hutool               | Java工具类库        | https://github.com/looly/hutool                |
| PageHelper           | MyBatis物理分页插件 | http://git.oschina.net/free/Mybatis_PageHelper |
| SpringDoc            | API文档生成工具      | https://github.com/springdoc/springdoc-openapi |
| Hibernator-Validator | 验证框架            | http://hibernate.org/validator                 |

#### 前端技术

| 技术       | 说明                  | 官网                                   |
| ---------- | --------------------- | -------------------------------------- |
| Vue        | 前端框架              | https://vuejs.org/                     |
| Vue-router | 路由框架              | https://router.vuejs.org/              |
| Vuex       | 全局状态管理框架      | https://vuex.vuejs.org/                |
| Element    | 前端UI框架            | https://element.eleme.io               |
| Axios      | 前端HTTP框架          | https://github.com/axios/axios         |
| v-charts   | 基于Echarts的图表框架 | https://v-charts.js.org/               |
| Js-cookie  | cookie管理工具        | https://github.com/js-cookie/js-cookie |
| nprogress  | 进度条控件            | https://github.com/rstacruz/nprogress  |

#### 移动端技术

| 技术         | 说明             | 官网                                    |
| ------------ | ---------------- | --------------------------------------- |
| Vue          | 核心前端框架     | https://vuejs.org                       |
| Vuex         | 全局状态管理框架 | https://vuex.vuejs.org                  |
| uni-app      | 移动端前端框架   | https://uniapp.dcloud.io                |
| mix-mall     | 电商项目模板     | https://ext.dcloud.net.cn/plugin?id=200 |
| luch-request | HTTP请求框架     | https://github.com/lei-mu/luch-request  |

#### 架构图

##### 系统架构图

![系统架构图](./document/resource/re_mall_system_arch.jpg)

##### 业务架构图

![业务架构图](./document/resource/re_mall_business_arch.jpg)

#### 模块介绍

##### 后台管理系统 `mall-admin`

- 商品管理：[功能结构图-商品.jpg](document/resource/mind_product.jpg)
- 订单管理：[功能结构图-订单.jpg](document/resource/mind_order.jpg)
- 促销管理：[功能结构图-促销.jpg](document/resource/mind_sale.jpg)
- 内容管理：[功能结构图-内容.jpg](document/resource/mind_content.jpg)
- 用户管理：[功能结构图-用户.jpg](document/resource/mind_member.jpg)

##### 前台商城系统 `mall-portal`

[功能结构图-前台.jpg](document/resource/mind_portal.jpg)

#### 开发进度

![项目开发进度图](./document/resource/re_mall_dev_flow.jpg)

## 环境搭建

### 开发环境

| 工具          | 版本号 | 下载                                                         |
| ------------- | ------ | ------------------------------------------------------------ |
| JDK           | 17     | https://www.oracle.com/java/technologies/javase/jdk17-archive-downloads.html |
| MySQL         | 5.7    | https://www.mysql.com/                                       |
| Redis         | 7.0    | https://redis.io/download                                    |
| MongoDB       | 5.0    | https://www.mongodb.com/download-center                      |
| RabbitMQ      | 3.10.5 | http://www.rabbitmq.com/download.html                        |
| Nginx         | 1.22   | http://nginx.org/en/download.html                            |
| Elasticsearch | 7.17.3 | https://www.elastic.co/downloads/elasticsearch               |
| Logstash      | 7.17.3 | https://www.elastic.co/cn/downloads/logstash                 |
| Kibana        | 7.17.3 | https://www.elastic.co/cn/downloads/kibana                   |

### 搭建步骤

> Windows环境部署

- Windows环境搭建请参考：[mall在Windows环境下的部署](document/reference/deploy-windows.md);
- 克隆`mall-admin-web`项目，并导入到IDEA中完成编译：[前端项目地址](https://github.com/macrozheng/mall-admin-web)。

> Docker环境部署

- **本项目完整Docker部署手册（含13条踩坑记录）**：[Docker部署手册](document/reference/deploy-docker.md)；

## 贡献

欢迎提交 Issue 和 Pull Request 参与贡献！

1. Fork 本仓库
2. 新建分支：`git checkout -b feature/your-feature`
3. 提交改动：`git commit -m 'Add some feature'`
4. 推送分支：`git push origin feature/your-feature`
5. 提交 Pull Request

## 许可证

[Apache License 2.0](https://github.com/macrozheng/mall/blob/master/LICENSE)

Copyright (c) 2018-2026 macrozheng
