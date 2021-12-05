---
layout: post
title: "Spring Security(2) - JWT token 적용"
author: "jhkim593"
tags: SpringSecurity
---

## JWT
JWT는 JSON Web Token의 약자로 전자 서명 된 URL-safe (URL로 이용할 수있는 문자 만 구성된)의 JSON입니다. JWT는 서버와 클라이언트 간 정보를 주고 받을 때 Http 리퀘스트 헤더에 JSON 토큰을 넣은 후 서버는 별도의 인증 과정없이 헤더에 포함되어 있는 JWT 정보를 통해 인증합니다. 이때 사용되는 JSON 데이터는 URL-Safe 하도록 URL에 포함할 수 있는 문자만으로 만듭니다. JWT는 HMAC 알고리즘을 사용하여 비밀키 또는 RSA를 이용한 Public Key/ Private Key 쌍으로 서명할 수 있습니다.

<br>

### JWT 토큰 구성

<img src="https://user-images.githubusercontent.com/53510936/136517990-60c0e485-6410-4745-8ab0-39b35f593c00.png"  width="600" height="220" style="margin-left: auto; margin-right: auto; display: block;"/>



JWT는 세 파트로 나누어지며, 각 파트는 점으로 구분하여 xxxxx.yyyyy.zzzzz 이런식으로 표현됩니다. 순서대로 헤더 (Header), 페이로드 (Payload), 서명 (Sinature)로 구성합니다.
Base64 인코딩의 경우 “+”, “/”, “=”이 포함되지만 JWT는 URI에서 파라미터로 사용할 수 있도록 URL-Safe 한  Base64url 인코딩을 사용합니다. <br><br>
Header는 토큰의 타입과 해시 암호화 알고리즘으로 구성되어 있습니다. 첫째는 토큰의 유형 (JWT)을 나타내고, 두 번째는 HMAC, SHA256 또는 RSA와 같은 해시 알고리즘을 나타내는 부분입니다.<br><br>
Payload는 토큰에 담을 클레임(claim) 정보를 포함하고 있습니다. Payload 에 담는 정보의 한 ‘조각’ 을 클레임이라고 부르고, 이는 name / value 의 한 쌍으로 이뤄져있습니다. 토큰에는 여러개의 클레임 들을 넣을 수 있습니다.
클레임의 정보는 등록된 (registered) 클레임, 공개 (public) 클레임, 비공개 (private) 클레임으로 세 종류가 있습니다.<br><br>
마지막으로 Signature는 secret key를 포함하여 암호화되어 있습니다.<br>

<br>

### JWT 장점과 단점

- JWT 장점
<br><br>
JWT 의 주요한 이점은 사용자 인증에 필요한 모든 정보는 토큰 자체에 포함하기 때문에 별도의 인증 저장소가 필요없다는 것입니다. 분산 마이크로 서비스 환경에서 중앙 집중식 인증 서버와 데이터베이스에 의존하지 않는 쉬운 인증 및 인가 방법을 제공합니다.

- JWT 단점
<br><br>
토큰은 클라이언트에 저장되어 데이터베이스에서 사용자 정보를 조작하더라도 토큰에 직접 적용할 수 없습니다.
더 많은 필드가 추가되면 토큰이 커질 수 있습니다. 비상태 애플리케이션에서 토큰은 거의 모든 요청에 대해 전송되므로 데이터 트래픽 크기에 영향을 미칠 수 있습니다.

### build.gradle에 library 추가
~~~gradle
implementation 'io.jsonwebtoken:jjwt:0.9.1'
~~~
<br>

### yml파일에 secretKey 추가
~~~yml
jwt:
  secret: secret@&$
~~~

<br>

### JwtTokenProvider 생성

~~~java
@RequiredArgsConstructor
@Component
public class JwtTokenProvider { // JWT 토큰을 생성 및 검증 모듈

    @Value("spring.jwt.secret")
    private String secretKey;

    private long tokenValidMilisecond = 1000L * 60 * 60; // 1시간만 토큰 유효

    private final CustomUserDetailsService userDetailsService;

    @PostConstruct
    protected void init() {
        secretKey = Base64.getEncoder().encodeToString(secretKey.getBytes());
    }

