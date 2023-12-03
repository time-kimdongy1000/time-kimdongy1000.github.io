---
title: Spring Secuirty 34 MAC 기반 Authentication 프로젝트
author: kimdongy1000
date: 2023-07-16 10:00
categories: [Back-end, Spring - Security]
tags: [ Spring-Security , JWT ]
math: true
mermaid: true
---

이번시간에는 회원가입 + 로그인시 form 로그인 이용 로그인이 완료되면 JWT 발급 이때 사용할 암호화 방식은 MAC 방식으로 진행할것입니다 그리고 로그인이 완료되면 JWT return 해서 
JWT 로 통신하는 방법에 대해서 기술하겠습니다 

## git 소스
https://gitlab.com/kimdongy1000/spring_security_web/-/tree/main_mac_authentication_project?ref_type=heads

## 사용기술
JDK 11
springboot 2.7.1
jpa 
spring-security-resource-server
lombok
spring-web 
thyleamf 
bootstrap

## maven 
```
<dependencies>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-security</artifactId>
	</dependency>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-web</artifactId>
	</dependency>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-test</artifactId>
		<scope>test</scope>
	</dependency>
	<dependency>
		<groupId>org.springframework.security</groupId>
		<artifactId>spring-security-test</artifactId>
		<scope>test</scope>
	</dependency>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
	</dependency>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-thymeleaf</artifactId>
	</dependency>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-data-jpa</artifactId>
	</dependency>
	<dependency>
		<groupId>com.h2database</groupId>
		<artifactId>h2</artifactId>
		<scope>runtime</scope>
	</dependency>
	<dependency>
		<groupId>org.projectlombok</groupId>
		<artifactId>lombok</artifactId>
		<version>1.18.30</version>
		<scope>provided</scope>
	</dependency>
</dependencies>
```

maven 은 간략하게 전달하겠습니다 

## 회원가입부터 회원가입은 생략 (자세한것은 git 소스 참조부탁드립니다)

## JpaAuthenticationProviderManager
```
public class JpaAuthenticationProviderManager implements AuthenticationProvider {

    @Autowired
    private JpaUSerDetailsService jpaUSerDetailsService;

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Override
    public Authentication authenticate(Authentication authentication) throws AuthenticationException {

        String email = authentication.getName();
        String password = authentication.getCredentials().toString();

        UserDetails user = jpaUSerDetailsService.loadUserByUsername(email);

        if(!passwordEncoder.matches(password ,  user.getPassword())) throw new BadCredentialsException("비밀번호가 일치하지 않습니다.");

        UsernamePasswordAuthenticationToken authenticationToken = UsernamePasswordAuthenticationToken.authenticated(user , authentication.getCredentials() , user.getAuthorities());


        return authenticationToken;
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return UsernamePasswordAuthenticationToken.class.isAssignableFrom(authentication);
    }
}
```

이 부분에서는 JPA 에 등록된 user 데이터를 가져와서 인증이 완료 되었으면 authenticationToken 을 반환합니다 이때 UserDetails 객체는 user 는 

## JpaUSerDetailsService

```
@Component
public class JpaUSerDetailsService implements UserDetailsService {

    @Autowired
    private UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {

        Optional<UserEntity> optionalUser = userRepository.findByUsername(username);

        if(!optionalUser.isPresent()) throw new UsernameNotFoundException(username);


        UserEntity userEntity = optionalUser.get();
        List<GrantedAuthority> grantedAuthorities = new ArrayList<>();
        grantedAuthorities.add(new SimpleGrantedAuthority(userEntity.getAuthority()));
        UserDetails user = new User(userEntity.getUsername() , userEntity.getPassword() ,  grantedAuthorities);

        return user;
    }
}

```
마찬가지로 UserDetailsSServicer 는 repository 에서 User 객체를 가져오고 username 으로 조회한뒤 있으면 UserDetails 을 일차적으로 반환 위의 JpaAuthenticationProviderManager
에서 비밀번호를 검증하게 됩니다 

다시 JpaAuthenticationProviderManager 넘어오게 되면

```

UsernamePasswordAuthenticationToken authenticationToken = UsernamePasswordAuthenticationToken.authenticated(user , authentication.getCredentials() , user.getAuthorities());

```

