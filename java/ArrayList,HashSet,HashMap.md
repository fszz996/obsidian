# ArrayList

```Java
import java.util.ArrayList;

ArrayList<string> names = new ArrayList();
//      集合中的类型 list名
```

ArrayList没有数组的长度限制,可以自由添加删除元素,可以允许有重复元素

```Java
//list 常用操作
name.add();
name.get();//获取指定位置的元素,start from 0
name.set();//替换
name.remove();//删除
name.size();//集合中元素的数量
```
# HashMap
**键-值**映射集合
```Java
HashMap<string,interger> age = new HashMap();
```

# HashSet

不允许重复元素,如果内容相同,那么hashcode一定要相同

```Java
HashSet<string> names = new HashSet();
```

在储存string时,eqaul()和Hashcode可以正确的用string的内容比较
但在储存其他自定义类时不行,需要@override equal和Hashcode函数

在实际使用上面的集合时,通常用通用接口list通过多态调用.