    // Jwt 토큰 생성
    public String createToken(String userPk
    ) {
        Claims claims = Jwts.claims().setSubject(userPk);
        Date now = new Date();
        return Jwts.builder()
                .setClaims(claims) // 데이터
                .setIssuedAt(now) // 토큰 발행일자
                .setExpiration(new Date(now.getTime() + tokenValidMilisecond)) // set Expire Time
                .signWith(SignatureAlgorithm.HS256, secretKey) // 암호화 알고리즘, secret값 세팅
                .compact();
    }

    // Jwt 토큰에서 회원 구별 정보 추출
    public String getUserPk(String token) {
        return Jwts.parser().setSigningKey(secretKey).parseClaimsJws(token).getBody().getSubject();
    }
    public String resolveToken(HttpServletRequest req) {
        return req.getHeader("Authorization");
    }

    //토큰 유효성 검증
    public boolean validateToken(String jwtToken) throws Exception {

        Jws<Claims> claims = Jwts.parser().setSigningKey(secretKey).parseClaimsJws(jwtToken);
        return !claims.getBody().getExpiration().before(new Date());


    }
}

~~~

JWT토큰 생성 및 유효성 검증을 하는 JwtTokenProvider를 생성합니다. claim 정보에 회원을 구분할수 있는 값을 세팅할 것이며 여기서 생성된 토큰은 리소스를 요청할 때 header에 넣어 사용자를 인증하는 용도로 사용될 것입니다.

<br>

### CustomAuthenticFilter 생성
~~~java
public class CustomAuthenticationFilter extends UsernamePasswordAuthenticationFilter {

    private ObjectMapper mapper=new ObjectMapper();
    private AuthenticationManager authenticationManager;
    private JwtTokenProvider jwtTokenProvider;
    public CustomAuthenticationFilter(AuthenticationManager authenticationManager, JwtTokenProvider jwtTokenProvider){
        this.authenticationManager=authenticationManager;
        this.jwtTokenProvider=jwtTokenProvider;
    }

    @Override
    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) throws AuthenticationException {

        Authentication authenticate=null;
        PrintWriter writer=null;

        try {
            response.setContentType("application/json");
            response.setCharacterEncoding("utf-8");
            writer=response.getWriter();
            UserLoginDto userLoginDto = mapper.readValue(request.getInputStream(), UserLoginDto.class);
            UsernamePasswordAuthenticationToken token=new UsernamePasswordAuthenticationToken(userLoginDto.getEmail(),userLoginDto.getPassword());
            authenticate = authenticationManager.authenticate(token);

        }catch (Exception e){
            response.setStatus(401);
            writer.print(mapper.writeValueAsString("로그인에 실패 했습니다."));
        }
        finally {
            return authenticate;
        }
    }

    @Override
    protected void successfulAuthentication(HttpServletRequest request, HttpServletResponse response, FilterChain chain, Authentication authResult) throws IOException, ServletException {
        CustomUserDetails principal = (CustomUserDetails) authResult.getPrincipal();
        String token = jwtTokenProvider.createToken(String.valueOf(principal.getId()));
        PrintWriter writer=response.getWriter();
        UserLoginResponseDto responseDto=new UserLoginResponseDto("JWT","Bearer "+token);
        writer.print(mapper.writeValueAsString(responseDto));

    }
}

~~~
/login 요청시 입력 받은 email,password를 이용하여 사용자 인증을 진행하는 필터입니다.  AuthenticationManager 기본 구현체인 ProviderManager는 여러 Provider들을 사용하게 되며 인증이 완료되면 Authentication 객체를 반환하게 되는데 여기서는 userDetailsService를 이용해서 입력받은 값에 대한 사용자 정보를 DB에서 조회하게 됩니다.이 후 successfulAuthentication에서 인증된 사용자 정보를 기반으로 토큰을 생성해서 보내줍니다.



<br>

### CustomAuthorizationFilter 생성

~~~java

public class CustomAuthorizationFilter extends OncePerRequestFilter {

    private JwtTokenProvider jwtTokenProvider;
    private UserRepository userRepository;
    private ObjectMapper objectMapper=new ObjectMapper();

