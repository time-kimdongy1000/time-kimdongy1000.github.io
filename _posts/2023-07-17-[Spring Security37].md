---
title: Spring Secuirty 37 React - Spring Security JWT 를 이용한 JWT 토큰 발급 1
date: 2023-07-17 12:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security , JWT ]
math: true
mermaid: true
---

## 소스 전체
<https://gitlab.com/kimdongy1000/public_project_amadeus/-/tree/main?ref_type=heads>

해당 소스는 민감한 정보를 제외한 순수 코드입니다 사용하실려면 application.yml 에 자신이 필요한 정보를 기입하시면 사용 가능합니다 
해당 글을 적는부분과 소스의 올라간 부분은 상당히 많이 다릅니다 

## 프로젝트 구성 
JDK - 11
Srping - boot 2.7.1
Srping - Security 
Spring - resource - server
Spring - web 
mysql 
React 

이렇게 구성이 되었습니다 라이브러리는 이보다 더 많은 라이브러리가 현재 사용중이지만 필요한 것들만 나열했습니다 

우리가 만들것은 JWT 토큰을 만들어서 Return 해주는 것입니다 

## 로그인 
```

@Autowired
private Gson gson;


@PostMapping("/user/login")
public ResponseEntity<?> loginUser(
        @Valid @RequestBody LoginDto loginDto)
{
    try{

        String jwtToken = loginService.loginUser(loginDto);

        Map<String , Object> result = new HashMap<>();
        result.put("JWT_TOKEN" , jwtToken);

        String to_json_result = gson.toJson(result);


        return new ResponseEntity<>(to_json_result , HttpStatus.OK);
    }catch (Exception e){
        throw new RuntimeException(e);
    }

}

```
로그인 Controller 은 아이디 비밀번호를 받아서 최종적으로 JWT_TOKEN 을 반환 해주는 것이 목적입니다 

## Gson Bean 구성

```
@Configuration
public class GsonConfig {

    @Bean
    public Gson gson(){

        return new Gson();
    }
}

```
이때 gson 은 bean 으로 만들어서 사용중입니다 이렇게 하는 이유는 gson 은 하나의 객체로만 움직여도 문제 없을 것이라고 판단해서 이렇게 사용했습니다 


## loginUser
```
public String loginUser(LoginDto loginDto) throws Exception{

    String jwtToken = "";

    Authentication nonAuthentication = new UsernamePasswordAuthenticationToken(loginDto.getEmail() , loginDto.getPassword());
    Authentication authenticatedUser = customAuthenticationService.authenticate(nonAuthentication);

    jwtToken  = jwtGenerator.jwtGenerator(authenticatedUser , javaWebKey);



    return jwtToken;
}

```
우리는 해당 LoginDto 를 통해서 들어오는 정보를 이용해서 Authentication 작성해서 해당 Authentication 을 이용해서 jwtToken 을 만들어나갈 예정입니다 
이때 UsernamePasswordAuthenticationToken 은 Security api 를 활용해서 인증되지 않은 Authentication 을 만들고 이를 활용해서 제가 만든 customAuthenticationService 을 활용해서 
인증된 Authentication 객체를 만드는 과정입니다

## LoginDto
```
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class LoginDto {

    @NotBlank(message = "email 은 필수값입니다.")
    private String email;

    @NotBlank(message = "password 은 필수값입니다.")
    private String password;
}

```

## customAuthenticationService
```
@Service
public class CustomAuthenticationService implements AuthenticationProvider {

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Autowired
    private CustomUserDetailService customUserDetailService;

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {

        try{

            String username = authentication.getName();
            String password = authentication.getCredentials().toString();

            UserDetails userDetails = customUserDetailService.loadUserByUsername(username);

            if(!passwordEncoder.matches(password , userDetails.getPassword())){
                throw new BadCredentialsException("비밀번호가 일치하지 않습니다");
            }


            UsernamePasswordAuthenticationToken userLoginToken = new UsernamePasswordAuthenticationToken(userDetails.getUsername(), userDetails.getPassword(), userDetails.getAuthorities());


            return userLoginToken;


        }catch(Exception e){
            throw new RuntimeException(e.getMessage());

        }

    }

    @Override
    public boolean supports(Class<?> authentication) {
        return UsernamePasswordAuthenticationToken.class.isAssignableFrom(authentication);

    }
}

```
제가 커스텀을 한 인증 서비스는 시큐리티의 구현체 AuthenticationProvider 을 활용해서 재정의 구현을 하게 됩니다 이때 사용자 정보는 customUserDetailService 에서 loadByUserName 을 통해서 가져오게 됩니다 그리고 그 정보를 가지고 입력한 password 와 , 저장된 password 를 활용해서 일치여부를 판단하고 인증된 UsernamePasswordAuthenticationToken 을 만들어서 return 을 해줍니다

## CustomUserDetailService
```
@Service
public class CustomUserDetailService implements UserDetailsService {


    @Autowired
    private LoginRepository loginRepository;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {

        Project_Amdeus_User_Dao projectAmdeusUserDao = loginRepository.loadUserByUsername(username);

        if(projectAmdeusUserDao == null){
            throw new UsernameNotFoundException("회원이 존재하지 않습니다.");
        }

        /*기본권한은 member*/



        return new SecurityUsers(
                projectAmdeusUserDao.getUser_email(),
                projectAmdeusUserDao.getUser_password(),
                Arrays.asList(new SimpleGrantedAuthority("ROLE_MEMBER")),
                true ,
                true ,
                true ,
                true
        );
    }
}

```
이 부분은 최초 입력된 로그인 아이디를 바탕으로 loadUserByUsername 호출해서 시큐리티 UserDetails 을 만들어줍니다 이때 loginRepository 는 DB 에 저장된 정보를 가지고 최초 데이터를 가져옵니다 (이 부분은 생략)

그래서 최종적으로 return 하는 것은 UserDetails 타입의 정보로서 그 정보는 SecurityUsers 인데 이것은 제가 커스텀을 했습니다 

## SecurityUsers 

```
@AllArgsConstructor
public class SecurityUsers implements UserDetails {

    private final String username;
    private final String password;
    private final List<GrantedAuthority> authorities;
    private final boolean accountNonExpired;
    private final boolean accountNonLocked;

    private final boolean credentialsNonExpired;

    private final boolean enabled;




    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return this.authorities;
    }

    @Override
    public String getPassword() {
        return this.password;
    }

    @Override
    public String getUsername() {
        return this.username;
    }

    @Override
    public boolean isAccountNonExpired() {
        return this.accountNonExpired;
    }

    @Override
    public boolean isAccountNonLocked() {
        return this.accountNonLocked;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return this.credentialsNonExpired;
    }

    @Override
    public boolean isEnabled() {
        return this.enabled;
    }
}

```
UserDetails 구현하면 이 클래스도 시큐리티User 로 만들 수 있습니다 이를 활용할 예정입니다 이 부분까지가 입력한 로그인 정보를 바탕으로 UserDetails 를 만들어서 그 정보를 바탕으로 
AuthenticationToken 을 만드는 과정이었습니다 다음 포스트에서는 이 정보를 가지고 JWT 토큰을 만드는 과정을 보여드리도록 하겠습니다 






