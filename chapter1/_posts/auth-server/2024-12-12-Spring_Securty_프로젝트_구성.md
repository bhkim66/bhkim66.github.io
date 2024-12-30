# 1.Spring Security, 프로젝트 구성

### 인증 흐름
![spring_security_1_1.png](/assets/img/chapter1/auth_server/spring_security_1_1.png)
- Refresh Token 값을 저장하기 위해 메모리 저장소인 `Redis`를 사용했다
- 재발급 API 요청 시 일반 DB에서 유저 정보를 가져오는 대신 Redis에서 가져온다
- **장점**:
    - **짧은 응답속도, 빠른 처리 속도**
    - **TTL(Time-to-Live) 관리**
        - Refresh Token의 만료 시간을 정해 유효기간 관리가 편리
    - **Stateless 인증 구현**
        - Refresh Token을 Redis에 저장하면 서버가 “상태를 가지지 않는” 형태로 구현하기 적합하다
        - 중앙 저장소 역활 수행
    - 로그인 된 유저 파악 및 비정상 회원 관리 가능

### Redis

- 메모리 기반의 데이터 저장소이다
- `Key - Value` 데이터 구조에 기반한 다양한 형태의 자료 구조를 제공하며, 데이터들을 저장할 수 있는 저장소이다
- `Redis` 는 메모리에 데이터를 저장하기 때문에 저장 공간에 제약이 있어 보조 데이터 저장소로 사용되나 **메모리를 사용하기 때문에 빠른 처리 속도가 장점**이다

---

기본적으로 HTTP 통신을 통한 요청/응답을 처리하고 있기 때문에 `Stateless` 한 토큰 방식의 인증 방식을 채택, JWT 토큰 방식을 사용 할 것이다

- [Stateless, Stateful, 세션, 토큰](/chapter2/_posts/security/2024-12-9-Stateless_Stateful_%EC%84%B8%EC%85%98_%ED%86%A0%ED%81%B0.md)
- 위 글에서 확인 했듯이 토큰은 DB에 따로 저장하지 않아도 되고 `Stateless` 한 특성을 가지고 있어 장점이 많다
- 하지만 토큰이 탈취될 경우 탈취한 공격자가 탈취한 토큰 정보를 가지고 피해자 행세를 할 수 있기 때문에 토큰에는 만료시간을 지정해서 관리할 필요가 있다
- `AccessToken` 같은 경우는 인증을 위해 주로 사용되는 토큰이기 때문에 짧은 시간을 지정하는 것이 좋고 `RefreshToken`은 토큰을 재발급 받기 위한 토큰이기 때문에 좀 더 여유로운 시간을 지정해도 괜찮다
    - 이 프로젝트에서는 `AccessToken`은 30분 `RefreshToken`은 24시간만 유효하게 설정했다
    - Redis에도 24시간 뒤면 Expire 되게 설정했다

      ![spring_security_1_2.png](/assets/img/chapter1/auth_server/spring_security_1_2.png)

### 스프링 시큐리티 설정

**SecurityConfig**

