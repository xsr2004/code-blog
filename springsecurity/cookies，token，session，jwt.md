# 为什么一些网站要询问您是否接受cookies

所以到底什么是cookie？token？session？

## cookie

> 网站Cookie是你正在访问的网站发送到你正在使用的设备上的小型文档，当你按下接受的时候，这些Cookie将会储存在设备的浏览器上，随后可以从浏览器追踪和收集你的使用数据，并且将该数据回传给网站经营者。这些数据上标记有你和你的计算机专有ID，网站服务器读取ID并分析保存下来的数据确定要为你提供哪些信息与服务



**为什么一些网站要询问您是否接受cookies**

> cookie可以保存：网站名称和唯一对话ID，浏览历史记录，首选项与权限，访问次数，对话持续时间，点击的链接，登录凭证（包含使用者名称和密码），地理标记（地理位置和IP地址），个人数据（如电话号码和邮政编码），购物车活动

因为有《GDPR 》法案限制，cookie被视为个人信息，网站服务方要获取时应该经过授权同意



**cookie实现登录基本原理**

http协议本身无状态，无法确定这一次请求和下一次请求是否“同一个人”

当客户端发送一个请求到服务器，服务端会生成一个httpSession，

然后给response装一个cookie（set-cookies），这个cookie一般是{"sessionId":xxx}

下一次客户端再发送请求时，可以把这个cookie加在request header里，服务端就可以识别该请求和上一次请求”是一个人“

缺点：

- **session存储在服务端的内存或者数据库，用户变多，压力很大**
- **cookie保存在客户端，由于登录等业务要求cookie长时间存留，如果被劫持，则全部信息可能被泄露**

**csrf跨站伪造**

比如csrf跨站伪造就是通过诱导用户访问钓鱼链接，从中获取浏览器存储的cookie，伪造用户身份

> 举个例子：
>
> 客户端和淘宝断开连接后，但是客户端保存着淘宝给的cookie，这个记录着客户端的信息，用于确认客户端本人，但是钓鱼网站，给客户端发链接，让它点击了该链接进入一个页面，这个钓鱼页面里面可能写了如下代码：
>
> 
>
> ```
> <img src='http://www.taobao.com/action?k1=v1&k2=v2' width=0 height=0 />
> ```
>
> 
>
> 这里width=0 height=0 表示图片是不可见的。这个语句会导致客户端浏览器器向另外的服务器（淘宝）发送一个请求。
>
> 
>
> 因为浏览器不管该图片url实际是否指向一张图片，只要src字段中规定了url，就会按照地址触发这个请求。（浏览器默认都是没有禁止下载图片，这是因为禁用图片后大多数web程序的可用性就会打折扣）。加载图片根本不考虑所涉及的图像所在位置（可以跨域）。如果A网站不小心提供了get接口就非常不幸得中招了
>
> 
>
> 意思是，客户端打开了钓鱼网站，却在不知情的情况下，发送了一个请求给淘宝，这个请求有可能是转账，付款等等，因为是客户端发出的请求，所以客户端会带上自己的cookie，这样就可以请求成功
>
> 转自：https://developer.aliyun.com/article/1105790#slide-2



## token

客户端登录通过后，服务器生成一堆客户端身份信息，包括用户名、用户组、有那些权限、过期时间等等。另外再对这些信息进行**签名**。

之后把**身份信息和签名**作为一个整体传给客户端。这个整体就叫做token。

之后，客户端负责保存该token，而服务器不再保存。

客户端每次访问该网站都要带上这个token。服务器收到请求后，把它**分割**成身份信息和签名，然后验证签名，若验证成功，就直接使用身份信息(用户名、用户组、有哪些权限等等)



**与cookie的比较**

优点：服务端不存东西，就保管一个密钥，压力小



## **JWT**

![image-20240518132503970](D:\picGo\images\image-20240518132503970.png)

 步骤翻译：

- 1、用户登录
- 2、服务的认证，通过后根据secret生成token
- 3、将生成的token返回给浏览器
- 4、用户每次请求携带token
- 5、服务端利用公钥解读jwt签名，判断签名有效后，从Payload中获取用户信息
- 6、处理请求，返回响应结果

> 转自：https://cloud.tencent.com/developer/article/2084586

jwt原理

https://en.wikipedia.org/wiki/JSON_Web_Token

> 主要就是Header，Payload，Signature
>
> header：使用哪种算法来生成签名（一般是HS256）
>
> Payload：存储一些用户信息，解码jwt后可以获取到。
>
> Signature：根据secret(自定义)，对header和payload加密运算（base64）得到一个字符串
>
> 如eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJsb2dnZWRJbkFzIjoiYWRtaW4iLCJpYXQiOjE0MjI3Nzk2Mzh9.gzSraSYS8EXBxLN_oWnFSRgCzcmJmMjLiuyu5CSpyHI



https://zhuanlan.zhihu.com/p/569770479

https://blog.csdn.net/MarcoAsensio/article/details/99223439