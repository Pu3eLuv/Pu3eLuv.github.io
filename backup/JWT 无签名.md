# JWT none算法绕过认证
JWT “none” 算法漏洞详解  
标签：`JWT` `安全` `漏洞分析` `Token` `Web`
由G老师扩写排版

---

## 目录

- 1. 何为 “none” 算法漏洞
        
- 2. JWT 的结构回顾（Header / Payload / Signature）
        
- 3. 漏洞形成的原因
        
- 4. 漏洞复现示例
        
    
    - 4.1 原始 JWT
        
    - 4.2 修改 Header 为 `alg=none`
        
    - 4.3 构造无签名的 Token
        
    - 4.4 完整 none-token 示例
        
- 5. 为什么这一攻击有效
        
- 6. 可利用的攻击点（权限提升示例）
        
- 7. 如何彻底修复该漏洞
        
- 8. 小结
        

---

# 1. 何为 “none” 算法漏洞

JWT 支持多种签名算法，如：

- HS256（对称 HMAC）
    
- RS256（非对称 RSA）
    

然而早期实现允许使用一种特殊算法：  
**alg = "none"**  
代表：**JWT 不进行任何签名验证**。  
这原本只用于内部测试环境，但许多后端库错误地对其进行了支持。

攻击者可通过将 JWT 的 Header 修改为：  
```json
{ 
"alg": "none"
 }  
```
从而绕过后端的签名校验，伪造任意合法 Token。

这是 JWT 最著名的历史漏洞之一。

---

# 2. JWT 的结构回顾

JWT 由三部分组成：  
Header.Payload.Signature

例如：  
aaaa.bbbb.cccc

- **Header**：声明算法，如 HS256
    
- **Payload**：主体数据，如用户 ID、权限等
    
- **Signature**：用于验证 Token 是否被篡改
    

当 `alg = "none"` 时，Signature 理论上应为空。

---

# 3. 漏洞形成的原因

漏洞出现的核心原因是：

> 某些后端框架对 `alg=none` 的处理不严格，认为“既然算法是 none，那就不需要验证签名”。

错误的实现逻辑可能是：

`if (header.alg === "none") {     // 不做签名验证     return payload; }`

攻击者利用这一点：

- 将 Header 改成 `{"alg":"none"}`
    
- 伪造任意 Payload
    
- 直接让后端认为 Token 是合法的
    

只要系统使用了脆弱的 JWT 库，这一攻击就能成功。

---

# 4. 漏洞复现示例

## 4.1 原始 JWT

例如以下合法 Token：

`eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWUsImlhdCI6MTUxNjIzOTAyMn0.KMUFsIDTnFmyG3nMiGM6H9FNFUROf3wh7SmqJp-QV30`

Header 解码：

```json
{
	"alg": "HS256",
	"typ": "JWT" 
}
```

Payload 解码：

```json
{
   "sub": "1234567890",
   "name": "John Doe",
   "admin": true,
   "iat": 1516239022 
}
```

---

## 4.2 将 Header 修改为 “none”

替换为：

```json
{   
	"alg": "none",   
	"typ": "JWT"
}
```

进行 Base64URL 编码：

`ewogICJhbGciOiAibm9uZSIsCiAgInR5cCI6ICJKV1QiCn0=`

---

## 4.3 构造无签名 Token

none 算法不需要签名，但 **第三段（Signature）后的点号必须保留**：

`header_base64.payload_base64.`

---

## 4.4 完整 none Token 示例

`ewogICJhbGciOiAibm9uZSIsCiAgInR5cCI6ICJKV1QiCn0=.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiYWRtaW4iOnRydWUsImlhdCI6MTUxNjIzOTAyMn0.`

不论 Payload 是否原样，只要服务端接受 `alg=none` 就能绕过验证。

---

# 5. 为什么这一攻击有效

攻击成功的原因包括：

1. **后端错误地信任 Header 中的 alg 字段**
    
2. **未检查签名算法是否与配置一致**
    
3. **部分旧库默认允许 none**
    
4. **后端没有强制要求签名**
    

一旦攻击者能构造 none Token，服务端完全不验证其合法性。

---

# 6. 可利用的攻击点（权限提升示例）

你可以在 Payload 中自行修改字段，例如：

将

`"admin": false`

改成：

`"admin": true`

构造出的 none Token：

`header_none.payload_modified.`

在脆弱系统中：  
**任何用户都可伪造成为管理员**。

攻击者也可伪造：

- 任意用户 ID
    
- 高权限角色
    
- 注册/付费状态
    
- API 访问范围
    

风险极其严重。

---

# 7. 如何彻底修复该漏洞

### 必须禁止 “none” 算法

绝大多数现代 JWT 库已经默认禁用，但仍需确保：

- 明确指定允许的算法
    
- 不接受客户端提供的算法字段
    

例如后端必须配置：

`acceptableAlgorithms = ["HS256", "RS256"]`

拒绝任何不在白名单的 Token。

---

### 不信任 Header 中的 alg 字段

正确做法：  
**由服务器自身配置算法，而不是根据 Token Header 决定。**

避免逻辑：

`verify(token, {alg: token.header.alg})`

---

### 使用成熟、安全的 JWT 库

避免自己实现验证逻辑。  
选择：

- jose
    
- jsonwebtoken（最新版，不要用旧版本）
    
- go-jose
    
- pyjwt（注意禁用 none）
    

---

### 对 Token 添加过期时间

所有 JWT **必须**包含 `exp` 字段，避免长期有效的伪造 Token。

---

# 8. 小结

- “none 算法漏洞”是 JWT 早期实现中的重大漏洞。
    
- 攻击者可通过将 Header 改为 `alg=none` 来绕过签名验证。
    
- 可伪造任意 Payload，从而实现权限提升等攻击。
    
- 漏洞根因是：后端错误地信任客户端提供的签名算法。
    
- 最佳修复方案：**禁用 none、强制使用服务器配置的算法、升级 JWT 库**。