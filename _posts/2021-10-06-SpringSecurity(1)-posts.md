---
layout: post
title: "Spring Security(1) - Rest API 구현을 위한 Spring Security 적용"
author: "jhkim593"
tags: SpringSecurity
---

## Spring Security
Spring Security는 Spring 기반의 애플리케이션의 보안(인증과 권한, 인가 등)을 담당하는 스프링 하위 프레임워크입니다. Spring Security는 '인증'과 '권한'에 대한 부분을 Filter 흐름에 따라 처리하고 있습니다.Filter는 Dispatcher Servlet으로 가기 전에 적용되므로 가장 먼저 URL 요청을 받지만, Interceptor는 Dispatcher와 Controller사이에 위치한다는 점에서 적용 시기의 차이가 있습니다.

### Authentication
Session에 저장되는 정보이며, Authentication은 역할에 따라서 principal, credentials, authorities, details로 구성된다.
- Principal
  - 식별된 사용자 정보를 보관한다. UserDetails의 인스턴스입니다.
  - 시스템에 따라 UserDetails 클래스를 상속하여, 커스텀한 형태로 유지할 수 있습니다.
- Credentials
  - 주체 (사용자)가 올바르다는 것을 증명하는 자격 증명이며, 보통 비밀번호를 의미합니다.
- Authorities
  - AuthenticationManager가 설정한 권한을 의미합니다.


<!-- ### AuthenticationManager
AutheticationManager는 인증을 처리하는 방법을 정의한 API이다.
- AuthenticationFilter에 의해 AuthenticationManager가 동작한다.
- 인증을 처리하면 SecurityContextHolder에 Authentication 값이 세팅된다.
 -->

<br>

### [ HTTP 상태 401(Unauthorized)]
HTTP 상태 중 401(Unauthorized)는 클라이언트가 인증되지 않았거나, 유효한 인증 정보가 부족하여 요청이 거부되었음을 의미하는 상태값입니다. 즉, 클라이언트가 인증되지 않았기 때문에 요청을 정상적으로 처리할 수 없음을 나타냅니다.

<br>

### [ HTTP 상태 403(Forbidden) ]
HTTP 상태 중 403(Forbidden)는 서버가 해당 요청을 이해했지만, 권한이 없어 요청이 거부되었음을 의미합니다. 즉, 클라이언트가 해당 요청에 대한 권한이 없다고 알려주는 것입니다. 로그인하여 인증되었지만 접근 권한이 없는 요청을 할 경우 나타나게됩니다.

<br>

### filter
SpringSecurity는 기능별 필터의 집합으로 되어있습니다.

1. SecurityContextPersistenceFilter : SecurityContextRepository에서 SecurityContext를 가져오거나 저장하는 역할을 한다.
2. LogoutFilter : 설정된 로그아웃 URL로 오는 요청을 감시하며, 해당 유저를 로그아웃 처리
3. (UsernamePassword)AuthenticationFilter : (아이디와 비밀번호를 사용하는 form 기반 인증) 설정된 로그인 URL로 오는 요청을 감시하며, 유저 인증 처리
 - AuthenticationManager를 통한 인증 실행
 - 인증 성공 시, 얻은 Authentication 객체를 SecurityContext에 저장 후 AuthenticationSuccessHandler 실행
 - 인증 실패 시, AuthenticationFailureHandler 실행
4. DefaultLoginPageGeneratingFilter : 인증을 위한 로그인폼 URL을 감시한다.
5. BasicAuthenticationFilter : HTTP 기본 인증 헤더를 감시하여 처리한다.
6. RequestCacheAwareFilter : 로그인 성공 후, 원래 요청 정보를 재구성하기 위해 사용된다.
7. SecurityContextHolderAwareRequestFilter : HttpServletRequestWrapper를 상속한 SecurityContextHolderAwareRequestWapper 클래스로 HttpServletRequest 정보를 감싼다. SecurityContextHolderAwareRequestWrapper 클래스는 필터 체인상의 다음 필터들에게 부가정보를 제공한다.
8. AnonymousAuthenticationFilter : 이 필터가 호출되는 시점까지 사용자 정보가 인증되지 않았다면 인증토큰에 사용자가 익명 사용자로 나타난다.
9. SessionManagementFilter : 이 필터는 인증된 사용자와 관련된 모든 세션을 추적한다.
10. ExceptionTranslationFilter : 이 필터는 보호된 요청을 처리하는 중에 발생할 수 있는 예외를 위임하거나 전달하는 역할을 한다.
11. FilterSecurityInterceptor : 이 필터는 AccessDecisionManager 로 권한부여 처리를 위임함으로써 접근 제어 결정을 쉽게해준다.

