# try-catch-finally
```Java
try{
3/0;
}
catch(ArithmeticException e){
system.out.println("错误");
}
finally{
}
```
try用于可能会有异常的语句
catch用于捕捉指定的异常类型
finally在通常情况下有异常未被捕捉也会执行
# throw and throws
## throw:
直接抛出一个异常对象,然后直接报错
## throws:
声明可能会出现异常,可以直接catch,也可以在函数里面先不处理,**向外传播**,在外部处理