```java
@EnableWebSecurity
@Configuration
@RequiredArgsConstructor
@EnableMethodSecurity
public class WebSecurityConfig {
    private final JwtAuthenticationFilter jwtAuthenticationFilter;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
                .csrf(AbstractHttpConfigurer::disable) // csrf 관련 정책 비활성화
                .headers(headers -> headers.frameOptions(HeadersConfigurer.FrameOptionsConfig::disable))
								// 1.
                .cors(cors -> cors.configurationSource(corsConfigurationSource()))
                .sessionManagement(sessionManagement ->
                        sessionManagement.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
                // 2. URL에 따라 인증/인가 설정 
                .authorizeHttpRequests(authorizeRequests -> authorizeRequests
                                .requestMatchers("/auth/**", "/public/**").permitAll()
                                .requestMatchers("/admin/**").hasAnyAuthority(ADMIN.name())
//                        .requestMatchers("/admin/**").access(new UserAuthorizationManger())
                                .anyRequest().authenticated() // 모든 요청은 인증 필요
                )
                //3. 인증되지 않은 요청 처리 
                .exceptionHandling(exceptionHandling ->
                        exceptionHandling.authenticationEntryPoint(new JwtAuthenticationEntryPoint()))
                .formLogin(AbstractHttpConfigurer::disable)
                // 4. FIlter 추가
                .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class)
                .build();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
//        configuration.addAllowedOrigin("*");
        configuration.addAllowedOriginPattern("http://localhost:*");
        configuration.setAllowedMethods(Arrays.asList("HEAD", "GET", "POST", "PUT", "DELETE"));
        configuration.addAllowedHeader("*");
        configuration.setAllowCredentials(true);
        configuration.setMaxAge(3600L);
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", configuration);
        return source;
    }

    // 5. 암호화에 필요한 PasswordEncoder Bean 등록
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

1. `corsConfigurationSource()` : `CORS` 허용 범위를 설정하기 위한 설정이다
    - [CORS란?](/chapter2/_posts/security/2024-12-11-CORS.md)
    - CORS 허용 범위를 와일드카드 `*` 로 설정하면 보안에 취약하기 때문에 직접 허용할 출처를 세팅하는 방법이 좋다
2. `"/auth"`, `"/public"` 으로 들어오는 요청은 모두 permitAll(), 접근을 허용해주었다
    - 권한 없이 실행 되야하거나, 인증이 따로 필요 없는 API들이다. 그 외는 모두 인증을 받아야 한다
    - `"/admin/”` 으로 시작하는 요청은 `ADMIN` 권한이 있어야 허용이 된다
3. 인증되지 않은 즉 `UNAUTHORIZED` Exception을 처리하기 위한 클래스를 지정했다
4. `UsernamePasswordAuthenticationFilter`는 유저가 로그인을 시도할 때 보내지는 요청에서 아이디와 패스워드 데이터를 가져온 후 **인증을 위한 토큰을 생성 후 인증을 다른 쪽에 위임하는 역할**을 하는 필터이다
    - `UsernamePasswordAuthenticationFilter`가 실행되기 전에 `OncePerRequestFilter`을 상속하여 구현한 `JwtAuthenticationFilter`가 먼저 실행된다
        - `OncePerRequestFilter`은 Http Request의 한 번의 요청에 대해 한 번만 실행하는 Filter이다
        - 포워딩으로 인해 Filter Chain이 다시 동작하여 여러번 Filter가 발생하는 불상사를 막아준다
        - `OncePerRequestFilter`을 구현하여 Request에 담겨져 있는 토큰값의 유효성 검사를 통해 인증/인가를 한번만 거치고 다음 로직을 수행할 수 있다
5. 암호화에 필요한 `PasswordEncoder` Bean으로 등록했다

**JwtAuthenticationEntryPoint**

```java
@Component
public class JwtAuthenticationEntryPoint implements AuthenticationEntryPoint {
    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response, AuthenticationException authException) throws IOException, ServletException {
        setErrorResponse(response, authException);
    }

    private void setErrorResponse(HttpServletResponse res, RuntimeException e) throws IOException {
        ApiResponseResult<String> result = ApiResponseResult.failure(UNAUTHORIZED_EXCEPTION);
        ObjectMapper objectMapper = new ObjectMapper();
        String resultSrt = objectMapper.writeValueAsString(result);

        res.setContentType("application/json; charset=UTF-8");
        res.getWriter().write(resultSrt);
        res.setStatus(SC_UNAUTHORIZED);
        res.flushBuffer();
    }
}
```

- 인증이 되지않은 유저가 요청을 했을때 동작한다

