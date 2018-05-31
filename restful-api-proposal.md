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
本节将尝试设计一个REST架构的教务系统API。将前后端分离是一个重要设计原则，后端的实现与前端无关。二者仅通过HTTP请求交互。

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
**功能：** 验证用户登陆信息

**返回：** 成功时返回`{accessToken, roles}`状态码`200`，失败时只返回状态码`204`

##### GET /api/login/userquery/:username
**功能：** 检验用户名`username`是否存在

**返回：** 成功时返回状态码`200`，失败时返回状态码`204`

**使用场景：** 用于登录界面下的用户名校验

##### GET /api/login/auth
**功能：** 检验是否仍然处于登录状态。客户端会将HTTP头的`authorization`设置为access token，在发送这一GET请求时，认证也会随之送往服务器。服务端会对这一票据做检查，检查时间戳是否超越最长登录时间。

**返回：** 成功时返回状态码`200`，失败时返回状态码`204`

#### 后端实现
完成登录认证的相关逻辑在后端代码的`modules/LoginAuth.js`中。导出的`LoginAuth`对象本身是一个构造器（用其他语言理解，相当于一个类和一个构造函数），接受一个用户信息参数。`LoginAuth`本身还有成员函数（用其他语言理解，从语法和语义上相当于Java或C++的静态成员）。

用`LoginAuth`构造对象：

```javascript
var a = LoginAuth(userinfo);
```

其中，`userinfo`是BASE64形式的用户登录信息`{username, password, timestamp}`。`LoginAuth`构造器会解码这个用户信息，如果认证成功，可以通过成员函数`.getAccessToken()`获得一个access token返回客户端。原型对象的成员（用Java或C++的话讲，非静态成员）如下：

```
.userinfo         // 用来构造该对象的用户登录信息
.isValidUser      // 布尔值，是否是有效用户
.isAuthorized     // 布尔值，是否已经被认证
.getAccessToken() // 获得一个access token
.getUserRoles()   // 获得用户角色数组
```

`LoginAuth`对象直接的成员有（可以用Java或C++的静态成员类比）：

```
LoginAuth.isUser(username)               // 判断是否是合法用户
LoginAuth.isVerifiedAccessToken(token)   // 判断access token是否被认证
LoginAuth.getRoles(token)                // 从access token中获得用户角色
LoginAuth.getUsername(token)             // 从access token中获得用户名
```

涉及到数据访问的操作在`dataModels/user.js`，目前还是假数据。在后面应该用相同的接口的代码替换，将数据库返回的结果以JSON的形式与其他逻辑相交互。

> 下面这部分API设计仅供启发思路，作为REST架构应用的一个风格倡议。

### 用户信息
#### 接口
##### POST /api/user/:userid
**前置条件：** 用户已认证，且具有管理员角色

**数据内容：** 
```javascript
BASE64(JSON.stringfy({
  username: <string>,
  password: <string>,
  name : <string>,
  sex  : <string|'male'/'female'/'others'>,
  phone: <string>,
  email: <string>,
  role : ['student', 'lecturer', 'admin'],
  depart: <string>,
  class: <string>,
  admissionYear: int,
  courses: ['123-2016-1', '456-2017-2'],
}))
```

**样例数据：**
```javascript
BASE64(JSON.stringfy({
  username: '3162319803',
  password: 'yangle250ERZHENG',
  name : '杨乐二正',
  sex  : 'male',
  phone: '12312344321',
  email: 'yangleer@zheng.moe',
  role : ['student'],
  depart: '计算机科学与技术',
  class: '1509班',
  admissionYear: 2016,
  courses: ['123-2016-1', '456-2017-2'],
}))
```

上文中的`BASE64`仅代表使用BASE64编码，具体形式取决于实现。

**行为：**
1. 检查必要字段完好性（如用户名、密码等）
2. 覆盖更新数据库内容

**异常：** 若未认证或认证用户不具有管理员角色，返回403

##### DELETE /api/user/:username
**前置条件：** 用户已认证，具有管理员角色，URL参数`username`存在。

**行为：** 删除（为了安全性，或懒惰删除）

**异常：** 若未认证或认证用户不具有管理员角色，返回403

##### PUT /api/user/:username
**前置条件：** 用户已认证，具有管理员角色，URL参数`username`存在。

**数据内容：** 与上文POST类同，但不要求包含所有必要字段。

**行为：** 合并更新用户数据

**异常：** 若未认证或认证用户不具有管理员角色，返回403

##### GET /api/user/:username
