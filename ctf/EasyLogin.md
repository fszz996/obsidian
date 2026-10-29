  进入后是一个登录系统,需要输入用户名,密码和图片上的验证码

  照样先f12看看,发现一条note:  管理员的密码格式是:4位字母+3位数字
字母加数字,就算只看小写字母,也有26^4\*999=456519024--4亿多个!!
算上大写字母只会多的更多,

  但无论如何还不知道要好很多的,至少理论上可以尝试爆破了
## 先尝试解决一下验证码的问题吧
brup抓包!!!
```http
GET / HTTP/1.1
Host: forward.vidar.club:31408
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/156.0.0.0 Safari/537.36 Edg/156.0.0.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6
Cookie: session=eyJjYXB0Y2hhX2NvZGUiOiI2NjQ0In0.auAvnw.uN83nJO7R1GKu584_xeJIqFeYyA
Connection: keep-alive247

```
  这个请求头里面似乎只有cookie信息比较可疑
  看起来像base64编码,尝试解读一下
```python
import base64

code = "eyJjYXB0Y2hhX2NvZGUiOiI2NjQ0In0"
len (code)
data = base64.b64decode
```

  发现会报错: binascii.Error: Incorrect padding
  会发现会有base64编码的问题.

## 那先了解一下base64编码吧:
 为了解决在不同系统和不同的传输协议里面会遇到的二进制不兼容问题,
 所以只用从ASIIC字符中挑了64个(26+26+10+/+'+'),保证支持全部所有所有()
 
 ==🟢*什么时候会用它呢*==,比如我们想发送一段文字:"hello"
 base64会将3个字节分成一组,也就是3\*8=24bit, 然后将这24bit分为4组,每组6bit
注意到2^6=64,也就是正好对应刚刚提到的64个字符
**总之3byte -> 4base64字符!!!**

那又产生了一个问题:如果原文的字符数%3 != 0 怎么办
base64规定:**直接补零**

举个栗子:hello的后两位是"lo",
先用python查一下ascii码
```python
ord("l")
ord("o")
```
再把3位十进制的ascii码化成2^3=8位2进制
注意到这里只有16bit,那后面就都变成零, 变成类似xxxxxxxx xxxxxxxx 00000000 00000000
然后分成4\*6位: xxxxxx xxxxxx xxxx00 000000
再化成10进制用base64的索引表 : X X X A
我们发现最后一位是因为补零所以是A,但这里的A不承载信息,
所以base64规定,将不携带原始信息的base64字符变为"="

这就是为什么我们看到的base64编码后面会带上1-2个"="啦

## 回到刚刚的题目
cookie携带的base64编码省掉了=,所以python的b64decode不能对其解码,
我们应该将它补为3的整数倍
但是神奇的我发现多补=不会对结果造成影响!!
所以无论是差多1/2/0个"=",直接加\==就行
```python
  
```
那验证码就被我们解决了,在爆破时只需要将上一次报文中cookie的验证码提取出来自动加入到post报文就行

发现题目还有提示()
我都准备直接开始爆破了,但是雀食时间也会太久了,发现在靶机外面还有个hint:  ==🟢密码的前四位的md5值是fd1859325f7119e9==

好吧,之前跳过的最终还是会找上门
## 看看md5吧
md5一般用来保证消息是否被篡改,提供完整性保护,是一种"指纹算法"
拥有以下特点:
1. 对于任意长度的信息,其输出结果长度不变--恒为128位2进制数(所以称为指纹算法)
2. 不可逆,只能通过查字典的方式推测原文
3. 运算速度很快,低位消息容易破解

**但是为什么在日常我们看到的md5都是像上面那样:数字与字母混合呢?**
由于128位的二进制显示出来实在太长,通常化为16进制数据,也就是用0-9-a-f来表示
我们来看看长度,由于2\^4=16,所以4位2进制数可以化为一个16进制数,那一个md5的密文化为16进制就会有--128/4 = 32位
**但其实日常使用的还有一种16位的md5** :非常简单,只需要把16位的md5提取中间部分,也就是4-12位

# 破解md5
ok既然知道了md5原文的位数和由字母组成,那就能很快的在本地脚本查字典得出原文
写一个python脚本
```python
import hashlib
from itertools import product

real = "fd1859325f7119e9"#已知16位md5哈希
def md5(str):
    m= hashlib.md5()#创建md5对象
    m.update(str.encode('utf-8'))
    md5_result = m.hexdigest()[8:24]
    #16位md5一般取中间,但有时也会取开头或结尾,都可以试一下
    return md5_result

allchar = []
for i in range(ord('a'),ord('z')+1): #先尝试小写字母吧,注意range是左闭右开
    allchar.append(chr(i))
  
for r in product(allchar, repeat=4):#product返回的是tuple(元组--不可变的列表)
    result = ""
    result = result + r[0]+r[1]+r[2]+r[3]#用[]取tuple里的值
    if md5(result) == real:
        print("密码是:",result)
        break
else:
    print("没有匹配的值")
```

**我们来看看这段代码**:
  首先,怎么理解:  m= hashlib.md5()
因为很多时候md5并不是一次性的,是可累加的
所以创建的md5对象可以分次upadte--"加入"新的数据,然后再通过hexdigest方法取出16进制的结果即可
**注意:**.update()方法只接受2进制数据,需要用字符串的.encode()方法转成二进制数(方法默认参数为utf-8)

在这里提一下python里面import和from - import的区别--因为我不知道():
import<>:  将整个模块作为对象导入,所以用的时候需要写<模块名>.<模块中对象>来访问对象
from<>import<>:只导入import后写的对象,写的时候不用带模块名,可以写\*来代表全部对象,但极易引发命名对象冲突,不推荐使用

最后得到原文:'mini'

## 最后爆破
只剩最后3位数字未知了,也就是1000种可能,爆破速度会非常快
这里用python写脚本,需要用到request库






