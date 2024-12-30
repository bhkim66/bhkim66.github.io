# @WebMvcTest를 이용한 Controller 테스트

Spring MVC 웹 계층 테스트에 사용

- `@WebMvcTest`는 Spring MVC의 **특정 컨트롤러를 대상**으로 한 테스트를 지원한다
- 이 어노테이션을 사용하면 웹 계층에 필요한 구성만 로드되므로, 전체 애플리케이션 컨텍스트를 로드하는 것보다 테스트 속도가 빠르다
- 내장된 `MockMvc` 인스턴스를 사용하여 HTTP 요청과 응답을 쉽게 테스트할 수 있다

```java
@WebMvcTest(TestController.class)
```

- 테스트 할 특정 컨트롤러를 지정한다

```java
@Slf4j
@ActiveProfiles("default") //테스트시 실행할 profile
@WebMvcTest(controllers = AuthController.class)
@Import({WebSecurityConfig.class})
class AuthControllerTest {
    @Autowired
    MockMvc mvc;
    @MockBean
    AuthServiceImpl authService;
    @MockBean
    JwtTokenProvider jwtTokenProvider;
```

- `@Import`를 통해 Spring Security의 필터 설정을 테스트 할 수 있다
- `MockMvc`를 주입받아 사용한다. `MockMvc`는 HTTP 요청을 디스패처 서블릿에 전송하고 결과를 받아 테스트하는데 사용한다
- 필요경우 `MockBean`을 사용하여 서비스나 리포지토리와 같은 다른 빈 들을 모의 객체로 생성하고 주입할 수 있다

**SecurtiyConfig**

```java
.authorizeHttpRequests(authorizeRequests -> authorizeRequests
        .requestMatchers("/auth/**", "/public/**").permitAll()
        .requestMatchers("/admin/**").hasRole(RoleEnum.USER.name())
        .anyRequest().authenticated() // 모든 요청은 인증 필요
)
```

- `"/auth/**"`, `"/public/**"` 로 시작하는 요청은 모두 허용한다
- `"/admin/**"`로 들어오는 요청은 ROLE `ADMIN`만 접근할 수 있다
- 이외 나머지 모든 요청은 인증이 필요하다

```java
@Test
void signIn() throws Exception {
    String requestJson = "{\"userId\":\"bhkim62\", \"password\": \"1234\"}";
    mvc.perform(post("/auth/sign-in")
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(requestJson))
            .andExpect(status().isOk())
            .andDo(print());
}
```

- `mockMvc` 인스턴스를 사용하여 HTTP 요청을 수행하고 응답을 검증한다
- `login()` 경우 post 요청이기 때문에 content에 requestBody 값을 보내줘야 한다
- `/auth/**` 패턴으로 시작하고 있기 때문에 인증 없이 요청을 받을 수 있다

```java
@Test
void 인증요청_필요한_api() throws Exception {
    mvc.perform(get("/user/sign-out").header(
                    "X-AUTH-TOKEN", ""))
            .andExpect(status().is4xxClientError())
            .andDo(print());
}
```

- `/user/**` 패턴으로 시작하고 있기 때문에 인증이 필요하다

```java
@Test
@WithMockUser(roles = "USER")
void ROLE이_다른경우() throws Exception {
    mvc.perform(get("/admin/health-check").header(
                    "X-AUTH-TOKEN", ""))
            .andExpect(status().is4xxClientError())
            .andDo(print());
}
```

- MockUser로 가상의 User을 만들고 `USER` 라는 권한을 주었다
- 하지만 `/admin/**` 패턴에 해당하는 요청은 `ADMIN` 권한만 접근 가능하기 때문에 접근 불가능하다

### MockMvc의 메소드

- `perform()`
    - HTTP 요청을 할 수 있다
    - 체이닝이 지원되어 여러 검증 기능을 이어서 선언할 수 있다
        - param / params : 쿼리 스트링 설정
        - cookie : 쿠키 설정
        - requestAttr : 요청 스코프 객체 설정
        - sessionAttr : 세션 스코프 객체 설정
        - content : 요청 본문 설정
        - header / headers : 요청 헤더 설정
        - contentType : 본문 타입 설정
- `andExcept()`
    - `mvc.perform`의 결과를 검증한다
    - `status()` : 상태 코드를 검증할 수 있다
        - isOk() : 200
        - isNotFound() : 404
        - isMethodNotAllowed() : 405
        - isInternalServerError() : 500
        - is(int status) : status 상태 코드
    - `view()` : 리턴하는 뷰에 대해 검증한다
        - ex) `andExcept(view().name(”/detailpage”))`
    - `redirectedUrl()` : 리다이렉트 url 을 검증한다
        - ex) `redirectedUrl("/where")`
    - `model()` : 컨트롤러의 Model에 관해 검증한다.
        - ex) `model().attribute("alcoholDetails", alcoholDetails)`
    - `content()`: 응답에 대한 정보를 검증한다.
    - `jsonPath()`: response body에 들어갈 json 데이터를 검증한다.
        - ex) `jsonPath("$.name").value("박병호")`
- `andDo()`
    - 요청/응답 전체 메세지를 확인할 수 있다
    - `print()` : 요청/ 응답 **전체 메세지**를 확인할 수 있다

### @Mockbean vs @Autowired

**@Autowired**

- **`@Autowired`** 어노테이션은 Spring의 의존성 주입을 위한 어노테이션이다
- **`@Autowired`** 가 붙은 필드, 생성자, 또는 메서드에 대해 Spring은 해당 타입의 빈을 찾아 자동으로 주입해준다
- 이는 Spring의 ApplicationContext에서 관리되는 실제의 빈 인스턴스이다
- **`@Autowired`** 는 주로 실제 객체를 사용해야 할 때 사용된다

**@MockBean**

- `@MockBean` 은 Spring Boot Test에서 제공하는 어노테이션으로, 해당 타입의 빈은 모킹버전으로 교체한다
- 이 모킹된 객체는 실제 로직을 수행하지 않고, 원하는 대로 동작을 설정할 수 있다
- `@MockBean` 은 주로 테스트 환경에서 특정 빈의 실제 동작을 원치 않고, 미리 정의된 동작을 수행하게 하기 위해 사용된다