<br>

클라이언트가 리소스를 요청 할 때 접근권한이 없는 경우 기본적으로 로그인 폼으로 보내게 되는데 그 역할을 하는 필터는 UsernamePasswordAuthenticationFilter입니다. RestAPI에서 로그인 폼이 따로 없으므로 인증 권한이 없다는 오류 처리를 UsernamePasswordAuthenticationFilter 전에 넣어야 합니다.

본격적으로 API서버에 Spring Security를 적용해보도록 하겠습니다.
API 서버는 유저의 세션을 관리하는 것이 아닌 요청 Request 헤더에 값을 보내주면 인증이 완료되고 api 기능을 사용할 권한을 갖게 됩니다.

<br>

### build.gradle에 library 추가
~~~gradle
implementation 'org.springframework.boot:spring-boot-starter-security'
~~~

~~~java
public class CustomUserDetails implements UserDetails {
    private Long id;
    private String emil;
    private String password;
    private GrantedAuthority authorities;

    public CustomUserDetails(Long id,String email, String password, GrantedAuthority authorities) {
        this.id=id;
        this.emil = email;
        this.password = password;
        this.authorities = authorities;

    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        ArrayList<GrantedAuthority> auth = new ArrayList<>();
        auth.add(authorities);
        return auth;

    }

    @Override
    public String getPassword() {
        return password;
    }

    public Long getId() {
        return id;
    }

    @Override
    public String getUsername() {
        return emil;
    }

    @Override
    public boolean isAccountNonExpired() {
        return true;
    }

    @Override
    public boolean isAccountNonLocked() {
        return true;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }

    @Override
    public boolean isEnabled() {
        return true;
    }
}
~~~

<br>

~~~java

@RequiredArgsConstructor
@Service
public class CustomUserDetailsService implements UserDetailsService{

    private final UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String userPk) {

        User user = userRepository.findUserByEmail(userPk).orElseThrow(UserNotFoundException::new);
        return new CustomUserDetails(
                user.getId(),
                user.getEmail(),
                user.getPassword(),
                new SimpleGrantedAuthority(user.getRole()));
    }


}

~~~
DB에서 사용자 정보를 읽어 인증 객체생성을 위한 CustomUserDetailsService, SpringSecurity Context 내에 저장 할 인증 정보를 위해 CustomUserDetails 생성합니다.


<br>




~~~java

@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    private final CustomUserDetailsService customUserDetailsService;

    @Override
     protected void configure(HttpSecurity http) throws Exception {

         http.
                 httpBasic().disable()
                 .csrf().disable()
                 .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                 .and()
                 .authorizeRequests() // 다음 리퀘스트에 대한 사용권한 체크
                 .antMatchers("/test"
                 ).permitAll()
                 .antMatchers("/user/test").hasRole("USER")
                 .antMatchers("/admin/test").hasRole("ADMIN")
                 .anyRequest().authenticated()// 그외 나머지 요청은 모두 인증된 회원만 접근 가능
                 .and()
                 .addFilterBefore(new  SecurityAuthenticationFilter(customUserDetailsService,passwordEncoder), UsernamePasswordAuthenticationFilter.class);

     }

}

~~~
클라이언트가 리소스를 요청할 때 접근 권한이 없는 경우 기본적으로 로그인폼으로 보내게 되는데 그 역할을 하는 필터는 UsernamePasswordAuthenticationFilter입니다. Rest API에서 로그인폼이 따로 없으므로 인증 권한 관련처리를 UsernamePasswordAuthenticationFilter 전에 넣어야 합니다.


<br>


