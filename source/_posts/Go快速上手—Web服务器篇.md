---
title: Go快速上手—Web服务器篇
date: 2021-05-05 22:14:58
tags: [go]	
categories: 技术
---

Go的一个比较流行的HTTP后台框架是Gin，若要搭建Go的HTTP后台，推荐直接学习Gin。

## Gin简介

Gin 特性
- 快速：路由不使用反射，基于Radix树，内存占用少。
- 中间件：HTTP请求，可先经过一系列中间件处理，例如：Logger，Authorization，GZIP等。这个特性和 NodeJs 的 Koa 框架很像。中间件机制也极大地提高了框架的可扩展性。
- 异常处理：服务始终可用，不会宕机。Gin 可以捕获 panic，并恢复。而且有极为便利的机制处理HTTP请求过程中发生的错误。
- JSON：Gin可以解析并验证请求的JSON。这个特性对Restful API的开发尤其有用。
- 路由分组：例如将需要授权和不需要授权的API分组，不同版本的API分组。而且分组可嵌套，且性能不受影响。
- 渲染内置：原生支持JSON，XML和HTML的渲染。

## 安装

1. go env -w GO111MODULE=on
2. go env -w GOPROXY=https://goproxy.io,direct
3. 设置后，重新运行： go get -u github.com/gin-gonic/gin，
4. 在项目文件夹下运行 go mod init gin
5. 在项目文件夹下运行 go mod edit -require github.com/gin-gonic/gin@latest

## 一个最基本的web框架实践
一个最简单的web框架，实现了get和post两种方式，幷绑定了端口9999进行监听。实现的基本功能：
1. 绑定指定端口9999进行监听
2. 实现了get和post两种方式
3. 实现了路由解析，不同的路由会由对用的函数进行处理请求
4. 实现了解析请求格式为query string的请求
5. 实现了解析参数格式为json的请求
6. 实现了json响应的回复

**路由方法有 GET, POST, PUT, PATCH, DELETE 和 OPTIONS，还有Any，可匹配以上任意类型的请求。**
```go

package main

import (
	"github.com/gin-gonic/gin"
	"net/http"
	"fmt"
)

func HelloWeb(c *gin.Context) {
	c.String(http.StatusOK, "Hello, Go\n")
}

func HiWeb(c *gin.Context) {
	c.String(http.StatusOK, "Hi, Go\n")
}

// 解析query string， 匹配users?name=xxx&role=xxx&age=xx，role可选
func QueryUser(c *gin.Context) {
	name := c.Query("name")
	age := c.Query("age")
	role := c.DefaultQuery("role", "teacher")
	resp := fmt.Sprintf("my name is %s, my age is %s, my role is %s\n", name, age, role)
	c.String(http.StatusOK, resp)
}

type Login struct {
	User string `json:"user"`
	Password string `json:"password"`
}

// 解析json格式的请求
func LoginCheck(c *gin.Context) {
	var req Login
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}
	resp := gin.H{"message": "someJSON", "status": 200}
	c.JSON(http.StatusOK, resp)
}

 type Response struct {
	Name    string
	Message string
	Status  int
}

func LoginCheck2(c *gin.Context) {
	var req Login
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}
	var resp Response
	resp.Name = req.User
	resp.Message = "OKOK"
	resp.Status = 200
	c.JSON(http.StatusOK, resp)
}

func main() {
	r := gin.Default()
	r.GET("/hello", func(c *gin.Context) {
		HelloWeb(c)
	})
	r.POST("/hi", func(c *gin.Context) {
		HiWeb(c)
	})
	r.GET("/user", func(c *gin.Context) {
		QueryUser(c)
	})
	r.POST("/login", func(c *gin.Context) {
		LoginCheck(c)
	})
	r.POST("/login2", func(c *gin.Context) {
		LoginCheck2(c)
	})
	r.Run(":9999") //  listen and serve on 0.0.0.0:9999
}
```

访问对应的URL
```shell
junshideMacBook-Pro:~ junshili$ curl -X POST http://localhost:9999/hi
Hi, Go
junshideMacBook-Pro:~ junshili$ curl http://localhost:9999/hello
Hello, Go

junshideMacBook-Pro:~ junshili$ curl "http://localhost:9999/user?name=James&age=19&role=student"
my name is James, my age is 19, my role is student

curl -X POST http://localhost:9999/login -d '{"user":"kk", "password":"123"}' -H "content-type:application/json"
{"message":"someJSON","status":200}

junshideMacBook-Pro:~ junshili$ curl -X POST http://localhost:9999/login2 -d '{"user":"kk", "password":"123"}' -H "content-type:application/json"
{"Name":"kk","Message":"OKOK","Status":200}
```

