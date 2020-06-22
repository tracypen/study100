> SpringSecurity允许我们在定义URL方法访问所应有的注解权限时使用SpringEL表达式,在定义所需的访问权限时如果对应的表达式返回结果为true则表示拥有对应的权限,反之则没有权限,会进入到我们配置的UserAuthAccessDeniedHandler(暂无权限处理类)中进行处理
### 基础配置

#### UserDetailsService实现

```java
/**
 * @author haopeng
 * @date 2020-06-12 14:50
 */
@Service("userService")
public class UserServiceImpl extends ServiceImpl<UserMapper, User> implements UserService , UserDetailsService {

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        QueryWrapper<User> queryWrapper = new QueryWrapper<>();
        queryWrapper.eq("name", username);
        User user = baseMapper.selectOne(queryWrapper);
        JwtUser jwtUser = new JwtUser();
        if (null != user) {
            BeanUtil.copyProperties(user, jwtUser);
        }
         //TODO 待完善
        jwtUser.setAuthorities(Collections.singletonList(new SimpleGrantedAuthority("user")));
        return jwtUser;
    }

    @Override
    public Set<String> selectPermissionByUserId(Long id) {
         //TODO 待完善
        return Sets.newHashSet("user:list");
    }
}

```

**注意：** 这里为了尽快演示效果，用户的角色和权限都是暂时写死的 

#### 创建jwt token

```java
public String createJwt(JwtUser jwtUser) {
        String jwt = Jwts.builder()
                .setId(jwtUser.getId().toString())
                .setSubject(jwtUser.getUsername())
                .setIssuedAt(DateUtil.date())
                .setIssuer("hp")
                .signWith(SignatureAlgorithm.HS256, jwtConfig.getSecret())
                .setExpiration(DateUtil.offsetMillisecond(DateUtil.date(), jwtConfig.getExpiration() * 1000))
                .claim("roles", null)
                .claim("authorities", JSONUtil.toJsonStr(jwtUser.getAuthorities().stream().map(GrantedAuthority::getAuthority).collect(Collectors.toList())))
                .compact();
        // 将生成的JWT保存至Redis
        stringRedisTemplate.opsForValue()
                .set(REDIS_JWT_KEY_PREFIX + jwtUser.getUsername(), jwt, jwtConfig.getExpiration(), TimeUnit.SECONDS);
        return jwt;
    }
```

这里是将`UserDetails`中的角色信息包装成字符串存储在JwtToken的claim中用于之后的``JwtAuthenticationTokenFilter`r`中解析使用

#### `JwtAuthenticationTokenFilter`r认证过滤器

```java
@Component
@Slf4j
public class JwtAuthenticationTokenFilter extends OncePerRequestFilter {

    @Autowired
    private JwtUtil jwtUtil;

    @Autowired
    JwtConfig jwtConfig;

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {

        if (checkIgnores(request)) {
            filterChain.doFilter(request, response);
            return;
        }
        try {
            String jwt = jwtUtil.getJwtFromRequest(request);
            Claims claims = jwtUtil.parseJwt(jwt);

            String username = claims.getSubject();
            String userId = claims.getId();

            if (!StringUtils.isEmpty(username) && !StringUtils.isEmpty(userId)) {
                List<GrantedAuthority> authorities = new ArrayList<>();
                String authority = claims.get("authorities").toString();
                if (!StringUtils.isEmpty(authority)) {
                    JSONArray array = JSONUtil.parseArray(authority);
                    authorities = array.stream().map(Object::toString).map(SimpleGrantedAuthority::new).collect(Collectors.toList());
                }
                JwtUser jwtUser = new JwtUser();
                jwtUser.setName(claims.getSubject());
                jwtUser.setId(Long.parseLong(claims.getId()));
                jwtUser.setAuthorities(authorities);

                UsernamePasswordAuthenticationToken authentication = new UsernamePasswordAuthenticationToken(jwtUser, null, authorities);
                SecurityContextHolder.getContext().setAuthentication(authentication);
            }
        } catch (ExpiredJwtException e) {
            ResponseUtil.renderJson(response, ResultCode.TOKEN_EXPIRED);
            return;
        } catch (Exception e) {
            ResponseUtil.renderJson(response, ResultCode.TOKEN_PARSE_ERROR);
            return;
        }
        filterChain.doFilter(request, response);
    }

    private boolean checkIgnores(HttpServletRequest request) {
        String[] ignores = jwtConfig.getAntMatchers().split(",");
        if (ArrayUtil.isNotEmpty(ignores)) {
            for (String ignore : ignores) {
                AntPathRequestMatcher matcher = new AntPathRequestMatcher(ignore);
                if (matcher.matches(request)) {
                    return true;
                }
            }
        }
        return false;
    }
}
```

