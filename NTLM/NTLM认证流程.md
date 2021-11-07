NTLM认证通常被用于Windows本地身份认证和网络身份认证，与之密切相关的就是哈希。本地用户的哈希存储在`C:/Windows/system32/config/sam`中，而域用户的哈希存储在`NTDS.dit`之中。

常说的`hash`一般指`NTLM hash`，还有`LM hash`、`Net-NTLM hash`等。`LM hash`因为强度不高、长度有限制等原因，目前已经基本被弃用，微软使用了强度更高的`NTLM hash`，`Windows Server 2003`之后的系统不再支持`LM hash`。

`Net-NTLM hash`是基于`NTLM hash`，再使用一定算法生成的，用于网络认证。

# 主要流程

`NTLM`认证流程主要分为三大步：

1. 协商：确认双方协议版本
2. 质询：挑战（Challenge）/响应（Response）认证机制
3. 验证：验证结果

将其放到域内，流程如下图：

![](https://adan0s-1256533472.cos.ap-nanjing.myqcloud.com/typora/image-20211107160343403.png)

1. `User`登录`Client Computer`
2. `Client Computer`向`Server`发送协商消息，确认支持的功能、版本等
3. `Server`发送质询消息进行响应，其中包含功能以及`Server`生成的`challenge`，`challenge`是16位的随机字符
4. `Client`发送验证消息响应质询，验证消息包含使用`challenge`加密的用户hash，即`response`
5. `Server`需要将`challenge`和用户hash加密生成的`response2`与得到的`response`进行比较。如果在域内环境，`Server`就需要使用`netlogon`协议请求`Domain Controller`，发送`username`、`challenge`和`response`
6. `Domain Controller`使用`challenge`和用户hash加密得到`response2`
7. `Domain Controller`比较`response`和`response2`，如果一致，代表验证成功

# 版本和等级

在验证消息中，包含了使用`challenge`和用户hash加密的`response`，`response`常见的类型有两种，即`NTLM v1`响应和`NTLM v2`响应.

前面提到了`Net-NTLM hash`，与之相对应的就是`Net-NTLM hash v1`和`Net-NTLM hash v2`，前者更容易被破解。

`LMCompatibilityLevel`即LAN身份验证级别，它决定了客户端所使用的身份验证协议等级以及服务端接收的身份验证协议等级等，配置项及含义如下：

![](https://adan0s-1256533472.cos.ap-nanjing.myqcloud.com/typora/image-20211107170452534.png)

- **发送 LM和NTLM 响应(&)**：客户端使用 LM 和 NTLM 身份验证，而决不会使用 NTLMv2 会话安全；域控制器接受 LM、NTLM 和 NTLMv2 身份验证

- **发送 LM & NTLM响应 - 如果协商一致，则使用 NTLMv2 会话安全(&)**：客户端使用 LM 和 NTLM 身份验证，并且在服务器支持时使用 NTLMv2 会话安全；域控制器接受 LM、NTLM 和 NTLMv2 身份验证

- **仅发送 NTLM 响应**：客户端仅使用 NTLM 身份验证，并且在服务器支持时使用 NTLMv2 会话安全；域控制器接受 LM、NTLM 和 NTLMv2 身份验证

- **仅发送 NTLMv2 响应**：客户端仅使用 NTLMv2 身份验证，并且在服务器支持时使用 NTLMv2 会话安全；域控制器接受 LM、NTLM 和 NTLMv2 身份验证

- **仅发送 NTLMv2 响应。拒绝 LM**：客户端仅使用 NTLMv2 身份验证，并且在服务器支持时使用 NTLMv2 会话安全；域控制器拒绝 LM (仅接受 NTLM 和 NTLMv2 身份验证)

- **仅发送 NTLMv2 响应。拒绝 LM & NTLM**：客户端仅使用 NTLMv2 身份验证，并且在服务器支持时使用 NTLMv2 会话安全；域控制器拒绝 LM 和 NTLM (仅接受 NTLMv2 身份验证)

默认情况下，此配置项为：

- Windows 2000、Windows XP：**发送 LM & NTLM响应 - 如果协商一致，则使用 NTLMv2 会话安全(&)**
- Windows Server 2003：**仅发送 NTLM 响应**
- Windows Vista、Windows Server 2008、Windows 7 以及 Windows Server 2008 R2及以上：**仅发送 NTLMv2 响应**