## 一个带有复杂路由项目实践


当项目大到一定程度后，上面再main.go里写处理函数的方式已经不再适用，一个更合适的方法是把处理函数迁移到一个文件里单独写逻辑，而main.go里越简单越好，只做路由选择，不做业务函数的实现。

因此，项目按照该树结构来组织，routers放各个业务代码的实现，一个文件对应一个子业务，比如这里的login和user。main.go里不再写业务逻辑，唯一的功能就是做路由注册（调用SetupUserRouter）。这样一来，复杂的web项目也能清晰管理了。

main.go
```go
package main

import (
    "fmt"
    "web_demo/routers"
	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()
    routers.SetupUserRouter(r)
	routers.SetupLoginRouter(r)
    if err := r.Run(":9999"); err != nil {
        fmt.Println("startup service failed, err:%v\n", err)
    }
}
```

user.go
```go
package routers

import (
	"net/http"
	"github.com/gin-gonic/gin"
	"fmt"
)

// 解析query string， 匹配users?name=xxx&role=xxx&age=xx，role可选
func QueryUser(c *gin.Context) {
	name := c.Query("name")
	age := c.Query("age")
	role := c.DefaultQuery("role", "teacher")
	resp := fmt.Sprintf("my name is %s, my age is %s, my role is %s\n", name, age, role)
	c.String(http.StatusOK, resp)
}

func SetupUserRouter(e *gin.Engine) {
	e.GET("/users", QueryUser)
}
```


login.go
```go
package routers

import (
	"net/http"
	"github.com/gin-gonic/gin"
)
type Login struct {
	User string `json:"user"`
	Password string `json:"password"`
}

// 解析json格式的请求
func LoginCheck(c *gin.Context) {
	var req Login
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}
	resp := gin.H{"message": "someJSON", "status": 200}
	c.JSON(http.StatusOK, resp)
}

func SetupLoginRouter(e *gin.Engine) {
	e.POST("/login", LoginCheck)
}
```

请求和响应
```shell
junshideMacBook-Pro:web junshili$ curl "http://localhost:9999/users?name=James&age=19&role=student"
my name is James, my age is 19, my role is student

junshideMacBook-Pro:web junshili$ curl -X POST http://localhost:9999/login -d '{"user":"kk", "password":"123"}' -H "content-type:application/json"
{"message":"someJSON","status":200}
```

## 分组路由
如果有一组路由，前缀都是/api/v1开头，是否每个路由都需要加上/api/v1这个前缀呢？答案是不需要，分组路由可以解决这个问题。利用分组路由还可以更好地实现权限控制，例如将需要登录鉴权的路由放到同一分组中去，简化权限控制。

比如我们改写login.go，请求URL调整为login/logincheck2和login/logincheck这两个路径，此时我们使用Group函数即可。

```go
package routers

import (
	"net/http"
	"github.com/gin-gonic/gin"
)
type Login struct {
	User string `json:"user"`
	Password string `json:"password"`
}

// 解析json格式的请求
func LoginCheck(c *gin.Context) {
	var req Login
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}
	resp := gin.H{"message": "someJSON", "status": 200}
	c.JSON(http.StatusOK, resp)
}

// 解析json格式的请求
func LoginCheck2(c *gin.Context) {
	var req Login
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}
	resp := gin.H{"message": "someJSON2", "status": 200}
	c.JSON(http.StatusOK, resp)
}

func SetupLoginRouter(e *gin.Engine) {
	v1 := e.Group("/login")
	v1.POST("/logincheck", LoginCheck)
	v1.POST("/logincheck2", LoginCheck2)
}
```
请求和响应
```shell
junshideMacBook-Pro:web junshili$ curl -X POST http://localhost:9999/login/logincheck2 -d '{"user":"kk", "password":"123"}' -H "content-type:application/json"
{"message":"someJSON2","status":200}
```

## 重定向

http 关于重定向的状态码：301，302

