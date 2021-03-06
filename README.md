## 统一身份认证报告


### 1.概述
>> 通常,如果一个公司有多个管理应用系统,各个应用系统有着自己独立的帐号管理系统. 当用户在各个系统中切换的时候,常常要输入不同的账户信息. 这种方式使得用户要记住不同的帐号,在不同的系统中登录. 而理想的模型就是,使用统一的身份认证方式. 用户登陆统一的认证服务后, 即可使用所有支持统一认证服务的管理应用系统.

### 2.流程
![统一身份认证.png](http://upload-images.jianshu.io/upload_images/1803273-c304d2b7a794523e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

  1. 用户进入业务系统(sys1).
  2. 业务系统发现用户未登陆或登陆失效. 业务系统跳转到认证中心.
  3. 认证中心发现用户没用登录, 便引导用户到登录页面. 用户登录后返回token.业务系统(sys)记录令牌.
  4. 用户系统进入业务系统(sys2).
  5. 业务系统(sys2)发现用户没有登录, 业务系统跳转认证中心.
  6. 认证中心发现用户已经登录过, 便直接返回token.

### 主要功能
  1. 用户管理:能实现用户与组织创建、删除、维护与同步等功能；
  2. 用户认证:通过SOA服务，支持第三方认证系统；
  3. 单点登录:共享多应用系统之间的用户认证信息，实现在多个应用系统间自由切换；
  3. 分级管理:实现管理功能的分散，支持对用户、组织等管理功能的分级委托；
  4. 权限管理:系统提供了统一的，可以扩展的权限管理及接口，支持第三方应用系统通过接口获取用户权限。
  5. 会话管理:查看、浏览与检索用户登录情况，管理员可以在线强制用户退出当前的应用登录；


### 总结
  现在基本上掌握了aaa业务的业务需求. 关注点在于认证服务和业务服务. 认证服务负责身份验证,授权, 检查token的有效性. 业务服务检查token的有效性, 请求授权.