其中核心逻辑就是解析JwtToken解析成功后（成功登陆了）再获取`authorities`保存至`SpringSecurityContex`上下文中

之后SpringSecurity就会只用提供的角色信息进行鉴权

### 常用注解

#####  @PreAuthorize  方法拦截注解
> 一般用在Controller中用于接口级别的权限控制

#####  @hasRole 判断是否有某种角色
例如：
`@PreAuthorize("hasRole('ROLE_ADMIN', 'ROLE_USER')")`
**这里注意：**与`UserDetails`中返回的`Collection<? extends GrantedAuthority>`集合,需要加上`ROLE`前缀.否则你会发现`@PreAuthorize `中的hasRole角色不能正确被识别，这是因为Spring Security框架内的`DefaultMethodSecurityExpressionHandler `在验证表达式时是有”ROLE_”前缀的。

```java
private String defaultRolePrefix = "ROLE_";
```
当然更优雅的解决办法就是，直接去掉前缀

```java
//自定义GrantedAuthority前缀
@Bean
GrantedAuthorityDefaults grantedAuthorityDefaults() {
    return new GrantedAuthorityDefaults(""); // Remove the ROLE_ prefix
}
```



##### @ hasPermission判断是否有某种权限
> 就是直接判断`UserDetails`中返回的`Collection<? extends GrantedAuthority>`集合中的数据，如果`@hasRole`去掉`ROLE_`前缀，那么两种的效果完全相同

#####  hasPermission

> 自定义**鉴权**规则，需要自己实现`PermissionEvaluator`来完成

```java
@GetMapping("/user")
    @PreAuthorize("hasPermission('/test/user','user:list')")
    public String user() {
        return "hello spring security .. ";
    }
```

一旦使用了如上注解，那么SpringSecurity就会自动执行我们自己实现的`PermissionEvaluator`的接口，进行权限的判断，有权限就返回true，没有就返回false，即不可以访问，最终到`AccessDeniedHandler` 中完成无权限的处理逻辑

代码如下：

- 实现`PermissionEvaluator`接口

```java
@Component
public class UserPermissionEvaluator implements PermissionEvaluator {

    @Autowired
    private UserService userService;

    /**
     * 这里仅仅判断PreAuthorize注解中的权限表达式
     * @param authentication 用户身份(在使用hasPermission表达式时Authentication参数默认会自动带上)
     * @param targetUrl 请求路径
     * @param per 请求路径权限
     * @return 是否通过
     */
    @Override
    public boolean hasPermission(Authentication authentication, Object targetUrl, Object per) {
        JwtUser jwtUser = (JwtUser) authentication.getPrincipal();
        Set<String> permissions;
        // TODO 待完善
        permissions = userService.selectPermissionByUserId(jwtUser.getId());

        if (permissions.contains(per.toString())) {
            return true;
        }
        return false;
    }

    @Override
    public boolean hasPermission(Authentication authentication, Serializable serializable, String s, Object o) {
        return false;
    }
}
```

- 在SpringSecurity配置类中进行配置

```java
 /**
     * 注入自定义PermissionEvaluator
     */
    @Bean
    public DefaultWebSecurityExpressionHandler userSecurityExpressionHandler(UserPermissionEvaluator permissionEvaluator){
        DefaultWebSecurityExpressionHandler handler = new DefaultWebSecurityExpressionHandler();
        handler.setPermissionEvaluator(permissionEvaluator);
        return handler;
    }
```

之后就可以像这样使用了

```java
   @GetMapping("/user")
    @PreAuthorize("hasRole('user') and hasPermission('/test/user','user:list')")
    public String user() {
        return "hello spring security .. ";
    }
```

#### 常用注解

![](https://upload-images.jianshu.io/upload_images/8387919-09a56e06c148a88e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 最佳实践

去掉默认前缀，在UserDetailsService中`getAuthorities`返回用户的所有权限（菜单）,直接在Controller层进行细粒度鉴权