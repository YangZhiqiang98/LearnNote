# 一、Spring Security 简介

## 1.1 核心功能

- authentication(认证)：
- authorization（授权）

## 1.2 整体架构

在 Spring Security 中，认证（authentication）和授权（Authorization）是分开的，无论使用什么样的认证方式，都不会影响授权。这种独立的好处就时 Spring Security 可以非常方便地整合一些外部的认证方案。

### 1.2.1 认证

在 Spring Security 中，用户的认证信息主要由 Authentication 的实现类来保存。Authentication 的接口定义如下：

```java
public interface Authentication extends Principal, Serializable {
	Collection<? extends GrantedAuthority> getAuthorities();
	Object getCredentials();
	Object getDetails();
	Object getPrincipal();
	boolean isAuthenticated();
	void setAuthenticated(boolean isAuthenticated) throws IllegalArgumentException;
}
```

- getAuthorities：用来获取用户的权限。
- getCredentials：用来获取用户凭证，一般来说就是密码。
- getDetails：用来获取用户携带的详细信息，可能是当前请求之类等。
- getPrincipal：用来获取当前用户，例如是一个用户名或者用户对象。
- isAuthenticated：当前用户是否认证成功。



认证工作主要是由 AuthenticationManager 接口来负责，定义如下：

```java
public interface AuthenticationManager {
	Authentication authenticate(Authentication authentication)
			throws AuthenticationException;
}
```

AuthenticationManager 是有一个 authenticate 方法用来做认证，该方法有三个不同的返回值：

- 返回 Authentication ：表示认证成功。
- 抛出 AuthenticationException：表示用户输入了无效的凭证。
- 返回 null:表示不能确定。

AuthenticationManager 最主要的实现类是 ProviderManager，ProviderManager 管理了众多的 AuthenticationProvider 实例，AuthenticationProvider 类似于 AuthenticationManager ，但是它多了一个 supports 方法用来判断是否支持给定的 Authentication 类型。

```java
public interface AuthenticationProvider {
	Authentication authenticate(Authentication authentication)
			throws AuthenticationException;
	boolean supports(Class<?> authentication);
}
```

由于 Authentication 有不同的实现类，这些实现类会由不同的 AuthenticationProvider 来处理，所以 AuthenticationProvider 会有一个 supports 方法，用来判断当前的 AuthenticationProvider 是否支持对应的 Authentication。



在一次完整的认证流程中，可能会同时存在多个 AuthenticationProvider，多个 AuthenticationProvider 统一由 ProviderManager 来管理。同时，ProviderManager 具有一个可选的 parent,如果所有的 AuthenticationProvider 都认证失败，那么就会调用 parent 进行认证。parent 相当于一个备用认证方式，即各个 AuthenticationProvider 都无法处理认证问题时，就由 parent 出场收拾残局。

```java
public class ProviderManager implements AuthenticationManager, MessageSourceAware,
		InitializingBean {
	private AuthenticationEventPublisher eventPublisher = new NullEventPublisher();
	private List<AuthenticationProvider> providers = Collections.emptyList();
	protected MessageSourceAccessor messages = SpringSecurityMessageSource.getAccessor();
	private AuthenticationManager parent;
	private boolean eraseCredentialsAfterAuthentication = true;
	...
}
```

### 1.2.2 授权

当完成认证后，接下来就是授权了。在 Spring Security 授权体系中，有两个关键接口：

- AccessDecisionManager
- AccessDecisionVoter

AccessDecisionVoter 是一个投票器，投票器会检查用户是否具备应有的角色，进而投出赞成、返回或者弃权票；AccessDecisionManager 则是一个决策器，来决定此次访问是否被允许。在 AccessDecisionManager 中会挨个遍历 AccessDecisionVoter，进而决定是否允许用户访问。