- 301 redirect: 301 代表永久性转移(Permanently Moved)
- 302 redirect: 302 代表暂时性转移(Temporarily Moved )
- 301表示旧地址A的资源已经被永久地移除了（这个资源不可访问了），搜索引擎在抓取新内容的同时也将旧的网址交换为重定向之后的网址；
- 302表示旧地址A的资源还在（仍然可以访问），这个重定向只是临时地从旧地址A跳转到地址B，搜索引擎会抓取新的内容而保存旧的网址。

修改users.go，实现永久重定向
```go
func SetupUserRouter(e *gin.Engine) {
	v1 := e.Group("/users")
	v1.GET("/user", QueryUser)

	v1.GET("/redirect", func(c *gin.Context) {
		c.Redirect(http.StatusMovedPermanently, "/users")
	})

}
```

请求和响应，通过响应可以看到，该url已经被重定向到/index了，请求发起者可以根据这个回复重新调整自己的请求路径。
```shell
junshideMacBook-Pro:web junshili$ curl "http://localhost:9999/users/redirect?name=James&age=19&role=student"
<a href="/index">Moved Permanently</a>.
```

## 同步异步

- goroutine机制可以方便地实现异步处理
- 另外，在启动新的goroutine时，不应该使用原始上下文，必须使用它的只读副本

考虑这样的场景：客户端请求web服务器进行一个复杂计算，这个计算任务耗时比较久，因此一个比较好的做法是异步处理，客户端只是提交计算任务，服务器收到任务后把任务存储进消息队列，就直接回复客户端任务已收到，断开本次连接，待消息队列的任务被处理完后，再主动发起请求通知客户端。这个就是一个典型的异步处理的场景。

```go
package main

import (
    "fmt"
	"github.com/gin-gonic/gin"
	"time"
	"net/http"
)


func main() {
	r := gin.Default()
	// 同步
	r.GET("/sync", func(c *gin.Context) {
		time.Sleep(3 * time.Second)
		fmt.Println("同步执行：", c.Request.URL.Path)
		c.String(http.StatusOK, "同步执行")
	})
	// 异步
    r.GET("/async", func(c *gin.Context) {
        // 需要搞一个副本
        copyContext := c.Copy()
        // 异步处理
        go func() {
            time.Sleep(3 * time.Second)
            fmt.Println("异步执行：" + copyContext.Request.URL.Path)
        }()
		c.String(http.StatusOK, "异步执行")
    })
	r.Run() //  listen and serve on 0.0.0.0:8080
}
```

请求和响应，可以看出异步请求是秒回的，而同步请求花费了3s。
```shell
junshideMacBook-Pro:web junshili$ time curl "http://localhost:8080/async"
异步执行
real	0m0.017s
user	0m0.007s
sys	0m0.007s
junshideMacBook-Pro:web junshili$ time curl "http://localhost:8080/sync"
同步执行
real	0m3.021s
user	0m0.008s
sys	0m0.008s

```

## token令牌
Web服务器中身份验证是个重要的功能，比如app用户需要先登录才能操作一些内部功能。其中令牌身份验证是个常用的手段。

JSON Web令牌（JWT）作为令牌系统而不是在每次请求时都发送用户名和密码，因此比其他方法（如基本身份验证）具有固有的优势。JWT主要有两个部分：提供用户名和密码以获取令牌；并根据请求检查该令牌。

jwt由以下三部分构成：
- Header:头部 （对应：Header）
- Claims:声明 (对应：Payload)
- Signature:签名 (对应：Signature)

![image](./1.png)

### Header头部
Header中指明jwt的签名算法，如
```
{
  "typ": "JWT",
  "alg": "HS256"
}
```

### Claims声明

声明中有jwt自身预置的，使用时可选。当然，我们也可以加入自定义的声明，
如下面例子中的Claims的UserId信息，但一定不要声明重要或私密的信息，因为这些信息是可破解的。

### Signature签名

在生成jwt的token（令牌的意思）串时，先将Header和Claims用base64编码,再用Header中指定的加密算法，
将编码后的2个字符串进行加密（签名）,作用是防止数据篡改。加密时需要用到一个signString签名串(例子中的jwtkey)，我们可指定自己的signString，
不同的signString生成的加密结果不一样（解密时可能也需要同样的串，视加密算法而定）。签名部分主要和token的安全性有关，Signature的生成依赖前面两部分。
首先将Base64编码后的Header和Payload用.连接在一起，

令牌的用法一般如下：
1. 客户端没有令牌时（第一次登陆或者令牌超时失效了）需要先请求生成令牌，此时可能需要玩家输入账号密码给服务器校验
2. 账号密码校验正确后，服务器会生成token返回给客户端
3. 后续客户端访问服务器只需要在header上带上token即可，无需再输入账号密码
4. 服务器从token解析出该token对应的玩家uid，即验证了玩家身份，使用该uid继续后面的逻辑处理。

```go
package main

import (
    "fmt"
    "net/http"
    "time"

    "github.com/dgrijalva/jwt-go"
    "github.com/gin-gonic/gin"
)

var jwtkey = []byte("lijunshi2015@163.com")  // 这个秘钥需要跟代码分开存储，这样才符合安全规范

type Claims struct {
    UserId string
    jwt.StandardClaims
}

func main() {
	r := gin.Default()
	r.GET("/get_token", genToken)
	r.GET("/check_token", checkToken)
	r.Run(":8080")
}

// 颁发token,请求参数为?uid=xxx
func genToken(ctx *gin.Context) {
	uid := ctx.Query("uid")
	expireTime := time.Now().Add(7 * 24 * time.Hour) // 24小时后token过期
	claims := &Claims {
		UserId : uid,
		StandardClaims: jwt.StandardClaims {
			ExpiresAt: expireTime.Unix(), // 过期时间
			IssuedAt: time.Now().Unix(), // 颁发时间
			Issuer: "127.0.0.1",  // token颁发者
			Subject: "user token",  // token主题
		},
	}
	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	fmt.Println(token)
	tokenString, err := token.SignedString(jwtkey)
	if err != nil {
		fmt.Println(err)
	}
	ctx.JSON(200, gin.H{"token": tokenString})
}


// 验证token
func checkToken(ctx *gin.Context) {
	tokenString := ctx.GetHeader("Authorization")  // 从请求头获取token
	if tokenString == "" {
		ctx.JSON(http.StatusUnauthorized, gin.H{"code": 401, "msg": "权限不足"})
        ctx.Abort()
        return
	}

	// 通过token解析出是哪个玩家(UserId)
	token, claims, err := parseToken(tokenString)
	if err != nil || !token.Valid {
		ctx.JSON(http.StatusUnauthorized, gin.H{"code": 401, "msg": "权限不足"})
        ctx.Abort()
        return
	}
	fmt.Println("token valid, uid:", claims.UserId)
	ctx.JSON(http.StatusUnauthorized, gin.H{"code": 200, "uid": claims.UserId})
}

func parseToken(tokenString string) (*jwt.Token, *Claims, error) {
	Claims := &Claims{}
	token, err := jwt.ParseWithClaims(tokenString, Claims, func(token *jwt.Token) (i interface{}, err error) {
        return jwtkey, nil
    })
    return token, Claims, err
}
```
没有token时，带上自己的uid请求生成token
```shell
junshideMacBook-Pro:web junshili$ curl "http://localhost:8080/get_token?uid=88998899"
{"token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJVc2VySWQiOiI4ODk5ODg5OSIsImV4cCI6MTYyMDU2Mzk5MywiaWF0IjoxNjE5OTU5MTkzLCJpc3MiOiIxMjcuMC4wLjEiLCJzdWIiOiJ1c2VyIHRva2VuIn0.-78Bw3jHNDZyGQgZPWbEfUTodPRIy9PlD0rVuoUO6ks"}
```
有token时请求头带上token，服务器从token解析出uid，完成身份验证。
```shell
junshideMacBook-Pro:web junshili$ curl "http://localhost:8080/check_token" -H "Authorization:eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJVc2VySWQiOiI4ODk5ODg5OSIsImV4cCI6MTYyMDU2Mzk5MywiaWF0IjoxNjE5OTU5MTkzLCJpc3MiOiIxMjcuMC4wLjEiLCJzdWIiOiJ1c2VyIHRva2VuIn0.-78Bw3jHNDZyGQgZPWbEfUTodPRIy9PlD0rVuoUO6ks"
{"code":200,"uid":"88998899"}
```

## 防篡改，防重放

篡改：API参数篡改就是恶意人通过抓包的方式获取到请求的接口的参数，通过修改相关的参数，达到欺骗服务器的目的，常用的防止篡改的方式是用签名以及加密的方式。