여기서 이제 인증이 성공이 되었으니 JpaUsernamePasswordSuccessHandler 에서 이제 JWT 토큰을 발급하기 위한 작업을 진행을 합니다 

## JpaUsernamePasswordSuccessHandler
```
public class JpaUsernamePasswordSuccessHandler implements AuthenticationSuccessHandler {

    private JwtGenerator jwtGenerator;

    private OctetSequenceKey jwtKeyGenerator;

    public JpaUsernamePasswordSuccessHandler(JwtGenerator jwtGenerator , OctetSequenceKey jwtKeyGenerator){
        this.jwtGenerator = jwtGenerator;
        this.jwtKeyGenerator = jwtKeyGenerator;
    }



    @Override
    public void onAuthenticationSuccess(HttpServletRequest request, HttpServletResponse response, Authentication authentication) throws IOException, ServletException {

        UserDetails userDetails = (UserDetails) authentication.getPrincipal();

        String jwtToken;
        try {

            jwtToken = jwtGenerator.jwtGenerator(userDetails , jwtKeyGenerator);
            Cookie cookie = new Cookie("jwtToken" , jwtToken);
            cookie.setPath("/");
            response.addCookie(cookie);
            response.sendRedirect("/main/");


        } catch (Exception e) {
            throw new RuntimeException(e);
        }


    }
}


```
결국에는 onAuthenticationSuccess 에서 UserDetails 를 받아서 jwtToken = jwtGenerator.jwtGenerator(userDetails , jwtKeyGenerator); 사용해서 jwtToken 을 만들어내고 
그것을 Cookie 로 반환을 합니다 (Header 로 못한 이유는 리다이렉시 header 는 같이 넘기지 못하는 이슈가 있어서 Cookie 로 넘깁니다)

## JwtGenerator
```
public class JwtGenerator {

    public String jwtGenerator(UserDetails userDetails , OctetSequenceKey jwtKeyGenerator) throws Exception{

		/*
		* OctetSequenceKey 는 JWK 세트에 속하는 키 중 하나로 대칭키를 표현할때 사용하게 됩니다 
		* JWK Json Web Key 디지털 서명을 위한 키를 나타내는 표준입니다 
		* 암호화와 복호화에 동일한 비밀 키를 사용하는 알고리즘에서 사용되는 키입니다
		*/
        OctetSequenceKey octetSequenceKey =  jwtKeyGenerator; 
		


		/*
        * octetSequenceKey 는 기본적으로 만들때
        * keySize 와 비밀키 및 알고리즘을 지정해서 대칭키 를 만들 수 있습니다
        * KeySize 는 key 길이를 나타내는것으로 spring - security 는 기본적으로 256 이상으로 만들어야 합니다 
        * algorithm 는 암호화 할때 사용하는 어떤 방식의 알고리즘을 사용할지 정하게 됩니다 
        * */
        JWSAlgorithm jwsAlgorithm = (JWSAlgorithm) octetSequenceKey.getAlgorithm();
		String keyId = octetSequenceKey.getKeyID();
        SecretKey secretKey =  octetSequenceKey.toSecretKey();

        List<String>  authorities = userDetails.getAuthorities().stream().map( x-> {return x.getAuthority();}).collect(Collectors.toList());
        authorities.add("EMAIL");
        authorities.add("PROFILE");

		/* JWSHeader 은 JWT 를 만들떄 헤더에 속하는 데이터로 이때는 알고리즘과 를 포함한 값의 길이를 반환합니다
         *
         */
        JWSHeader jwtHeader = new JWSHeader.Builder((JWSAlgorithm) octetSequenceKey.getAlgorithm()).keyID(keyId).build();

		/*
        * JWTClaimsSet 는 payload 를 나타내는것으로 이에 대해서는 앞전에 한번 설명드린적이 있으므로 pass 하겠습니다
        *
        * */
        JWTClaimsSet jwtPayload = new JWTClaimsSet.Builder()
                .subject("user")
                .issuer("httpL//localhost:8080")
                .claim("username" , userDetails.getUsername())
                .claim("authority" , authorities)
                .expirationTime(new Date(new Date().getTime() + 60 * 1000 * 5))
                .build();

		/*
        * 그리고 이 부분이 서명부분이다 MACSigner 부분으로 secretKey 서명을 만들고
		*
        * */
        MACSigner jwsSigner = new MACSigner(secretKey);


		/*
        * JWT 를 서명할때는 헤더와 payload 가 필요함으로 SignedJWT 객체에 첨부후
        *
        * */
        SignedJWT signedJWT = new SignedJWT(jwtHeader , jwtPayload);

		 /*
        * 만든 SecretKey 로 서명을 하게 됩니다
        *
        * */
        signedJWT.sign(jwsSigner);

        String token = signedJWT.serialize();

        return token;
    }
}

```
JwtGenerator 는 User 정보를 기반으로 JWT 객체를 만드는 역활입니다 


