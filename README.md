# 智练营平台

## 一、项目介绍

基于Next.js服务端渲染 + Spring Boot + Redis + Mysql + Elasticsearch 的面试刷题平台。

- 管理员可以创建题库、题目和题解，并批量关联题目到题库；
- 用户可以注册登录、分词检索题目、在线刷题并查看刷题记录日历等。

### 架构设计图

![image](https://github.com/user-attachments/assets/6cebf681-e2f1-4003-81fa-c8e8427ac838)
![image](https://github.com/user-attachments/assets/5f1e8c54-802d-44c2-afde-3b355a50bf17)
![image](https://github.com/user-attachments/assets/eab094e0-9383-419c-a4a2-786841501e13)

以上：COS 为对象存储，如图片资源；

### 技术选型

> 后端

- Java Spring Boot 框架 + Maven 多模块构建；
- MySQL 数据库 + Mybatis-Plus 框架 + Mybatis X；
- Redis 分布式缓存 + Caffeine 本地缓存；
- Redission 分布式锁 + BitMap + BloomFilter；
- Elasticsearch 搜索引擎；
- Druid 数据库连接池 + 并发编程；
- Sa-Token 权限控制；
- HotKey 热点探测；
- Sentinel 流量控制；
- Nacos 配置中心；
- 多角度优化：性能、安全性、可用性；

> 前端

- React 18框架；
- Next.js 服务端渲染；
- Redux 状态管理；
- Ant Design 组件库；
- 前端工程化：ESlint + Prettier + TypeScript；
- OpenAPI 前端代码生成；

## 二、项目构建

### 项目功能拆解

#### 1、用户模块

- 用户注册
- 用户登录
- 管理员（管理用户 - 增删改查）

#### 2、题库模块

- 查看题库列表
- 查看题库详情（展示题库下的题目）
- 管理员（管理题库 - 增删改查）

#### 3、题目模块

- 题目搜索
- 查看题目详情（进入刷题界面）
- 管理员（管理题目 - 增删改查）

#### 4、高级功能

- 题目批量管理（批量向题库添加题目、批量从题库移除题目、批量删除题目）
- 分词题目搜索
- 用户刷题记录日历图
- 自动缓存热门题目
- 网站流量控制和熔断
- 动态IP黑白名单过滤
- 同端登录冲突检测
- 分级题目反爬虫策略；

### 后端项目构建

#### 1、环境准备

- 因为本地缓存 Caffeine 要求 JDK 适配版本为11，这里选择 JDK 版本为11；
- Node.js 版本 >= 18.18.0；（这里使用nvm包管理器便捷切换不同版本应用）

#### 2、库表设计

项目数据库为：**zhilian_sys**；数据表分为四个核心表：用户表、题库表、题目表、题库题目关联表；

```sql
-- 创建库
create database if not exists zhilian_sys;

-- 切换库
use zhilian_sys;
```

##### 用户表

其中，unionId、mpOpenId是为了实现公众号登录，每个微信用户在同一家公司的 unionId 是唯一的，在同一个公众号的 mpOpenId 是唯一的；

```sql
-- 用户表
create table if not exists user
(
    id           bigint auto_increment comment 'id' primary key,
    userAccount  varchar(256)                           not null comment '账号',
    userPassword varchar(512)                           not null comment '密码',
    unionId      varchar(256)                           null comment '微信开放平台id',
    mpOpenId     varchar(256)                           null comment '公众号openId',
    userName     varchar(256)                           null comment '用户昵称',
    userAvatar   varchar(1024)                          null comment '用户头像',
    userProfile  varchar(512)                           null comment '用户简介',
    userRole     varchar(256) default 'user'            not null comment '用户角色：user/admin/ban',
    editTime     datetime     default CURRENT_TIMESTAMP not null comment '编辑时间',
    createTime   datetime     default CURRENT_TIMESTAMP not null comment '创建时间',
    updateTime   datetime     default CURRENT_TIMESTAMP not null on update CURRENT_TIMESTAMP comment '更新时间',
    isDelete     tinyint      default 0                 not null comment '是否删除',
    index idx_unionId (unionId)
) comment '用户' collate = utf8mb4_unicode_ci;
```

> 扩展点

（1）实现会员功能

- 给 userRole 字段增加枚举值 VIP 表示会员用户，根据该值判断用户权限；
- 新增会员过期时间字段，可用于记录会员有效期；
- 新增会员兑换码字段，可用于记录会员开通方式；
- 新增会员编号字段，可便于定位用户并提供额外服务，增加会员归属感；

对应SQL：

```sql
vipExpireTime datetime     null comment '会员过期时间',
vipCode       varchar(128) null comment '会员兑换码',
vipNumber     bigint       null comment '会员编号'
```

（2）实现用户邀请功能

- 新增 shareCode 分享码字段，用于记录每个用户的唯一邀请标识，可拼接到邀请网址后面；
- 新增 inviteUser 字段，用于记录该用户被哪个用户邀请了，可通过这个字段查询某用户邀请的用户列表；

对应SQL：

```sql
shareCode varchar(20) DEFAULT null comment '分享码',
inviteUser bigint     DEFAULT null comment '邀请用户id'
```



##### 题库表

```sql
-- 题库表
create table if not exists question_bank
(
    id          bigint auto_increment comment 'id' primary key,
    title       varchar(256)                       null comment '标题',
    description text                               null comment '描述',
    picture     varchar(2048)                      null comment '图片',
    userId      bigint                             not null comment '创建用户 id',
    editTime    datetime default CURRENT_TIMESTAMP not null comment '编辑时间',
    createTime  datetime default CURRENT_TIMESTAMP not null comment '创建时间',
    updateTime  datetime default CURRENT_TIMESTAMP not null on update CURRENT_TIMESTAMP comment '更新时间',
    isDelete    tinyint  default 0                 not null comment '是否删除',
    index idx_title (title)
) comment '题库' collate = utf8mb4_unicode_ci;
```

> 扩展点

（1）如果要实现题库创建审核功能

- 新增审核状态字段，用枚举值表示待审核、通过和拒绝；
- 新增审核信息字段，用于记录未过审的原因等；
- 新增审核人id字段，便于审计操作；比如出现了违规内容过审情况，可以追责到人；
- 新增审核时间字段，也是便于审计；

对应SQL：

```sql
reviewStatus  int default 0   not null comment '审核状态：0-待审核，1-通过，2-拒绝',
reviewMessage varchar(512)    null     comment '审核信息'，
reviewerId    bigint          null     comment '审核人id',
reviewTime    datetime        null     comment '审核时间'
```

（2）如果实现题库排序功能

- 可以新增整型的排序字段，并且根据该字段排序；

- 该字段还可以用于快速实现题库精选和置顶功能，比如优先级 = 1000 的题库表示精选，优先级 = 10000 的题库表示置顶；

对应SQL：

```sql
priority int default 0 not null comment '优先级'
```

（3）如果要实现题库浏览功能

- 可以新增题库浏览数字段，每进入题库该字段 + 1，还可以根据该字段对题库热门度进行排序；

对应SQL：

```sql
viewNum int default 0 not null comment '浏览量'
```



##### 题目表

```sql
-- 题目表
create table if not exists question
(
    id         bigint auto_increment comment 'id' primary key,
    title      varchar(256)                       null comment '标题',
    content    text                               null comment '内容',
    tags       varchar(1024)                      null comment '标签列表（json 数组）',
    answer     text                               null comment '推荐答案',
    userId     bigint                             not null comment '创建用户 id',
    editTime   datetime default CURRENT_TIMESTAMP not null comment '编辑时间',
    createTime datetime default CURRENT_TIMESTAMP not null comment '创建时间',
    updateTime datetime default CURRENT_TIMESTAMP not null on update CURRENT_TIMESTAMP comment '更新时间',
    isDelete   tinyint  default 0                 not null comment '是否删除',
    index idx_title (title),
    index idx_userId (userId)
) comment '题目' collate = utf8mb4_unicode_ci;
```

**几个重点：**

- 题目标题 title 和题目创建人 userId 是常用的题目搜索条件，所以添加索引提升查询性能；
- 题目可能有多个标签，为了简化设计，没有采用关联表，而是以 JSON 数组字符串方式存储，比如：["java","python"]；
- 题目内容（详情）和题目答案可能很长，所以使用 text 类型存储；

> 扩展点

（1）审核功能

```sql
reviewStatus  int default 0   not null comment '审核状态：0-待审核，1-通过，2-拒绝',
reviewMessage varchar(512)    null     comment '审核信息'，
reviewerId    bigint          null     comment '审核人id',
reviewTime    datetime        null     comment '审核时间'
```

（2）评价题目的指标

可能通过浏览数、点赞数、收藏数等来进行题目质量排序；

```sql
viewNum   int default 0 not null comment '浏览量',
thumbNum  int default 0 not null comment '点赞数',
favourNum int default 0 not null comment '收藏数'
```

（3）实现题目排序、精选和置顶

```sql
priority int default 0 not null comment '优先级'
```

（4）版权风险

如果题目是从其他网站或途径获取到的，担心有版权风险，可以增加题目来源字段，最简单的实现方式就是直接存储来源名称信息；

```sql
source varchar(512) null comment '题目来源'
```

（5）设置部分题目会员可见

- 可以新增一个 ”是否仅会员可见“ 字段，本质上是布尔类型，用1表示会员可见；

```sql
needVip tinyint default 0 not null comment '仅会员可见：1-会员可见；0-不限'
```



##### 题库题目关联表

```sql
-- 题库题目表（硬删除）
create table if not exists question_bank_question
(
    id             bigint auto_increment comment 'id' primary key,
    questionBankId bigint                             not null comment '题库 id',
    questionId     bigint                             not null comment '题目 id',
    userId         bigint                             not null comment '创建用户 id',
    createTime     datetime default CURRENT_TIMESTAMP not null comment '创建时间',
    updateTime     datetime default CURRENT_TIMESTAMP not null on update CURRENT_TIMESTAMP comment '更新时间',
    UNIQUE (questionBankId, questionId)
) comment '题库题目' collate = utf8mb4_unicode_ci;
```

**几个重要点：**

- 题库表和题目表两个是多对多关系；
- 上面 userId 表示添加题目到题库的用户 id，仅管理员可操作；
- 由于关联表中的数据记录并没有那么重要，这里直接使用硬删除方式；
- 通过设置 “题库id” 和 “题目id” 添加**联合唯一索引**，防止题目被重复添加到题库中，这里还使用索引**最左前缀原则**将题库id放到最前面，使得查询更高效；

> 扩展点

（1）如果需要对题库内题目进行排序

- 这里可以增加题目顺序字段；

```sql
questionOrder int default 0 not null comment '题目顺序（题号）'
```

#### 3、后端项目初始化

1. 使用后端项目模板快速修改结构，创建数据库表并且生成相应的逻辑层；
2. 新建数据库和表后，使用 Mybatis-X 生成对应模块的实体类、service、mapper层代码；
3. 使用万用模板生成不同实体类的dto、service、controller等基本crud功能；
4. 修改代码逻辑，补充校验规则等，实现增删改查功能；
5. 根据实际业务修改数据模型，如前端需要交互的数据结构，后端返回到前端的数据结构；
6. 待办：controller业务数据校验阶段：

#### 4、业务代码

##### 核心业务分析

在以下业务中，题库、题目创建修改更新都仅限于管理员用户来操作，普通用户进入界面只能查找信息和刷题操作；

###### 1、用户模块

- 用户注册：已完成
- 用户登录：已完成
- 【管理员】管理用户：已完成

###### 2、题库模块

- 查看题库列表：分页获取题库接口
- 查看题库详情（展示题库下的题目）：根据 id 获取题库详情接口
- 【管理员】管理题库：增删改查

###### 3、题目模块

- 题目搜索：分页获取题目接口
- 查看题目详情（进入刷题界面）：根据 id 获取题目详情接口
- 【管理员】管理题目：增删改查
- 【管理员】按照题库查询题目：根据题库 id 获取题目列表
- 【管理员】修改题目所在的题库信息等；

### 前端项目构建

#### 1、客户端渲染和服务端渲染

> 什么是客户端渲染和服务端渲染？

**客户端渲染**：客户端渲染就比如浏览器，客户端会先向服务器发送 HTML 请求，服务器会返回一个基础的 HTML 文件，包含 JavaScript 脚本，动态请求后端数据后再渲染到页面上。

在客户端渲染中，页面加载时文档预览为页面标题和一些执行的 js 脚本。

**服务端渲染**：服务端渲染和客户端渲染是相对的，服务端渲染是将要返回的数据一并返回渲染到页面，不用再动态地向后端发送请求。当用户请求一个页面时，服务器会提前调用后端数据并生成完整的 HTML 文档，然后将其发送到客户端。

在服务端渲染中，页面加载后文档预览包含网页数据及完整的 HTML。

> 服务端渲染和客户端渲染区别？

**客户端渲染**：

- 开发方便灵活：开发者不需要区分哪些数据要在服务器加载，便于实现更加复杂和动态的用户界面，适合构建单页面应用和需要频繁交互的应用。
- 减少服务器压力：由于渲染主要由客户端完成，只需要提供静态资源，服务器负载相对较小。

**服务端渲染**：

- 减少初始加载时间：SSR(服务端渲染) 页面可以在首次加载时展示完整内容，减少白屏时间；而 CSR(客户端渲染) 通常需要等待 JS 加载和执行后才能展示内容。
- SEO 友好：SSR 更有利于 SEO(让浏览器更先搜索到你的网站)
- SSR 将渲染任务交给服务器，可能会加重服务器负载和压力，所以 SSR 更适合追求性能和 SEO 的企业级项目。

**静态网站**：静态网站生成（SSG），静态网站生成是在构建时生成页面，所有页面都是以静态文件的形式存在。

- 高性能：由于数据不变化，加载快，特别适合 CDN 缓存加速；
- SEO 友好：搜索引擎最喜欢静态 HTML 文件，可以轻松索引并提升 SEO 效果。

静态网站适合内容数量有限、内容基本不变的网站，比如个人博客、资讯网站、产品官网页面；像 VuePress、Hugo、Hexo、Astro 都是主流的静态网站生成器。

在客户端、服务端、静态页面三种方式选择中，使用选择优先级：静态 ——> 服务端 ——> 客户端；三者结合使用来加快网站加载速度。

#### 2、Next.js 服务端渲染

##### 项目初始化

实现服务端渲染的方式很多，以前有 Java 的 JSP 、PHP 等等，现在有基于 React 的 Next.js 和基于 Vue 的 Nuxt.js 框架。

Next.js 文档：[Introduction | Next.js](https://nextjs.org/docs)

项目文档初始安装流程：node 版本必须是18.18以上

```bash
# 在项目文件下安装
npx create-next-app@latest

# 切换 npm 镜像仓库为国内源
npm get registry
npm config set registry https://registry.npmmirror.com/
```

![image](https://github.com/user-attachments/assets/7410b1b4-83bd-4db6-bfd1-689daab7c8f6)


然后使用编辑器打开运行 `run dev` ，访问本地3000端口。

##### 修改环境

如果要整合 `Prettier` 在项目目录下安装一下命令：

`npm install --save-dev eslint-config-prettier`

`npm install --save-dev --save-exact prettier`

修改 `.eslintrc.json` 文件：

```json
{
  "extends": ["next/core-web-vitals", "prettier"]
}
```

修改编辑器配置：

![image](https://github.com/user-attachments/assets/58cfeee0-167d-4cb4-b392-c601415d9dbf)


然后语法校验规则就生效了，代码格式化：Ctrl + Alt + L

##### 引入组件库

Ant Design 是 React 项目主流的组件库，并且这个组件是支持服务端渲染 Next.js ；本项目使用这个组件开发：[组件总览 - Ant Design](https://ant-design.antgroup.com/components/overview-cn/?from=msidevs.net)

```bash
# 执行安装组件
npm install antd --save

# 安装优化
npm install @ant-design/nextjs-registry --save

# 在 app/layout.tsx 中使用，修改文件，这是一个全局文件
import { AntdRegistry } from '@ant-design/nextjs-registry';
<AntdRegistry>{children}</AntdRegistry>
```

引入 Ant Design Procomponents 高级组件，使用表单、下拉框、文本域等高级套件：[简介 - ProComponents](https://procomponents.ant.design/docs/intro)

```bash
npm i @ant-design/pro-components --save
```

##### Next.js 开发规范

###### 1、路由

Next.js 使用**约定式路由**，根据文件夹的结构和名称，自动将对应的 URL 地址映射到页面文件。

> 路由

- 基础路由：以 app 目录作为根目录，根据文件夹名称和嵌套层级，自动映射为 URL 地址。只有目录下包含 page 文件才会被识别。
- 路由组：可以通过 (xxx) 创建虚拟路由，实际指向为下一个文件夹
- 动态路由：可以通过 [xxx] 语法，让多个不同参数的 URL 复用一个页面，比如 app/question/:id: 来动态拼接不同项目id。

###### 2、静态资源

Next.js 约定在 public 目录下存放静态资源。在其中新建 assets 目录，可以存放图片等静态资源文件。

然后就可以使用 Image 组件加载静态资源，比如：

```bash
<Image src={'/assets/logo.png'} alt={alt} width="64" height="64" />
```

Next.js 会针对该组件进行特定的图像优化，提升性能。

对于网站标识小图标，Next.js 约定将其放到 app 目录下，还有 robots.txt 也是放到 app 目录下。

###### 3、文件组织形式

首先，项目中的每个页面和组件都是单独的文件夹。每个页面目录中需要添加 `page.tsx` 和 `index.css` 样式文件；

对于项目中多个页面公用组件，放在 src/components 目录下。对于私有组件，在页面目录下新建 components 目录。

###### 4、页面开发规范

Next.js 支持 React 语法，可以用函数的方式声明页面和组件。

为了区分客户端和服务端渲染，在每个页面（组件）开头都要有以下标识：

- 客户端渲染：**use server**
- 服务端渲染：**use client**

在服务端渲染开发过程中，不要使用 window 相关，这是客户端渲染的，容易冲突。
