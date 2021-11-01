# 一、Kerberos简介

Kerberos协议由麻省理工学院开发，如今主要被用于Windows网络认证，Windows 2000和之后的操作系统使用Kerberos协议为其默认认证方法。加入Windows域则代表启动Kerberos作为客户端到服务端认证的默认协议，如果双方其一或全都未加入Windows域，则使用NTLM进行身份验证。

# 二、标准认证流程

Kerberos协议认证过程中主要存在三种角色：

- `Client`，客户端，访问服务

- `Server`，服务端，提供服务

- `KDC`（Key Distribution Center），密钥分发中心


Kerberos协议的认证流程如下图所示：

![](https://adan0s-1256533472.cos.ap-nanjing.myqcloud.com/uPic/20211027112800783vop.png)

## 1. KRB_AS_REQ

`Client`向`KDC`的`AS（Authentication Service）`服务发送`KRB_AS_REQ`请求，目的是获得`TGT`（Ticket Granting Ticket），发送的内容包括使用`Client`用户哈希加密的时间戳，这也就是用户凭证。

这一步也叫做预认证，证明`Client`有域内的用户和对应的哈希，可以进行下一阶段的请求。

![](https://adan0s-1256533472.cos.ap-nanjing.myqcloud.com/typora/image-20211023131511329.png#alt=image-20211023131511329)

抓包来看，`KRB_AS_REQ`包含的字段如下：

- `pvno`，Kerberos协议版本号，现在常用的是`Kerberos 5`

- `msg-type`，消息类型，`10`即`0x0a`，代表`krb-as-req`

- `padata`，认证相关信息

- `req-body`，里面包含`kdc-options`、`cname`、`sname`、`till`、`nonce`、`etype`等

![](https://adan0s-1256533472.cos.ap-nanjing.myqcloud.com/typora/image-20211023132742014.png#alt=image-20211023132742014)


`padata`即`PA_DATA`，其中在`KRB_AS_REQ`阶段时用到的主要是：
1. `ENC_TIMESTAMP`，用户哈希加密的时间戳
2. `PA_PAC_REQUEST`，启动PAC支持的扩展

![](https://adan0s-1256533472.cos.ap-nanjing.myqcloud.com/typora/image-20211023132903876.png#alt=image-20211023132903876)

在`req-body`里，主要有：
1. `kdc-options`，一些flag字段
2. `cname`，发起请求的用户名
3. `sname`，请求的服务名，在`KRB_AS_REQ`中为`krbtgt`
4. `realm`，域名
5. `till`，到期时间
6. `nonce`，随机生成的一个数
7. `etype`，加密类型

## 2. KRB_AS_REP

KDC在收到`KRB_AS_REQ`之后，使用发起请求的用户的哈希解密凭证得到时间戳，如果解密成功并且时间在一定范围内，则通过预认证。

预认证通过后，`KDC`会返回`TGT`票据，此票据是使用`KDC`的`krbtgt`用户的hash加密的，理论上无法被伪造和解密。

![](https://adan0s-1256533472.cos.ap-nanjing.myqcloud.com/typora/image-20211023135414817.png#alt=image-20211023135414817)

`KRB_AS_REP`的主要字段有：
- `pvno`， 版本
- `msg-type`，消息类型
- `crealm`，域名
- `cname`，请求的用户名
- `ticket`，加密的票据
- `enc-part`，里面有下阶段的认证密钥

![](https://adan0s-1256533472.cos.ap-nanjing.myqcloud.com/typora/image-20211023135946426.png#alt=image-20211023135946426)

可以看到，在`ticket`字段里，主要包含了域名、服务名、使用`krbtgt`用户哈希加密的密文等。

![](https://adan0s-1256533472.cos.ap-nanjing.myqcloud.com/typora/image-20211023141143421.png#alt=image-20211023141143421)

而`enc-part`里也是密文，但这里的密文是使用请求用户的哈希加密的，所以是可以解密的，里面是下一阶段的认证密钥。

## 3. KRB_TGS_REQ

通过预认证之后，`Client`向`KDC`的`TGS（Ticket Granting Service）`服务发起请求，请求中包含了之前申请的TGT。

主要的字段和`KRB_AS_REQ`相同，但内容有区别。

![](https://adan0s-1256533472.cos.ap-nanjing.myqcloud.com/typora/image-20211023151121665.png#alt=image-20211023151121665)

`padata`字段里的`ap-req`字段里的`ticket`就是上面申请的TGT，而`authenticator`是在`KRB_AS_REP`中得到的下一阶段认证密钥。

![](https://adan0s-1256533472.cos.ap-nanjing.myqcloud.com/typora/image-20211023170353176.png#alt=image-20211023170353176)

`req-body`里主要有这些字段：
- `kdc-options`，一些特殊选项，比如是否可转发等
- `cname`，请求的用户名
- `realm`，域名
- `sname`，需要申请访问的服务名，如`cifs`等，如果服务是`krbtgt`，那得到的ST（服务票据）是可以当做TGT来使用的
- `till`，到期时间
- `nonce`，随机数

## 4. KRB_TGS_REP

`KDC`验证`TGT`票据和对应服务后，会返回`ST`票据给`Client`，这张票据对应着相关服务的访问权限，也就是俗称的白银票据。此票据是使用所请求的服务账户hash加密的，理论上也不可伪造和解密。

![](https://adan0s-1256533472.cos.ap-nanjing.myqcloud.com/typora/image-20211023170921736.png#alt=image-20211023170921736)

这些字段就不再解释，和之前步骤的大致相同，这里同样有`enc-part`，它是使用前面`KRB_AS_REP`阶段得到的`key`字段加密的，解密后同样得到一个`key`，作为下一阶段的认证密钥。

## 5. KRB_AP_REQ

`Client`请求访问对应的服务，提供ST和`KRB_TGS_REP`阶段得到的认证密钥加密的字段，以获取访问权限。通过设置`ap-options`的`MUTUAL-REQUIRED`可以进行双向认证。

## 6. KRB_AP_REP

除非`Client`要求进行双向认证，否则服务端一般不回复`KRB_AP_REP`.

# 三、扩展后的认证流程

Kerberos协议支持S4U扩展，通过此扩展可以实现委派的功能。

这时流程中存在四个角色：

- `Client`，客户端
- `KDC`
- `Service1`，代理用户请求的服务
- `Service2`，要被访问的服务

`S4U`扩展有两个子协议：`S4U2self`和`S4U2proxy`，前者让`Service1`代表用户获得对`Service1`自身的票据，也就是获得了`Client`对`Service1`的票据；后者则允许`Service1`代表用户去请求`Service2`的服务权限。

这里要说明的是，委派功能并不完全等同于`S4U`扩展，还能使用forwarded TGT的方法实现，流程如下图：

![](https://adan0s-1256533472.cos.ap-nanjing.myqcloud.com/typora/image-20211023173355835.png#alt=image-20211023173355835)

从上面可以看出，这种方式要求`Client`要先通过Kerberos的方式进行认证，而`S4U2self`可以让用户使用其他的验证方式来获得对`Service1`的访问权限，这样就简化了整体流程，也扩展了适用场景。

使用`S4U`扩展后，委派的流程如下：

![](https://adan0s-1256533472.cos.ap-nanjing.myqcloud.com/typora/image-20211023184240382.png#alt=image-20211023184240382)

## 1. Request

`Client`可以使用Kerberos以外的验证方式，来获得对`Service1`的访问权限，这里`Service1`并不需要得到`Client`的相关凭据。

## 2. S4U2self

`Service1`已经从KDC处得到了TGT，那么它会代表`Client`去请求KDC，以获得自身可转发的ST.

![](https://adan0s-1256533472.cos.ap-nanjing.myqcloud.com/typora/image-20211023185753064.png#alt=image-20211023185753064)

可以看到`padata`里多了`pA-FOR-USER`，里面的`KerberosString`字段表明`Service1`是代表哪个用户进行请求的。

![](https://adan0s-1256533472.cos.ap-nanjing.myqcloud.com/typora/image-20211023190222231.png#alt=image-20211023190222231)

`req-body`里能看到`cname`和`sname`都是`Service1`，因为`Service1`代替了`Client`向KDC发起请求，请求的目标服务也是`Service1`.

要想获得可转发的ST，需要满足：
- 使用的TGT是可转发的
- `Service1`配置了约束委派
- 请求时设置了可转发标志
- `Client`不是敏感账户

## 3. Service ticket, as if from user

KDC返回对`Service1`的可转发ST给`Service1`

## 4. Response

这一步实际上没什么作用，因为单靠`S4U2self`协议并不能请求到其他服务，这需要靠`S4U2proxy`完成。

## 5. Request

`Client`向`Service1`发起请求，需要`Service1`以`Client`的身份去访问`Service2`，这里就要求`Service1`通过前几个步骤来获得对自身的可转发ST.

## 6. S4U2proxy for Service 2

`Service1`代表`Client`向KDC请求获得`Service2`的ST，这个ST也应是可转发的，所以`kdc-options`应设置`forwardable`和`cname-in-addl-tkt`，后者是为了标志`S4U2proxy`功能。

![](https://adan0s-1256533472.cos.ap-nanjing.myqcloud.com/typora/image-20211023195410300.png#alt=image-20211023195410300)

与普通的`KRB_TGS_REQ`一样，使用的TGT放在`padata`中。这里的`cname`是`Client`，`sname`为`Service2`.

![](https://adan0s-1256533472.cos.ap-nanjing.myqcloud.com/typora/image-20211023195458675.png#alt=image-20211023195458675)

这次还多了一个`Service1`的ST，放在了`req-body`的`additional-tickets`中。

## 7. Ticket for Service 2

如果上一步的请求中存在特权属性证书（PAC），那么KDC会验证PAC，在PAC有效或不存在PAC的情况下，返回`Service2`的ST给`Service1`。此ST中`cname`和`crealm`字段存储的是`Client`的信息，不是`Service1`的信息。

## 8. Request As User

`Service1`向`Service2`发起请求，后者会将其看作是`Client`，并且认为`Client`已经通过了KDC验证。

## 9. Response

`Service2`响应`Service1`

## 10.Response

`Service1`响应用户
