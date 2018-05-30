# RESTful架构设计提案

关于如何设计系统架构，有诸多意见。这里提议设计RESTful Web API，理由如下：
* RESTful可以实现前后端的最大程度脱耦，减少沟通成本。每个人都可以用自己熟悉的技术。
* REST服务端是无状态的，不需要储存用户会话数据，逻辑更为简单。
* 关于REST的无状态，可以认为，已登录用户会在每次API调用时携带上自己的认证信息。
* 工作方式易于理解：
  * 渲染过的前端页面作为静态资源被部署在特定目录（例如服务器根目录下）直接访问
  * 当浏览器访问Web应用时，页面和前端脚本被下载到客户端
  * 前端页面与后端的交流通过API调用完成
  * 调用的方式为：使用一个HTTP请求，对一个URL关联的资源做读写操作
  * 服务端会将资源以一个JSON字符串返回客户端
  * 前端页面利用返回的JSON实现页面的数据显示

## 快速理解RESTful架构
### 概念与API设计
* 阮一峰：[RESTful API 设计指南](http://www.ruanyifeng.com/blog/2014/05/restful_api.html)

里面提到了GitHub本身的RESTful架构设计，可以借鉴。

### 实现
* Java: [用 Java 技术创建 RESTful Web 服务](https://www.ibm.com/developerworks/cn/web/wa-jaxrs/index.html)
* Node: [Node.js RESTful API](http://www.runoob.com/nodejs/nodejs-restful-api.html)

### REST API测试工具
* [Postman](https://www.getpostman.com/)

## API设计
本节将尝试设计一个REST架构的教务系统API。将前后端分离是一个重要的特性，后端的实现与前端无关。二者仅通过HTTP请求交互。

### 登录认证
#### 登录流程如下：
* 客户端将用户名、密码和时间戳（以Unix时钟秒计）`{username, password, timestamp}`以BASE64编码，POST到服务器的`/api/login/userinfo`上。
* 服务端收到用户信息后，检查时间戳是否过期，用户名和密码是否匹配
* 如果用户名、密码都得到了认证，服务器会利用密钥计算一个access token，和查询到的用户角色信息一起返回给客户端`{accessToken, roles}`，并返回`200`；如果不匹配，服务器返回`204`
  * 客户端收到服务器返回的数据后，会将`{accessToken, roles, username}`保存在本地数据存储`window.localStorage`中。`accessToken`的载荷是`{username, roles, iat}`，分别为用户名、角色和时间戳（以Unix时钟秒计），用JWT的方式、HS256算法加密，密钥保存在后端代码的配置文件中
  * 如果客户端收到的是`204`，会自行解决错误
* 客户端在认证成功的情况下，会将access token作为自己HTTP请求头中`authorization`的默认值，然后会跳转至用户面板页面

> 注意：本地数据存储`window.localStorage`中的数据形式都是字符串，`roles`角色是一个数组对象。例如`['student', 'instructor', 'admin']`在本地数据存储中会以`'student, instructor, admin'`形式保存。
> 角色数组中可能存有三种角色类型，都以字符串的形式表示。它们分别是`student`（学生）、`lecturer`（教师）和`admin`（管理员）
#### 相关接口
##### POST /api/login/userinfo