## JwtKeyGenerator
```
@Configuration
public class JwtKeyGenerator {

    @Value("${spring.security.keySize:256}")
    private int keySeize;

    @Value("${spring.security.secretKey:application}")
    private String secretKey;


    @Bean
    public OctetSequenceKey createToken() throws Exception{
        OctetSequenceKey octetSequenceKey = new OctetSequenceKeyGenerator(keySeize).keyID(secretKey).algorithm(JWSAlgorithm.HS256).generate();
        return octetSequenceKey;
    }
}


```
이 부분이 위에서 JwtGenerator 에서 사용하는 MAC 타입의 JWK 를 만드는 부분입니다 보시다 싶히 keySize 와 알고리즘 그리고 secretKey를 넣어서 생성을 해서 이를 공통적으로 사용을 해야 하기 때문에 이를 Bean 으로 등록을 합니다 이때 value 값은 application.properties 에 저장을 하고 그게 없으면 설정한 디폴트 값을 설정했습니다 


## JwtAuthenticationFilter
처음에 form 로그인으로 사용자를 찾아서 jwt 를 발급받아서 되돌아오게 되면 이제는 jwt 를 통해서 통신을 할 수 있게끔 만들어야 합니다 

```
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private OctetSequenceKey jwtKeyGenerator;

    public JwtAuthenticationFilter(OctetSequenceKey jwtKeyGenerator){
        this.jwtKeyGenerator = jwtKeyGenerator;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {


        /*
        * 쿠키 탐색 쿠키가 없으면 그냥 다음 filter 통과
        * */
        Cookie[]  cookies = request.getCookies();
        if(cookies == null ){
            filterChain.doFilter(request , response);
            return;
        }

        /*
        * 쿠키 탐색을 하면서 우리가 앞에서 넣어주는 쿠를 찾게 됩니다
        *
        * */

        for (Cookie cookie : cookies) {
            String request_cookie_name = cookie.getName();

            if("jwtToken".equals(request_cookie_name)){
                
                /*
                * 쿠키에서 값을 분리 
                * 
                * */
                String token =  cookie.getValue();
                
                try{

                    /* bean 으로 등록한 MAC 기반의 JWK 를 불러와서 넣음 
                     * 
                     * */
                    OctetSequenceKey octetSequenceKey = jwtKeyGenerator;
                    
                    /*
                     *
					 * token 을 parse 로 넣어서 서명을 검토하기 위한 SignedJWT 객체를 만들게 됩니다
                     */
                    SignedJWT signedJWT = SignedJWT.parse(token);
                    
                    /*
                    * Bean 으로 등록한 JWK 를 기반으로 서명이 일치하는지 아닌지 확인을 하기 위해서 
                    * MACVerifier 객체를 만들게 됩니다 
                    * */
                    MACVerifier macVerifier = new MACVerifier(octetSequenceKey.toSecretKey());

                    /*
                    * 그리고 JWT 의 서명부분과 시크릿 key 를 넣은 MACVerifier 를 통해서 verify 를 불러오게 넣게 되면 
                    * true , false 를 반환하게 됩니다 
                    * 
                    * */
                    boolean verify = signedJWT.verify(macVerifier);

                    /*
                    * true 가 되면 여기서 발급한 jwt 가 맞기 때문에 
                    * 이제 인증과정으로 가게 됩니다 
                    * 
                    * */
                    if(verify){
                        
                        /*
                        * 
                        * JWTClaimsSet 으로 값을 분리 
                        * 
                        * */
                        JWTClaimsSet jwtClaimsSet = signedJWT.getJWTClaimsSet();
                        String username = (String)jwtClaimsSet.getClaim("username");
                        List<String> authority = (List<String>) jwtClaimsSet.getClaim("authority");

                        if(username == null) throw new UsernameNotFoundException("username 을 찾을 수 없습니다");
                        if(authority.isEmpty()) throw new RuntimeException("등록된 권한이 없습니다");

                        List<GrantedAuthority> array_authority = authority.stream().map(x -> new SimpleGrantedAuthority(x)).collect(Collectors.toList());

                        /*
                        * JWT 에서 분리한 값에 username , authority 로 새로운 UserDetails 를 만들게 됩니다 
                        * 이때 비밀번호는 없기 때문에 임시 비밀번호를 발급해서 새로운 User 를 넣게 되고 
                        * 
                        * */
                        UserDetails userDetails = new User(username , UUID.randomUUID().toString() , array_authority);
                        
                        /*
                        * 인증객체를 UsernamePasswordAuthenticationToken 넘겨서 인증을 받게끔 위임합니다 
                        * 
                        * */

                        Authentication authenticationToken = new UsernamePasswordAuthenticationToken(userDetails , null , array_authority);
                        
                        /*
                        * 그리고 인증이 완료되면 Authentication 객체를 SecurityContextHolder 심는것으로 끝입니다 
                        * */

                        SecurityContextHolder.getContext().setAuthentication(authenticationToken);

                        filterChain.doFilter(request , response);
                        return;
                    }
					
                }catch(Exception e){
                    throw new RuntimeException(e);
                }
            }
        }
        filterChain.doFilter(request , response);
        return;
    }
}
```

