---
title: Spring Secuirty 39 React - Spring Security JWT 를 이용한 JWT 토큰 유효성 검증
author: kimdongy1000
date: 2023-07-17 14:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security , JWT ]
math: true
mermaid: true
---


지난시간에는 토큰을 만들어서 Front-end 로 쿠키 형태로 내보냈다면 이번시간에는 이 토큰을 바탕으로 해석을 해서 인증을 하는 과정을 그려보도록 하겠습니다 



## networkApiCall
```

export async function networkApiCall(api , method , param){

    const headers = new Headers();
    const project_name = f1().project_name;
   
    headers.append("Content-Type" , "application/json");
   

    const access_token = localStorage.getItem(`${project_name}_ACCESS_TOKEN`);
    if(access_token) headers.append("Authorization" , `Bearer ${access_token}`);

    let options = {

        headers : headers,
        url : f1().api_base_url + api ,
        method : method
    }

    if(!("get"== method.toLowerCase("get")) && param) options.body = JSON.stringify(param);

    return await fetch(options.url , options).then( (response) => {

       

        if(response.status === 401){

            const project_name = f1().project_name;
            localStorage.removeItem(`${project_name}_ACCESS_TOKEN`)
            window.location.href = '/login';
           
           
        }else {

            return response.json();
        }
       
       
   
    }).then( (response2) => {


        if(response2.HttpStatus === 500){        
            throw new Error(response2.errorMsg);
        }
               
        return response2
    })
}

```

제가 사용하는 React 에서 공통 api 호출입니다 여기서 localStorage JWT 를 가져와서 해당 토큰이 존재하면 헤더에 Authorization 에 Bearer 에 저장하고 항상 요청시 날아가게 만들었습니다 

```
const access_token = localStorage.getItem(`${project_name}_ACCESS_TOKEN`);
if(access_token) headers.append("Authorization" , `Bearer ${access_token}`);

```
그 부분이 이 부분에 해당하게 됩니다 




## CustomJwtAuthenticationFilter
```
public class CustomJwtAuthenticationFilter extends OncePerRequestFilter {

    private CustomJWTParser jwtParser;

    public CustomJwtAuthenticationFilter(CustomJWTParser jwtParser) {
        this.jwtParser = jwtParser;
    }

    @Autowired
    private Gson gson;



    private static final String[] ALLOW_URL = {
                "/user/register", "/user/login"
                , "/oauth2/google/oauth2_login_address"
                , "/oauth2_google_login"
                , "/oauth2/naver/oauth2_login_address"
                , "/oauth2_naver_login"
    };

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {

        String authorizationHeader = request.getHeader("Authorization");
        final String url = request.getRequestURI();
        String method = request.getMethod();

        List<String> allow_url_matcher = Arrays.stream(ALLOW_URL).filter(x -> url.equals(x)).collect(Collectors.toList());


        if (allow_url_matcher.size() > 0 || "OPTIONS".equals(method.toUpperCase())) {

            filterChain.doFilter(request, response);   

        } else {

            if (authorizationHeader != null && authorizationHeader.startsWith("Bearer ")) {

                String accessToken = authorizationHeader.substring(7); // "Bearer " 다음의 값만 추출

                try {

                    /*

                        1. JWT 토큰을 분석해서 유효성 검증
                        2. 해당 토큰으로 만들어진 것을 바탕으로 시큐리티 contextHolder 에 setAuthentication 에 저장을 하게 됩니다
                        3. 그리고 인증이 완료 되었으면 다음 filter 로 이동을 합니다 

                    */

                    Authentication authenticationToken = jwtParser.jwtParse(accessToken); 
                    SecurityContextHolder.getContext().setAuthentication(authenticationToken); 
                    filterChain.doFilter(request, response);

                } catch (Exception e) {

                    /*
                        토큰 검증 중에 에러가 발생하면 토큰 일치 여부와 관계 없이 401 에러를 뿜고 return 합니다
                    */


                    response.setStatus(HttpStatus.UNAUTHORIZED.value());
                    response.getWriter().write("UnAuthorized User");
                }
            } else {

                /*
                    토큰이 존재 하지 않아도 마찬가지로 401 에러를 return 하게 됩니다 
                */

                response.setStatus(HttpStatus.UNAUTHORIZED.value());
                response.getWriter().write("UnAuthorized User");

            }
        }
    }
}

```
이 커스텀Filter 로 JWT 를 파싱해서 다음 필터로 넘길지 아니면 401 에러를 발생시킬지 결정하는 부분입니다 

