## 一、整体设计
### 业务流程
**对称加密下业务服务处理流程**
![对称加密算法下的认证流程.png](http://upload-images.jianshu.io/upload_images/1803273-989ba3ad6afaf2df.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

**非对称加密下业务服务处理流程**
![非对称加密算法下的认证流程.png](http://upload-images.jianshu.io/upload_images/1803273-b1dc151f4a7adcb4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 二、技术方案
#### 关键字 
 * client_key、client_secret: 注册业务服务后所返回的唯一标识用户的信息
 * APP_SECRET_REFREH_API：  业务服务同步密钥的借口（必须）
 * APP_API_SECRET：         业务服务同步密钥时所需要的校验签名
 * HTTP_X_AUTH_HMAC_SHA256：  HTTP首部信息， 认证服务根据APP_API_SECRET所计算的哈希值保存在存放在ＨTTP首部。 用来校验更新密钥请求和合法性。
 
#### 业务服务申请加入
 1. 业务服务填写应用名称、应用简介、应用地址、应用图表创建应用。
 2. 认证服务确认申请，返回client_key 和 client_secret 。
 3. 应用申请成功后在应用管理界面配置应用密钥更新接口APP_SECRET_REFREH_API(用于对称密钥的同步), 配置校验签名APP_API_SECRET（用来验证所受到的密钥是否来自于服务器）。

## 三、具体细节


#### 对称加密算法密钥同步
1. 业务服务提供同步接口, 该借口接受来自于认证服务的POST请求。
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