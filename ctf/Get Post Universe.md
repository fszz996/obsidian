直接使用burp抓包

1.要用**get**格式+/?来传递money参数
```http
POST /?money=10000 http/1.1
```

2.然后题目要求用POST传递card_id
尝试直接在正文中写:
```http
card_id=xxxx
```
未果

再试试json格式
```http
{
"card_id"="xxxxxx"
}
```
未果

gpt说还有一种格式:**multipart/form-data**,试试看
```http
POST /?money=10000 HTTP/1.1
Host: forward.vidar.club:32741
Content-Type: multipart/form-data; boundary=----Boundary123  

------Boundary123
Content-Disposition: form-data; name="card_id"

7890234
------Boundary123--
```
成功