## CustomJWTParser

```
@Component
public class CustomJWTParser {

    @Autowired
    private JWK jwk;

    @Value("${spring.security.access_tokenExpireTime}")
    private Long access_tokenExpireTime;

    public Authentication jwtParse(String accessToken) throws ParseException, JOSEException {

        /*
        
            1. JWT_TOKEN 을 통해서 SignedJWT 객체를 먼저 만들어냅니다
            2. 앞전에 만든 JWK(JAVA_WEB_KEY) 를 통해서 RSAVerifier 객체를 만들고 이때 객체는 RSA 의 pubKey 로 만들어줍니다
            3. RSA 같은 비대칭키는 비밀키로 암호화 하고 공개키로 복호화를 합니다

        */

        SignedJWT signedJWT = SignedJWT.parse(accessToken); 
        RSASSAVerifier rsassaVerifier = new RSASSAVerifier((RSAKey) jwk.toPublicJWK());  

        /*verify 함수를 통해서 서명이 먼저 일치하는지 판단 */
        boolean verify = signedJWT.verify(rsassaVerifier); 


        if(!verify){
            throw new BadCredentialsException("잘못된 서명입니다.");
        }

        /*
            서명이 일치하면 일단 JWTClaimsSet 분리를 시작하고 토큰의 유효시간을 검증합니다 
        */

        JWTClaimsSet jwtClaimsSet = signedJWT.getJWTClaimsSet();  

        Date jwt_expire_time =  jwtClaimsSet.getExpirationTime();
        Date today = new Date();
        Date today_add_expire_time = new Date(today.getTime());

        /*
            서명이 일치하면 이제는 토큰의 만료시간을 체크합니다 토큰이 만료시간이 끝아면 Exception 을 던지고 끝이납니다
            사실 이때는 BadCredentialsException 보다는 다른 RunTimeException 을 추천합니다 
        */

        if(jwt_expire_time.getTime() <= today_add_expire_time.getTime()){ // 
            throw new BadCredentialsException("시간이 만료된 토큰입니다");
        }

        /*

        만료시간 유효성까지 검증이 완료되면 새로운 UserDetails 만들어서 Authentication 을 만들고 이를 return 을 하게 됩니다
        그러면 시큐리티 컨텍스트 안에는 JWT 로 만들어진 UserDetails 정보가 만들어서 들어가게 됩니다 

        */

        String username = (String)jwtClaimsSet.getClaim("username");
        List<String> authority = (List<String>) jwtClaimsSet.getClaim("authority");


        List<GrantedAuthority> array_authority = authority.stream().map(x -> new SimpleGrantedAuthority(x)).collect(Collectors.toList());

        UserDetails userDetails = new User(username , UUID.randomUUID().toString() , array_authority);
        Authentication authenticationToken = new UsernamePasswordAuthenticationToken(userDetails , null , array_authority);

        return authenticationToken;
    }
}

```
이 부분은 filter 에서 넘겨받은 access_token 으로 시큐리티컨텍스트를 만드는 과정입입니다 


## SecurityConfig

```

@Autowired
private CustomJWTParser jwtParser;


@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity httpSecurity) throws Exception{

    httpSecurity.authorizeRequests().antMatchers(   "/user/register" ,
                                                    "/user/login" ,
                                                    "/oauth2/google/oauth2_login_address" ,
                                                    "/oauth2_google_login" ,
                                                    "/oauth2/naver/oauth2_login_address" ,
                                                    "/oauth2_naver_login"
                                                ).permitAll();
    httpSecurity.authorizeRequests().anyRequest().authenticated();

    httpSecurity.csrf().disable();

    httpSecurity.formLogin().disable();
    httpSecurity.httpBasic().disable();

    httpSecurity.addFilterAfter(new CustomJwtAuthenticationFilter(jwtParser) , CorsFilter.class);


    return httpSecurity.build();
}

```

그리고 Filter 를 CorsFilter 다음으로 설정합니다 그러면 우리는 이 과정으로 인해서 Front 에서 넘어오는 발급된 JWT 토큰을 가지고 새로운 Authentication 을 만들어서 인증을 완료하게 된것입니다 
