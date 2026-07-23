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

