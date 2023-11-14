# JWT 与 Session

## 用户认证流程

1. 用户向服务器发送用户名和密码；
2. 服务器验证通过后，在当前会话（Session）里面保存相关数据，比如用户角色、登录时间等；
3. 服务器向用户返回一个 Session ID，写入用户的 Cookie；
4. 用户随后的每一次请求，都会通过 Cookie，将 Session ID 发送到服务器；
5. 服务器收到 Session ID，找到前期保存的数据，由此得知用户的身份。

这种模式的问题在于，扩展性（scaling）不好。单机当然没有问题，如果是服务器集群，或者跨域的服务导向架构，就要求 Session 数据共享，每台服务器都能够读取 Session。

举例来说，A 网站和 B 网站是同一家公司的关联服务。现在要求，用户只要在其中一个网站登录，再访问另一个网站就会自动登录，请问怎么实现？

一种解决方案是 session 数据持久化，写入数据库或别的持久层。各种服务收到请求后，都向持久层请求数据。这种方案的优点是架构清晰，缺点是工程量比较大。另外，持久层万一挂了，就会单点失败。

另一种方案是服务器索性不保存 Session 数据了，所有数据都保存在客户端，每次请求都发回服务器。JWT 就是这种方案的一个代表。

## JWT 与 Session 的区别

**JSON Web Tokens (JWT)** 和 **Session** 是两种不同的认证和会话管理方法，它们在多种应用中被用于验证用户身份和管理用户会话。以下是它们之间的主要区别：

1. **存储位置**
   - **Session**: 通常存储在服务器上。当用户登录时，服务器创建一个会话，并将会话 ID 存储在用户的浏览器中的 cookie 里。每次用户发出请求时，都会发送这个会话 ID，服务器通过它来识别用户。
   - **JWT**: 存储在客户端（例如浏览器的 localStorage 中）。JWT 是一个包含用户身份信息的加密字符串，每次用户发出请求时，这个 token 会被发送到服务器以验证用户身份。
2. **状态性（Statefulness）**
   - **Session**: 是有状态的（stateful）。服务器需要存储会话数据来跟踪用户状态。
   - **JWT**: 是无状态的（stateless）。服务器不需要存储任何会话数据，因为所有需要的信息都存储在 token 本身中。
3. **扩展性**
   - **Session**: 在大规模分布式系统中，由于需要在服务器之间共享会话信息，因此可能会出现扩展性问题。
   - **JWT**: 更适合大规模分布式系统。由于它是无状态的，不需要在服务器之间共享会话信息，因此扩展起来更容易。
4. **安全性**
   - **Session**: 安全性依赖于服务器和用户之间的连接（如使用 HTTPS）。会话 ID 如果被截获，可能会导致会话劫持。
   - **JWT**: 本身包含加密信息，更难被篡改。但如果 token 被截获，可能会导致安全问题，因为它包含用户的身份验证信息。
5. **性能**
   - **Session**: 每次请求都需要服务器查找和读取会话数据，这可能影响性能。
   - **JWT**: 由于其无状态性质，不需要额外的服务器查询，这可能导致更快的响应时间。
6. **过期管理**:
   - **Session**: 服务器控制会话的过期时间，并可以随时使会话失效。
   - **JWT**: 过期时间嵌入在 token 中，一旦发出，无法更改，除非发出新的 token。

根据应用的具体需求和场景，选择适当的认证方法非常关键。例如，对于需要高安全性和可扩展性的应用，JWT 可能是更好的选择；而对于需要精细会话控制的应用，使用 Session 可能更合适。

## JWT 原理

JWT 的原理很简单，就是服务器认证以后，生成一个 JSON 对象，发回给用户，就像下面这样。

```json
{
  "姓名": "张三",
  "角色": "管理员",
  "到期时间": "2018年7月1日0点0分"
}
```

以后，用户与服务端通信的时候，都要发回这个 JSON 对象。服务器完全只靠这个对象认定用户身份。为了防止用户篡改数据，服务器在生成这个对象的时候，会加上签名（signature）。

### JWT 的三个部分依次如下

- Header（头部）
  - 是一个 JSON 对象
    - alg 属性表示签名的算法（algorithm），默认是 HMAC SHA256 写成（HS256）；
    - typ 属性表示这个令牌（token）的类型（type），JWT 令牌统一写成 JWT。
  - 使用 Base64URL 算法转成字符串，拼成 JWT 的第一部分。
- Payload（负载）
  - 也是一个 JSON 对象，用来存放实际需要传递的数据。
    - iss（issuer）：签发人
    - exp（expiration time）：过期时间
    - sub（subject）：主题
    - aud（audience）：受众
    - nbf（Not Before）：生效时间
    - iat（Issued At）：签发时间
    - jti（JWT ID）：编号
  - 除了以上 7 个官方字段，还可以自定义字段。
  - 注意，JWT 默认是不加密的，任何人都可以读到，所以不要把秘密信息放在这个部分。
  - 使用 Base64URL 算法转成字符串。
- Signature（签名）
  - 是对前两部分的签名，防止数据篡改。
  - 这个密钥（secret）只有服务器才知道，不能泄露给用户。

```Json
    // header
    {
        "alg": "HS256",
        "typ": "JWT"
    }
    // payload
    {
        "sub": "1234567890",
        "name": "John Doe",
        "admin": true
    }
    // secret
    your-256-bit-secret

    HMACSHA256(base64UrlEncode(header) + "." + base64UrlEncode(payload),secret)
```

上面代码中，HMACSHA256 是一个哈希算法，需要服务器保存的密钥（secret）参与计算。签名的目的，是为了保证数据不会被篡改。

```json
    eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
    eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6Ikpva
    G4gRG9lIiwiYWRtaW4iOnRydWV9.
    TJVA95OrM7E2cBab30RMHrHDcEfxjoYZgeFONFh7HgQ
```

上面这个字符串就是 JSON Web Token 的原始形式，也是它最基本的构成，这三个部分拼成一个长字符串，每个部分之间用"点"（`.`）分隔，注意，JWT 内部是没有换行的，这里只是为了便于展示，将它写成了几行。

在 JWT 中，消息体是透明的，使用签名可以保证消息不被篡改。但不能实现数据加密功能。

### Base64URL

前面提到，Header 和 Payload 串行化的算法是 Base64URL。这个算法跟 Base64 算法基本类似，但有一些小的不同。

JWT 作为一个令牌（token），有些场合可能会放到 URL （比如 api.example.com/?token=xxx）。Base64 有三个字符`+`、`/`和`=`，在 URL 里面有特殊含义，所以要被替换掉：`=`被省略、`+`替换成`-`、`/`替换成`_`。这就是 Base64URL 算法。

[jwt 调试网站](https://jwt.io)

### JWT 的使用方式

客户端收到服务器返回的 JWT，可以储存在 Cookie 里面，也可以储存在 localStorage。

此后，客户端每次与服务器通信，都要带上这个 JWT。你可以把它放在 Cookie 里面自动发送，但是这样不能跨域，所以更好的做法是放在 HTTP 请求的头信息 Authorization 字段里面。

```js
    Authorization: Bearer <token>
```

另一种做法是，跨域的时候，JWT 就放在 POST 请求的数据体里面。

### JWT 的几个特点

1. JWT 默认是不加密，但也是可以加密的。生成原始 Token 以后，可以用密钥再加密一次。
2. JWT 不加密的情况下，不能将秘密数据写入 JWT。
3. JWT 的最大缺点是，由于服务器不保存 session 状态，因此无法在使用过程废止某个 token，或者更改 token 的权限。也就是说，一旦 JWT 签发了，在到期之前就会始终有效，除非服务器部署额外的逻辑。
4. JWT 本身包含认证信息，一旦泄露，任何人都可以获得该令牌的所有权限。为了减少盗用，JWT 的有效期不宜设置太长。对于某些重要操作，用户在使用时应该每次都进行进行身份验证。
5. 为了减少盗用，JWT 不应该使用 HTTP 协议明码传输，要使用 HTTPS 协议传输。

### JWT 的解决方案

[https://jwt.io/](https://jwt.io/)

### 在 Node.js 中使用 JWT

推荐：[https://github.com/auth0/node-jsonwebtoken](https://github.com/auth0/node-jsonwebtoken)
