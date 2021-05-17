## Spring Security 基本认证

### 1、快速入门

首先新建一个SpringBoot项目，并引入Web和SpringSecuity依赖即可。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

加入springsecurity 依赖后，编写控制层

```java
@RestController
public class HelloController {
    @GetMapping("/hello")
    public String hello(){
        return "hello spring security";
    }
}
```

此时， /hello已经被自动保护起来了，当用户访问/hello接口时，会自动跳转到springsecurity的默认登录页面。

![image-20210517220904284](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/image-20210517220904284.png)

默认登录用户名时user,登录密码则是一个随机生成的UUID字符串，在项目启动日志中可以看到，这样说明，每次项目重启，密码都会变化。

```java
Using generated security password: 33b07716-59ff-4414-a7e2-2d10f0638032
```

这就是SpringSecurity的强大之处，一个依赖就保护了所有接口。

