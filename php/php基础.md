# 基本骨架
用`<?php`和`?>`包裹代码
所以php可以直接和html写在一起
每句代码后加 ;
用echo输出
# 变量 
以$符开头,像python一样不用声明类型
但与python不同的是,拼接字符串用.号

# 数组
两种写法
```php
$arr = array("1","2","3");
$arr2 = ["1","2","3"];
```

# 函数
定义一个函数
```php
function <函数名>(<参数>):<返回类型>
{}
```

# 超全局变量
可以在任何地方直接用,php内置
- $\_POST    一个数组,储存post表单提交
- $\_GET     url里面?后的参数
- $\_SERVER
- $\_SESSION

# session
```php
session_start();  // 开启会话
$_session['username'] = 'admin' 
```