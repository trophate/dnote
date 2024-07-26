# 配置

```java
@EnableWebSecurity// 开启
public class SecurityConfig {

    // 生成验证安全的链式过滤器
    @Bean
    public SecurityFilterChain chain(HttpSecurity http) throws Exception {
        http
                .authorizeRequests()
                .antMatchers("/login").permitAll()
                .anyRequest()
                .authenticated()
                .and()
                .formLogin();
        return http.build();
    }
}
```





# 登录验证

核心对象

![](../img/SecurityContextHolder.png)

SecurityContextHolder存储已登录的用户信息。security不关心它如何被填充，只要存在值，它就会被当做当前已登录用户。

通过SecurityContextHolder能够获取当前登录用户。



## 密码存储

BCryptPasswordEncoder

生成：

```java
// 加密强度 调整大小使得系统验证时间在1s左右。
public static final int BCRYPT_LEN = 10;

BCryptPasswordEncoder encoder = new BCryptPasswordEncoder(BCRYPT_LEN);
// 存储格式为：加密方式+加密串
String encryptedPassword = "{bcrypt}" + encoder.encode(password);
```



## 用户对象

用户（Authentication）被定义为UserDetails。

UserDetails由UserDetailsService返回。

自定义用户对象实现UserDetails和UserDetailsService即可。





# 授权

在security中没有实质上区分角色和权限，权限在实现上等同于角色，即只有用户-角色/权限。

核心对象：GrantedAuthority

GrantedAuthority用于返回一个确定的权限，它只定义了一个方法：getAuthority()。

GrantedAuthority在UserDetails中以列表存储：

```java
@Override
public Collection<? extends GrantedAuthority> getAuthorities() {
    var authorities = new ArrayList<SimpleGrantedAuthority>();
    for (String permission : permissions) {
        authorities.add(new SimpleGrantedAuthority(permission));
    }
    return authorities;
}
```



## 方法级别授权（注解）

配置：

```java
@EnableMethodSecurity
```

注解：@PreAuthorize、@PostAuthorize

例：

```java
@PreAuthorize(hasAuthority('xxx'))
```

hasRole和hasAuthority本质上没有区别，hasRole会在权限前加上前缀”ROLE_“然后进行判断。

```java
@PreAuthorize(hasAuthority('xxx'))
// =
@PreAuthorize(hasRole('ROLE_xxx'))
```





# 自动填充数据

注解：@PrePersist（插入前）、@PreUpdate（修改前）

例：

监听器：

```java
public class CustomEntityListener {

    @PrePersist
    public void setValueForInsert(Object obj) {
        if (obj instanceof SuperEntity) {
            SuperEntity entity = (SuperEntity) obj;
            CurrentUser user = SecurityUtils.getCurrentUser();
            entity.setCreatedById(user.getId());
            LocalDateTime now = LocalDateTime.now();
            entity.setCreatedAt(now);
            entity.setUpdatedById(user.getId());
            entity.setUpdatedAt(now);
        }
    }

    @PreUpdate
    public void setValueForUpdate(Object obj) {
        if (obj instanceof SuperEntity) {
            SuperEntity entity = (SuperEntity) obj;
            CurrentUser user = SecurityUtils.getCurrentUser();
            entity.setUpdatedById(user.getId());
            entity.setUpdatedAt(LocalDateTime.now());
        }
    }
}
```

实体类：

```java
// 开启监听
@EntityListeners({CustomEntityListener.class})
public class SuperEntity {

    private Integer createdById;
    private LocalDateTime createdAt;
    private Integer updatedById;
    private LocalDateTime updatedAt;
}
```





# 审计

追踪并填充创建人、创建时间、更新人、更新时间。

注解：@CreatedBy、@CreatedDate、@LastModifiedBy、@LastModifiedDate

配置：

```java
// 启用审计
@EnableJpaAuditing
public class JpaConfig {

    @Bean
    public AuditorAware<Integer> auditorProvider() {
        return () -> Optional.of(SecurityUtils.getCurrentUser().getId());
    }
}
```

泛型对应被@CreatedBy、@LastModifiedBy标识的字段的类型。

实体类：

