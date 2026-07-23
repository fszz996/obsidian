## if语句

格式:
```python 
if age>=18:
    print('adualt')
    print('pass')
elif <条件2>:
    do
else:
    do    
```

notice: 从上向下判断,符合条件就结束


## match 语句

### 简单匹配
类似于c的switch

格式: 
```python
match score:
    case a:
        do
    case b:
        do
    case _:    #其他情况
        do 
```
### 复杂匹配

    case x if x < 10    # 匹配10以下的范围,并将值赋给x
    case a|b|c              # 或,匹配多个值
eg:
```python
match score:
    case 100:
        do
    case x if x<10:
        do
    case 60|70|90:
        do
```

### 匹配列表

case \[<元素>]: