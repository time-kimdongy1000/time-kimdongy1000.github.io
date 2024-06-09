---
title: Spring Secuirty 36 CORS Fitler
author: kimdongy1000
date: 2023-07-17 11:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security , JWT ]
math: true
mermaid: true
---

오늘은 조그마한 미니 프로젝트 도중 일어난 일에 대해서 서술을 할려고 한다 바로 CORS 와 관련한 것이다 현재 하고 있는 미니 프로젝트는 조만간 블로그에 옮길 예정이다 다만 그전에 오늘 겪은 일에 대해서 정리를 할려고 한다 

## 소스 전체
<https://gitlab.com/kimdongy1000/public_project_amadeus/-/tree/main?ref_type=heads>

해당 소스는 민감한 정보를 제외한 순수 코드입니다 사용하실려면 application.yml 에 자신이 필요한 정보를 기입하시면 사용 가능합니다 
해당 글을 적는부분과 소스의 올라간 부분은 상당히 많이 다릅니다 


## 프로젝트 개요 
현재 하고 있는 미니 프로젝트는 React <> Spring-boot 로 간단한 웹페이지를 만들고 있다 이때 사용하는 방법은 내가 이제까지 배우고 공부한 것들을 정리하면서 하나씩 만들어가고 있는데 특히 오늘은 인증에 관한 개발을 진행하고 있다 

웹페이지에서 로그인을 하면 -> 서버에서는 JWT 토큰을 주고 그 토큰을 반환하고 그 토큰으로 다시 -> 클라이언트 반환후 -> 서버로 검증하는 로직을 만들었다 

이번에 삽질을 하게된 부부은 바로 토큰을 서버로 검증하는 로직 이 부분이다 

## CustomJwtAuthenticationFilter

```
public class CustomJwtAuthenticationFilter extends OncePerRequestFilter {

    private JWK jwk;

    public CustomJwtAuthenticationFilter(JWK jwk){
        this.jwk = jwk;
    }



    private static final String[] ALLOW_URL = {"/user/register" , "/user/login"};

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {

        String authorizationHeader = request.getHeader("Authorization");
        final String url = request.getRequestURI();
        String method = request.getMethod();

        List<String> allow_url_matcher = Arrays.stream(ALLOW_URL).filter(x-> url.equals(x)).collect(Collectors.toList());


        if(allow_url_matcher.size() > 0 || "OPTIONS".equals(method.toUpperCase())){
            filterChain.doFilter(request, response);
        }else{

            if (authorizationHeader != null && authorizationHeader.startsWith("Bearer ")) {

                try{

                    String accessToken = authorizationHeader.substring(7); // "Bearer " 다음의 값만 추출
                    SignedJWT signedJWT = SignedJWT.parse(accessToken);

                    RSASSAVerifier rsassaVerifier = new RSASSAVerifier((RSAKey) jwk.toPublicJWK());
                    boolean verify = signedJWT.verify(rsassaVerifier);

                    if(verify){

                        JWTClaimsSet jwtClaimsSet = signedJWT.getJWTClaimsSet();
                        String username = (String)jwtClaimsSet.getClaim("username");
                        List<String> authority = (List<String>) jwtClaimsSet.getClaim("authority");

                        List<GrantedAuthority> array_authority = authority.stream().map(x -> new SimpleGrantedAuthority(x)).collect(Collectors.toList());

                        UserDetails userDetails = new User(username , UUID.randomUUID().toString() , array_authority);
                        Authentication authenticationToken = new UsernamePasswordAuthenticationToken(userDetails , null , array_authority);

                        SecurityContextHolder.getContext().setAuthentication(authenticationToken);
                        filterChain.doFilter(request , response);
                    }else{

                        response.setStatus(HttpStatus.FORBIDDEN.value());
                        response.getWriter().write("Access Denied");
                    }

                }catch(Exception e){
                    throw new RuntimeException(e);
                }

            } else {
                // Bearer 토큰이 아닌 경우 403 Forbidden 응답을 보냄
                response.setStatus(HttpStatus.FORBIDDEN.value());
                response.getWriter().write("Access Denied");
            }


        }


    }
}

```

현재 이 필터는 들어오는 요청사항에 대해서 헤더의 Authorization 을 받아와서 안에 있는 토큰이 내가 만든 토큰인지 검증하고 그것이 맞다면 시큐리티 컨텍스트에 심는 로직을 개발하고 있다 
이곳은 별로 문제 되지 않는다 비대칭키로 들어온 JWT 을 헤더에서 분리해서 검증을 거친 간단한 로직이다 다만 이곳이 문제가 아니라 계속해서 CORS 오류가 발생하고 있었다 

## addCorsMappings

```
@Configuration
public class WebMvcConfig implements WebMvcConfigurer {

    @Value("${application.front.server}")
    private String front_server;


    /*
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                .allowedOrigins(front_server)
                .allowedMethods("GET" , "POST" , "PUT" , "PATCH" , "DELETE" , "OPTIONS")
                .allowedHeaders("Access-Control-Allow-Headers","Origin, X-Requested-With, Content-Type, Accept, Authorization, x-xsrf-token")
                .allowCredentials(true)
                .maxAge(3600);
    }
    */
}


```
계속 삽질을 푸게 된 이유는 이 부분인데 이 cors 가 계속해서 토큰에 넘어오는 Authorization 을 받아오지 못하고 계속해서 null 처리 되는 이슈가 발생되었다 아무리 노력을 해도 이 부분에서는 
헤더에 있는 Authorization key 값을 불러올 수 없었다 계속해서 찾고 찾다 보니 CORSFIlter 를 아예 Bean 으로 만들어서 return 하고 우선순위를 최고 우선순위로 잡으라고 한 
stack-over-flow 의 형님의 말씀이 있어서 위를 주석을 처리하고 아래와 같이 만들었다 

## CustomCorsFilter
```
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class CustomCorsFilter extends OncePerRequestFilter {

    @Value("${application.front.server}")
    private String front_server;
    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        response.setHeader("Access-Control-Allow-Origin", front_server);
        response.setHeader("Access-Control-Allow-Credentials", "true");
        response.setHeader("Access-Control-Allow-Methods", "POST, GET, PUT, OPTIONS, DELETE");
        response.setHeader("Access-Control-Max-Age", "3600");
        response.setHeader("Access-Control-Allow-Headers",
                "Origin, X-Requested-With, Content-Type, Accept, Authorization, x-xsrf-token");

        // allow cros preflight
        if ("OPTIONS".equalsIgnoreCase(request.getMethod())) {
            response.setStatus(HttpServletResponse.SC_OK);
        } else {
            filterChain.doFilter(request, response);
        }
    }
}


```

이게 지금 만들려고 하는 커스텀 CORSFilter 이다 Fitler 에 Bean 으로 정의해서 올리면 자동으로 등록이 되는 특징이 있다 이렇게 하고 나니 리액트에서 보낸 Authorization 값이 인식이 되기 시작했다 왜 그런것일까?

## 내가 생각하는것은 
```

// allow cros preflight
if ("OPTIONS".equalsIgnoreCase(request.getMethod())) {
    response.setStatus(HttpServletResponse.SC_OK);
} else {
    filterChain.doFilter(request, response);
}

```
이 부분이다 사실 CORS 라는 것은 httpMethod 중에 option 이라는 것이 있는데 이는 실제로 본요청을 하기 전에 아무것도 없는 option 으로 가서 요청을 넣을 수 있는지 없는지 먼저 선판단을 한다 이떄 헤더가 포함이 되지 않기 때문에 이 부분은 filter 에서 자동으로 걸러주어야 하는데 그렇지 못하고 있는것이다 addCorsMappings 그렇기에 좀더 세밀한 조정을 할려고 한다면 
addCorsMappings 보다는 직접 정의해서 사용하는 CustomCorsFilter 부분이 더 유연하게 개발을 할 수 있다고 생각합니다 