**JwtAuthenticationFilter**

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {
    private final JwtTokenProvider jwtTokenProvider; //1. JwtTokenProvider 의존성 주입

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        //2. Request Header 에서 JWT 토큰 추출
        Optional<String> extractToken = extractToken(request);

        try {
            if(extractToken.isPresent()) {
                String token = extractToken.get();
                //3. 토큰 유효성 검사
                jwtTokenProvider.validateToken(token);
                Authentication authentication = jwtTokenProvider.getAuthentication(token);
                //4. SecurityContext에 Authentication 객체 저장 
                SecurityContextHolder.getContext().setAuthentication(authentication);
            }
        } catch (JwtException e) {
            log.error("e message : {} ", e.getMessage());
            throw new ApiException(MEMBER_REQUIRED);
        }
        filterChain.doFilter(request, response);
    }
    
    //2. Request Header 에서 JWT 토큰 추출
    private Optional<String> extractToken(ServletRequest request) {
       return Optional.ofNullable(((HttpServletRequest) request).getHeader(KEY_HEADER_AUTHORIZATION));
    }
}
```

1. Jwt 토큰 관련 처리를 도와줄 클래스를 주입 받는다
2. Request에 `“Authorization”` Key 값으로 넘어온 Header 값을 리턴한다
3. 토큰의 값이 있을 경우 토큰의 유효성 검사를 실시한다
4. 토큰으로 부터 꺼내온 `Authentication` 객체를  `SecurityContext`에 저장한다.
    - 저장된`Authentication` 값은 어디서든지 접근할 수 있다. `SecutiryContextHolder`가  ****`SecurityContext`을 스레드 로컬 변수에 저장하기 때문이다

**JwtTokenProvider**

```java
@Slf4j
@Component
@RequiredArgsConstructor
public class JwtTokenProvider {
    private final JwtHandler jwtHandler; //1. Jwt 토큰 생성 및 파싱 

    @Getter
    @AllArgsConstructor
    public static class PrivateClaims { //2. Claims 클래스
        private String userId;
        private RoleEnum role;
    }

    public String generateToken(PrivateClaims privateClaims, Long expireTime) {
        return jwtHandler.createJwt(Map.of(USER_ID, privateClaims.getUserId(), ROLE, privateClaims.getRole()), expireTime);
    }

    //3. Refresh Token 유효 확인
    public PrivateClaims parseRefreshToken(String refreshToken, String redisRefreshToken) {
        if (!refreshToken.equals(redisRefreshToken)) {
            throw new ApiException(INVALID_TOKEN_VALUE);
        }
        return jwtHandler.parseClaims(refreshToken).map(this::convert).orElseThrow();

    }

    //4. 토큰 검증
    public boolean validateToken(String token){
        try {
            Optional<Claims> claims = jwtHandler.parseClaims(token);
//            existSignOutMemberInRedis(token);
            return true;
        } catch (io.jsonwebtoken.security.SecurityException | MalformedJwtException e) {
            log.error("Invalid JWT Token");
        } catch (ExpiredJwtException e) {
            log.error("Expired JWT Token");
        } catch (UnsupportedJwtException e) {
            log.error("Unsupported JWT Token");
        } catch (IllegalArgumentException e) {
            log.error("JWT claims string is empty.");
        }
        throw new ApiException(INVALID_TOKEN_VALUE_ERROR);
    }

    //5. JWT 토큰을 복호화 Authentication 객체 생성 
    public Authentication getAuthentication(String token) {
        // 토큰 복호화
        Claims claims = jwtHandler.parseClaims(token).orElseThrow();
        // 클레임에서 권한 정보 가져오기
        Collection<? extends GrantedAuthority> authorities = Collections.singleton(new SimpleGrantedAuthority("ROLE_" +
                claims.get(ROLE)));

        User user = User.builder()
                .id((String) claims.get(USER_ID))
                .role(RoleEnum.valueOf((String) claims.get(ROLE)))
                .build();
        // UserDetails 객체를 만들어서 Authentication 리턴
        CustomUserDetail principal = new CustomUserDetail(user);
        return new UsernamePasswordAuthenticationToken(principal, token, authorities);
    }

		//6. 헤더에 있는 RefreshToken을 가져옴
   public String getTokenInHeader() {
        String refreshToken = ((ServletRequestAttributes) Objects.requireNonNull(RequestContextHolder.getRequestAttributes()))
                .getRequest().getHeader(GET_HEADER_REFRESH_TOKEN);
        return validateToken(refreshToken) ? refreshToken : "";
    }