    public CustomAuthorizationFilter(JwtTokenProvider jwtTokenProvider, UserRepository userRepository){
        this.jwtTokenProvider=jwtTokenProvider;
        this.userRepository=userRepository;
    }
    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {

        String token = jwtTokenProvider.resolveToken(request);
        PrintWriter writer=null;

        if(token == null || !token.startsWith("Bearer")) {
            filterChain.doFilter(request, response);
            return;
        }
        token=token.replace("Bearer ","");
        try{
            response.setCharacterEncoding("utf-8");
            response.setContentType("application/json");


        if(token != null && jwtTokenProvider.validateToken(token)) {

            Long userId= Long.valueOf(jwtTokenProvider.getUserPk(token));
            User user = userRepository.findById(userId).orElseThrow(UserNotFoundException::new);
            CustomUserDetails customUserDetails=new CustomUserDetails(user.getId(),user.getEmail(),user.getPassword(),new SimpleGrantedAuthority(user.getRole()));
            UsernamePasswordAuthenticationToken authenticationToken=new UsernamePasswordAuthenticationToken(customUserDetails,null, customUserDetails.getAuthorities());
            SecurityContextHolder.getContext().setAuthentication(authenticationToken);
        }

        filterChain.doFilter(request, response);
      }
        catch (UserNotFoundException e){
            writer= response.getWriter();
            response.setStatus(401);
            writer.print(objectMapper.writeValueAsString("등록된 회원이 아닙니다."));
        }
        catch(ExpiredJwtException e) {
            writer= response.getWriter();
            response.setStatus(401);
            writer.print(objectMapper.writeValueAsString("토큰 유효기간이 만료되었습니다."));
        }
        catch (Exception e) {
            writer= response.getWriter();
            response.setStatus(401);
            writer.print(objectMapper.writeValueAsString("토큰 인증에 실패했습니다."));

        }
    }

}
~~~
header에 token을 넣어 제한된 리소스를 요청하게 되는데 인증정보를 SecurityContext에 담아 저장하게 됩니다.

<br>


### SecurityCofig 수정
~~~java
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig extends WebSecurityConfigurerAdapter {


    private final JwtTokenProvider jwtTokenProvider;
    private final UserRepository userRepository;

    @Override
    protected void configure(HttpSecurity http) throws Exception {

        http.
                csrf().disable()
                .formLogin().disable()
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                .and()
                .authorizeRequests() // 다음 리퀘스트에 대한 사용권한 체크
                .antMatchers("/test"
                ).permitAll()
                .antMatchers("/user/test").hasRole("USER")
                .antMatchers("/admin/test").hasRole("ADMIN")
                .anyRequest().authenticated()// 그외 나머지 요청은 모두 인증된 회원만 접근 가능
                .and()
                .addFilterBefore(new CustomAuthorizationFilter(jwtTokenProvider,userRepository),UsernamePasswordAuthenticationFilter.class)
                .addFilter(new CustomAuthenticationFilter(authenticationManager(),jwtTokenProvider));

    }

}
~~~

<br>
### 테스트
테스트를 위해 DB User 테이블의 email에는 "test" password에는 "test"를 PasswordEncoder로 인코딩 한 값을 권한은 ROLE_USER를 삽입해 저장했습니다.

~~~java
@SpringBootTest
@AutoConfigureMockMvc

public class SecurityTest {
    @Autowired
    MockMvc mockMvc;

    @Autowired
    ObjectMapper objectMapper;

    @Test
    public void AuthenticationTest()throws Exception{
        //given
        String content = objectMapper.writeValueAsString(new UserLoginDto("test", "test"));

        mockMvc.perform(MockMvcRequestBuilders.post("/login")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(content))
                        .andDo(MockMvcResultHandlers.print())
                .andExpect(status().isOk());


    }

}

~~~

~~~console
MockHttpServletResponse:
           Status = 200
    Error message = null
          Headers = [Content-Type:"application/json;charset=utf-8", X-Content-Type-Options:"nosniff", X-XSS-Protection:"1; mode=block", Cache-Control:"no-cache, no-store, max-age=0, must-revalidate", Pragma:"no-cache", Expires:"0", X-Frame-Options:"DENY"]
     Content type = application/json;charset=utf-8
             Body = {"type":"JWT","accessToken":"Bearer eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIyIiwiaWF0IjoxNjM0MDI1MzExLCJleHAiOjE2MzQwMjg5MTF9.prbjm2UiFruPUKFaxQHkjgJfkihVz_XFlkUQazhPmyc"}
    Forwarded URL = null
   Redirected URL = null
          Cookies = []
~~~

/login 요청시 사용자 인증이 완료되어 토큰이 발급되는 것을 확인했습니다.

<br>
<br>

