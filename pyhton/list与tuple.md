## list

一种有序的集合,有点像c中的数组,但元素可以为任意类型
可以嵌套,类似n维数组
**用中括号**

eg:
classmates=\['fszz','fszz2','fszz3']

- 用clssmates\[<索引>]来访问单个元素   (从0开始,最后的元素为-1)

- <list名>.+
			- append    末尾加元素
			- insert(n,'')   中间加元素        
			- pop()         删除元素
			
- 替换某个元素:直接赋值			


## tuple

就是元素不可变的list,但可以嵌套list,其中list中元素可变
**用小括号**

eg:
	classmates=('1','2','3')

**notice** :对于只有一个元素的tuple,为了与普通的数学()区分,在元素后加逗号,



### 其他

- range(n):生成从0到n-1的list
- list(<list名>)  :列出完整list 