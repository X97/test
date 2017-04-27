## 一、整体设计
### 业务流程
**对称加密下业务服务处理流程**

![对称加密算法下的认证流程(1).png](http://upload-images.jianshu.io/upload_images/1803273-6132291953978c58.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

对称加密算法下先由业务服务验证token的有效性。

**非对称加密下业务服务处理流程**
![非对称加密算法下的认证流程(1).png](http://upload-images.jianshu.io/upload_images/1803273-56822b9b34f4d982.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

非对称加密算法下由认证服务验证token的有效性。

### 业务设计
#### 数据库
![登陆认证SQL.png](http://upload-images.jianshu.io/upload_images/1803273-3343019d76fdf97c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

 * 存放注册应用相关的信息/client
name、 description、 domain、 clilent_key、 client_secret
 * 存放用户相关信息/user
mobile_phone、 email、password、 avator 
 * 管理每个客户端/ clientmanager

#### 类

![身份认证类图.png](http://upload-images.jianshu.io/upload_images/1803273-d6e5307aa187efa0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 二、技术方案
#### 关键字 
 * client_key、client_secret: 注册业务服务后所返回的唯一标识用户的信息
 * APP_SECRET_REFREH_API：  业务服务同步密钥的接口（必须）
 * APP_API_SECRET：         业务服务同步密钥时所需要的校验签名
 * HTTP_X_AUTH_HMAC_SHA256：  HTTP首部信息， 认证服务根据APP_API_SECRET所计算的哈希值保存在存放在ＨTTP首部。 用来校验更新密钥请求和合法性。
 
### 申请应用
 1. 业务服务填写应用名称、应用简介、应用地址、应用图表创建应用。
 2. 认证服务确认申请，返回client_key 和 client_secret 。
 3. 应用申请成功后在应用管理界面配置应用密钥更新接口APP_SECRET_REFREH_API(用于对称密钥的同步), 配置校验签名APP_API_SECRET（用来验证所受到的密钥是否来自于服务器）。
### 放置登录标识
### OAuth2.0对接
 * 用户打开客户端以后，客户端要求用户给予授权。
 * 用户同意给予客户端授权。
 * 客户端使用上一步获得的授权，向认证服务器申请令牌。
 * 认证服务器对客户端进行认证以后，确认无误，同意发放令牌。
## 三、具体细节


### 1. 密钥同步验证策略

1. 业务服务提供同步接口, 该接口接受来自于认证服务的POST请求。
2. 通过校验签名APP_API_SECRET判断POST请求是否来自于认证服务器。
  * 认证服务器将新的对称加密密钥跟用户所填写的校验签名APP_API_SECRET进行哈希运算， 将哈希运算的结果放在HTTP首部HTTP_X_AUTH_HMAC_SHA256中。
  * 认证服务送POST请求到业务服务器提供的密钥更新接口。 
  * 业务服务收到更新密钥的请求后对收到的密钥跟自己所填写的校验签名APP_API_SECRET签名进行哈兮运算。并将运算结果跟HTTP首部中HTTP_X_AUTH_HMAC_SHA256的内容进行比对。 如果比对成功则证明是来自认证服务器的更新密钥请求。否则应当忽略该请求。
 
 ```
    def get_hmac(body, secret):
        """
        Calculate the HMAC value of the given request body and secre
        """
        hash = hmac.new(secret.encode('utf-8'), body, hashlib.sha256)
        return base64.b64encode(hash.digest()).decode()


    def hmac_is_valid(body, secret, hmac_to_verify):
        """
        Return True if the given hmac_to_verify matches that calculated from the given body and secret.
        """
        return get_hmac(body, secret) == hmac_to_verify

    def check_valid_request():
        hmac = request.META['HTTP_X_AUTH_HMAC_SHA256']
        if not hmac_is_valid(request.body, settings.SHOPIFY_APP_API_SECRET, hmac):
            return False
        return True
  ```
    
3. 更新对称密钥。

4. 业务服务器更新后返回确认信息。 若业务服务没有及时返回确认信息， 那么每过2的i(0 < N) 次方时间重新发送一次密钥(Ｎ的大小取决于生成密钥的周期)。 当i大于Ｎ时， 即使再有新的密钥更新时也不再对该业务服务器发送同步请求。直到业务服务器重新请求新初始化加密数据为止。

### 2. 密钥同步策略
认证服务器要保证不同的业务服务器使用不同的密钥， 即密钥的唯一性。假设共有50个业务服务。每一个密钥的生命周期为60分钟。
那么每过60分钟就要对到期的密钥进行一次旋转。

![密钥同步结构设计(1).png](http://upload-images.jianshu.io/upload_images/1803273-4c57de87405e580b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

具体做法：
1. 创建一个0到60的环形队列， 环上的每个slot是一个业务服务器的集合。
2. 设定一个指针。 指向环上的某一个slot， 每隔一分钟移动一下指针。
3. 遍历指针所指向slot业务服务集合， 对每一个业务服务的密钥进行旋转同步。
4. 当有新的业务服务到来时， 初始化密钥。 并将该业务服务加入到指针所指向的上一个slot中。
5. 当某个业务服务长时间未进行确认回复时， 将该指针从对应的集合中移除。

### 3. 获得初始化对称加密密钥
 * 认证服务器提供初始化密钥借口接口， 当业务服务器第一次启动、或者宕机后重新启动应该请求该接口。
 
  ```
  GET /initial_secret/?client_id=<client_id>&client_secret=<client_secret>
  Host: www.example.com
  ```
 
 ## 问答
 #### 如何及时有效的同步密钥信息？
   答： 数据库中会记录每个注册应用的接受密钥更新请求的地址。 对所有有效的应用发送更新请求。 当发送更新请求后， 如火认证服务没有受到确认回复。 认证请求会重复发送， 直到超出预期发送次数。 超出预期发送次数后， 认证服务器会标记该应用为不可用。 密钥更新后不会对该应用发送密钥更新请求。
