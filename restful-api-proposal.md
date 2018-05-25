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

### 登录认证
在RESTful这种无状态架构中，一般使用OAuth：
* 阮一峰：[理解OAuth 2.0](http://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html)

## API设计
本节将尝试设计一个REST架构的教务系统API。
