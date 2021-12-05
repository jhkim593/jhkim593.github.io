---
layout: post
title:  "Spring Security(3) - 인증 , 인가 Exception Handling"
author: "jhkim593"
tags: SpringSecurity

---

http.exceptionHandling() 을 이용해서 예외 처리 기능이 작동하도록 해보겠습니다.
AuthenticationException 인증 예외를 AuthenticationEntryPoint 처리 할 것이며, AccessDeniedException 인가 예외를 AccessDeniedHandler로 처리 할 것입니다.

<br>

### CustomAccessDeniedHandler 생성

~~~java
@Component
public class CustomAccessDeniedHandler implements AccessDeniedHandler {

    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response, AccessDeniedException accessDeniedException) throws IOException, ServletException {

        response.setContentType("application/json");
        response.setStatus(HttpServletResponse.SC_FORBIDDEN);
        response.getWriter().print("{ \"message\": \"" + "보유하신 권한으로 접근 할 수없습니다." + "\" }");
    }
}

~~~

<br>

### CustomAuthenticationEntryPoint 생성

~~~java
@Component
public class CustomAuthenticationEntryPoint implements AuthenticationEntryPoint {

    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response, AuthenticationException ex) throws IOException,
            ServletException {
        response.setContentType("application/json");
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        response.getWriter().print("{ \"message\": \"" + "인증에 실패 했습니다." + "\" }");
    }
}

~~~

<br>

### SecurityConfig exceptionHandling추가

~~~java
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
               .exceptionHandling().accessDeniedHandler(new CustomAccessDeniedHandler()).and()
               .exceptionHandling().authenticationEntryPoint(new CustomAuthenticationEntryPoint()).and() //exceptionHandling 추가   
               .addFilterBefore(new CustomAuthorizationFilter(jwtTokenProvider,userRepository),UsernamePasswordAuthenticationFilter.class)
               .addFilter(new CustomAuthenticationFilter(authenticationManager(),jwtTokenProvider));

   }

}
~~~

<br>


### CustomAuthorizationFilter 수정

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

}
~~~

<br><br>
### 테스트
테스트 코드를 작성해서 인증, 인가 예외 처리가 되는지 확인해보겠습니다.

~~~java
@Test
public void authorizationTest()throws Exception{
    mockMvc.perform(MockMvcRequestBuilders.get("/user/test")
                    .contentType(MediaType.APPLICATION_JSON).
                    header(HttpHeaders.AUTHORIZATION,"Bearer test"))
            .andExpect(status().isUnauthorized())
            .andDo(MockMvcResultHandlers.print());
}

}
~~~

~~~console
Body = { "message": "인증에 실패 했습니다." }
~~~
인증 예외 처리 확인을 위해 header에 올바르지 않은 토큰 값을 넣었는데
인증 예외 처리가 되어 메세지가 오는 것을 확인했습니다.

<br>
~~~java
@Test
public void authorizationTest()throws Exception{
    mockMvc.perform(MockMvcRequestBuilders.get("/admin/test")
                    .contentType(MediaType.APPLICATION_JSON).
                    header(HttpHeaders.AUTHORIZATION,"Bearer eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIyIiwiaWF0IjoxNjM0MDg3NDE4LCJleHAiOjE2MzQwOTEwMTh9.QjeH0TsuHbjqfoA1JcMgiBzJX02v1NCt6G9GxGJ-3XU"))
            .andExpect(status().isForbidden())
            .andDo(MockMvcResultHandlers.print());
          }

}

~~~

~~~console
Body = { "message": "보유하신 권한으로 접근 할 수없습니다." }
~~~
다음 인가 예외 처리에 대해서 확인 해 본것입니다. 현재 User는 USER 권한만을 가지고있기 때문에 /admin/test에 접근 하지 못하고
예외처리 문구가 오는 것을 확인했습니다.

<br>

-----
## Reference

https://fenderist.tistory.com/344
