## dict 

也就是doctionary,也叫key-value储存,用哈希算法

特点:
- 用空间换时间,占用内存大,搜索快    
    与list对比
- 由于hash算法,key不能变(不能用list做key)
- 无序

格式:
```python
d={'A':95,'b':90}
print(d['A'])
```

**notice**:
- key不存在会报错
        - 判断key是否存在< dict名>.get('要判断的元素')
- 删除key
        - < dict>.pop ('key')

## set

只储存key,没有对应的value,
相当于一个集合,元素不能重复,可取交并等

格式:用大括号
```python
s = {1,2,3}
s = {[1,2,3]}        #用list作为输入集合
```

- .add(key)
- .remove(key)