重放：API重放攻击就是把之前窃听到的数据原封不动的重新发送给接收方.

解决方案：
- 防篡改：签名
- 防重放：timestamp时间戳 + nonce随机数


timestamp的作用： 每次HTTP请求，都需要加上timestamp参数，然后把timestamp和其他参数一起进行数字签名。HTTP请求从发出到达服务器一般都不会超过60s，所以服务器收到HTTP请求之后，首先判断时间戳参数与当前时间相比较，是否超过了60s，如果超过了则认为是非法的请求。

nonce的作用： 每次HTTP请求，都需要加上nonce随机数。我们将每次请求的nonce参数存储到一个“集合”中，服务器每次处理HTTP请求时，首先判断该请求的nonce参数是否在该“集合”中，如果存在则认为是非法请求。nonce参数在首次请求时，已经被存储到了服务器上的“集合”中，再次发送请求会被识别并拒绝。

**nonce的一次性可以解决timestamp参数60s(防止重放攻击)的问题，timestamp可以解决nonce参数“集合”越来越大的问题。=**

### 两种常用的请求签名的方式
1. Md5(url+key) 的方式进行，URL+Key字符串拼接后的值用MD5加密生成签名，将签名发送到服务器端，同时服务器端已同样的方式计算出签名，然后比较俩个MD5的值是否相同，来确定URL是否被篡改。
2. AES 对称加密，使用URL和秘钥进行加密。

**MD5组合加密解密**

```
appKey     = "mhxy"
appSecret  = "xxx"
encryptStr = "param_1=xxx&param_2=xxx&ak="+appKey+"&ts=xxx"+"nonce=xxx"

// 自定义验证规则
sn = MD5(appSecret + encryptStr)
```
加密解密都是同一套流程，验证客户端发过来的sn与自己服务器计算的sn是否一致即可。


**AES 对称加密**
```
appKey     = "mhxy"
appSecret  = "xxx"
encryptStr = "param_1=xxx&param_2=xxx&ak="+appKey+"&ts=xxx"+"nonce=xxx"

sn = AesEncrypt(encryptStr, appSecret)
```

解密：
```
decryptStr = AesDecrypt(sn, app_secret)
```

将加密前的字符串与解密后的字符串做个对比。相同，表示签名验证成功。


利用python生成随机秘钥：
```python
>>> import base64
>>> import os
>>> a = os.urandom(24) 
>>> base64.b64encode(a)
b'eUxqsXD/FkNlMR6nIpGvQh8MVlrNTsP4'
```

### 利用MD5做请求签名的例子

这个请求验证的例子做了以下的请求验证：

1. 时间戳验证，时间过期或明显不合理的不能通过验证；
2. appKey验证，不是注册业务的请求不能通过验证；
3. 指定时间内收到相同的随机数的请求，不能通过验证，因为有可能是请求被重放了；
4. 验证签名是否一致，验证请求参数是否被篡改；

