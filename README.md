苹果规定 从2017年1月1日起,新提交的 app 不允许使用NSAllowsArbitraryLoads来绕过ATS(全称：App Transport Security)的限制。

 

以前为了能兼容http和不满足规定的https，我们采用了最偷懒的做法：设置NSAllowsArbitraryLoads 为YES。即：

    <key>NSAppTransportSecurity</key>
    <dict>
        <key>NSAllowsArbitraryLoads</key>
        <true/>
　　　　
　　</dict>
现在这个方式被苹果给禁止了。 

 

一、使用默认的ATS设置要满足：

1、https 要基于TLS 1.2或以上版本。

2、证书的加密的算法要至少要SHA256的算法，用至少是2048位的RSA的key 或至少是256位的Elliptic-Curve(ECC)的key所产生的证书

3、加密算法也是有限制，就是ATS中的ForwardSecrecy（超前的密码保护算法）配置项，需要在以下列表中，详见：苹果文档。

TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA384
TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA
TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA256
TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA
TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA384
TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256
TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA

 

如果不符合上述3各要求，请求接口会报错：

NSURLSession/NSURLConnection HTTP load failed (kCFStreamErrorDomainSSL, -9802)

 

二、如何适配

1、让服务端或运维的小伙伴 配置下tomcat或nginx的TSL版本和证书算法，更新最新的SDK，比如友盟统计SDK刚刚可以支持ATS了，新浪微博等用的还是TLS1.0版本。需要配置一下。

2、我们公司的证书算法达不到苹果的要求，运维的童鞋又很忙（不想改），因为苹果只是禁用了NSAllowsArbitraryLoads选项，我们可以通过其他的选项来兼容以前的接口。如下：

 

复制代码
    <key>NSAppTransportSecurity</key>
    <dict>
        <key>NSExceptionDomains</key>
        <dict>
            <key>xxxx.com</key>
            <dict>
                <key>NSExceptionMinimumTLSVersion</key>
                <string>TLSv1.0</string>
                <key>NSThirdPartyExceptionRequiresForwardSecrecy</key>
                <false/>
                <key>NSIncludesSubdomains</key>
                <true/>
            </dict>
        </dict>
    </dict>
复制代码
 

扩展：我们公司用的 Red ware（类似F5） 不支持  ECDHE算法，所以上面我们配置了NSThirdPartyExceptionRequiresForwardSecrecy。

目前最常用的密钥交换算法有 RSA 和 ECDHE：RSA 历史悠久，支持度好，但不支持 PFS（Perfect Forward Secrecy）；而 ECDHE 是使用了 ECC（椭圆曲线）的 DH（Diffie-Hellman）算法，计算速度快，支持 PFS。

在 RSA 密钥交换中，浏览器使用证书提供的 RSA 公钥加密相关信息，如果服务端能解密，意味着服务端拥有证书对应的私钥，同时也能算出对称加密所需密钥。密钥交换和服务端认证合并在一起。

在 ECDHE 密钥交换中，服务端使用证书私钥对相关信息进行签名，如果浏览器能用证书公钥验证签名，就说明服务端确实拥有对应私钥，从而完成了服务端认证。密钥交换和服务端认证是完全分开的。

可用于 ECDHE 数字签名的算法主要有 RSA 和 ECDSA，也就是目前密钥交换 + 签名有三种主流选择：

RSA 密钥交换（无需签名）；
ECDHE 密钥交换、RSA 签名；
ECDHE 密钥交换、ECDSA 签名；
有兴趣了解更多的可以看： ECC证书

 

扩展2：如果用了nginx，可以直接在nginx.conf 中配置ssl_ciphers 设置加密选项。

 

三、ATS配置说明：
http://www.cnblogs.com/dahe007/p/6093874.html

  

 

详细说明：

NSAppTransportSecurity : 配置ATS的跟属性。


NSAllowsArbitraryLoads : 是否允许所有的连接，包括http。默认值是No，如果设置成YES，就可以使用http了。2017年1月1号后该选项被禁用。

 

NSExceptionDomains : 用于配置例外的域名，即在该配置项中的域不需要通过ATS的验证。


< domain-name-for-exception-as-string > : 需要添加例外的域名字符串，如:baidu.com


NSExceptionMinimumTLSVerion :最低支持的TSL的版本号，可用的配置有TLSv1.0、TLSv1.1以及TLSv1.2三个配置项。


NSExceptionRequiresForwardSecrecy ：是否满足上文中列举的加密算法。因为我们服务端不支持这些算法，所以我们设置为NO。


NSExceptionAllowsInsecureHTTPLoads : 是否为HTTPS的服务器。用这个配置可用访问那些没有证书、自签名证书、过期证书以及证书与域名匹配不上的服务器。默认值是NO。


NSIncludesSubdomains : 子域名是否使用同样的配置。默认值是NO。如果设置为YES，则子域名如api.baidu.com也都使用相同的配置


NSThirdPartyExceptionMinimumTLSVersion : 如果是域名为第三的域名，且开发人员无法控制的情况下进行配置。设置最低支持的TSL的版本号


NSThirdPartyExceptionRequiresForwardSecrecy : 如果是域名为第三的域名，且开发人员无法控制的情况下进行配置ForwardSecrecy。

 

 