```java
@EntityListeners(AuditingEntityListener.class)
public class SuperEntity {

    @CreatedBy
    private Integer createdById;
    @CreatedDate
    private LocalDateTime createdAt;
    @LastModifiedBy
    private Integer updatedById;
    @LastModifiedDate
    private LocalDateTime updatedAt;
}
```





# 逻辑删除

```java
public interface CustomRepository<T, ID> extends JpaRepository<T, ID> {

    @Modifying
    @Transactional
    @Query(value = "UPDATE #{#entityName} SET deleted = 1, deleted_at = NOW() WHERE id = :id", nativeQuery = true)
    void deleteByIdInLogical(@Param("id") Integer id);
}
```

```java
public interface UserRepository extends CustomRepository<User, Integer> {}
```

！用@Query自定义增删改sql时@Modifying、@Transactional是必须的。

！#{#entityName}是SpEL表达式，用于插入实体类型。如果@Entity设置了name，它会取name值，否则会取实体类的简单类名。





# 拦截器链

拦截器链将在启动时以info级别日志打印。





# 自定义认证

目标：自定义认证过程、认证结果处理

实现参考UsernamePasswordAuthenticationFilter的实现。

```java
public class CustomerUsernamePasswordAuthenticationFilter extends AbstractAuthenticationProcessingFilter {

    private static final AntPathRequestMatcher DEFAULT_ANT_PATH_REQUEST_MATCHER =
            new AntPathRequestMatcher("/login", "POST");

    private boolean postOnly = true;

    private final Logger logger = LogUtils.getLogger(this);

    public CustomerUsernamePasswordAuthenticationFilter(AuthenticationManager authenticationManager) {
        super(DEFAULT_ANT_PATH_REQUEST_MATCHER, authenticationManager);
    }

    {
        // 认证成功处理
        super.setAuthenticationSuccessHandler(new AuthenticationSuccessHandler() {
            @Override
            public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication)
                    throws IOException, ServletException {
                var objectMapper = new ObjectMapper();
                response.setContentType("application/json");
                response.setCharacterEncoding("UTF-8");
                PrintWriter out = response.getWriter();
                Result res = Result.success();
                out.print(objectMapper.writeValueAsString(res));
                out.flush();
                out.close();
            }
        });
        // 认证失败处理
        super.setAuthenticationFailureHandler(new AuthenticationFailureHandler() {
            @Override
            public void onAuthenticationFailure(HttpServletRequest request, HttpServletResponse response, AuthenticationException exception)
                    throws IOException, ServletException {
                var objectMapper = new ObjectMapper();
                response.setContentType("application/json");
                response.setCharacterEncoding("UTF-8");
                PrintWriter out = response.getWriter();
                Result res = Result.fail().setMessage("账号或密码错误");
                out.print(objectMapper.writeValueAsString(res));
                out.flush();
                out.close();
            }
        });
    }

    // 此处可以自定义认证逻辑，这里将表单传参改为了body传参。
    @Override
    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response)
            throws AuthenticationException {
        if (this.postOnly && !request.getMethod().equals("POST")) {
            throw new AuthenticationServiceException("Authentication method not supported: " + request.getMethod());
        }
        LoginDTO loginDTO = null;
        try {
            StringBuilder bodyJson = new StringBuilder();
            BufferedReader reader = request.getReader();
            String line;
            while ((line = reader.readLine()) != null) {
                bodyJson.append(line);
            }
            reader.close();
            var objectMapper = new ObjectMapper();
            loginDTO = objectMapper.readValue(bodyJson.toString(), LoginDTO.class);
        } catch (Exception e) {
            logger.error(e.getLocalizedMessage());
        }
        String username = "";
        String password = "";
        if (loginDTO != null) {
            username = loginDTO.getUsername() != null ? loginDTO.getUsername() : "";
            password = loginDTO.getPassword() != null ? loginDTO.getPassword() : "";
        }
        UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(username, password);
        setDetails(request, authRequest);
        return this.getAuthenticationManager().authenticate(authRequest);
    }

    protected void setDetails(HttpServletRequest request, UsernamePasswordAuthenticationToken authRequest) {
        authRequest.setDetails(this.authenticationDetailsSource.buildDetails(request));
    }

    public void setPostOnly(boolean postOnly) {
        this.postOnly = postOnly;
    }
}
```

