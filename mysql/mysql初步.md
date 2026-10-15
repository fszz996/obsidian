# 基本结构
**sql**:数据库
**MySQL**:与SQL交互的语言
启动sql数据库:
```powershell
C:\mysql\bin\mysqld.exe --console #mysqld:服务器端
C:\mysql\bin\mysql.exe -u root -p  #客户端
```
## 包含
- 数据库 DATABADE
    - 表 TABLE
        - 字段 column 列--一类数据
        - 记录 row 行--一个对象

# 基本命令
不强制大写命令,但一般大写
```MySQL
-- 查看数据库
SHOW DATABASES

-- 切换到其中的一个数据库
USE learning
show TABLES -- 查看其中的表
 
-- 创建
CREATE DATABASE learning
CREATE TABLE student(
    id INT PRIMARY KEY AUTO_INCREMENT,-- 主键,代表每个row的唯一标识,且自动递增
    name VARCHAR (50),-- 最大值
    age INT
);

-- 创建/插入一个TABLE实例
INSERT INTO studnet(id,name,age)
value(DEFAULT,fszz,18);--  使用默认值

-- 查询row
SELECT * FROM student;-- 显示所有行列-整个表
-- *表示所有的column
SELECT name FROM studnet-  -- 只显示name这一column
WHERE name = 'fszz';-- 查找fszz这一row
```

# 模糊查找
**通配符**
- *    -- 在字符串中表示任意长度的字符,包括0
- -         表示一个长度的字符

# 统计功能
```mysql
SELECT COUNT(name) From student;
SELECT AVG(name) From student;
SELECT SUM(name) From student;
SELECT MAX(name) From student;
SELECT MIN(name) From student;

-- 按年龄分组
GROUP BY age 

```

# join合并
```mysql
SELECT * -- 所有列
FROM studnet
JOIN scores
ON student.id=scores.id;-- 将id相同的合并并输出
JOIN ---
ON ---- -- 可以链接多张表
```
join / leftjoin
join合并两表都有的,没有的舍弃
leftjoin合并from表的全部,join没有的会写入NULL