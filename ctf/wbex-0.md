题目拿到手以后是一个登录界面,提示输入用户名与密码
先试探一下 ,随便输一个试试
并没有发生任何事...
抓包一下,发现我们的输入根本没有被发送出去!?
尝试一下自己用post方法发送登录请求,也没有结果

那还剩的突破口就只有用dirsearch扫一下看看有没有什么目录了
但在扫目录之前可以自己手动看看一些常见目录,比如我们的`/robots.txt`
简单了解一下这个目录只干什么的:
用来告知搜索引擎和爬虫**如何**抓取网络的界面 -- 一个指示文件,告诉爬虫哪些页面是被允许访问的

好了那都说到这步了,robots.txt肯定是有东西的
```
User-agent: *       
Disallow: /backdoor

session=E:OH48NH`6f!1E6dco2c"hESB8UzwP"QlXAeA?TN!1E]F8Xj{7w2@yrj(T?W$DKU$w[bP
```
这里的文件标明`/backdoor`这个地址是不被允许的,那更要试试了xwx
顺便提一下:useragent标明的是所处理的特定爬行器的名称,而这里的\*表示的当然是全部啦

果然是不行的返回内容:`Only admin can do this`
看来是我们缺少权限,
重点将目光看向下面的session

## 什么是sesision
简单来说,就是一个类似字典的**键值对**,保存了用户的信息
由于http通信的无记忆性,所以需要seesion让服务器辨别你到底是谁

### 了解一下简单session的原理
首先客户端向服务器会送http请求
服务器依据用户信息生成session,并加密后发给用户浏览器
浏览器储存为cookie,并在接下来的每次请求带上该cookie
服务器通过加密算法判断session,从而识别用户

## 回到题目的session
试图把session放到cookie里面,但cookie要求两个值(上面提到了session是个键值对嘛)
这里根本无法拆成两个值,所以只能把目光放回解密这里的session上了

## 解密
先观察密文包含哪些字符: 大小写英文,数字,还有各种符号
常见的编码base64/32,unicode,url,md5都不太符合

但是经过我的探究(ai()),发现这个特征满足base编码家族的一种:baese91(密文由91个字符`（0-9，a-z，A-Z,!#$%&()*+,./:;<=>?@[]^_{|}~”）组成)`

那在网上随便就可以找到解码base91的工具,得到如下
`RzQ0VE1OSlhHTTNEU05SUkdaQ0RHUUpYR1UzVEdOUlZHNFpBPT09PQ==`
其他的不认识,这个还是很有特点的:以\==结尾!!!  base64无疑了
再次解码:
`G44TMNJXGM3DSNRRGZCDGQJXGU3TGNRVG4ZA====`
这个字符串似乎也长得很有特点:以等号结尾,只含大写字符和数字
对照着解码网站的编码表,不难发现这是base32的特征:
`79657369616D3A75736572`
这样了已经很容易猜到了,只含有16进制字符,用base16
最终得到解密后的session:`yesiam:user`

## "登录"
根据题目的hint : 无需再加密
那直接把user改成admin,再放到cookie里面
这样就能成功进入backdoor啦

可以远程执行命令,那还说啥了
`ls -a` `cat flag`
**收工!**