## MainController
```
@Controller
@RequestMapping("/main")
public class MainController {

    @GetMapping("/")
    @ResponseBody
    public Authentication mainPage(Authentication authentication){

        return authentication;
    }
}
```
그러면 로그인 성공시 main 으로 핸들러가 찾아올때 현재 로그인한 상태 이때는 form 로그인 상태가 아닌 JWT 토큰 으로 발급받은 Authentication 타입의 객체입니다 

## SecurityConfig
```
@Configuration
public class SecurityConfig {


    @Autowired
    private OctetSequenceKey keygenerator;


    @Bean
    public SecurityFilterChain securityFilterChain (HttpSecurity http) throws Exception {

        http.authorizeRequests().antMatchers("/" ,
                                                        "/signUp/**" ,
                                                        "/bootStrap/css/**" ,
                                                        "/bootStrap/js/**" ,
                                                        "/resources/css/**" ,
                                                        "/resources/js/**" ,
                                                        "/favicon.ico"

                                                            ).permitAll().anyRequest().authenticated();


        http.formLogin()
                .loginPage("/login/")
                .loginProcessingUrl("/login/")
                .usernameParameter("email")
                .passwordParameter("password")
                .successHandler(authenticationSuccessHandler())
                .permitAll();


        http.sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS);
        http.authenticationProvider(authenticationProvider());

        http.addFilterBefore(jwtAuthenticationFilter(keygenerator) , UsernamePasswordAuthenticationFilter.class);



        http.csrf().disable();

        return http.build();
    }

    @Bean
    public AuthenticationProvider authenticationProvider(){
        return new JpaAuthenticationProviderManager();
    }

    @Bean
    public AuthenticationSuccessHandler authenticationSuccessHandler(){
        return new JpaUsernamePasswordSuccessHandler(jwtGenerator() , keygenerator);
    }
    @Bean
    public JwtGenerator jwtGenerator(){
        return new JwtGenerator();
    }

    @Bean
    public JwtAuthenticationFilter jwtAuthenticationFilter(OctetSequenceKey keygenerator){

        return new JwtAuthenticationFilter(keygenerator);
    }
}

```
그리고 마지막으로 SecurityConfig bean 을 만들어서 이를 추가했습니다 각각 필요한 bean 을 만들고 필요한곳에 첨부했습니다 이렇게 해서 우리는 MAC 기반으로 JWT 토큰을 만들어서 
form 로그인과 연계를 진행을 해보았습니다 