```go
package main

import (
    "fmt"
    "time"
    "github.com/gin-gonic/gin"
	"crypto/md5"
	"encoding/hex"
	"errors"
	"strconv"
	"math/rand"
	"net/http"
)

var secretKey string = "eUxqsXD/FkNlMR6nIpGvQh8MVlrNTsP4" // 安全规范要求秘钥跟代码要分开存储
var reqMap map[string]map[string]int64  // 一般都是放在redis，用expire控制key的存活时间
var appRegist = map[string]int {
	"mhxy" : 1,
	"lol" : 1,
}

var expireTime int64 = 600


func MD5(str string) string {   
	s := md5.New()   
	s.Write([]byte(str))   
	return hex.EncodeToString(s.Sum(nil))   
}

// 验证签名
func md5VerifySign(c *gin.Context) error {
	ak := c.Query("ak")  // appKey，业务标记
	sn := c.Query("sn")  // sign，签名
	ts := c.Query("ts")  // timestamp，时间戳
	nonce := c.Query("nonce") // 随机数

	name := c.Query("name")
	uid := c.Query("uid")
	pay := c.Query("pay")

	now := time.Now().Unix()

	// 验证appkey是否已注册
	if _, ok := appRegist[ak]; !ok {
		return errors.New("appkey error")
	}

	// 验证过期时间
	tsInt, _ := strconv.ParseInt(ts, 10, 64)
	if tsInt > now || now - tsInt > expireTime {
		return errors.New("ts error")
	}

	//验证随机数是否重复
	err := getNonce(uid, nonce)
	if err != nil {
		return err
	}

	//验证签名
	// url param需要排序
	urlParmString := fmt.Sprintf("ak=%s&name=%s&nonce=%s&pay=%s&ts=%s&uid=%s",ak, name, nonce, pay, ts, uid)
	fmt.Println("urlParmString:", urlParmString)
	if sn == "" || sn != createMd5Sign(urlParmString) {
		return errors.New("sn error")
	}
	setNonce(uid, nonce)
	return nil
}

// 创建签名
func createMd5Sign(str string) string {
	// 自定义 MD5 组合
	return MD5(secretKey + str)
}

func setNonce(uid string, nonce string) {
	now := time.Now().Unix()
	c := make(map[string]int64)
	c[nonce] = now
	if _, ok := reqMap[uid]; !ok {
		reqMap[uid] = make(map[string]int64)
	}
	reqMap[uid] = c
}

func getNonce(uid string, nonce string) error {
	now := time.Now().Unix()
	 if _, ok := reqMap[uid]; !ok {
		 return nil
	 }

	if _, ok := reqMap[uid][nonce]; !ok {
		return nil
	}

	if reqMap[uid][nonce] > 0 && now - reqMap[uid][nonce] > expireTime {
		delete(reqMap[uid], nonce)
		return nil
	}
	return errors.New("nonce err")
}

func genSign(c *gin.Context) string {
	ak := c.Query("ak")  // appKey，业务标记
	ts := time.Now().Unix()  // timestamp，时间戳
	nonce := rand.Intn(100) // 随机数

	name := c.Query("name")
	uid := c.Query("uid")
	pay := c.Query("pay")

	urlParmString := fmt.Sprintf("ak=%s&name=%s&nonce=%d&pay=%s&ts=%d&uid=%s",ak, name, nonce, pay, ts, uid)
	sn := createMd5Sign(urlParmString)
	return fmt.Sprintf("%s&sn=%s", urlParmString,sn)
}

func main() {
	reqMap = map[string]map[string]int64{}
	r := gin.Default()
	r.GET("/gen_sign", func(c *gin.Context) {
		sign := genSign(c)
		fmt.Println("生成签名：", sign)
		c.String(http.StatusOK, sign)
	})

    r.GET("/check_sign", func(c *gin.Context) {
		err := md5VerifySign(c)
		if err != nil {
			fmt.Println("验证签名失败：", err)
			c.String(http.StatusOK, fmt.Sprintf("验证签名失败, 原因：%+v", err))
			return
		}
		fmt.Println("验证签名成功！")
		c.String(http.StatusOK, "验证签名成功")
    })
	r.Run() //  listen and serve on 0.0.0.0:8080
}


```

### 实验过程

请求gen_sign获得签名和拼接好的url param请求串
```shell
junshideMacBook-Pro:web junshili$ curl "http://localhost:8080/gen_sign?ak=mhxy&name=james&pay=100&uid=9988998" 
ak=mhxy&name=james&nonce=81&pay=100&ts=1620020732&uid=9988998&sn=48247171223ff1b4739ca300e3b7d32djunshide

```

如果我们篡改请求串然后再去请求签名验证，比如pay字段我们改为1000，会提示验证失败，原因是签名对不上失败了
```shell
junshideMacBook-Pro:web junshili$ curl "http://localhost:8080/check_sign?ak=mhxy&name=james&nonce=81&pay=1000&ts=1620020732&uid=9988998&sn=48247171223ff1b4739ca300e3b7d32d" 
验证签名失败, 原因：sn error
```


如果我们不修改请求串，直接请求验证签名，签名验证成功
```shell
junshideMacBook-Pro:web junshili$ curl "http://localhost:8080/check_sign?ak=mhxy&name=james&nonce=81&pay=100&ts=1620020732&uid=9988998&sn=48247171223ff1b4739ca300e3b7d32d" 
验证签名成功
```

我们重放这个请求，提示签名失败，原因是随机数重复了
```shell
junshideMacBook-Pro:web junshili$ curl "http://localhost:8080/check_sign?ak=mhxy&name=james&nonce=81&pay=100&ts=1620020732&uid=9988998&sn=48247171223ff1b4739ca300e3b7d32d" 
验证签名失败, 原因：nonce err
```