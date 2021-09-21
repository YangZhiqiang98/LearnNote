## CORS

Spring 对跨域的处理有三种处理方案，

### 一、@CrossOrigin

@CrossOrigin 可以用在方法和 Controller 上。当添加到类上，表示 Controller 中的所有接口都支持跨域。

```java
@CrossOrigin(origins = "http://localhost:8081")
@GetMapping("/verifyCode")
public void verifyCode(HttpServletRequest request, HttpServletResponse resp) throws IOException {
    VerificationCode code = new VerificationCode();
    BufferedImage image = code.getImage();
    String text = code.getText();
    HttpSession session = request.getSession(true);
    session.setAttribute("verify_code", text);
    VerificationCode.output(image,resp.getOutputStream());
}
```

@CrosOrigin 注解的各属性含义如下：

- allowCredentials: 浏览器是否应当发送凭证信息，如 Cookie。
- allowHeaders: 设置请求被允许的请求头字段，* 表示所有字段。
- exposedHeaders：哪些响应头可以作为响应的部分暴露出来。
- maxAge: 预检请求的有效期，有效期内不必再次发送预检请求，默认为 1800 秒。
- methods: 允许的请求方法，* 代表允许所有的请求方法。
- origins: 允许的请求域， * 代表允许所有的请求域。



CrossOrigin 最终会构建出一个 CorsInterceptor 拦截器，然后在拦截器中对跨域请求进行校验。

### 二、addCorsMappings

通过重写 WebMvcConfigurer#addCorsMappings 方法来实现。

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                .allowedOrigins("*")
                .allowedMethods("*")
                .allowedHeaders("*")
                .allowCredentials(false)
                .exposedHeaders("")
                .maxAge(1800);
    }
}
```

这种配置方式最终处理结果和 @CrossOrigin 注解相同，都是在 CorsInterceptor  拦截器中对跨域请求进行校验。



### 三、CorsFilter

CorsFilter 是 Spring Web 中提供的一个处理跨域的过滤器。

```java
@Configuration
public class WebMvcConfig {
    @Bean
    FilterRegistrationBean<CorsFilter> corsFilterFilter(){
        FilterRegistrationBean<CorsFilter> registrationBean = new FilterRegistrationBean<>();
        CorsConfiguration corsConfiguration = new CorsConfiguration();
        corsConfiguration.setAllowedHeaders(Arrays.asList("*"));
        corsConfiguration.setAllowedMethods(Arrays.asList("*"));
        corsConfiguration.setAllowedOrigins(Arrays.asList("*"));
        corsConfiguration.setMaxAge(3600L);
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", corsConfiguration);
        registrationBean.setFilter(new CorsFilter(source));
        registrationBean.setOrder(-1);
        return registrationBean;
        
    }
}
```



这种方式与前两种不同的是， CorsFilter 是在过滤器中处理跨域的，而前面两种方案则是在 DispatchServlet 中触发跨域处理，从处理时间上看，**CorsFilter 对于跨域的处理时机要早于前面两种**。

因此，需要说明的是：

- @CorssOrigin 注解 + 重写 addCorsMappings 方法同时配置，这两种方式关于跨域的配置会自动合并，跨域在 CorsInterceptor 中只处理了一次。
- @CorsOrigin 注解 + CorsFilter 同时配置，或者重写 addCorsMappings 方法 + CorsFilter 同时配置，都会导致跨域在 CorsInterceptor 和 CorsFilter 中各处理了一次，降低程序运行效率，这两种组合不可取。

### Spring Security 中的CORS

在 Spring  Security 中，通过 @CrossOrigin 注解或者重写 addCorsMappings 方法配置跨域会失效；通过 CorsFilter 配置的跨域，有没有失效则要看过滤器的优先级。

由于非简单请求都要发送预检请求（preflight request），而预检请求没有携带认证信息，就有被 Spring Security 拦截的可能。如果通过 @CrossOrigin 注解或者重写 addCorsMappings 方法配置跨域，都要在 CorsInterceptor 中对跨域请求进行校验，要进入 CorsInterceptor ,就肯定会经过 Spring Security 过滤器链，因为没有携带认证信息，就会被拦截下来。

如果使用了 CorsFilter 配置跨域，只要过滤器的优先级高于 Spring Security 过滤器，就不会有问题。

#### 特殊处理 OPTIONS 请求

因此，如果在引入 Spring Security 后，还想继续通过 @CorssOrigin 注解或者重写 addCorsMappings 方法配置跨域，那么可以通过给 OPTIONS 请求单独放行，来解决预检请求被拦截的问题。

```java
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .antMatchers(HttpMethod.OPTIONS).permitAll()
                .anyRequest().authenticated()
                .and()
                .httpBasic()
                .and()
                .csrf().disable();
    }
}
```



#### 继续使用 CorsFilter

只需要将 CorsFilter 的优先级设置高于 Spring Security 过滤器的优先级。

```java
@Configuration
public class WebMvcConfig {
    @Bean
    FilterRegistrationBean<CorsFilter> corsFilterFilter(){
        FilterRegistrationBean<CorsFilter> registrationBean = new FilterRegistrationBean<>();
        CorsConfiguration corsConfiguration = new CorsConfiguration();
        corsConfiguration.setAllowedHeaders(Arrays.asList("*"));
        corsConfiguration.setAllowedMethods(Arrays.asList("*"));
        corsConfiguration.setAllowedOrigins(Arrays.asList("*"));
        corsConfiguration.setMaxAge(3600L);
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", corsConfiguration);
        registrationBean.setFilter(new CorsFilter(source));
        registrationBean.setOrder(Ordered.HIGHEST_PRECEDENCE);
        return registrationBean;
        
    }
}
```



#### 专业解决方案

```java
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .anyRequest().authenticated()
                .and()
                .httpBasic()
                .and()
                .cors()
                .configurationSource(corsConfigurationSource())
                .and()
                .csrf().disable();
    }

    @Bean
    CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration corsConfiguration = new CorsConfiguration();
        corsConfiguration.setAllowedHeaders(Arrays.asList("*"));
        corsConfiguration.setAllowedMethods(Arrays.asList("*"));
        corsConfiguration.setAllowedOrigins(Arrays.asList("*"));
        corsConfiguration.setMaxAge(3600L);
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", corsConfiguration);
        return source;
    }
}
```

首先需要提供一个 CorsConfigurationSource 实例，将跨域的各项配置都填充进去，然后在 configure(HttpSecurity http) 方法中，开启 

  .cors() 开启跨域配置，并将 CorsConfigurationSource 实例 设置进去即可。