    private PrivateClaims convert(Claims claims) {
        return new PrivateClaims(claims.get(USER_ID, String.class), RoleEnum.valueOf(claims.get(ROLE, String.class)));
    }
}
```

1. `Jwt 토큰` 생성 및 파싱을 위한 클래스 의존성을 주입한다
2. Jwt 토큰에 들어갈 Claim을 내부클래스로 구현했다
3. Request의 `RefreshToken`과 Redis에 저장된 `RefreshToken`값을 비교한다
4. Token 값의 유효성 검사를 한다
    - 값이 유효하지 않거나 유효기간이 지났거나 jwt 토큰 형식이 아닐 경우 예외가 발생한다
    - 토큰은 유효하지 않을 경우 보안상 위험이 있기 때문에 DB를 접근하는 것 보다 토큰을 버리고 다시 **재생성 하는 것이 더 효율적**이다
5. 토큰 값을 복호화 하여 다시 `Authentication` 객체를 만들어 리턴한다
    - 리턴한 `Authentication` 객체를 `SecurityContext`에 저장한다
6. 토큰 재발급을 할 때 RefreshToken 값을 Header에 넣어 전달한다.
    - Header에 있는 RefreshToken 값을 꺼내 전달한다

**JwtHandler**

```java
@Component
public class JwtHandler {
    private static final String BEARER_TYPE = "Bearer";
    private final Key key;

    public JwtHandler() {
        this.key = Keys.secretKeyFor(SignatureAlgorithm.HS256); //1.key 생성
    }

    //2. 유저 정보를 가지고 AccessToken, RefreshToken 을 생성하는 메서드
    public String createJwt(Map<String, Object> privateClaims, Long expireTime) {
        Date now = new Date();
        Date accessTokenExpiresIn = new Date(now.getTime() + expireTime);

        return Jwts.builder()
                .setIssuedAt(now) // 토큰 발급 시간
                .setIssuer("auth.bhkim.com") // 토큰 발급자
                .setClaims(privateClaims)
                .setExpiration(accessTokenExpiresIn) // 만료 시간
                .signWith(key) // 키 셋팅
                .compact();
    }

		//3. token을 파싱하여 Claims를 추출한다 
    public Optional<Claims> parseClaims(String token) {
        return Optional.of(Jwts.parserBuilder()
                .setSigningKey(key)
                .build()
                .parseClaimsJws(token)
                .getBody());
    }
}
```

1. Jwt 토큰 암호화를 위해  `io.jsonwebtoken.security.Keys` 유틸 클래스는 강력하고 간단한 키를 제공한다
    - 기존 문자열을 받아 바이트 배열로 값을 받았지만 안전하지 않고 적당하지 않은 키를 인수로 사용했기에 변경되었다
2. `PrivateClaims` 클래스로 만든 사용자 정보를 토큰 `Claims`에 넣는다. 토큰 파싱을 통해 값을 다시 꺼낼 수 있다
    - `setExpiration()` 메소드를 통해서 토큰의 유효기간을 정할 수 있다
3. 인자값으로 넘어온 `Token`을 파싱하여 `Claims`를 반환해준다

**CustomUserDetail**

```java
public class CustomUserDetail implements UserDetails {
    private User user;

    public CustomUserDetail(User user) {
        this.user = user;
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        Collection<GrantedAuthority> authorities = new ArrayList<>();
        authorities.add(new SimpleGrantedAuthority(user.getRole().getValue()));
        return authorities;
    }

    @Override
    public String getPassword() {
        return user.getPassword();
    }

    @Override
    public String getUsername() {
        return user.getName();
    }

    public String getId() {
        return user.getId();
    }