在 Spring Security 中，请求一个资源所需要的角色会被封装成一个 ConfigAttribute 对象，该对象的 getAttribute 会返回角色的名称。一般来说，角色名称都带有一个 ROLE_ 前缀，投票器 AccessDecisionVoter 所做的就是比较用户所具备的角色和请求某个资源所需的 ConfigAttribute  之间的关系。

```java
public interface ConfigAttribute extends Serializable {
	String getAttribute();
}
```

### 1.2.3 Web 安全

在 Spring Security 中，认证、授权等功能都是基于过滤器来完成的。下表是常见的过滤器：

| 过滤器                                   | 过滤器作用                                                   | 是否默认加载 |
| :--------------------------------------- | ------------------------------------------------------------ | ------------ |
| ChannelProcessingFilter                  | 过滤请求协议，如 HTTPS 和 HTTP                               | NO           |
| WebAsyncManagerIntegrationFilter         | 将 WebSayncManager 与 Spring Security 上下文进行集成         | YES          |
| ==SecurityContextPersistenceFilter==     | 在处理请求之前，将安全信息加载到 SecurityContextHolder 中以方便后续使用。请求结束后，再擦除 SecurityContextHolder 中的信息。 | YES          |
| HeaderWriterFilter                       | 头信息加入到响应中                                           | YES          |
| CorsFilter                               | 处理跨域问题                                                 | NO           |
| CsrfFilter                               | 处理 CSRF 攻击                                               | YES          |
| LogoutFilter                             | 处理注销登录                                                 | YES          |
| OAuth2AuthorizationRequestRedirectFilter | 处理 OAuth2 认证重定向                                       | NO           |
| Saml2WebSsoAuthenticationRequestFilter   | 处理 SAML认证                                                | NO           |
| X509AuthenticationFilter                 | 处理 X509 认证                                               | NO           |
| AbstractPreAuthenticatedProcessingFilter | 处理预认证问题                                               | NO           |
| CasAuthenticationFilter                  | 处理 CAS 单点登录                                            | NO           |
| OAuth2LoginAuthenticationFilter          | 处理 OAuth2 认证                                             | NO           |
| Saml2WebSsoAuthenticationFilter          | 处理 SAML 认证                                               | NO           |
| ==UsernamePasswordAuthenticationFilter== | 处理表单登录                                                 | YES          |
| OpenIDAuthenticationFilter               | 处理 OpenID 认证                                             | NO           |
| DefaultLoginPageGeneratingFilter         | 配置默认登录页面                                             | YES          |
| DefaultLogoutPageGeneratingFilter        | 配置默认注销页面                                             | YES          |
| ConcurrentSessionFilter                  | 处理 Session 有效期                                          | NO           |
| DigestAuthenticationFilter               | 处理 HTTP 摘要认证                                           | NO           |
| BearerTokenAuthenticationFilter          | 处理 OAuth2 认证时的 Access Token                            | NO           |
| BasicAuthenticationFilter                | 处理 HttpBasic 登录                                          | YES          |
| RequestCacheAwareFilter                  | 处理请求缓存                                                 | YES          |
| SecurityContextHolderAwareRequestFilter  | 包装原始请求                                                 | YES          |
| JaasApiIntegrationFilter                 | 处理 JAAS 认证                                               | NO           |
| ==RememberMeAuthenticationFilter==       | 处理 RememberMe 登录                                         | NO           |
| AnonymousAuthenticationFilter            | 配置匿名认证                                                 | YES          |
| OAuth2AuthorizationCodeGrantFilter       | 处理 OAuth2 认证中的授权码                                   | NO           |
| ==SessionManagementFilter==              | 处理 Session 并发问题                                        | YES          |
| ExceptionTranslationFilter               | 处理异常认证/授权中的情况                                    | YES          |
| FilterSecurityInterceptor                | 处理授权                                                     | YES          |
| SwitchUserFilter                         | 处理账户切换                                                 | NO           |