~~~java
@Test
public void AuthenticationTest()throws Exception{
    //given
    String content = objectMapper.writeValueAsString(new UserLoginDto("t", "test"));

    mockMvc.perform(MockMvcRequestBuilders.post("/login")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(content))
                    .andDo(MockMvcResultHandlers.print())
            .andExpect(status().isOk());
}
~~~


~~~console
MockHttpServletResponse:
           Status = 401
    Error message = null
          Headers = [Content-Type:"application/json;charset=utf-8", X-Content-Type-Options:"nosniff", X-XSS-Protection:"1; mode=block", Cache-Control:"no-cache, no-store, max-age=0, must-revalidate", Pragma:"no-cache", Expires:"0", X-Frame-Options:"DENY"]
     Content type = application/json;charset=utf-8
             Body = "로그인에 실패 했습니다."
    Forwarded URL = null
   Redirected URL = null
          Cookies = []
~~~
DB User 테이블 정보와 일치하지 않으면 토큰이 발급되지 않는 것을 확인했습니다.
<br>
<br>


~~~java
@Test
    public void authorizationTest()throws Exception{
        mockMvc.perform(MockMvcRequestBuilders.get("/user/test")
                        .contentType(MediaType.APPLICATION_JSON).
                        header(HttpHeaders.AUTHORIZATION,"Bearer eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIyIiwiaWF0IjoxNjM0MDI2NTE3LCJleHAiOjE2MzQwMzAxMTd9.J7IPodrbxX2J8FTtKXINFgePZI_0IoiLCG2ZISUlYJg"))
                .andExpect(status().isOk())
                .andDo(MockMvcResultHandlers.print());
    }
~~~
~~~console
MockHttpServletResponse:
           Status = 200
    Error message = null
          Headers = [Content-Type:"application/json;charset=utf-8", Content-Length:"8", X-Content-Type-Options:"nosniff", X-XSS-Protection:"1; mode=block", Cache-Control:"no-cache, no-store, max-age=0, must-revalidate", Pragma:"no-cache", Expires:"0", X-Frame-Options:"DENY"]
     Content type = application/json;charset=utf-8
             Body = userTest
    Forwarded URL = null
   Redirected URL = null
          Cookies = []
~~~
발급 받은 token을 헤더에 담아 요청을 보내서 권한을 확인받아 리소스 사용이 허용된 것을 확인했습니다.

<br><br>
~~~java
@Test
    public void authorizationTest()throws Exception{
        mockMvc.perform(MockMvcRequestBuilders.get("/admin/test")
                        .contentType(MediaType.APPLICATION_JSON).
                        header(HttpHeaders.AUTHORIZATION,"Bearer eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIyIiwiaWF0IjoxNjM0MDI2NTE3LCJleHAiOjE2MzQwMzAxMTd9.J7IPodrbxX2J8FTtKXINFgePZI_0IoiLCG2ZISUlYJg"))
                .andExpect(status().isForbidden())
                .andDo(MockMvcResultHandlers.print());
    }
~~~

~~~console
MockHttpServletResponse:
           Status = 403
    Error message = Forbidden
          Headers = [Content-Type:"application/json;charset=utf-8", X-Content-Type-Options:"nosniff", X-XSS-Protection:"1; mode=block", Cache-Control:"no-cache, no-store, max-age=0, must-revalidate", Pragma:"no-cache", Expires:"0", X-Frame-Options:"DENY"]
     Content type = application/json;charset=utf-8
             Body =
    Forwarded URL = null
   Redirected URL = null
          Cookies = []
~~~
테스트에 사용된 USER는 USER 권한만을 갖고있기 때문에 /admin/test 접근이 허용되지 않는 것을 확인했습니다.

## @PreAuthorize , @Secured
권한 설정이 필요한 리소스에 @PreAuthorize , @Secured 어노테이션을 이용해 권한을 세팅할 수 있습니다.
둘다 역할은 같지만 아래와 같은 차이가 있습니다.

<br>

@PreAuthorize : 표현식 사용 가능

~~~java
@PreAuthorize("hasRole('ROLE_USER') and hasRole('ROLE_ADMIN')")
~~~
@Secured : 표현식 사용 불가능
~~~java
@Secured({"ROKE_USER","ROLE_ADMIN"})
~~~
어노테이션으로 권한 설정을 하려면 GlobalMethodSecurity를 활성화해야합니다.
~~~java
@EnableGlobalMethodSecurity(securedEnabled = true, prePostEnabled = true)
~~~