    public RoleEnum getRole() {
        return user.getRole();
    }
}
```

- Spring Security에서 제공해주는 UserDetail 인터페이스를 구현했다
- 앞으로 `AuthenticationToken`을 만들때 `CustomUserDetail`의 형태로 `principal`이 생성된다
- `getAuthorities()`은 유저가 가진 권한을 리턴해주는 메소드이다
    - 지금은 Role만 있기 때문에 회원이 가진 Role 값을 전달해준다
    - `SimpleGrantedAuthority`은 `GrantedAuthority`의 간단한 구현체로써 String으로 Role 값을 받는다

**CustomUserDetailsService**

```java
@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor(access = AccessLevel.PROTECTED)
public class CustomUserDetailsService implements UserDetailsService {
    private final UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String userId) throws UsernameNotFoundException {
        return userRepository.findById(userId)
                .map(this::createUserDetails)
                .orElseThrow(() -> new UsernameNotFoundException("해당하는 유저를 찾을 수 없습니다"));
    }

    // 해당하는 User 의 데이터가 존재한다면 UserDetails 객체로 만들어서 리턴
    private CustomUserDetail createUserDetails(User user) {
       return new CustomUserDetail(user);
    }
}
```

- **`CustomUserDetailsService`**은 ****`UserDetail`을 구현한 **** `CustomUserDetail`을 반환한다
- `loadUserByUsername()`  userId로 DB를 조회해서  유저를 찾아오고 `createUserDetails()` 메소드를 통해서  `CustomUserDetail` 형태로 반환한다
- 로그인 요청이 들어왔을 때 `CustomUserDetailsService` 에서 만든 `loadUserByUsername` 메서드가 실행된다
    - 이를 통해 `principal`을 `CustomUserDetail` 형태로 강제할 수 있다

**JwtAuthenticationEntryPoint**

```java
/**
 * 인증되지 않는 요청 오류 처리
 */
@Component
public class JwtAuthenticationEntryPoint implements AuthenticationEntryPoint {
    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response, AuthenticationException authException) throws IOException, ServletException {
        setErrorResponse(response, authException);
    }

    private void setErrorResponse(HttpServletResponse res, RuntimeException e) throws IOException {
        ApiResponseResult<String> result = ApiResponseResult.failure(UNAUTHORIZED_EXCEPTION);
        ObjectMapper objectMapper = new ObjectMapper();
        String resultSrt = objectMapper.writeValueAsString(result);
        
        res.setContentType("application/json; charset=UTF-8");
        res.getWriter().write(resultSrt);
        res.setStatus(SC_UNAUTHORIZED);
        res.flushBuffer();
    }
}
```

- 인증되지 않는 요청에 관해서 처리하는 클래스이다
- `setErrorResponse()` 동일한 결과 형태를 반환하기 위해 커스텀한 `response` 형태로 반환해준다

![spring_security_1_3.png](/assets/img/chapter1/auth_server/spring_security_1_3.png)
- 인증 예외 발생 후 `AuthenticationEntryPoint`를 구현해서 오류 처리를 한다

**JwtExceptionFilter**

```java
/**
 * Security 인증/인가 과정에서 생기는 오류 처리
 */
@Slf4j
@Component
public class JwtExceptionFilter extends OncePerRequestFilter {
    private final ObjectMapper objectMapper = new ObjectMapper();

    private void setErrorResponse(HttpServletResponse res, ApiException e) throws IOException {
        ApiResponseResult<String> result = ApiResponseResult.failure(e.getException());
        String resultSrt = objectMapper.writeValueAsString(result);

        res.setContentType("application/json; charset=UTF-8");
        res.getWriter().write(resultSrt);
        res.setStatus(SC_BAD_REQUEST);
        res.flushBuffer();
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        response.setCharacterEncoding("utf-8");
        try {
            filterChain.doFilter(request, response); // go to 'JwtAuthenticationFilter'
        } catch (ApiException e) {
            log.error("ApiException : {}", e.getMessage());
            setErrorResponse(response, e);
        }
    }
}
```

- `filterChain.doFilter()`에서 발생하는 모든 오류처리를 하는 필터를 추가했다
- `setErrorResponse()` 동일한 결과 형태를 반환하기 위해 커스텀한 `response` 형태로 반환해준다

- **[Redis 연동](/chapter1/_posts/auth-server/2024-12-10-Redis_%EC%97%B0%EB%8F%99.md)**