开发者所见到的 Spring Security 提供的功能都是通过这些过滤器实现的，这些过滤器按照既定的优先级排列，最终形成一个过滤器链。开发者也可自定义过滤器。

需要注意的是，默认过滤器并不是直接放在 Web 项目的原生过滤器链中，而是通过一个 FilterChainProxy 来统一管理。Spring Security 的过滤器链通过 FilterChainProxy 嵌入到 Web 项目的原生过滤器链中。FilterChainProxy 作为一个顶层管理者，将统一管理 Security Filter，FilterChainProxy  本身将通过 Spring 框架提供的 [DelegatingFilterProxy](https://docs.spring.io/spring-security/site/docs/5.4.6/reference/html5/#servlet-delegatingfilterproxy) 整合到原生过滤器链中。



![securityfilterchain](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20211104203028.png)



在 Spring Security 中，这样的过滤器链可能会有多个。当存在多个过滤器链时，多个过滤器链之间要指定优先级，当请求到达之后，会从 FilterChainProxy 进行分发，先和哪个过滤器链匹配上，就用哪个过滤器链进行处理。



![multi-securityfilterchain](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20211104203330.png)





### 1.2.4 登录数据保存

当用户登录成功后，Spring Security 会将登录成功的用户信息保存到 SecurityContextHolder 中。SecurityContextHolder 中的数据保存默认是通过 ThreadLocal 来实现的，也就是用户数据和请求绑定在一起，当登录请求处理完毕后，Spring Security 会将 SecurityContextHolder 中的数据拿出来保存到 Session 中，同时将 SecurityContextHolder 中的数据清空。以后每当有请求到来时，Spring Security 就会先从 Session 中取出用户登录数据，保存到 SecurityContextHolder 中，方便在该请求的后续处理过程中使用，同时在请求结束时将 SecurityContextHolder 中的数据拿出来保存到 Session 中，然后将 SecurityContextHolder 中的数据清空。

- 优势：方便用户在 Controller 和 Service 层获取当前登录用户数据。
- 问题：在子线程中获取用户登录数据较为麻烦。

如果开发者使用 `@Async` 注解来开启异步任务的话，只需要添加如下配置，使用 Spring Security 提供的异步任务代理，就可以在异步任务中从 SecurityContextHolder 里边获取当前登录用户的信息。

```java
@Configuration
public class ApplicationConfiguration extends AsyncConfigurerSupport {
    @Override
    public Executor getAsyncExecutor() {
        return new DelegatingSecurityContextExecutor(Executors.newFixedThreadPool(5));
    }
}
```

![securitycontextholder](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20211105065548.png)



# 二、Spring Security 认证

### 2.1 Spring Security 基本认证

#### 2.1.1 快速入门

在 Spring Boot 中，默认我们引入如下依赖，Spring Security 会将所有的接口都自动保护起来。

```xml
<dependency>
<	groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

提供一个测试接口：

```java
@RestController
public class HelloController {
    @GetMapping("/hello")
    public String hello(){
        return "hello spring security";
    }
}
```

当访问 hello 时，会跳转到登录页面。

![image-20211106164359818](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20211106164401.png)





#### 2.1.2 流程分析

![默认登录流程](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20211106170118.png)



1). 客户端（浏览器）发起请求去访问 /hello 接口，这个接口默认是需要认证后才能访问。

2). 这个请求会走一遍 Spring Security 中的过滤器链，因未登录认证，在最后的 FilterSecurityInterceptor 过滤器中被拦截下来，接下来抛出 AccessDeniedException 异常。

3). 抛出的 AccessDeniedException 异常在 ExceptionTranslationFilter 过滤器中被捕获，ExceptionTranslationFilter 过滤器通过调用 LoginUrlAuthenticationEntryPoint#commence 方法给客户端返回 302,要求客户端重定向到 /login 页面。

4). 客户端发送 /login 请求。

5). /login 请求被 DefaultLoginPageGeneratingFilter 过滤器拦截下阿里，并在该过滤器中返回登录页面。

#### 2.1.3 原理分析

当加入上节的依赖后，Spring Boot 背后做了很多事：

- 开启 Spring Security 自动化配置，开启后，会自动创建一个名为 springSecurityFilterChain 的过滤器，并注入到 Spring 容器中，此过滤器代理了 Spring Security 中的过滤器链。
- 创建一个 UserDetailsService 实例，默认用户数据是基于内存的用户（user）,密码则是随机生成的 UUID 字符串。
- 给用户生成一个默认的登录页面。
- 开启 [CSRF](https://baike.baidu.com/item/%E8%B7%A8%E7%AB%99%E8%AF%B7%E6%B1%82%E4%BC%AA%E9%80%A0/13777878?fromtitle=CSRF&fromid=2735433&fr=aladdin) 攻击防御。
- 开启[会话固定攻击](https://www.cnblogs.com/phpstudy2015-6/p/6776919.html)防御。
- 集成 X-XSS-Protection。
- 集成 X-Frame-Options 以防止[单击劫持](https://zhuanlan.zhihu.com/p/150121109)。

##### 2.1.3.1 默认用户生成

Spring Security 中定义了 UserDetails 接口来规范开发者自定义的**用户对象**。

```java
public interface UserDetails extends Serializable {
	Collection<? extends GrantedAuthority> getAuthorities();
    
	String getPassword();
    
	String getUsername();
    
	boolean isAccountNonExpired();
    
	boolean isAccountNonLocked();
    
	boolean isCredentialsNonExpired();
    
	boolean isEnabled();
}
```



该接口中一共定义了 7 个方法：

（1）getAuthorities：返回当前账户所具备的权限。

（2）getPassword：返回当前账户的密码。

（3）getUsername：返回当前账户的用户名。

（4）isAccountNonExpired：返回当前账户是否未过期。

（5）isAccountNonLocked：返回当前账户是否未锁定。

（6）isCredentialsNonExpired：返回当前账户凭证（如密码） 是否未过期。

（7）isEnabled：返回当前账户是否可用。



这时用户对象的定义，而负责提供用户数据源的接口是 UserDetailsService，接口中只有一个查询用户的方法：

```java
public interface UserDetailsService {
	UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
}
```

在实际项目中，一般需要开发者自定义 UserDetailsService 的实现。如果没有自定义实现，Spring Security 也为 UserDetailsService 提供了默认实现：

![UserDetailService 的默认实现](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20211106221832.png)



- UserDetailsManager 在 UserDetailsService 的基础上，继续定义了添加用户、更新用户、删除用户、修改密码以及判断用户是否存在共 5 种方法。

  ```java
  public interface UserDetailsManager extends UserDetailsService {
  	void createUser(UserDetails user);
  
  	void updateUser(UserDetails user);
  
  	void deleteUser(String username);
  
  	void changePassword(String oldPassword, String newPassword);
  
  	boolean userExists(String username);
  
  }
  ```

- JdbcDaoImpl 在 UserDetailsService 的基础上，通过 spring-jdbc 实现了从数据库中查询用户的方法。
- InMemoryUserDetailsManager 实现了 UserDetailManager 中关于用户的增删改查方法，不过都是基于内存的操作，数据并没有持久化。（**默认**）
- JdbcUserDetailsManager 继承自 JdbcDaoImpl 同时又实现了 UserDetailManager 接口，因此可以持久化对用户的增删改查操作；不过有一个局限性，JdbcUserDetailsManager 中操作数据库中用户的 SQL 都是提前写好的，不够灵活。
- CachingUserDetailsService 的特点就是会将 UserDetailsService 缓存起来。
- ReactiveUserDetailsServiceAdapter 是 webflux-web-security 模块定义的 UserDetailsService 的实现。



Spring Boot 中针对 UserDetailsService 的自动化配置类是 UserDetailsServiceAutoConfiguration。

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(AuthenticationManager.class)
@ConditionalOnBean(ObjectPostProcessor.class)
@ConditionalOnMissingBean(
		value = { AuthenticationManager.class, AuthenticationProvider.class, UserDetailsService.class },
		type = { "org.springframework.security.oauth2.jwt.JwtDecoder",
				"org.springframework.security.oauth2.server.resource.introspection.OpaqueTokenIntrospector" })
public class UserDetailsServiceAutoConfiguration {

	private static final String NOOP_PASSWORD_PREFIX = "{noop}";

	private static final Pattern PASSWORD_ALGORITHM_PATTERN = Pattern.compile("^\\{.+}.*$");

	private static final Log logger = LogFactory.getLog(UserDetailsServiceAutoConfiguration.class);

	@Bean
	@ConditionalOnMissingBean(
			type = "org.springframework.security.oauth2.client.registration.ClientRegistrationRepository")
	@Lazy
	public InMemoryUserDetailsManager inMemoryUserDetailsManager(SecurityProperties properties,
			ObjectProvider<PasswordEncoder> passwordEncoder) {
		SecurityProperties.User user = properties.getUser();
		List<String> roles = user.getRoles();
		return new InMemoryUserDetailsManager(
				User.withUsername(user.getName()).password(getOrDeducePassword(user, passwordEncoder.getIfAvailable()))
						.roles(StringUtils.toStringArray(roles)).build());
	}

	private String getOrDeducePassword(SecurityProperties.User user, PasswordEncoder encoder) {
		String password = user.getPassword();
		if (user.isPasswordGenerated()) {
			logger.info(String.format("%n%nUsing generated security password: %s%n", user.getPassword()));
		}
		if (encoder != null || PASSWORD_ALGORITHM_PATTERN.matcher(password).matches()) {
			return password;
		}
		return NOOP_PASSWORD_PREFIX + password;
	}

}
```

从上述代码可以看出，有两个比较重要的条件促使系统自动提供一个 InMemoryUserDetailsManager 的实例：

- 当前 classpath 下存在 AuthenticationManager 类。
- 当前项目下，系统没有提供 AuthenticationManager、AuthenticationProvider、UserDetailsService 以及 ClientRegistrationRepository 实例。

默认情况下，上面的条件都会满足，此时 Spring Security 会提供一个 InMemoryUserDetailsManager 实例。从 inMemoryUserDetailsManager 方法中可以看到，用户数据源自 SecurityProperties#getUser 方法：

```java
@ConfigurationProperties(prefix = "spring.security")
public class SecurityProperties {
	private User user = new User();
	
	public User getUser() {
		return this.user;
	}

	public static class User {

		private String name = "user";

		private String password = UUID.randomUUID().toString();

		private List<String> roles = new ArrayList<>();

		private boolean passwordGenerated = true;
		
		}
}
```

可以看出，默认用户名为 user，默认密码是一个 UUID 字符串。

定制 SecurityProperties.User 类中的各属性的值：

```properties
spring.security.user.name=test
spring.security.user.password=123
spring.security.user.roles=admin,user
```

##### 2.1.3.2 默认页面生成

在 Spring Security 过滤器中，DefaultLoginPageGeneratingFilter 用来生成默认的登录页面，DefaultLogoutPageGeneratingFilter 用来生成默认的注销页面。

DefaultLoginPageGeneratingFilter 部分核心源码：

```java
public class DefaultLoginPageGeneratingFilter extends GenericFilterBean {
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
			throws IOException, ServletException {
		HttpServletRequest request = (HttpServletRequest) req;
		HttpServletResponse response = (HttpServletResponse) res;

		boolean loginError = isErrorPage(request);
		boolean logoutSuccess = isLogoutSuccess(request);
		if (isLoginUrlRequest(request) || loginError || logoutSuccess) {
			String loginPageHtml = generateLoginPageHtml(request, loginError,
					logoutSuccess);
			response.setContentType("text/html;charset=UTF-8");
			response.setContentLength(loginPageHtml.getBytes(StandardCharsets.UTF_8).length);
			response.getWriter().write(loginPageHtml);

			return;
		}

		chain.doFilter(request, response);
	}
}
```

在 doFilter 方法中，首先判断出当前请求是否为登录出错请求、注销成功请求或者登录请求。如果是这三种请求中的任意一个，就会通过 generateLoginPageHtml 方法生成登录页面（字符串拼接的方式）返回，在该方法中，会根据不同的登录场景，生成不同的登录页面，如果有异常信息，就把异常信息一同返回给前端，否则请求继续往下走，执行下一个过滤器。



DefaultLogoutPageGeneratingFilter 部分核心源码：

```java
public class DefaultLogoutPageGeneratingFilter extends OncePerRequestFilter {
	private RequestMatcher matcher = new AntPathRequestMatcher("/logout", "GET");
    @Override
	protected void doFilterInternal(HttpServletRequest request,HttpServletResponse response, FilterChain filterChain)
			throws ServletException, IOException {
		if (this.matcher.matches(request)) {
			renderLogout(request, response);
		} else {
			filterChain.doFilter(request, response);
		}
	}
    
    private void renderLogout(HttpServletRequest request, HttpServletResponse response) throws IOException {
		String page =  "";
		response.setContentType("text/html;charset=UTF-8");
		response.getWriter().write(page);
	}
}    
```

请求到来之后，会先判断是否是注销请求 /logout,如果是 /logout 请求，则渲染一个注销请求的页面返回给客户端；否则请求继续往下走，执行下一个过滤器。

### 2.2 登录表单配置

#### 2.2.1 基本配置

```java
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .anyRequest().authenticated()
                .and()
                .formLogin()
                .loginPage("/login.html")
                .loginProcessingUrl("/doLogin")
                .defaultSuccessUrl("/index")
                .failureUrl("/login.html")
                .usernameParameter("uname")
                .passwordParameter("passwd")
                .permitAll()
                .and()
                .csrf().disable();
    }
}
```

在 Spring Security 中，如果自定义配置，基本上都是继承自 WebSecurityConfigurerAdapter 来实现的。

配置详解：

（1）首先 configure 中是一个链式配置，当然也可以不用链式配置。

（2）authorizeRequests（）方法表示开启权限配置， .anyRequest().authenticated() 表示所有的请求都要认证之后才能访问。

（3）formLogin() 表示开启表单登录配置，loginPage 用来配置登录页面地址；loginProcessingUrl 用来配置登录接口地址；defaultSuccessUrl 表示登录成功后的跳转地址；failureUrl 表示登录失败后的跳转（重定向）地址；usernameParameter 表示登录用户名的参数名称；passwordParameter 表示登录密码的参数名称；permitAll 表示等登录相关的页面和接口不做拦截，直接通过。

（4）csrf().disable() 表示禁用 CSRF 防御功能。



#### 2.2.2 配置细节

##### 2.2.2.1 登录成功

###### 前后端不分离

在 Spring Security 中，和登录成功重定向 URL 相关的方法有两个：

- defaultSuccessUrl
- successForwardUrl

在配置的时候，defaultSuccessUrl 和 successForwardUrl 只需要配置一个即可，具体配置哪个，则要看你的需求，两个的区别如下：

1. defaultSuccessUrl 表示当前用户登录成功之后，会**自动重定向到登录之前的地址**。
2. successForwardUrl 则不会考虑用户之前的访问地址，只要用户登录成功，就会通过**服务器端跳转到 successForwardUrl 所指定的页面**。
3. defaultSuccessUrl 还有一个重载的方法，第二个参数如果不设置默认为 false，也就是我们上面的的情况，如果手动设置第二个参数为 true，则 defaultSuccessUrl 的效果和 successForwardUrl 一致。不同之处在于，**defaultSuccessUrl 是通过重定向实现的跳转（客户端跳转），而successForwardUrl 则是通过服务器端跳转实现的**。

```java
.and()
.formLogin()
.loginPage("/login.html")
.loginProcessingUrl("/doLogin")
.usernameParameter("name")
.passwordParameter("passwd")
.defaultSuccessUrl("/index")
.successForwardUrl("/index")
.permitAll()
.and()
```

**「注意：实际操作中，defaultSuccessUrl 和 successForwardUrl 只需要配置一个即可。」**



无论是 defaultSuccessUrl 还是 successForwardUrl 最终所配置的都是 AuthenticationSuccessHandler 接口的实例。AuthenticationSuccessHandler 专门用来处理登录成功事项：

```java
public interface AuthenticationSuccessHandler {
	default void onAuthenticationSuccess(HttpServletRequest request,
			HttpServletResponse response, FilterChain chain, Authentication authentication)
			throws IOException, ServletException{
		onAuthenticationSuccess(request, response, authentication);
		chain.doFilter(request, response);
	}

	void onAuthenticationSuccess(HttpServletRequest request,
			HttpServletResponse response, Authentication authentication)
			throws IOException, ServletException;

}
```

AuthenticationSuccessHandler 接口中定义了两个方法，其中一个是 default 方法，此方法是 Spring Security 5.2 加入进来的，在处理特定的认证请求 Authentication Filter 中会用到；另外一个非 default 方法，则用来处理登录成功的具体事项，其中 authentication 参数保存了登录成功的用户信息。



AuthenticationSuccessHandler 接口共有三个实现类：



![AuthenticationSuccessHandler  实现类](https://cdn.jsdelivr.net/gh/YangZhiqiang98/ImageBed/20211107111229.png)



（1）SimpleUrlAuthenticationSuccessHandler 继承自 AbstractAuthenticationTargetUrlRequestHandler，通过 AbstractAuthenticationTargetUrlRequestHandler 中的 handle 方法实现请求重定向。

（2）SavedRequestAwareAuthenticationSuccessHandler  在 SimpleUrlAuthenticationSuccessHandler 的基础上增加了请求缓存的功能，可以记录之前的请求地址，进而在登录成功后重定向到一开始访问的地址。（defaultSuccessUrl 配置的实例）

（3）ForwardAuthenticationSuccessHandler 就是一个服务端跳转。（successForwardUrl 配置的实例）



SavedRequestAwareAuthenticationSuccessHandler 源码：

```java
public class SavedRequestAwareAuthenticationSuccessHandler extends
		SimpleUrlAuthenticationSuccessHandler {
	protected final Log logger = LogFactory.getLog(this.getClass());

	private RequestCache requestCache = new HttpSessionRequestCache();

	@Override
	public void onAuthenticationSuccess(HttpServletRequest request,
			HttpServletResponse response, Authentication authentication)
			throws ServletException, IOException {
		SavedRequest savedRequest = requestCache.getRequest(request, response);

		if (savedRequest == null) {
			super.onAuthenticationSuccess(request, response, authentication);

			return;
		}
		String targetUrlParameter = getTargetUrlParameter();
		if (isAlwaysUseDefaultTargetUrl()
				|| (targetUrlParameter != null && StringUtils.hasText(request
						.getParameter(targetUrlParameter)))) {
			requestCache.removeRequest(request, response);
			super.onAuthenticationSuccess(request, response, authentication);

			return;
		}

		clearAuthenticationAttributes(request);

		// Use the DefaultSavedRequest URL
		String targetUrl = savedRequest.getRedirectUrl();
		logger.debug("Redirecting to DefaultSavedRequest Url: " + targetUrl);
		getRedirectStrategy().sendRedirect(request, response, targetUrl);
	}

	public void setRequestCache(RequestCache requestCache) {
		this.requestCache = requestCache;
	}
}
```

核心方法是 onAuthenticationSuccess：

（1）首先从 requestCache 获取缓存下来的请求，如果没有获取到缓存请求，就直接调用父类 onAuthenticationSuccess 方法来处理，最终重定向到 defaultSuccessUrl 指定的地址。

（2）接下里获取一个 targetUrlParameter，这时用户指定的、希望登录成功重定向的地址。例如用户发送的登录请求时 `http://localhost:8080/doLogin?target=/hello`，这就表示用户登录成功之后，希望自动重定向到 `/hello`接口。

（3）如果 targetUrlParameter 存在，或者用户设置了 alwaysUseDefaultTargetUrl 为 true,这时候缓存下来的请求就没有意义了。此时会直接调用父类方法完成重定向。

（4）如果以上条件都不满足，最终会从缓存请求 savedRequest 中获取重定向地址，然后进行重定向操作。



自定义 SavedRequestAwareAuthenticationSuccessHandler

```java
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
                .anyRequest().authenticated()
                .and()
                .formLogin()
                .loginPage("/login.html")
                .loginProcessingUrl("/doLogin")
                .successHandler(successHandler())
                .failureUrl("/login.html")
                .usernameParameter("uname")
                .passwordParameter("passwd")
                .permitAll()
                .and()
                .csrf().disable();
    }

    SavedRequestAwareAuthenticationSuccessHandler successHandler(){
        SavedRequestAwareAuthenticationSuccessHandler successHandler = new SavedRequestAwareAuthenticationSuccessHandler();
        successHandler.setDefaultTargetUrl("/index");
        successHandler.setTargetUrlParameter("target");
        return successHandler;
    }
```

###### 前后端分离

前后端分离开发中，用户登录成功后，就不再需要页面跳转了，只需要给前端返回一个 JSON 数据即可。可通过自定义 AuthenticationSuccessHandler 的实现类实现 onAuthenticationSuccess 方法来完成。

```java
 http.authorizeRequests()
                .anyRequest().authenticated()
                .and()
                .formLogin()
                .loginPage("/login.html")
                .loginProcessingUrl("/doLogin")
                .successHandler((req,resp,authentication) ->{
                    Object principal = authentication.getPrincipal();
                    resp.setContentType("application/json;charset=utf-8");
                    PrintWriter out = resp.getWriter();
                    out.write(new ObjectMapper().writeValueAsString(principal));
                    out.flush();
                    out.close();
                })
                .failureUrl("/login.html")
                .usernameParameter("uname")
                .passwordParameter("passwd")
                .permitAll()
                .and()
                .csrf().disable();
```

##### 2.2.2.2 登录失败

与登录成功相似，登录失败也是有两个方法：

- failureForwardUrl
- failureUrl

**「这两个方法在设置的时候也是设置一个即可」**。failureForwardUrl 是登录失败之后会发生服务端跳转，**服务器端跳转的好处是可以携带登录异常信息**。failureUrl 则在登录失败之后，会发生重定向。重定向是一种客户端跳转，不方便携带请求失败的异常信息（只能放在 URL 中）。



























































































































































































































































# 参考文章

[1] [Spring Security Reference](https://docs.spring.io/spring-security/site/docs/5.4.6/reference/html5/#authentication)

[2] [Spring security 专栏](https://mp.weixin.qq.com/mp/appmsgalbum?__biz=MzI1NDY0MTkzNQ==&action=getalbum&album_id=1319828555819286528&scene=173&from_msgid=2247488106&from_itemidx=1&count=3&nolastread=1&uin=&key=&devicetype=Windows+10+x64&version=6302019a&lang=zh_CN&ascene=0&fontgear=3)

[3] 王松，《深入浅出spring security》

[4] 

