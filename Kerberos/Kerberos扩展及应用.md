# 一、PAC


Kerberos协议解决了身份验证的问题，但却没有身份控制的部分，也就是说每个域内用户都可以自由申请TGT来获得域内任意服务的访问权限，这显然是不符合实际需求的。微软为此引入了PAC（特权属性证书），里面包含了用户信息、凭据信息、配置文件等。


引入PAC后，Kerberos流程发生了一些改变：


1. 在`Client`向`KDC`发起`KRB_AS_REQ`后，KDC验证成功会返回TGT，而在TGT里包含了PAC
2. `Client`使用TGT发起`KRB_TGS_REQ`，KDC验证成功后返回对应服务的ST
3. `Client`使用ST去请求对应的服务，而服务在验证ST正确后，会带着PAC请求KDC，验证`Client`是否有访问服务的权限，如果有才允许访问



可以看到，PAC让Kerberos多了一道验证用户权限的流程，这样就确保了服务只被有权限的用户访问。


但是并不是所有服务都有请求KDC来验证PAC的步骤，否则白银票据攻击也就无法进行了。


## PAC结构


![](https://adan0s-1256533472.cos.ap-nanjing.myqcloud.com/typora/image-20211024174034552.png#id=uaDmF&originHeight=413&originWidth=518&originalType=binary&ratio=1&status=done&style=none)


PAC整体是一个`AuthorizationData`结构：


```c
AuthorizationData       ::= SEQUENCE OF SEQUENCE {
              ad-type         [0] Int32,
              ad-data         [1] OCTET STRING
}
```


`ad-type`有如下几种：


```
AD-IF-RELEVANT                     1
AD-INTENDED-FOR-SERVER             2
AD-INTENDED-FOR-APPLICATION-CLASS  3
AD-KDC-ISSUED                      4
AD-AND-OR                          5
AD-MANDATORY-TICKET-EXTENSIONS     6
AD-IN-TICKET-EXTENSIONS            7
AD-MANDATORY-FOR-KDC               8
Reserved values                 9-63
OSF-DCE                           64
SESAME                            65
AD-OSF-DCE-PKI-CERTID             66 (hemsath@us.ibm.com)
AD-WIN2K-PAC                     128 (jbrezak@exchange.microsoft.com)
AD-ETYPE-NEGOTIATION             129  (lzhu@windows.microsoft.com)
```


在PAC中，`ad-type`为`AD-IF-RELEVANT`，而`ad-data`还是一个`AuthorizationData`结构，这个`AuthorizationData`结构里`ad-type`为`AD-WIN2K-PAC`，`ad-data`里存放着相关数据。


在`ad-data`里，最先是一个`PACTYPE`结构，作为PAC元素的头部。`PACTYPE`结构包含了：


- `cBuffers`，4字节，一个32位无符号整数，定义`PAC_INFO_BUFFER`数组的个数
- `Version`，4字节，一个32位无符号整数，定义PAC版本
- `Buffer`，`PAC_INFO_BUFFER`结构数组



在`PACTYPE`后面还有若干个`PAC_INFO_BUFFER`，这些结构用作指向后面PAC数据的指针。


`PAC_INFO_BUFFER`结构是键值对型，重要的有：


-  `0x00000001`，登录信息，PAC结构必须包含一个此种类型的缓冲区，使用它来验证用户身份 
-  `0x0000000A`，客户端名称和票证信息，PAC结构必须包含一个此种类型的缓冲区，包含客户端ID、名称、名称长度。 
-  `0x00000006`，服务端校验和，PAC结构必须包含一个此种类型的缓冲区，包含签名类型、签名值和`RODCIdentifier`，使用服务hash加密 
-  `0x00000007`，KDC校验和，PAC结构必须包含一个此种类型的缓冲区，内容同上，使用KDC hash加密 



服务端校验和可以避免客户端自己生成`PAC`放入票证中，KDC校验和可以避免不受信任的服务伪造包含无效PAC的票据。


## 伪造PAC


`MS14-068`这个漏洞允许域内任意用户提升至域管理员权限，本质上就是KDC没有正确检查上述的两个有效签名，导致用户可以自行构造PAC。


计算校验和使用的是HMAC类的CHECKSUM算法，需要key作为输入，这里的key就是服务或KDC的hash，但此处还可以使用其他不需要key的CHECKSUM算法，这样能更改PAC的内容后再自己生成校验和。


登录信息是一个`KERB_VALIDATION_INFO`结构，其中`UserId`是用户的SID，`GroupIds`是用户所在的组SID，如果把组改为域管理员组，那么KDC解密`PAC`时就会把对应的用户当做域管理员组成员。


在`KRB_TGS_REQ`阶段，`TGT`票据在`PA-DATA`的`AP_REQ`的`ticket`字段中，将构造好的PAC加密之后放入`enc-authorization-data`里，把解密用的key放在`AP_REQ`的`authenticator`的`subkey`中，KDC获取key并解密KDC。


# 二、委派


Kerberos协议允许通过代理的方式间接访问服务，这就是委派，即`Client`让`Service1`代替自己去访问`Service2`。它在Active Directory最初发布时就存在，是为了解决用户能通过Kerberos协议向Web服务进行身份验证并更新数据的情况而推出的。


## 1. 非约束委派


在最初发布的版本中，委派是无约束的，也就是一个服务账户被配置了非约束委派后，它就可以接受**任何用户**的委派去请求**任何服务**。只有服务账户或计算机账户能被配置委派。


![](https://adan0s-1256533472.cos.ap-nanjing.myqcloud.com/typora/image-20210921231125726.png#id=iC02E&originHeight=668&originWidth=487&originalType=binary&ratio=1&status=done&style=none#id=G6IAp&originHeight=668&originWidth=487&originalType=binary&ratio=1&status=done&style=none)


`Client`通过委派让`Service1`去访问服务`Service2`，`Client`会把`TGT`发送给`Service1`，之后`Service1`就拿着`TGT`去KDC处申请访问`Service2`的`ST`，但这里的`TGT`是不受限制的，完全可以拿着它去请求另一个服务。


实际上此时并没用到`S4U`扩展，而是使用转发TGT的方式来实现的委派。


## 2. 约束委派


在Windows Server 2003之后，约束委派可用。相较于非约束委派，约束委派明确了所代理的服务，即不能再去访问任意的服务了，这是通过`S4U`扩展实现的。


约束委派只能由具有`SeEnableDelegation`特权的用户（通常为域管理员）来为服务配置，并且限定为单个域。


![](https://adan0s-1256533472.cos.ap-nanjing.myqcloud.com/typora/image-20210921233406950.png#id=oMDVS&originHeight=674&originWidth=491&originalType=binary&ratio=1&status=done&style=none#id=s5wdS&originHeight=674&originWidth=491&originalType=binary&ratio=1&status=done&style=none)

配置了约束委派的账户，其属性`userAccountControl`会有一个值为`TRUSTED_TO_AUTH_FOR_DELEGATION`：

![image-20211025224205025](https://adan0s-1256533472.cos.ap-nanjing.myqcloud.com/typora/image-20211025224205025.png)

如果没有这个值，虽然`S4U2self`扩展还能工作，但生成的ST不是可转发的。

属性`msDS-AllowedToDelegateTo`里是`Service2`的SPN：

![image-20211025230416470](https://adan0s-1256533472.cos.ap-nanjing.myqcloud.com/typora/image-20211025230416470.png)

## 3. 基于资源的约束委派


在Windows Server 2012及之后，我们可以使用基于资源的约束委派。因为约束委派配置所需的特权很敏感，通常只有域管理员具有，使用基于资源的约束委派可以让资源配置受信任的账户委派给它们。


假设现在需要配置`Service A`到`Service B`的委派，传统约束委派是让域管理员等特权用户在`Service A`上进行配置，定义从A到B的“传出”信任；而基于资源的约束委派不同，是在`Service B`上进行配置，定义从A到B“传入”信任。


![](https://adan0s-1256533472.cos.ap-nanjing.myqcloud.com/typora/t01fe78dcd0b3af3aa4.jpg#id=PlPCv&originHeight=764&originWidth=1044&originalType=binary&ratio=1&status=done&style=none#id=YvDXi&originHeight=764&originWidth=1044&originalType=binary&ratio=1&status=done&style=none)


相较于传统的约束委派，基于资源的约束委派把信任决定权交给了资源，资源可以决定应该信任谁。

在流程上，基于资源的约束委派在`S4U2proxy`阶段，使用的`S4U2self`阶段获取的`Service1`的ST，即使不是可转发的，也不影响后面的使用。

只需要对`Service2`有修改权限就可以配置基于资源的约束委派，要修改的就是`Service2`的属性`msDS-AllowedToActOnBehalfOfOtherIdentity`值。
