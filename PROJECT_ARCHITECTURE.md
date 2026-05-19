# fuint 餐饮会员营销系统 — 系统架构与全栈设计说明书

> **版本**：1.0.0  
> **最后更新**：2026-05-19  
> **官网**：https://www.fuint.cn  
> **许可证**：开源（学习/论文用途免费，商用需授权）

---

## 目录

1. [项目概述](#1-项目概述)
2. [系统全景架构](#2-系统全景架构)
3. [多端协同体系](#3-多端协同体系)
4. [技术栈全矩阵](#4-技术栈全矩阵)
5. [后端多模块依赖与分层架构](#5-后端多模块依赖与分层架构)
6. [核心业务模块类职责详解](#6-核心业务模块类职责详解)
7. [Ehcache + Redis 双重缓存体系](#7-ehcache--redis-双重缓存体系)
8. [数据库设计](#8-数据库设计)
9. [Uniapp 前端架构与桌码参数解析](#9-uniapp-前端架构与桌码参数解析)
10. [Cashier 收银台实时推单机制](#10-cashier-收银台实时推单机制)
11. [API 接口设计规范](#11-api-接口设计规范)
12. [安全架构](#12-安全架构)
13. [定时任务与分布式锁](#13-定时任务与分布式锁)
14. [部署架构](#14-部署架构)
15. [附录](#15-附录)

---

## 1. 项目概述

### 1.1 系统定位

fuint 是一套面向**实体门店**的**全场景会员营销与收银管理系统**，覆盖餐饮、零售、酒吧、酒店、汽车服务、花店、甜品店、农家乐等多种业态。系统以「会员生命周期管理」为核心，提供电子优惠券、预存卡、集次卡、会员积分、会员等级、分佣返利、多店铺结算、桌码点单、收银核销等全链路功能。

### 1.2 核心业务能力矩阵

| 业务域 | 核心能力 |
|--------|----------|
| **会员管理** | 会员注册/登录（手机号、微信OAuth）、会员等级体系、会员分组、会员标签、标签规则引擎、会员行为跟踪 |
| **卡券系统** | 优惠券（满减券/折扣券）、预存卡（余额储值）、集次卡（按次消费）；券组打包、领取码发放、批量发放、转赠、过期回收 |
| **积分体系** | 积分获取（消费/签到/注册赠送）、积分消耗（兑换卡券/商品）、积分变动明细 |
| **余额体系** | 会员余额充值与消费、余额变动明细、后台手动调整 |
| **分佣分销** | 会员二级分销关系树、员工销售提成、按比例/固定金额两种提成方式、佣金提现结算 |
| **商品管理** | 多规格SKU、商品分类、店铺商品关联、库存出入库、库存明细 |
| **订单系统** | 堂食点单（桌码）、外卖配送、到店自提；订单支付、退款售后、发票管理 |
| **桌码系统** | 桌台区域管理、桌台二维码生成、桌台状态（空闲/占用）、转台、挂单、清台 |
| **收银台** | 后台收银台初始化和商品搜索、会员查询与识别、购物车挂单/取单、桌台管理、商品取消 |
| **核销系统** | 优惠券二维码扫码核销、订单核销码确认、核销记录追踪 |
| **内容管理** | Banner轮播、文章资讯、导航菜单、自定义页面（可视化装修） |
| **消息通知** | 微信订阅消息模板推送、阿里云短信发送、站内消息 |
| **报表统计** | 日报表（收银统计、分类销售统计、商品销售统计）、会员活跃统计 |
| **多租户** | 平台—商户—店铺三级架构，数据按商户/店铺完全隔离 |

---

## 2. 系统全景架构

### 2.1 总体架构图

```
┌──────────────────────────────────────────────────────────────────────────┐
│                           前端多端矩阵 (Frontend)                         │
├──────────────┬─────────────────────────┬────────────────────┬─────────────┤
│  fuintAdmin  │     fuintUniapp         │   fuintCashier     │  第三方接入   │
│  (Web管理端) │  (会员端/商户端H5+小程序) │  (Electron桌面收银) │  (微信公众号) │
│  Vue2 +      │  Uni-app + uView UI     │  Electron +        │  WeChat OAuth │
│  Element UI  │  + Vuex + mescroll      │  electron-builder  │  + JSSDK     │
│  Port: 8081  │  + WeChat Mini Program  │  Windows Installer │              │
└──────┬───────┴────────────┬────────────┴─────────┬──────────┴──────┬───────┘
       │                    │                      │                 │
       │         HTTPS / RESTful API (JSON)        │                 │
       │   Header: Access-Token, merchantNo,       │                 │
       │   storeId, tableId, platform, isWechat    │                 │
       └────────────────────┬─────────────────────┘                 │
                            │                                        │
┌───────────────────────────┴──────────────────────────────────────────────┐
│                         Nginx 反向代理层 (Reverse Proxy)                   │
│  ┌──────────────┐                                                         │
│  │ URL Rewrite  │  /clientApi/*  → 会员端API                               │
│  │ Static File  │  /backendApi/* → 管理端API                               │
│  │ Serve        │  /merchantApi/*→ 商户端API                               │
│  └──────────────┘                                                         │
└───────────────────────────┬──────────────────────────────────────────────┘
                            │
┌───────────────────────────┴──────────────────────────────────────────────┐
│                    Java Spring Boot 2.5 后端服务层 (Backend)               │
│                                                                           │
│  ┌───────────────────────────────────────────────────────────────┐       │
│  │                  Interceptor Chain（拦截器链）                   │       │
│  │  CORSFilter → AdminUserInterceptor / ClientUserInterceptor    │       │
│  └───────────────────────────────────────────────────────────────┘       │
│                                                                           │
│  ┌─────────────────────┐  ┌─────────────────────┐                       │
│  │   Backend API Layer  │  │   Client API Layer   │                       │
│  │   @PreAuthorize 鉴权  │  │   Token 鉴权 (Redis)  │                       │
│  │   30+ Controllers    │  │   25+ Controllers    │                       │
│  └──────────┬──────────┘  └──────────┬──────────┘                       │
│             │                        │                                    │
│             └───────────┬────────────┘                                    │
│                         │                                                 │
│  ┌──────────────────────┴──────────────────────────────────────┐        │
│  │                  Service Layer（47 个 Service）               │        │
│  │  CouponService / OrderService / MemberService /              │        │
│  │  GoodsService / CartService / TableService /                 │        │
│  │  PaymentService / SettlementService / CommissionService ...  │        │
│  └──────────────────────┬──────────────────────────────────────┘        │
│                         │                                                 │
│  ┌──────────────────────┴──────────────────────────────────────┐        │
│  │            Repository Layer (MyBatis-Plus)                   │        │
│  │  Mapper Interfaces (72) + XML SQL (65)                      │        │
│  │  45+ Domain Models (Mt*/T*)                                  │        │
│  └──────────────────────┬──────────────────────────────────────┘        │
│                         │                                                 │
│  ┌──────────────────────┴──────────────────────────────────────┐        │
│  │                  Caching Layer（双重缓存）                     │        │
│  │  ┌──────────────────┐    ┌──────────────────────┐          │        │
│  │  │  Ehcache 2.10.2  │    │   Redis              │          │        │
│  │  │  本地JVM堆内缓存   │    │   分布式缓存中间件      │          │        │
│  │  │  一级缓存 L1       │    │   二级缓存 L2          │          │        │
│  │  │  TTL: 120s       │    │   共享Session/Tokens  │          │        │
│  │  │  200k entries    │    │   分布式锁 (Lua)       │          │        │
│  │  └──────────────────┘    └──────────────────────┘          │        │
│  └─────────────────────────────────────────────────────────────┘        │
└───────────────────────────┬──────────────────────────────────────────────┘
                            │
              ┌─────────────┼─────────────┐
              │             │             │
     ┌────────┴────┐  ┌────┴───────┐  ┌─┴──────────┐
     │   MySQL 8   │  │   Redis    │  │ OSS / 云   │
     │  fuint-food │  │ Session/   │  │ 阿里云OSS  │
     │  40+ Tables │  │ Cache/Lock │  │ 阿里云短信  │
     └─────────────┘  └────────────┘  └────────────┘
```

### 2.2 项目目录结构总览

```
fuintCatering/
├── README.md                            # 项目说明
├── PROJECT_ARCHITECTURE.md              # 本文件 — 系统架构说明书
├── db/
│   └── fuint-food.sql                   # 完整数据库DDL（40+张表）
├── docs/
│   └── fuint餐饮系统功能列表.xlsx         # 功能清单文档
├── fuintBackend/                        # Java Spring Boot 后端（Maven多模块）
│   ├── pom.xml                          # 父POM — 版本依赖统一管理
│   ├── sbin/                            # 启停脚本
│   │   ├── start.sh / stop.sh / restart.sh / kill.sh
│   │   └── yanhe.fuint                  # 守护进程配置
│   ├── fuint-utils/                     # 通用工具模块
│   ├── fuint-repository/                # 数据访问层模块
│   ├── fuint-framework/                 # 框架共享模块
│   └── fuint-application/               # 主应用模块（Spring Boot启动入口）
├── fuintAdmin/                          # 管理端前端（Vue2 + Element UI）
│   ├── README.md
│   └── check-node-version.js            # Node版本检查脚本
├── fuintCashier/                        # 桌面收银端（Electron）
│   ├── README.md
│   └── build/                           # electron-builder构建产物
│       ├── builder-debug.yml
│       ├── latest.yml                   # 自动更新元数据
│       └── fuint收银系统 Setup 1.0.0.exe.blockmap
└── fuintUniapp/                         # Uni-app跨端前端（H5 + 微信小程序 + App）
    ├── package.json
    ├── pages.json                       # 页面路由配置
    ├── manifest.json                    # 跨端编译配置
    ├── config.js                        # 环境配置（API地址、商户号）
    ├── main.js                          # Vue入口（注册uView、Vuex、全局方法）
    ├── App.vue                          # 根组件
    ├── api/                             # API请求模块（30个模块）
    ├── common/                          # 公共常量、枚举、模型
    ├── components/                      # 通用UI组件（70+）
    ├── core/                            # 启动引导、平台适配
    ├── pages/                           # 主包页面（会员端核心页面）
    ├── subPages/                        # 分包页面（预约、集次卡等）
    ├── merchantPages/                   # 商户端分包（店员功能）
    ├── store/                           # Vuex状态管理
    ├── uni_modules/                     # uni-app官方插件
    ├── utils/                           # 前端工具（请求库、存储、验证等）
    ├── static/                          # 静态资源
    └── uview-ui/                        # uView UI组件库（含70+组件）
```

---

## 3. 多端协同体系

### 3.1 三端角色与用户画像

```
┌─────────────────────────────────────────────────────────────────┐
│                        端侧角色矩阵                              │
├────────────┬──────────────┬──────────────┬──────────────────────┤
│            │  fuintAdmin  │ fuintUniapp  │    fuintCashier      │
│            │  (Web管理端)  │ (H5/小程序)   │   (Electron桌面端)    │
├────────────┼──────────────┼──────────────┼──────────────────────┤
│ 核心用户    │ 平台管理员    │ C端消费者      │ 收银员/店员           │
│            │ 商户管理员    │ 商户店员       │                      │
├────────────┼──────────────┼──────────────┼──────────────────────┤
│ 运行环境    │ 桌面浏览器    │ 微信H5/小程序   │ Windows桌面          │
│            │ Chrome/Edge  │ 原生App       │ Electron Shell       │
├────────────┼──────────────┼──────────────┼──────────────────────┤
│ API前缀     │ /backendApi  │ /clientApi   │ /backendApi/cashier  │
│            │              │ /merchantApi │                      │
├────────────┼──────────────┼──────────────┼──────────────────────┤
│ 核心功能    │ 商品/库存管理 │ 扫码点单      │ 收银开台              │
│            │ 会员/等级管理 │ 领取优惠券     │ 商品搜索加入购物车     │
│            │ 卡券发放/核销 │ 下单/支付      │ 挂单/取单             │
│            │ 报表统计      │ 会员中心       │ 会员识别              │
│            │ 系统配置      │ 积分/余额      │ 优惠券/订单核销        │
│            │ 权限管理      │ 预约服务       │ 发票管理              │
│            │              │ 商户管理       │                      │
└────────────┴──────────────┴──────────────┴──────────────────────┘
```

### 3.2 跨端通信流程

#### 3.2.1 会员自助点单流程（Uniapp 扫码 → 后端 → 收银端）

```
  顾客扫码桌台二维码
       │
       ▼
  Uniapp 解析 URL 参数
  ?tableId=xxx&storeId=xxx&merchantNo=xxx
       │
       ├── tableId 存入 localStorage
       ├── storeId 存入 localStorage
       └── 跳转 pages/category/index (点单页)
       │
       ▼
  请求 Header 自动携带:
    tableId, storeId, merchantNo, Access-Token
       │
       ▼
  后端 ClientCartController / ClientOrderController
    ├── 根据 tableId 关联桌台
    ├── 根据 storeId 加载对应店铺商品
    └── 下单时更新 mt_order.TABLE_ID
       │
       ▼
  支付完成后
    ├── 后端更新 mt_table 状态 → OCCUPIED (占用中)
    └── [Cashier端] 轮询 /backendApi/cashier/getTableList
        实时感知桌台状态变更
```

#### 3.2.2 收银员核销流程（Cashier 扫码 → 后端验证 → 核销记录）

```
  收银员在 Cashier 端扫描顾客会员码/券码
       │
       ▼
  请求 /backendApi/cashier/getMemberInfo
    或 /merchantApi/order/confirm (核销)
       │
       ▼
  后端验证:
    ├── 优惠券状态检查（未过期、未核销、适用店铺）
    ├── 会员身份验证（商户归属）
    ├── 预存卡余额扣减
    └── 集次卡次数校验
       │
       ▼
  写入核销记录:
    mt_confirm_log (核销记录表)
    mt_user_coupon (更新卡券状态)
    mt_balance (余额变动)
       │
       ▼
  返回核销结果 → Cashier 端展示成功/失败
```

---

## 4. 技术栈全矩阵

### 4.1 后端技术栈

| 层级 | 技术选型 | 版本 | 用途说明 |
|------|---------|------|---------|
| **运行环境** | JDK | 1.8 | Java运行环境 |
| **核心框架** | Spring Boot | 2.5.12 | 应用框架、自动配置、起步依赖 |
| **Web容器** | Jetty (替代Tomcat) | — | 嵌入式Servlet容器 |
| **ORM** | MyBatis-Plus | 3.1.0 | 增强型MyBatis，含分页插件、逻辑删除、乐观锁 |
| **分页** | PageHelper | 1.2.5 | MyBatis物理分页插件 |
| **连接池** | HikariCP | — | 高性能数据库连接池（Spring Boot默认） |
| **数据库** | MySQL | 8.0.25 | 关系型数据库 |
| **一级缓存** | Ehcache | 2.10.2 | 本地JVM堆内缓存，LRU淘汰策略 |
| **二级缓存** | Redis | — | 分布式缓存中间件（spring-boot-starter-data-redis） |
| **Session** | Spring Session + Redis | — | 分布式HttpSession共享 |
| **序列化** | Kryo | 4.0.2 | 高性能对象序列化（替代Java原生序列化） |
| **安全框架** | Spring Security | 5.x | 权限注解@PreAuthorize + BCryptPasswordEncoder |
| **认证方式** | 自定义Interceptor + Redis Token | — | Token存Redis，拦截器校验，替代Security Filter |
| **接口文档** | Swagger / SpringFox | 2.9.2 | 在线API文档 http://localhost:8080/swagger-ui.html |
| **AOP** | Spring AOP + AspectJ + Javassist | — | 日志切面、操作审计切面、参数名解析 |
| **微信SDK** | weixin-popular | 2.8.0 | 微信公众号/小程序集成（OAuth、消息、菜单） |
| **微信支付** | IJPay-WxPay | 2.9.6 | 微信支付V2/V3 |
| **支付宝** | IJPay-AliPay | 2.9.6 | 支付宝支付 |
| **云存储** | Aliyun OSS SDK | 3.10.2 | 阿里云对象存储（图片/文件上传） |
| **短信** | Aliyun SMS SDK | 4.4.6 | 阿里云短信服务（验证码、营销短信） |
| **HTTP客户端** | OkHttp | 3.8.1 | HTTP请求工具 |
| **二维码** | ZXing | 3.3.0 | 二维码生成与解析 |
| **验证码** | Google Kaptcha | 0.0.9 | 图片验证码生成 |
| **Excel** | Apache POI | 3.14 | Excel导入导出 |
| **加密** | BouncyCastle | 1.58 | RSA/AES等加密算法 |
| **URL重写** | Tuckey UrlRewriteFilter | 4.0.3 | URL重写规则 |
| **日志** | SLF4J + Logback | — | 日志框架，含Sentry异常监控集成 |
| **代码生成** | Apache Velocity | 2.3 | 代码模板引擎 |
| **工具类** | Apache Commons Lang3 | 3.12.0 | 通用工具 |
| **简化代码** | Lombok | — | @Data/@AllArgsConstructor等注解 |
| **序列化** | Jackson | — | JSON序列化（Spring Boot默认） |
| **打包** | spring-boot-maven-plugin | — | 可执行Fat JAR |

### 4.2 Uniapp 前端技术栈

| 层级 | 技术选型 | 版本 | 用途说明 |
|------|---------|------|---------|
| **框架** | Uni-app (Vue 2) | 2.x | 跨端开发框架（H5/微信小程序/App） |
| **UI库** | uView UI | 1.x | 70+组件的移动端UI库（含表单、弹窗、日历等） |
| **状态管理** | Vuex | 3.x | 全局状态管理（token、userInfo、platform） |
| **HTTP** | uni.request + 自定义拦截器 | — | 统一请求封装（自动附加商户号、店铺ID、桌码ID等Header） |
| **滚动** | mescroll-uni | — | 下拉刷新+上拉加载更多 |
| **富文本** | jyf-parser | — | 富文本HTML解析渲染组件 |
| **数字键盘** | neoceansoft-keyboard | — | 自定义数字键盘组件 |
| **微信小程序** | WeChat Mini Program | — | 微信小程序原生API（扫码、支付、定位、授权） |
| **支付** | 微信JSAPI / H5支付 | — | 微信支付集成 |
| **存储** | uni.storageSync | — | 本地键值存储（持久化token/tableId等） |
| **CSS** | SCSS | — | CSS预处理器 |
| **平台检测** | 自定义 bootstrap.js | — | 自动检测H5/微信/支付宝/百度/头条/App平台 |
| **构建** | HBuilderX | — | DCloud官方IDE编译发布 |

### 4.3 Admin 管理端技术栈

| 层级 | 技术选型 | 说明 |
|------|---------|------|
| **框架** | Vue 2.x | 渐进式JavaScript框架 |
| **UI库** | Element UI | 饿了么桌面端组件库 |
| **路由** | Vue Router | SPA路由管理 |
| **状态管理** | Vuex | 全局状态管理 |
| **构建工具** | Vue CLI / Vite | 工程化构建 |
| **Node版本** | >= 16 (推荐18.x/20.x LTS) | 运行时环境 |
| **端口** | 8081 | 开发服务器端口 |

### 4.4 Cashier 桌面收银技术栈

| 层级 | 技术选型 | 说明 |
|------|---------|------|
| **框架** | Electron | 跨平台桌面应用框架 |
| **打包** | electron-builder + NSIS | Windows安装包构建 |
| **自动更新** | electron-updater | 基于latest.yml的增量更新 |
| **安装包大小** | ~76MB | Windows .exe安装程序 |
| **界面** | HTML/CSS/JS (Chromium内核) | 与Web技术同构的渲染进程 |

---

## 5. 后端多模块依赖与分层架构

### 5.1 Maven 模块拓扑图

```
fuintBackend (父POM, pom)
├── fuint-utils (jar)
│   └── 无内部依赖
│
├── fuint-repository (jar)
│   └── 无内部依赖
│
├── fuint-framework (jar)
│   ├── 依赖 fuint-utils
│   └── 依赖 fuint-repository
│
└── fuint-application (jar, Spring Boot)
    └── 依赖 fuint-framework → (传递依赖 fuint-utils + fuint-repository)
```

### 5.2 各模块详细职责

#### 5.2.1 fuint-utils（通用工具模块）

**定位**：纯工具库，无框架依赖，可被任何Java项目复用。

| 包路径 | 包含类 | 职责 |
|--------|--------|------|
| `com.fuint.utils` | AES.java, AESUtil.java | AES对称加密/解密 |
| | ArrayUtil.java | 数组操作工具 |
| | Base64Util.java | Base64编解码 |
| | BeanToMapUtil.java | Java Bean → Map 转换 |
| | ClassUtil.java | 反射工具（类信息、方法调用） |
| | CommonUtil.java | 通用杂项工具 |
| | ContextUtils.java | Spring上下文持有者（供静态方法获取Bean） |
| | Digests.java | 摘要算法（SHA-1/MD5） |
| | Encodes.java | 编码工具（Hex/Base64） |
| | HttpUtil.java | HTTP请求工具 |
| | IDCard.java | 中国身份证号码校验 |
| | IpUtil.java | IP地址解析 |
| | MD5Util.java | MD5哈希 |
| | ObjectUtil.java | 对象操作工具 |
| | PropertiesUtil.java | 属性文件加载（国际化错误消息） |
| | QRCodeUtil.java | ZXing二维码生成 |
| | RSAKeys.java | RSA密钥对生成 |
| | SeqUtil.java | UUID序列号生成 |
| | StringUtil.java | 字符串处理增强 |
| | TimeUtils.java | 日期时间工具 |
| | ValidationUtil.java | 数据验证工具 |
| `com.fuint.text` | CharsetKit.java | 字符集处理 |
| | Convert.java | 类型转换 |
| | StrFormatter.java | 字符串格式化 |
| `com.fuint.exception` | Exceptions.java | 异常工具 |

**外部依赖**：ZXing 3.3.0、BouncyCastle 1.58、Apache POI 3.14、commons-lang3 3.12.0

---

#### 5.2.2 fuint-repository（数据访问层模块）

**定位**：纯数据层，定义所有实体模型、Mapper接口和MyBatis XML映射文件。

```
fuint-repository/
├── src/main/java/com/fuint/repository/
│   ├── base/
│   │   └── MyMapper.java              → 继承 BaseMapper<T>（MyBatis-Plus通用CRUD接口）
│   ├── model/                           → 45+个实体类（@TableName + @Data + @TableId）
│   │   ├── base/
│   │   │   ├── AutoIncrementIdModel.java
│   │   │   ├── ElasticSearchModel.java
│   │   │   └── RedisCache.java         → 自定义@RedisCache注解（预留，暂未使用）
│   │   ├── Mt*.java                     → 42个业务实体（前缀Mt = Model Table）
│   │   └── T*.java                      → 8个系统实体（前缀T = Table）
│   ├── mapper/                          → 72个Mapper接口
│   │   ├── Mt*Mapper.java               → 每个实体对应一个Mapper
│   │   └── T*Mapper.java                → 系统表Mapper
│   └── bean/                            → 9个查询结果Bean（非实体，用于复杂查询结果的承载）
│       ├── ColumnBean.java
│       ├── CouponNumBean.java
│       ├── GoodsBean.java
│       ├── GoodsTopBean.java
│       ├── MemberTopBean.java
│       └── StoreDistanceBean.java
└── src/main/resources/
    └── mapper/                          → 65个MyBatis XML映射文件
        ├── MtAddressMapper.xml
        ├── MtCartMapper.xml
        ├── MtOrderMapper.xml
        ├── ... (每个Mapper对应一个XML)
        └── TAccountMapper.xml
```

**关键实体核心字段一览**：

| 实体类 | 对应表 | 核心字段 | 业务含义 |
|--------|--------|----------|---------|
| `MtUser` | mt_user | MOBILE, OPEN_ID, GRADE_ID, BALANCE, POINT | 会员主表 |
| `MtOrder` | mt_order | TABLE_ID, USER_ID, AMOUNT, PAY_STATUS, TYPE | 订单主表 |
| `MtCart` | mt_cart | TABLE_ID, USER_ID, SKU_ID, GOODS_ID, HANG_NO | 购物车（含桌码关联） |
| `MtCoupon` | mt_coupon | TYPE(C/P/T), NAME, AMOUNT, EXPIRE_TYPE, TOTAL | 卡券定义 |
| `MtUserCoupon` | mt_user_coupon | CODE, TYPE, COUPON_ID, USER_ID, AMOUNT, BALANCE | 用户持有的卡券 |
| `MtStore` | mt_store | MERCHANT_ID, QR_CODE, LATITUDE, LONGITUDE | 店铺（含定位） |
| `MtTable` | mt_table | CODE, AREA_ID, MAX_PEOPLE, STORE_ID | 桌台（含二维码编码） |
| `MtGoods` | mt_goods | TYPE, CATE_ID, PRICE, STOCK | 商品/SKU |
| `MtSetting` | mt_setting | MERCHANT_ID, TYPE, NAME, VALUE(longtext) | 系统设置（键值对） |
| `MtMerchant` | mt_merchant | NO, NAME, WX_APP_ID, WX_MCH_ID | 商户信息 |
| `TAccount` | t_account | ACCOUNT_KEY, PASSWORD, ROLE_IDS, MERCHANT_ID, STORE_ID | 后台账户 |

---

#### 5.2.3 fuint-framework（框架共享模块）

**定位**：提供Controller基类、分页模型、全局异常处理、操作日志注解等跨模块的框架级抽象。

| 类 | 路径 | 职责 |
|----|------|------|
| `BaseController` | framework.web | 提供 `getSuccessResult(data)` / `getFailureResult(code)` / `getFailureResult(code, msg)` 等统一响应构建方法 |
| `ResponseObject` | framework.web | 统一响应体 `{ code: Integer, message: String, data: Object }` |
| `GlobalExceptionHandler` | framework.exception | `@RestControllerAdvice` 全局异常拦截，统一包装为 ResponseObject |
| `BusinessCheckException` | framework.exception | 业务校验异常（Checked），含 errorKey/errorParam |
| `BusinessRuntimeException` | framework.exception | 业务运行时异常（Unchecked） |
| `PaginationRequest` | framework.pagination | 分页请求 DTO：currentPage、pageSize、sortColumn[]、sortType、searchParams |
| `PaginationResponse` | framework.pagination | 分页响应 DTO：包装 Spring Data `Page<T>` |
| `@OperationServiceLog` | framework.annoation | 自定义注解，标记需要记录操作日志的方法 |
| `ExportService` / `ExportServiceImpl` | framework.service | 文件导出抽象服务 |

---

#### 5.2.4 fuint-application（主应用模块）

**定位**：Spring Boot 启动入口 + 全部业务逻辑 + API控制器 + 定时任务 + 配置类。

这是系统中最庞大的模块，按功能组织为以下包结构：

```
fuint-application/src/main/java/com/fuint/
├── fuintApplication.java                → @SpringBootApplication + @EnableScheduling
├── common/
│   ├── Constants.java                   → 全局常量（SESSION_TOKEN前缀、分页默认值等）
│   ├── config/
│   │   ├── RedisConfig.java             → Redis配置（RedisTemplate、KeyGenerator、Spring Session）
│   │   ├── SecurityConfig.java          → Spring Security配置（CSRF关闭，API全放行）
│   │   ├── WebConfig.java               → 拦截器注册、CORS、静态资源
│   │   ├── MybatisPlusConfig.java       → MyBatis-Plus插件（分页、乐观锁、逻辑删除）
│   │   ├── SwaggerConfig.java           → Swagger2 API文档
│   │   ├── CaptchaConfig.java           → Kaptcha图片验证码配置
│   │   └── Message.java                 → 错误码常量定义
│   ├── aspect/
│   │   ├── LogAop.java                  → 全局Controller日志切面（Before advice）
│   │   ├── TActionLogAop.java           → 操作审计日志切面（@OperationServiceLog）
│   │   └── RedisModelAspect.java        → Redis缓存切面（预留空实现）
│   ├── service/                          → 47个业务服务接口
│   │   └── impl/                         → 47个服务实现类
│   ├── dto/                              → 数据传输对象（按业务域分包：book/cashier/commission/...）
│   ├── enums/                            → 40+个枚举类
│   ├── util/                             → 应用级工具类（35个）
│   │   ├── RedisUtil.java               → Redis缓存静态工具
│   │   ├── RedisLock.java               → Redis分布式锁（SET NX + Lua脚本原子性解锁）
│   │   ├── TokenUtil.java               → 用户/管理员Token管理（生成/验证/销毁）
│   │   ├── QRCodeUtil.java              → 二维码生成
│   │   ├── PrinterUtil.java             → 云打印机连接
│   │   └── ...
│   ├── web/
│   │   ├── CORSFilter.java              → 跨域过滤器（允许所有Origin + 自定义Header）
│   │   ├── AdminUserInterceptor.java    → 管理端Token拦截器（/backendApi/**）
│   │   ├── ClientUserInterceptor.java   → 会员端Token拦截器（/clientApi/**）
│   │   ├── CommandInterceptor.java      → 命令拦截器（限制localhost访问）
│   │   ├── SpringContextHolder.java     → Spring上下文持有者
│   │   └── SystemInit.java              → 系统初始化
│   └── param/                            → 请求参数对象
└── module/
    ├── backendApi/                       → 管理端API（/backendApi/**）
    │   ├── controller/                   → 按业务域分包（book/cashier/commission/content/coupon/goods/member/merchant/message/order/report/system）
    │   │   ├── cashier/
    │   │   │   └── BackendCashierController.java    → 收银管理（商品搜索/会员查询/挂单/桌台）
    │   │   ├── order/
    │   │   │   ├── BackendOrderController.java      → 订单管理
    │   │   │   ├── BackendRefundController.java     → 售后/退款
    │   │   │   ├── BackendSettlementController.java → 结算管理
    │   │   │   └── BackendInvoiceController.java    → 发票管理
    │   │   ├── member/
    │   │   │   ├── BackendMemberController.java     → 会员管理
    │   │   │   ├── BackendBalanceController.java    → 余额管理
    │   │   │   ├── BackendPointController.java      → 积分管理
    │   │   │   ├── BackendUserGradeController.java  → 等级管理
    │   │   │   ├── BackendUserTagController.java    → 标签管理
    │   │   │   └── BackendMemberGroupController.java→ 分组管理
    │   │   ├── coupon/                               → 7个Controller（卡券CRUD/核销/发放/转赠）
    │   │   ├── goods/                                → 3个Controller（商品/分类/库存）
    │   │   ├── merchant/                             → 6个Controller（商户/店铺/员工/桌台/区域/打印机）
    │   │   ├── system/                               → 8个Controller（登录/账户/角色/权限/验证码/日志/代码生成）
    │   │   ├── content/                              → 3个Controller（文章/Banner/导航）
    │   │   └── report/                               → 1个Controller（报表统计）
    │   └── response/
    │       └── LoginResponse.java
    ├── clientApi/                        → 会员端API（/clientApi/**）
    │   └── controller/                   → 25个Controller（扁平结构）
    │       ├── ClientCartController.java     → 购物车（含桌码关联）
    │       ├── ClientOrderController.java    → 订单
    │       ├── ClientPayController.java      → 支付
    │       ├── ClientCashierController.java  → 收银端会员查询
    │       ├── ClientConfirmController.java  → 核销
    │       ├── ClientStoreController.java    → 店铺（含距离排序）
    │       ├── ClientGoodsController.java    → 商品
    │       ├── ClientCouponController.java   → 卡券领取
    │       ├── ClientUserController.java     → 会员中心
    │       ├── ClientSignController.java     → 登录注册
    │       ├── ClientBookController.java     → 预约
    │       └── ...
    ├── merchantApi/                      → 商户端API（/merchantApi/**）
    │   └── controller/                   → 11个Controller
    │       ├── MerchantController.java       → 商户仪表盘
    │       ├── MerchantOrderController.java  → 订单核销
    │       ├── MerchantMemberController.java → 会员管理
    │       ├── MerchantCouponController.java → 卡券管理
    │       └── ...
    └── schedule/                         → 定时任务
        ├── OrderCancelJob.java           → 订单超时取消（每2分钟）
        ├── OrderAutoJob.java             → 订单自动确认/完成（每5分钟）
        ├── CouponExpireJob.java          → 卡券过期回收（每1分钟）
        ├── CommissionJob.java            → 佣金结算（每5分钟）
        ├── MessageJob.java               → 微信消息推送（每1分钟）
        └── UploadShippingInfoJob.java    → 微信物流上传（每5分钟）
```

---

## 6. 核心业务模块类职责详解

### 6.1 服务层类图（核心业务域）

```
                          ┌──────────────────┐
                          │   IService<T>     │
                          │ (MyBatis-Plus)    │
                          └────────┬─────────┘
                                   │ extends
          ┌────────────────────────┼────────────────────────┐
          │                        │                         │
┌─────────┴─────────┐   ┌─────────┴─────────┐   ┌──────────┴──────────┐
│   OrderService    │   │   CouponService   │   │   MemberService     │
├───────────────────┤   ├───────────────────┤   ├─────────────────────┤
│+ createOrder()    │   │+ sendCoupon()     │   │+ queryMemberById()  │
│+ cancelOrder()    │   │+ receiveCoupon()  │   │+ queryMemberByMobile│
│+ setOrderPayed()  │   │+ expireCoupon()   │   │+ searchMembers()    │
│+ removeGoods()    │   │+ giveCoupon()     │   │+ addMember()        │
│+ getOrderDetail() │   │+ preStorePay()    │   │+ updateMember()     │
│+ removeTakenTable │   │+ confirmCoupon()  │   │+ getMemberAssets()  │
└───────────────────┘   └───────────────────┘   └─────────────────────┘

┌───────────────────┐   ┌───────────────────┐   ┌─────────────────────┐
│   CartService     │   │   TableService    │   │   GoodsService       │
├───────────────────┤   ├───────────────────┤   ├─────────────────────┤
│+ saveCart()       │   │+ getTableList()   │   │+ getGoodsDetail()   │
│+ getCartList()    │   │+ getTableDetail() │   │+ getStoreGoodsList()│
│+ setTableId()     │   │+ getHangUpList()  │   │+ getGoodsCateList() │
│+ removeCartById() │   │+ turnTable()      │   │+ getSkuDetail()     │
│+ clearCart()      │   │+ updateUseStatus()│   │+ getSpecDetail()    │
│+ removeCartBy     │   │+ getTableByCode() │   │+ initGoodsSaleCount()│
│  TableId()        │   └───────────────────┘   └─────────────────────┘
└───────────────────┘

┌───────────────────┐   ┌───────────────────┐   ┌─────────────────────┐
│  PaymentService   │   │ SettlementService │   │  CommissionService  │
├───────────────────┤   ├───────────────────┤   ├─────────────────────┤
│+ prePay()         │   │+ submitSettlement │   │+ calculateCommission│
│+ doPay()          │   │+ getSettlement    │   │+ addCommissionLog() │
│+ wxPay()          │   │  List()           │   │+ getCommissionCash()│
│+ aliPay()         │   │+ confirmSettlement│   │+ commissionCash()   │
│+ balancePay()     │   └───────────────────┘   └─────────────────────┘
│+ createPayOrder() │
└───────────────────┘
```

### 6.2 BackendCashierController 收银台核心API详解

`BackendCashierController`（位于 `fuint-application/.../backendApi/controller/cashier/`）是连接管理端收银界面与后端业务的核心枢纽，提供以下完整收银流程：

| API端点 | HTTP方法 | 功能描述 | 关键逻辑 |
|---------|----------|---------|---------|
| `/init/{userId}` | GET | 收银台初始化 | 加载店铺信息、商品列表（分页+分类）、会员信息（若userId>0） |
| `/searchGoods` | POST | 商品搜索 | 按关键字搜索商品，支持分页 |
| `/getGoodsInfo/{id}` | GET | 商品详情 | 返回商品信息+规格列表+SKU列表（含多级规格处理） |
| `/getMemberInfo` | POST | 会员识别 | 按手机号→会员号→用户名优先级查询，多商户隔离校验 |
| `/getMemberInfoById/{userId}` | GET | 按ID查会员 | 商户归属校验 |
| `/doHangUp` | POST | 挂单 | 将购物车商品关联到桌台（设置tableId），支持游客模式 |
| `/getHangUpList` | GET | 挂单列表 | 按商户/店铺筛选未结账的挂单 |
| `/getTableList` | GET | 桌台列表 | 获取所有桌台及其占用状态 |
| `/getTableDetail/{tableId}` | GET | 桌台详情 | 查看某桌已点商品列表 |
| `/cleanTable/{tableId}` | GET | 清台 | 清空购物车+移除桌台订单关联+恢复桌台为可用状态 |
| `/turnTable` | POST | 转台 | 将一个桌台的全部订单/购物车转移到另一桌台 |
| `/removeGoods` | POST | 取消商品 | 从订单中移除指定商品（未支付前可操作） |

### 6.3 支付服务类职责

```
PaymentServiceImpl
├── 依赖 WeixinService    → 微信支付（JSAPI / H5 / Native）
├── 依赖 AlipayService    → 支付宝支付
├── 依赖 BalanceService   → 余额支付（会员储值余额扣减）
├── 依赖 CouponService    → 优惠券金额计算与核销
└── 依赖 OrderService     → 订单状态更新、支付完成回调
```

支持的支付方式枚举 `PayTypeEnum`：`WXPAY`（微信支付）、`ALIPAY`（支付宝）、`BALANCE`（余额支付）、`CASH`（现金支付，收银端使用）。

---

## 7. Ehcache + Redis 双重缓存体系

### 7.1 架构设计

系统采用 **L1（Ehcache 本地）+ L2（Redis 分布式）** 的两级缓存架构：

```
┌────────────────────────────────────────────────────────────────────┐
│                      缓存查询流程                                   │
│                                                                   │
│  业务请求                                                          │
│      │                                                            │
│      ▼                                                            │
│  ┌─────────────┐    命中     ┌─────────────┐                      │
│  │  L1: Ehcache │──────────→│  返回数据     │                     │
│  │  (本地JVM)   │            └─────────────┘                      │
│  └──────┬──────┘                                                  │
│         │ 未命中                                                   │
│         ▼                                                         │
│  ┌─────────────┐    命中     ┌─────────────┐                      │
│  │  L2: Redis  │──────────→│  返回数据     │                     │
│  │  (分布式)    │            │  回填L1缓存   │                     │
│  └──────┬──────┘            └─────────────┘                      │
│         │ 未命中                                                   │
│         ▼                                                         │
│  ┌─────────────┐                                                  │
│  │   MySQL     │  ← 查询数据库，结果回填 L2 → L1                     │
│  └─────────────┘                                                  │
└────────────────────────────────────────────────────────────────────┘
```

### 7.2 Ehcache 配置详解（一级缓存）

**配置文件**：`ehcache.xml`

```xml
<defaultCache
    maxElementsInMemory="200000"    <!-- 最大堆内200,000元素 -->
    eternal="false"                 <!-- 非永久有效 -->
    timeToIdleSeconds="120"        <!-- 120秒空闲失效 -->
    timeToLiveSeconds="120"        <!-- 120秒最大存活 -->
    overflowToDisk="true"          <!-- 内存溢出写入磁盘 -->
    maxElementsOnDisk="10000000"   <!-- 最大磁盘10,000,000元素 -->
    diskPersistent="false"
    diskExpiryThreadIntervalSeconds="120"
    memoryStoreEvictionPolicy="LRU" <!-- LRU淘汰策略 -->
/>

<cache name="store"
    maxElementsInMemory="100"      <!-- 店铺缓存：最多100条 -->
    overflowToDisk="false"         <!-- 不写磁盘 -->
    timeToIdleSeconds="0"          <!-- 不因空闲失效 -->
    timeToLiveSeconds="300"        <!-- 5分钟绝对过期 -->
    memoryStoreEvictionPolicy="LRU"
/>
```

**Ehcache 应用场景**：

| 缓存区域 | TTL | 最大条目 | 应用场景 |
|---------|-----|---------|---------|
| defaultCache | 120s | 200,000 | 通用缓存：商品列表、分类树、枚举值等低频变更的查询结果 |
| store | 300s | 100 | 店铺信息缓存（店铺数量通常较少，但查询频繁）；TTL较长（5分钟）以减少DB查询 |

**配置注入**：`application.yml` 中配置 `spring.cache.type: ehcache`，Spring Cache抽象层自动适配。

### 7.3 Redis 配置详解（二级缓存/分布式中间件）

**配置类**：`RedisConfig.java`

```java
@Configuration
@EnableCaching                       // 启用Spring Cache抽象
@EnableRedisHttpSession              // 启用Redis HttpSession共享
public class RedisConfig extends CachingConfigurerSupport {

    // 自定义KeyGenerator: className + methodName + params
    @Bean
    public KeyGenerator keyGenerator() { ... }

    // RedisTemplate: Key用String序列化，Value用Jackson2Json序列化
    @Bean
    RedisTemplate<String, Object> redisTemplate(...) { ... }

    // Jackson序列化器（支持泛型Object反序列化）
    @Bean
    Jackson2JsonRedisSerializer<Object> jackson2JsonRedisSerializer(...) { ... }
}
```

**Redis 的五大核心用途**：

#### 用途一：分布式 Session 共享

```java
@EnableRedisHttpSession  // 替代传统HttpSession，存储在Redis中
```
Spring Session 将 HttpSession 数据序列化到 Redis，支持多实例部署时的会话共享。用户登录后 Session 存储于 Redis，任意实例均可验证。

#### 用途二：Token 身份认证存储

```java
// TokenUtil.java — 用户Token
RedisUtil.set(
    Constants.SESSION_USER + token,   // key: "FOOD_USER_{token}"
    userInfo,                          // value: UserInfo对象(JSON)
    TOKEN_OVER_TIME                    // TTL: 604800秒 (7天)
);

// TokenUtil.java — 管理员Token
RedisUtil.set(
    Constants.SESSION_ADMIN_USER + token,  // key: "FOOD_ADMIN_USER_{token}"
    accountInfo,                            // value: AccountInfo对象
    TOKEN_OVER_TIME                         // TTL: 604800秒
);
```

拦截器 `AdminUserInterceptor` 和 `ClientUserInterceptor` 通过读取请求Header中的 `Access-Token`，在 Redis 中验证 Token 有效性。

#### 用途三：分布式锁（Lua脚本原子性操作）

```java
// RedisLock.java — 尝试获取锁
public boolean tryLock(String lockKey, String requestId, long expireTime) {
    // SET key value NX EX expireTime
    Boolean result = redisTemplate.opsForValue()
            .setIfAbsent(lockKey, requestId, expireTime, TimeUnit.SECONDS);
    return Boolean.TRUE.equals(result);
}

// RedisLock.java — 释放锁（Lua脚本保证原子性）
public boolean unlock(String lockKey, String requestId) {
    String script =
        "if redis.call('get', KEYS[1]) == ARGV[1] then " +
        "    return redis.call('del', KEYS[1]) " +
        "else return 0 end";
    Long result = redisTemplate.execute(redisScript,
        Collections.singletonList(lockKey), requestId);
    return result != null && result == 1L;
}
```

所有定时任务均使用此锁机制，防止多实例部署时的重复执行。

#### 用途四：验证码临时存储

```java
// CaptchaServiceImpl.java
RedisUtil.set(uuid, codeText, 1800);  // 验证码30分钟有效

// 验证时
String vCode = RedisUtil.get(uuid);   // 取出验证码比对
```

#### 用途五：微信 AccessToken 缓存

```java
// WeixinServiceImpl.java
String tokenKey = "wx_access_token_" + appId;
Object token = RedisUtil.get(tokenKey);               // 先查Redis
if (token == null) {
    // 调微信API获取新token
    RedisUtil.set(tokenKey, access_token, 7200);       // 缓存2小时
}
```

### 7.4 双重缓存策略总结

| 维度 | Ehcache (L1) | Redis (L2) |
|------|-------------|-----------|
| **部署方式** | JVM进程内（堆内存） | 独立进程（远程服务器） |
| **数据范围** | 本实例独占 | 多实例共享 |
| **访问延迟** | 纳秒级（内存直读） | 毫秒级（网络IO） |
| **存储容量** | 受JVM堆限制（200k条目） | 受机器内存限制 |
| **适用数据** | 高频读取、低频变更（商品列表、分类） | 会话状态、Token、锁、跨实例共享数据 |
| **一致性** | 实例间不一致（需TTL兜底） | 全局一致 |
| **故障影响** | 实例重启丢失（溢写磁盘可恢复） | 独立于应用生命周期 |
| **序列化** | Kryo（高性能二进制序列化） | Jackson2 JSON |

---

## 8. 数据库设计

### 8.1 数据库概览

- **数据库名**：`fuint-food`
- **引擎**：InnoDB
- **字符集**：utf8
- **表总数**：40+ 张表
- **命名规范**：
  - 业务表前缀 `mt_`（Model Table）：会员、订单、商品、卡券等核心业务表
  - 系统表前缀 `t_`：账号、角色、权限、操作日志等系统管理表

### 8.2 核心业务表 ER 关系

```
                        ┌──────────────┐
                        │  mt_merchant │  ← 商户（顶级租户）
                        │  id / no     │
                        └──────┬───────┘
                               │ 1:N
                        ┌──────┴───────┐
                        │   mt_store   │  ← 店铺（含QR_CODE、经纬度）
                        └──────┬───────┘
                               │
          ┌────────────────────┼────────────────────┐
          │                    │                    │
   ┌──────┴──────┐    ┌───────┴───────┐    ┌───────┴───────┐
   │  mt_table   │    │ mt_table_area│    │  mt_goods_cate│
   │ 桌台/桌码    │    │ 桌台区域      │    │ 商品分类       │
   └──────┬──────┘    └───────────────┘    └───────┬───────┘
          │                                       │
          │ TABLE_ID                               │ CATE_ID
          ▼                                       ▼
   ┌──────────────┐                        ┌──────────────┐
   │   mt_order   │◄──────────────────────│   mt_goods   │
   │  订单主表     │  ORDER_ID              │  商品主表     │
   └──────┬───────┘                        └──────┬───────┘
          │                                       │
          │ 1:N                                   │ 1:N
          ▼                                       ▼
   ┌──────────────┐                        ┌──────────────┐
   │mt_order_goods│  ← ORDER_ID + GOODS_ID │ mt_goods_sku │
   │ 订单商品明细  │                        │ 商品规格SKU   │
   └──────────────┘                        └──────────────┘

   ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
   │   mt_user    │────▶│ mt_user_coupon│◀────│  mt_coupon   │
   │  会员主表     │     │ 用户持有卡券   │     │  卡券定义     │
   └──────┬───────┘     └──────┬───────┘     └──────────────┘
          │                    │
          │                    │ CODE (扫码核销)
          ▼                    ▼
   ┌──────────────┐     ┌──────────────┐
   │ mt_balance   │     │mt_confirm_log│
   │ 余额变动记录  │     │  核销记录     │
   └──────────────┘     └──────────────┘
```

### 8.3 关键表设计说明

#### 桌台表 (mt_table) — 堂食场景核心

```sql
CREATE TABLE mt_table (
    ID          INT AUTO_INCREMENT,
    CODE        VARCHAR(30),        -- 桌台编码（用于生成二维码）
    AREA_ID     INT,                -- 所属区域（mt_table_area）
    MERCHANT_ID INT,                -- 所属商户
    STORE_ID    INT,                -- 所属店铺
    MAX_PEOPLE  INT,                -- 最大就餐人数
    SORT        INT,                -- 排序
    STATUS      CHAR(1),            -- 状态：A可用/D删除
    PRIMARY KEY (ID)
)
-- 桌台状态通过 TableUseStatusEnum 枚举在业务层管理：
-- AVAILABLE(空闲) / OCCUPIED(占用中) / RESERVED(已预订)
```

#### 购物车表 (mt_cart) — 挂单与桌码关联

```sql
CREATE TABLE mt_cart (
    ID          INT AUTO_INCREMENT,
    USER_ID     INT,                -- 会员ID（游客为0）
    TABLE_ID    INT,                -- 桌码ID（关联堂食订单）
    IS_VISITOR  CHAR(1),            -- 是否游客 Y/N
    HANG_NO     VARCHAR(10),        -- 挂单号（收银端先挂单后结算）
    SKU_ID      INT,                -- SKU ID
    GOODS_ID    INT,                -- 商品ID
    NUM         DOUBLE(10,2),       -- 数量
    PRIMARY KEY (ID)
)
```

#### 卡券表 (mt_coupon) — 三种卡券类型

```sql
CREATE TABLE mt_coupon (
    ID          INT AUTO_INCREMENT,
    TYPE        CHAR(1),            -- C:优惠券 / P:预存卡 / T:集次卡
    CONTENT     INT,                -- 1:满减券 / 2:折扣券
    NAME        VARCHAR(100),       -- 券名称
    AMOUNT      DECIMAL(10,2),      -- 面额
    TOTAL       INT,                -- 发行数量
    LIMIT_NUM   INT DEFAULT 1,      -- 每人持有数量限制
    EXPIRE_TYPE VARCHAR(30),        -- 过期类型
    EXPIRE_TIME INT,                -- 有效期（天）
    STORE_IDS   VARCHAR(100),       -- 适用店铺（逗号分隔）
    BEGIN_TIME  DATETIME,           -- 开始有效期
    END_TIME    DATETIME,           -- 结束有效期
    IN_RULE     VARCHAR(1000),      -- 获取规则说明
    OUT_RULE    VARCHAR(1000),      -- 核销规则说明
    STATUS      CHAR(1) DEFAULT 'A',
    PRIMARY KEY (ID)
)
```

---

## 9. Uniapp 前端架构与桌码参数解析

### 9.1 应用入口与启动流程

```
App.vue (根组件)
  │
  ├── main.js → 初始化流程：
  │   1. import uView UI → Vue.use(uView)
  │   2. import Vuex Store (app + user modules)
  │   3. import bootstrap → 从storage恢复token/userId到Vuex
  │   4. 挂载全局方法：$toast / $success / $error / $navTo / $getShareUrlParams
  │   5. 平台检测：$platform (H5 / MP-Weixin / MP-Alipay / App)
  │
  └── pages.json → 页面路由配置：
      ├── TabBar (4个主Tab)
      │   ├── pages/index/index      → 首页
      │   ├── pages/category/index    → 点单
      │   ├── pages/order/index       → 订单
      │   └── pages/user/index        → 我的
      │
      ├── 主包页面 (30+ pages)
      │   ├── pages/login/            → 登录(auth.vue, index.vue)
      │   ├── pages/cart/             → 结算确认
      │   ├── pages/pay/              → 支付 (index, cashier, result)
      │   ├── pages/goods/            → 商品详情/列表
      │   ├── pages/confirm/          → 核销
      │   ├── pages/user/             → 会员中心/会员码/会员卡
      │   └── ...
      │
      ├── 分包 subPages/ (预加载：首页加载时预下载)
      │   ├── subPages/book/          → 预约服务
      │   ├── subPages/coupon/        → 卡券详情/领取
      │   ├── subPages/prestore/      → 预存卡购买
      │   └── subPages/timer/         → 集次卡
      │
      └── 商户分包 merchantPages/ (预加载：会员中心加载时预下载)
          ├── merchantPages/index.vue     → 商户仪表盘
          ├── merchantPages/member/       → 会员管理
          ├── merchantPages/order/        → 订单核销
          ├── merchantPages/coupon/       → 卡券管理
          ├── merchantPages/staff/        → 员工管理
          └── merchantPages/refund/       → 退款处理
```

### 9.2 HTTP 请求拦截器 — 多租户/多店铺/桌码自动化参数注入

**文件**：`utils/request/index.js`

前端请求拦截器在每次API请求时自动附加以下请求头，实现多租户上下文的无感传递：

```javascript
// 请求前拦截器 — 自动附加上下文参数
$http.requestStart = options => {
    options.header['platform']   = store.getters.platform;       // 平台类型
    options.header['Access-Token'] = store.getters.token;         // 用户Token
    options.header['merchantNo'] = uni.getStorageSync("merchantNo") || config.merchantNo; // 商户号
    options.header['storeId']    = uni.getStorageSync("storeId") || 0;      // 店铺ID
    options.header['orderId']    = uni.getStorageSync("orderId") || 0;      // 订单ID
    options.header['tableId']    = uni.getStorageSync("tableId") || 0;      // ★ 桌码ID
    options.header['latitude']   = uni.getStorageSync("latitude") || '';    // 纬度
    options.header['longitude']  = uni.getStorageSync("longitude") || '';   // 经度
    options.header['isWechat']   = isWechat() ? 'Y' : 'N';                  // 是否微信环境
    return options;
}
```

**后端CORSFilter 对应允许的自定义Header**：`Access-Token`, `platform`, `latitude`, `longitude`, `storeId`, `merchantNo`, `isWechat`, `tableId`, `orderId`

### 9.3 桌码参数解析 — 完整的扫码点单链路

#### 步骤 1：扫描桌台二维码

顾客使用微信扫一扫功能扫描贴在餐桌上的二维码。二维码由后端 `QRCodeUtil` (ZXing) 生成，内容为一个带参数的URL：

```
https://www.fuint.cn/fuint-food/#/pages/category/index?tableId=520&storeId=12&merchantNo=10001
```

#### 步骤 2：页面加载参数解析

`pages/category/index.vue` 页面在 `onLoad` 生命周期中解析URL参数：

```javascript
onLoad(options) {
    const { tableId, orderId } = options;
    if (tableId) {
        uni.setStorageSync('tableId', tableId);  // 持久化到本地存储
    }
    if (orderId) {
        uni.setStorageSync('orderId', orderId);
    }
    // 后续所有API请求自动携带此tableId
    this.getStoreInfo();
    this.getCateList();
}
```

#### 步骤 3：就餐人数确认

当 `tableId > 0` 且系统配置 `peopleNum == 'Y'` 时，页面弹出就餐人数选择弹窗（1-12人）。人数选择后，开始加载该店铺的菜单商品。

#### 步骤 4：店铺信息加载

根据 `storeId`（从URL或配置获取），调用 `ClientStoreController.getStoreInfo()` 获取店铺信息。店铺信息通过回显在页面顶部的 `Location` 组件展示：

```
┌─────────────────────────────────────┐
│  📍 店铺名称                          │
│  桌码：A05                           │
│  切换店铺 >                          │
└─────────────────────────────────────┘
```

#### 步骤 5：商品浏览与加购

`mt_table` 表中 `CODE` 字段作为桌码标识，与 `mt_cart.TABLE_ID` 关联。用户添加商品到购物车时，`tableId` 通过请求Header传递给后端 `ClientCartController.saveCart()`，购物车记录自动关联桌台。

#### 步骤 6：下单提交

提交订单时，`ClientSettlementController` 读取Header中的 `tableId`，写入 `mt_order.TABLE_ID` 字段。订单创建后，后台更新 `mt_table` 对应桌台的使用状态为 **OCCUPIED**。

#### 步骤 7：支付完成

支付成功后：
- 订单状态更新为 `PAID`
- 如果是余额支付，扣除 `mt_user.BALANCE`
- 如果使用优惠券，核销 `mt_user_coupon` 并写入 `mt_confirm_log`
- 计算佣金（如果有分销关系），写入 `mt_commission_log`

### 9.4 会员二维码身份识别

`pages/user/code.vue` 页面生成会员个人二维码，供收银员扫描识别会员身份：

```javascript
// 调用 clientApi/user/qrCode 接口获取二维码
const qrCodeData = await UserApi.qrCode();
// qrCodeData 包含用户编码（userNo），在Canvas上渲染为二维码图片
```

收银员在 Cashier 端扫描后调用 `/backendApi/cashier/getMemberInfo`，通过手机号或会员号识别会员，自动关联到当前收银操作中。

### 9.5 Vuex 状态管理

```
store/
├── index.js              → Vuex Store 创建
├── getters.js            → 全局计算属性
├── mutation-types.js     → Mutation 常量
└── modules/
    ├── app.js            → 应用状态：platform, config, location
    ├── index.js          → 根模块
    └── user.js           → 用户状态：token, userId, userInfo, isLogin
```

关键状态流转：
```
Login → dispatch('Login', token)
     → commit('SET_TOKEN', token)
     → commit('SET_USERID', userId)
     → uni.setStorageSync('token', token)

Logout → dispatch('Logout')
      → commit('CLEAR_TOKEN')
      → uni.removeStorageSync('token')
      → uni.removeStorageSync('tableId')
      → uni.removeStorageSync('userId')
```

---

## 10. Cashier 收银台实时推单机制

### 10.1 收银台架构概述

fuintCashier 是基于 **Electron** 构建的 Windows 桌面应用程序，通过 electron-builder 打包为约 76MB 的安装程序，支持自动更新（latest.yml）。收银端与前端的协同分为 **Web管理端收银** 和 **Electron桌面收银** 两种形态。

### 10.2 BackendCashierController 收银核心流程

收银台的操作流程完全由后端 `BackendCashierController` 驱动：

```
收银员操作流程
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  1. 收银台初始化                                           │
│     GET /backendApi/cashier/init/{userId}                │
│     → 加载商品列表 + 分类 + 店铺信息 + 会员信息               │
│                                                          │
│  2. 会员识别                                               │
│     POST /backendApi/cashier/getMemberInfo               │
│     → 手机号→会员号→用户名 优先级查询                        │
│                                                          │
│  3. 商品搜索/加购                                          │
│     POST /backendApi/cashier/searchGoods                 │
│     GET  /backendApi/cashier/getGoodsInfo/{id}           │
│     → 多规格SKU选择 → 加入购物车 (CartService)              │
│                                                          │
│  4. 挂单（暂存订单）                                        │
│     POST /backendApi/cashier/doHangUp                    │
│     → cartIds + tableId → 购物车关联桌台                   │
│                                                          │
│  5. 桌台管理                                               │
│     GET /backendApi/cashier/getTableList                 │
│     → 实时查看各桌台状态（空闲/占用/已预订）                   │
│                                                          │
│  6. 桌台详情（查看已点商品）                                  │
│     GET /backendApi/cashier/getTableDetail/{tableId}     │
│     → 查看某桌的挂单商品明细                                 │
│                                                          │
│  7. 转台（换桌）                                            │
│     POST /backendApi/cashier/turnTable                   │
│     → 将一个桌台的全部订单/购物车转移到另一桌台                 │
│                                                          │
│  8. 取消商品                                               │
│     POST /backendApi/cashier/removeGoods                 │
│     → 从订单中移除指定商品（orderService.removeGoods）        │
│                                                          │
│  9. 清台（结账后清理）                                       │
│     GET /backendApi/cashier/cleanTable/{tableId}         │
│     → 清空购物车 + 移除桌台关联 + 恢复桌台状态为AVAILABLE      │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### 10.3 实时桌台状态感知机制

虽然系统当前未使用 WebSocket 长连接，但收银端通过**轮询+Redis状态共享**的方式实现近似实时的桌台状态同步：

```
┌──────────────────┐         ┌──────────────────┐
│   fuintCashier   │         │   fuintUniapp    │
│   (收银员操作)     │         │   (顾客扫码点单)   │
└────────┬─────────┘         └────────┬─────────┘
         │                            │
         │ 顾客下单完成                 │ 扫码点单 → 支付成功
         │                            │
         ▼                            ▼
┌────────────────────────────────────────────────┐
│              Spring Boot Backend               │
│                                                │
│  CartService.setTableId(cartId, tableId)       │
│  OrderService.createOrder(...) → TABLE_ID      │
│  TableService.updateUseStatus(                 │
│      tableId, OCCUPIED, orderId)               │
│                                                │
│  → 更新 mt_table 状态                           │
│  → 关联 mt_cart.TABLE_ID                        │
│  → 关联 mt_order.TABLE_ID                       │
└────────────────────────┬───────────────────────┘
                         │
         ┌───────────────┴───────────────┐
         │                               │
         ▼                               ▼
┌──────────────────┐            ┌──────────────────┐
│  Cashier 轮询     │            │  Merchant 端查询  │
│  getTableList()  │            │  (店员App)       │
│  间隔: 手动刷新   │            │                  │
│  或定时轮询       │            │                  │
│                  │            │                  │
│ 返回:             │            │                  │
│ [{ tableId: 5,   │            │                  │
│    useStatus:     │            │                  │
│    OCCUPIED,     │            │                  │
│    orderAmount:  │            │                  │
│    258.00 }]     │            │                  │
└──────────────────┘            └──────────────────┘
```

### 10.4 挂单机制（Hang Up）

"挂单"是餐饮行业的术语，指顾客已点菜但在未结账时暂存购物车。fuint 的挂单机制如下：

```
挂单操作:
  输入: cartIds (多个购物车条目ID) + tableId + userId
  执行: CartService.setTableId(cartId, tableId, isVisitor)
        → 将购物车条目标记为属于某个桌台

取单操作:
  请求: GET /backendApi/cashier/getHangUpList
        参数: merchantId, storeId, tableId(可选)
  返回: List<HangUpDto> {
          tableId,          // 桌台ID
          tableCode,        // 桌台编码
          cartInfo: [{      // 该桌台的购物车条目
              goodsId,
              goodsName,
              skuId,
              num,
              price,
              createTime
          }],
          total,            // 金额合计
          hangNo            // 挂单号
        }

清台操作:
  请求: GET /backendApi/cashier/cleanTable/{tableId}
  执行: cartService.removeCartByTableId(tableId)
        orderService.removeTakenTableId(tableId)
        tableService.updateUseStatus(tableId, AVAILABLE, null)
```

### 10.5 核销确认机制

收银员扫描顾客出示的二维码（会员码或券码）后，后端核销流程如下：

```
POST /backendApi/coupon/doConfirm  (管理端核销)
或
POST /merchantApi/order/confirm    (商户端核销)

请求参数:
{
    code: "xxx",           // 券码或核销码
    amount: 100.00,        // 本次核销金额
    remark: "核销备注"
}

后端校验链:
  1. 验证 code 是否存在
  2. 验证卡券状态是否为 UNUSED (未使用)
  3. 验证卡券是否未过期
  4. 验证适用店铺 (STORE_IDS 匹配)
  5. 如果是预存卡(P): 验证余额 >= 核销金额，扣减余额
  6. 如果是集次卡(T): 验证剩余次数 > 0，扣减次数
  7. 如果是优惠券(C): 更新状态为 USED

写入记录:
  → mt_confirm_log: 核销记录 (CODE, AMOUNT, COUPON_ID, USER_COUPON_ID, ORDER_ID)
  → mt_user_coupon: 更新卡券状态/余额/次数
  → mt_balance: 若涉及余额，记录变动明细
```

---

## 11. API 接口设计规范

### 11.1 三大 API 前缀体系

```
┌──────────────────────────────────────────────────────────────┐
│                      API 路由架构                             │
├──────────────────┬──────────────────┬───────────────────────┤
│  /clientApi/*    │  /backendApi/*   │   /merchantApi/*      │
│  会员端 (C端)     │  管理端 (B端)    │   商户端 (M端)         │
├──────────────────┼──────────────────┼───────────────────────┤
│ 认证方式:         │ 认证方式:         │ 认证方式:              │
│ Token (Redis)    │ Token + @Pre     │ Token (Redis)        │
│ 拦截器:           │ Authorize权限    │                      │
│ ClientUser       │ 拦截器:           │                      │
│ Interceptor      │ AdminUser        │                      │
│                  │ Interceptor      │                      │
├──────────────────┼──────────────────┼───────────────────────┤
│ 公开接口:         │ 公开接口:         │                      │
│ /sign/* (登录)   │ /login/* (登录)   │                      │
│ /store/* (店铺)  │ /captcha/*       │                      │
│ /goods/* (商品)  │                   │                      │
│ /coupon/* (卡券) │                   │                      │
│ /system/config  │                   │                      │
└──────────────────┴──────────────────┴───────────────────────┘
```

### 11.2 统一响应格式

```json
{
    "code": 200,
    "message": "操作成功",
    "data": { }
}
```

**错误码体系**：

| 错误码 | 含义 | 触发场景 |
|--------|------|---------|
| `200` | 成功 | 正常响应 |
| `201` | 业务错误 | 参数错误、数据不存在、校验失败 |
| `202` | 参数错误 | 必填参数缺失或格式不正确 |
| `402` | 用户不存在 | 会员查询无结果 |
| `403` | 登录错误 | 账号密码不匹配 |
| `1001` | 未登录 | Token过期或无效（前端自动跳转登录页） |
| `1004` | 商户不匹配 | 查询的会员不属于当前商户 |
| `5000` | 服务端错误 | 未预期的运行时异常 |

### 11.3 分页规范

**请求参数**：
```json
{
    "currentPage": 1,
    "pageSize": 20,
    "sortColumn": ["createTime"],
    "sortType": "desc",
    "searchParams": {
        "keyword": "搜索关键字",
        "status": "A"
    }
}
```

**响应格式**：
```json
{
    "code": 200,
    "data": {
        "content": [],
        "totalElements": 150,
        "totalPages": 8,
        "number": 0,
        "size": 20,
        "first": true,
        "last": false
    }
}
```

---

## 12. 安全架构

### 12.1 认证与授权流程

```
请求到达
    │
    ▼
┌─────────────────┐
│   CORSFilter    │  ← 设置Cross-Origin响应头 (允许任何来源 + 自定义Header)
└────────┬────────┘
         │
         ▼
┌─────────────────────────────┐
│  Spring Security Filter     │  ← /clientApi/**, /backendApi/**, /merchantApi/** 全部放行
│  (CSRF Disabled, STATELESS) │     静态资源、Swagger UI 放行
└────────┬────────────────────┘
         │
         ▼
┌─────────────────────────────┐
│  Custom Interceptors        │  ← 真正的鉴权在此层
│                             │
│  /clientApi/**  → ClientUserInterceptor
│    ├── 排除列表：/sign/**, /store/**, /goods/**, /coupon/**, /article/**, /captcha/**, /sms/**, /page/**, /share/**, /system/config
│    └── 其他URL：读取Header Access-Token → Redis查询 → 注入UserInfo到Request
│
│  /backendApi/** → AdminUserInterceptor
│    ├── 排除列表：/login/**, /captcha/**, /common/**, /file/**
│    └── 其他URL：读取Header Access-Token → Redis查询 → 注入AccountInfo到Request
│         └── @PreAuthorize("@pms.hasPermission('xxx')") 方法级权限校验
│
│  /cmd/**       → CommandInterceptor (仅允许localhost)
└────────┬────────────────┘
         │
         ▼
    Controller
```

### 12.2 权限模型（RBAC）

```
t_account (账号)
    │ N:M (t_account_duty)
    ▼
t_duty (角色)
    │ N:M (t_duty_source)
    ▼
t_source (菜单/权限资源)
    ├── SOURCE_NAME     → 权限名称
    ├── SOURCE_CODE     → 权限编码（如 cashier:index）
    ├── PATH            → 前端路由路径
    └── SOURCE_LEVEL    → 菜单层级
```

权限校验通过 `PermissionService` 实现：

```java
@Service("pms")
public class PermissionService {
    public boolean hasPermission(String permission) {
        // 1. 获取当前登录用户（从TokenUtil）
        // 2. 根据用户角色查询权限列表
        // 3. 判断是否包含所需权限
    }
}
```

Controller中使用：
```java
@PreAuthorize("@pms.hasPermission('cashier:index')")
public ResponseObject init(...) { ... }
```

### 12.3 Token 安全机制

- **Token 生成**：UUID + 时间戳 → MD5 哈希
- **Token 存储**：Redis `SET FOOD_USER_{token} UserInfo EX 604800`（7天）
- **Token 传递**：HTTP Header `Access-Token: {token}`
- **Token 校验**：每次请求通过拦截器从 Redis 读取验证

### 12.4 多租户数据隔离

- **商户隔离**：通过请求Header中的 `merchantNo` 确定当前商户上下文
- **店铺隔离**：通过 `storeId` 进一步限定数据范围
- **后端校验**：在 Service 层确保查询/操作的数据属于当前商户/店铺

---

## 13. 定时任务与分布式锁

### 13.1 定时任务总览

| 任务类 | CRON表达式 | 执行频率 | 职责 | 分布式锁Key |
|--------|-----------|---------|------|------------|
| `OrderCancelJob` | `0 0/2 * * * ?` | 每2分钟 | 超时30分钟未支付订单自动取消；已支付订单自动确认 | `lock:orderCancelJob:deal` |
| `OrderAutoJob` | `0 0/5 * * * ?` | 每5分钟 | 已收货10天自动确认完成；已完成1天自动关闭 | `lock:orderAutoJob:deal` |
| `CouponExpireJob` | `0 0/1 * * * ?` | 每1分钟 | 过期卡券状态更新；到期前3天微信提醒推送 | `lock:couponExpireJob:deal` |
| `CommissionJob` | `0 0/5 * * * ?` | 每5分钟 | 已支付订单的佣金计算与分配 | `lock:commissionJob:deal` |
| `MessageJob` | `0 0/1 * * * ?` | 每1分钟 | 定时微信订阅消息推送 | `lock:messageJob:deal` |
| `UploadShippingInfoJob` | `0 0/5 * * * ?` | 每5分钟 | 将物流信息上传至微信小程序 | `lock:uploadShippingInfoJob:deal` |

### 13.2 分布式锁防重复执行机制

每个定时任务采用统一的加锁/解锁范式，以 `OrderCancelJob` 为例：

```java
@Component("orderCancelJob")
public class OrderCancelJob {

    @Autowired
    private RedisLock redisLock;

    @Scheduled(cron = "${orderCancel.job.time:0 0/2 * * * ?}")
    @Transactional(rollbackFor = Exception.class)
    public void dealOrder() throws BusinessCheckException {
        String lockKey = "lock:orderCancelJob:deal";
        String requestId = SeqUtil.getUUID();   // 唯一标识本次执行

        try {
            // 尝试获取锁，60秒自动过期（防止死锁）
            if (redisLock.tryLock(lockKey, requestId, 60)) {

                // 环境变量控制开关
                String theSwitch = environment.getProperty("orderCancel.job.switch");
                if ("1".equals(theSwitch)) {
                    // === 业务处理逻辑 ===
                    // 1. 查询超时未支付订单
                    // 2. 取消订单 / 确认支付
                }
            }
        } finally {
            // 释放锁（Lua脚本保证原子性：只有持有者才能释放）
            redisLock.unlock(lockKey, requestId);
        }
    }
}
```

**分布式锁关键特性**：
- **SET IF ABSENT + EXPIRE**：原子操作，避免死锁
- **Lua脚本解锁**：确保只有锁的持有者才能释放锁（通过 `requestId` 比较）
- **自动过期**：即使进程崩溃，锁也会在60/30/10秒后自动释放
- **环境开关**：通过 `application.yml` 中的 `xxx.job.switch` 属性控制任务是否执行

---

## 14. 部署架构

### 14.1 推荐部署拓扑

```
┌───────────────────────────────────────────────────────────┐
│                      公网入口                              │
│                    https://www.fuint.cn                    │
└───────────────────────────┬───────────────────────────────┘
                            │
                    ┌───────┴───────┐
                    │    Nginx      │
                    │  (反向代理)    │
                    │               │
                    │ / → 静态文件   │
                    │ /fuint-food/  │
                    │   → 后端API    │
                    └───────┬───────┘
                            │
         ┌──────────────────┼──────────────────┐
         │                  │                  │
  ┌──────┴──────┐   ┌───────┴───────┐  ┌───────┴───────┐
  │ 静态资源     │   │ Spring Boot   │  │   Uni-app    │
  │ (Nginx直接  │   │ JAR 实例      │  │   H5页面      │
  │  返回)      │   │ (可多实例)     │  │  (Nginx托管)  │
  └─────────────┘   └───────┬───────┘  └───────────────┘
                            │
              ┌─────────────┼─────────────┐
              │             │             │
     ┌────────┴────┐  ┌────┴───────┐  ┌──┴──────────┐
     │   MySQL 8   │  │   Redis    │  │ 阿里云OSS    │
     │  (主数据库)  │  │ (缓存/Session│  │ (图片/文件)  │
     │             │  │  /分布式锁)  │  │             │
     └─────────────┘  └────────────┘  └─────────────┘
```

### 14.2 启动流程

```bash
# 1. 数据库初始化
mysql -u root -p < db/fuint-food.sql

# 2. 后端启动
cd fuintBackend
mvn clean package -DskipTests
java -jar fuint-application/target/fuint-application-1.0.0.jar \
    --spring.datasource.url=jdbc:mysql://localhost:3306/fuint-food \
    --spring.datasource.username=root \
    --spring.datasource.password=yourpassword \
    --spring.redis.host=localhost \
    --spring.redis.port=6379

# 3. Uniapp前端编译（HBuilderX IDE）
# 导入fuintUniapp项目 → 修改config.js中的apiUrl → 编译发布

# 4. Admin管理端启动（独立仓库）
cd fuintAdmin
npm install
npm run dev    # 开发模式，端口8081

# 5. Cashier桌面端
# 直接安装 fuint收银系统 Setup 1.0.0.exe
```

### 14.3 系统配置项

| 配置项 | 位置 | 说明 |
|--------|------|------|
| 数据库连接 | application.yml / 启动参数 | MySQL8 URL + 用户名密码 |
| Redis连接 | application.yml / 启动参数 | Redis主机 + 端口 |
| Ehcache配置 | ehcache.xml | 本地缓存参数 |
| API地址 | fuintUniapp/config.js | 后端接口基地址 |
| 商户号 | fuintUniapp/config.js | 默认商户编号 |
| 微信AppID | application.yml | 微信公众号/小程序AppID |
| 微信商户号 | application.yml | 微信支付商户号 |
| 阿里云AK | application.yml | OSS + SMS 访问密钥 |
| 定时任务开关 | application.yml | `xxx.job.switch=1` 启用 |
| Sentry DSN | sentry.properties | 异常监控上报地址 |

---

## 15. 附录

### 15.1 数据库表完整清单

| 序号 | 表名 | 中文名称 | 行数规模（参考） |
|------|------|---------|-----------------|
| 1 | mt_address | 会员地址表 | <100 |
| 2 | mt_article | 文章表 | <100 |
| 3 | mt_balance | 余额变动记录表 | 动态增长 |
| 4 | mt_banner | Banner焦点图 | <500 |
| 5 | mt_book | 预约服务表 | <10 |
| 6 | mt_book_cate | 预约分类表 | <10 |
| 7 | mt_book_item | 预约记录表 | 动态增长 |
| 8 | mt_cart | 购物车表 | 动态变化 |
| 9 | mt_commission_cash | 佣金提现记录 | <100 |
| 10 | mt_commission_log | 佣金明细记录 | 动态增长 |
| 11 | mt_commission_relation | 分销关系树 | 动态增长 |
| 12 | mt_commission_rule | 分佣规则表 | <10 |
| 13 | mt_commission_rule_item | 分佣规则项 | <50 |
| 14 | mt_confirm_log | 核销记录表 | 动态增长 |
| 15 | mt_coupon | 卡券信息表 | <100 |
| 16 | mt_coupon_goods | 卡券适用商品 | <50 |
| 17 | mt_coupon_group | 券组表 | <50 |
| 18 | mt_freight | 运费模板 | <10 |
| 19 | mt_freight_region | 运费适用地区 | <100 |
| 20 | mt_give | 卡券转赠记录 | 动态增长 |
| 21 | mt_give_item | 转赠明细 | 动态增长 |
| 22 | mt_goods | 商品主表 | 动态变化 |
| 23 | mt_goods_cate | 商品分类 | <100 |
| 24 | mt_goods_sku | 商品SKU | 动态变化 |
| 25 | mt_goods_spec | 商品规格 | 动态变化 |
| 26 | mt_invoice | 发票记录 | 动态增长 |
| 27 | mt_merchant | 商户主表 | <100 |
| 28 | mt_message | 站内消息 | 动态增长 |
| 29 | mt_open_gift | 注册礼包定义 | <20 |
| 30 | mt_open_gift_item | 注册礼包领取 | 动态增长 |
| 31 | mt_order | 订单主表 | 动态增长 |
| 32 | mt_order_address | 订单收货地址 | 动态增长 |
| 33 | mt_order_goods | 订单商品明细 | 动态增长 |
| 34 | mt_point | 积分变动记录 | 动态增长 |
| 35 | mt_printer | 云打印机配置 | <10 |
| 36 | mt_refund | 退款/售后记录 | 动态增长 |
| 37 | mt_region | 省市区数据 | 全国行政区划 |
| 38 | mt_send_log | 卡券发放记录 | 动态增长 |
| 39 | mt_setting | 系统设置表 | <500 |
| 40 | mt_settlement | 商户结算单 | 动态增长 |
| 41 | mt_settlement_order | 结算-订单关联 | 动态增长 |
| 42 | mt_sms_sended_log | 短信发送记录 | 动态增长 |
| 43 | mt_sms_template | 短信模板 | <20 |
| 44 | mt_staff | 员工表 | <100 |
| 45 | mt_stock | 库存记录主表 | 动态增长 |
| 46 | mt_stock_item | 库存明细 | 动态增长 |
| 47 | mt_store | 店铺表 | <100 |
| 48 | mt_store_goods | 店铺商品关联 | 动态变化 |
| 49 | mt_table | 桌台/桌码表 | <200 |
| 50 | mt_table_area | 桌台区域 | <20 |
| 51 | mt_upload_shipping_log | 物流上传记录 | 动态增长 |
| 52 | mt_user | 会员主表 | 动态增长 |
| 53 | mt_user_action | 会员行为日志 | 动态增长 |
| 54 | mt_user_coupon | 用户持有卡券 | 动态增长 |
| 55 | mt_user_grade | 会员等级定义 | <20 |
| 56 | mt_user_group | 会员分组 | <50 |
| 57 | mt_user_tag | 会员标签定义 | <50 |
| 58 | mt_user_tag_relation | 会员-标签关系 | 动态增长 |
| 59 | mt_user_tag_rule | 标签规则定义 | <50 |
| 60 | mt_verify_code | 短信验证码 | 临时数据 |
| 61 | t_account | 后台账户 | <100 |
| 62 | t_account_duty | 账户-角色关联 | <200 |
| 63 | t_action_log | 操作审计日志 | 动态增长 |
| 64 | t_duty | 角色定义 | <50 |
| 65 | t_duty_source | 角色-权限关联 | <500 |
| 66 | t_gen_code | 代码生成器配置 | <50 |
| 67 | t_platform | 平台配置 | <5 |
| 68 | t_source | 菜单/权限资源 | <200 |

### 15.2 枚举值速查表

| 枚举类 | 枚举值 | 含义 |
|--------|--------|------|
| `CouponTypeEnum` | C | 优惠券（满减/折扣） |
| | P | 预存卡（余额储值） |
| | T | 集次卡（按次消费） |
| `OrderStatusEnum` | CREATED | 已创建 |
| | PAID | 已支付 |
| | DELIVERED | 已发货 |
| | RECEIVED | 已收货 |
| | COMPLETED | 已完成 |
| | CANCELED | 已取消 |
| `PayStatusEnum` | WAIT | 待支付 |
| | SUCCESS | 支付成功 |
| | FAIL | 支付失败 |
| `PayTypeEnum` | WXPAY | 微信支付 |
| | ALIPAY | 支付宝 |
| | BALANCE | 余额支付 |
| | CASH | 现金（收银端） |
| `TableUseStatusEnum` | AVAILABLE | 空闲 |
| | OCCUPIED | 占用中 |
| | RESERVED | 已预订 |
| `PlatformTypeEnum` | MP | 微信小程序 |
| | H5 | 移动端H5 |
| | APP | 原生App |
| | PC | 桌面端/管理端 |
| `StaffCategoryEnum` | 1 | 店长/管理员 |
| | 2 | 收银员 |
| | 3 | 销售员 |
| | 4 | 服务员 |
| `MerchantTypeEnum` | restaurant | 餐饮 |
| | retail | 零售 |
| | service | 服务 |
| | other | 其他 |

### 15.3 核心配置文件索引

| 文件路径 | 用途 |
|---------|------|
| `fuintBackend/pom.xml` | Maven父POM（版本统一管理） |
| `fuintBackend/fuint-application/src/main/resources/application.yml` | Spring Boot主配置 |
| `fuintBackend/fuint-application/src/main/resources/ehcache.xml` | Ehcache缓存策略 |
| `fuintBackend/fuint-application/src/main/resources/logback-spring.xml` | 日志配置 |
| `fuintBackend/fuint-application/src/main/resources/captcha-conf.properties` | Kaptcha验证码 |
| `fuintBackend/fuint-application/src/main/resources/sentry.properties` | Sentry异常监控 |
| `fuintBackend/fuint-application/src/main/resources/urlRewrite.xml` | URL重写规则 |
| `fuintBackend/fuint-application/src/main/resources/international/message_zh_CN.properties` | 中文国际化消息 |
| `fuintBackend/fuint-application/src/main/resources/international/message_en_US.properties` | 英文国际化消息 |
| `fuintUniapp/config.js` | Uni-app环境配置 |
| `fuintUniapp/pages.json` | Uni-app页面路由配置 |
| `fuintUniapp/manifest.json` | Uni-app跨端编译配置 |
| `db/fuint-food.sql` | 完整数据库DDL |

---