- csrf().disable() : 세션을 사용하지 않고 JWT 토큰을 활용하여 진행하고 REST API를 만드는 작업이기때문에 csrf 사용은 disable 처리합니다.
- sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS) : 현재 스프링 시큐리티에서 세션을 관리하지 않겠다는 뜻입니다. 서버에서 관리되는 세션 없이 클라이언트에서 요청하는 헤더에 인증 정보를 담아보낸다면 서버에서 토큰을 확인하여 인증하는 방식을 사용할 것이므로 서버에서 관리되어야할 세션이 필요없어지게 됩니다.
- permitAll() : 스프링 시큐리티에서 인증이 되지 않더라도 사용자를 허용합니다.
- authenticated() : 요청내에 스프링 시큐리티 컨텍스트 내에서 인증이 완료되어야 api를 사용할 수 있습니다. 인증이 되지 않은 요청은 403(Forbidden)에러가 발생합니다.

<br>

~~~java

public class SecurityAuthenticationFilter extends OncePerRequestFilter {
    private CustomUserDetailsService customUserDetailsService;
    private PasswordEncoder passwordEncoder;



    public SecurityAuthenticationFilter(CustomUserDetailsService customUserDetailsService,PasswordEncoder passwordEncoder){
        this.customUserDetailsService=customUserDetailsService;
        this.passwordEncoder=passwordEncoder;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {

        String header = request.getHeader("Authorization");
        if(header == null || !header.startsWith("Basic ")) {
            filterChain.doFilter(request, response);
            return;
        }
        header=header.replace("Basic ","");
        // Decode
        byte[] e = decode(header);
        String usernameWithPassword = new String(e);

        if(!usernameWithPassword.isEmpty()&&!usernameWithPassword.contains(":")){
            filterChain.doFilter(request, response);
            return;
        }
        // Split the username from the password
        String username = usernameWithPassword.substring(0, usernameWithPassword.indexOf(":"));
        String password = usernameWithPassword.substring(usernameWithPassword.indexOf(":") + 1);

        UserDetails userDetails = customUserDetailsService.loadUserByUsername(username);
        if(passwordEncoder.matches(password,userDetails.getPassword())) {

            UsernamePasswordAuthenticationToken token = new UsernamePasswordAuthenticationToken(userDetails.getUsername(), userDetails.getPassword(), userDetails.getAuthorities());
            SecurityContextHolder.getContext().setAuthentication(token);

        }
        filterChain.doFilter(request, response);
    }
}
~~~

<br>

Filter는 Request 요청마다 한번씩 호출하는 OncePerRequestFilter을 상속해 커스텀 필터를 생성 할 것이
헤더에 Base64로 인코딩된 사용자 ID와 비밀번호를 담아 API를 요청 하겠습니다.

~~~java
@RestController
public class TestController {


    @GetMapping("/test")
    public String test() {
        return "test";
    }
    @GetMapping("/user/test")
    public String userTest() {
        return "userTest";
    }
    @GetMapping("/admin/test")
    public String adminTest() {
        return "adminTest";
    }
  }
~~~
간단하게 Controller를 만들어 테스트 해보도록 하겠습니다.
User
- username : "test"
- password : "test"를 passwordEncoder를 이용하여 encode
- role     : "ROLE_USER"

~~~java
@AutoConfigureMockMvc
@SpringBootTest
public class SecurityTest {
    @Autowired
    MockMvc mockmvc;

    @Test
    public void getTest()throws Exception{

        mockmvc.perform(get("/test")
        ).andExpect(status().isOk());

    }

    @Test
    public void getUserTest()throws Exception{
        String username="test:";
        String password="test";
        byte[] bytes = (username + password).getBytes();
        String header = new String(Base64Coder.encode(bytes));
        mockmvc.perform(get("/user/test").header("Authorization","Basic "+header)
            ).andExpect(status().isOk());

    }
    @Test
    public void getAdminTest()throws Exception{
        String username="test:";
        String password="test";
        byte[] bytes = (username + password).getBytes();
        String header = new String(Base64Coder.encode(bytes));
        mockmvc.perform(get("/admin/test").header("Authorization","Basic "+header)
        ).andExpect(status().isForbidden());

    }
}

~~~

/test는 모든 사용자에게 허용되었고 /user/test는 권한이 USER인 사용자에게 허용되었고
/admin/test는 권한이 ADMIN이 아니었기 때문에 403에러가 발생하는 것을 확인했습니다.
