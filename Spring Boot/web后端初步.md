# 注解
使用@符号开头,提供额外信息供编译器或框架识别
eg:
- @override
- @RestController
- @GetMapping
- @RequstParam
   ...

# 基本实例
```Java
@RestController
public calss HelloController{
    @GetMapping(/hello)
    public stirng hello(){
        return "nihao";
    }
    
    //用@RequstParamc传入参数
    @GetMapping(/user)
    public string user(@RequstParamc String name){
        return "hello"+name
    }
    
    //用@PathVariable传入参数
    @GetMapping(/number/{id})
    public int number(@PathVariable int id){
        return "user's id is"+id;
    }
}
```
**用@RequstPramc传参时**
地址:/示例?name=xxx&name2=xxx
**用@PathVariable时**
/示例/xxx

# 项目结构分层

**Controller**:处理http请求
**Servce**:业务的底层逻辑
**User**:数据结构

一个简单的调用流程:
浏览器
   |     发送http请求
Controller
   |      调用Service的方法
Servise
   |       处理数据逻辑
Controller
   |       return返回值json
浏览器


# 前端传入Json
```java
  @PostMapping("/register")

    public String rigister(@Valid @RequestBody RegisterRequest requst) {

        return userService.fangfa1(requst);

    }
```
@PostMapping将post请求映射待/register
@RequsetBody将前端通过http传进的json,自动解析到java对象requst中

# 参数校验
如上代码
@vailid触发spring的对对象的参数检验
```java
public class RegisterRequest {

  

    @NotBlank(message = "用户名不能为空")

    public String username;

  

    @Min(value = 0, message = "年龄不能小于0")

    public int age;

  

    @Email(message = "邮箱格式不正确")

    public String email;

}
```
在定义类的时候加入对应的规则注解即可,messege定义了错误信息

# 返回错误信息与状态码