配置

```java
@Bean
public SecurityFilterChain chain(HttpSecurity http, CustomSecurityContextRepository repo, AuthenticationConfiguration authConfig)
    throws Exception {
    http
        ...
        .addFilterBefore(new CustomerUsernamePasswordAuthenticationFilter(authConfig.getAuthenticationManager()),
                         SecurityContextHolderAwareRequestFilter.class)
        ...;
    return http.build();
}
```





# 获取AuthenticationManager

默认的AuthenticationManager是不能被直接获取的，而AuthenticationConfiguration被注册为了bean，因此我们可以通过AuthenticationConfiguration获取AuthenticationManager。

```java
authenticationConfiguration.getAuthenticationManager()
```





# 使用redia缓存SecurityContent

用户的认证信息保存在SecurityContent中。

自定义上下文仓库：

```java
@Component
public class CustomSecurityContextRepository extends HttpSessionSecurityContextRepository {

    /**
     * 上下文缓存键
     */
    private final static String CONTEXT_CACHE_KEY = "auth:context";

    @Resource
    private RedisTemplate<Object, Object> redisTemplate;

    @Override
    public SecurityContext loadContext(HttpRequestResponseHolder requestResponseHolder) {
        SecurityContext context = getContext(requestResponseHolder.getRequest());
        if (context == null) {
            context = generateNewContext();
            if (this.logger.isTraceEnabled()) {
                this.logger.trace(LogMessage.format("Created %s", context));
            }
        }
        return context;
    }

    @Override
    public void saveContext(SecurityContext context, HttpServletRequest request, HttpServletResponse response) {
        String secrid = SecridBuilderFilter.secrid;
        redisTemplate.opsForHash().put(CONTEXT_CACHE_KEY, secrid, context);
        redisTemplate.expire(CONTEXT_CACHE_KEY, SecurityConstant.CONTEXT_CACHE_SURVIVAL_TIME, TimeUnit.SECONDS);
    }

    @Override
    public boolean containsContext(HttpServletRequest request) {
        return getContext(request) != null;
    }

    /**
     * 获取上下文
     *
     * @param request 请求
     * @return SecurityContext
     */
    private SecurityContext getContext(HttpServletRequest request) {
        String secrid = HttpUtils.getCookieVal(request, SecurityConstant.SECRID_COOKIE_NAME);
        SecurityContext context = null;
        if (secrid != null) {
            context = redisTemplate.<String, SecurityContext>opsForHash().get(CONTEXT_CACHE_KEY, secrid);
            if (context != null) {
                redisTemplate.expire(CONTEXT_CACHE_KEY, SecurityConstant.CONTEXT_CACHE_SURVIVAL_TIME, TimeUnit.SECONDS);
            }
        }
        return context;
    }
}
```

saveContext中需要生成唯一标识（secrid）并以cookie的形式返回给前端，但调用该方法是在响应（chain.doFilter()）之后，所以不能直接在该方法里向response中添加该cookie。

secrid构建器拦截器：

```java
public class SecridBuilderFilter extends GenericFilterBean {

    public static String secrid;

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        doFilter((HttpServletRequest) request, (HttpServletResponse) response, chain);
    }

    public void doFilter(HttpServletRequest request, HttpServletResponse response, FilterChain chain) throws IOException, ServletException {
        Cookie secridCookie = HttpUtils.getCookie(request, SecurityConstant.SECRID_COOKIE_NAME);
        if (secridCookie == null) {
            secrid = RandomStringUtils.randomAlphabetic(10) + System.currentTimeMillis();
        } else {
            secrid = secridCookie.getValue();
        }
        secridCookie = new Cookie(SecurityConstant.SECRID_COOKIE_NAME, secrid);
        response.addCookie(secridCookie);
        chain.doFilter(request, response);
    }
}
```

配置：

```java
@Bean
public SecurityFilterChain chain(HttpSecurity http, CustomSecurityContextRepository repo, AuthenticationConfiguration authConfig)
    throws Exception {
    http
        ...
        .addFilterBefore(new TokenBuilderFilter(), SecurityContextHolderAwareRequestFilter.class)
        ...;
    return http.build();
}
```

