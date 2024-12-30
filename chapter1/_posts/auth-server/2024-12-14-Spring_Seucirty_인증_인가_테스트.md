# 3.Spring Security, 인증/인가 테스트

앞서 Token을 만들어 반환해줬기 때문에 Token값을 받아서 인증/인가가 제대로 작동하는지 테스트를 진행한다

### 인증 테스트

```java
@WebMvcTest(controllers = {UserController.class})
@Import(WebSecurityConfig.class)
class UserControllerTest {
    @Autowired
    MockMvc mvc;

    @MockBean
    JwtTokenProvider jwtTokenProvider;

    @MockBean
    UserService userService;

    @MockBean
    UserDetailsService userDetailsService; // MockBean 사용

    @Test
    void 인증_없는_요청_제한() throws Exception {
        mvc.perform(get("/user/test"))
                .andExpect(status().is4xxClientError())
                .andDo(print());
    }

    @Test
    @WithCustomMockUser
    void 인증_정보가_있는_요청() throws Exception {
        UserRequestDTO.UpdateUserInfo resource = UserRequestDTO.UpdateUserInfo.builder()
                .name("정병호")
                .age(31)
                .sex(M)
                .build();

        ObjectMapper objectMapper = new ObjectMapper();
        String resourceJson = objectMapper.writeValueAsString(resource);

        mvc.perform(put("/user/update")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(resourceJson))
                .andExpect(status().isOk())
                .andDo(print());
    }

    @Test
    @WithCustomMockUser(id = "bhkim622", role = USER)
    void USER_권한_유저_테스트() throws Exception {
        UserRequestDTO.UpdateUserInfo resource = UserRequestDTO.UpdateUserInfo.builder()
                .name("정병호")
                .age(31)
                .sex(M)
                .build();

        ObjectMapper objectMapper = new ObjectMapper();
        String resourceJson = objectMapper.writeValueAsString(resource);

        mvc.perform(put("/user/update")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(resourceJson))
                .andDo(print())
                .andExpect(status().isOk());
    }

    @Test
    @WithCustomMockUser(id = "guest011", role = GUEST)
    void user만_접근_가능한_guest_권한이_접근() throws Exception {
        UserRequestDTO.UpdateUserInfo resource = UserRequestDTO.UpdateUserInfo.builder()
                .name("정병호")
                .age(31)
                .sex(M)
                .build();

        ObjectMapper objectMapper = new ObjectMapper();
        String resourceJson = objectMapper.writeValueAsString(resource);

        mvc.perform(put("/user/update")
                        .contentType(MediaType.APPLICATION_JSON)
                        .content(resourceJson))
                .andDo(print())
                .andExpect(status().is4xxClientError());
    }
}
```

- 인증/인가 테스트는 Controller에 Request가 도달할 때 테스트가 되므로 `@WebMvcTest`를 진행했다
    - [@WebMvcTest를 이용한 Controller테스트](/chapter2/_posts/tdd/2024-12-14-WebMvcTest를_이용한_Controller_테스트.md)
- `@Import(WebSecurityConfig.class)` : 선언적 테스트를 위해서 config 파일을 import한다

    ```java
    .authorizeHttpRequests(authorizeRequests -> authorizeRequests
          .requestMatchers("/auth/**", "/public/**").permitAll()
          .requestMatchers("/user/**").hasAnyAuthority(USER.getValue())
        .anyRequest().authenticated()
    ```


### 인증_없는_접근_제한

- 테스트는 `@WithCustomMockUser` 어노테이션을 추가하지 않았기 때문에 인증된 객체가 없다

![spring_security_3_1.png](/assets/img/chapter1/auth_server/spring_security_3_1.png)

- `AuthorizeHttpRequestsConfigurer` 클래스의 `hasAuthority()` 메소드에 인자값으로 넘어온 String을 초기값으로 가지는 **AuthorityAuthorizationManger** 객체를 생성한다

![spring_security_3_2.png](/assets/img/chapter1/auth_server/spring_security_3_2.png)

- 이후 **RequestMatcherDelegatingAuthorizationManager** 클래스에서 check 메소드를 통해 가장 적절한 Manager를 통해 인증/인가 체크를 한다

![spring_security_3_3.png](/assets/img/chapter1/auth_server/spring_security_3_3.png)

- 요청 request 패턴과 **RequestMatcherDelegatingAuthorizationManager**에 전달된 ****매핑 값들 중에 있기 때문에 해당 인자값으로 넘어온 “ROLE_USER” 값이 있는지 체크하게 된다
- .requestMatchers("/user/**").hasAnyAuthority(USER.getValue())

![spring_security_3_4.png](/assets/img/chapter1/auth_server/spring_security_3_4.png)

- 해당 테스트는 인증 및 인가 값이 없는 User의 테스트 이므로 `false`와 `AuthorizationDecision` 값을 반환하게 된다

![spring_security_3_5.png](/assets/img/chapter1/auth_server/spring_security_3_5.png)

- 401 오류가 발생하면서 테스트에 성공한다

### 인증_정보가_있는_요청 & USER_권한_유저_테스트

- `@WithCustomMockUser` 어노테이션을 추가하여 임시 인증 정보를 주입했다

![spring_security_3_6.png](/assets/img/chapter1/auth_server/spring_security_3_6.png)

- `check()` 결과 값으로 true와 `AuthorizationDecision` 반환해준다

![spring_security_3_7.png](/assets/img/chapter1/auth_server/spring_security_3_7.png)

- 200 리턴해주면서 테스트 성공했다

![spring_security_3_8.png](/assets/img/chapter1/auth_server/spring_security_3_8.png)

### 권한이 없는 요청

- 테스트를 위해 `GUEST` 권한을 추가했다

```java
public enum RoleEnum {
    ADMIN("ROLE_ADMIN"),
    USER("ROLE_USER"),
    GUEST("ROLE_GUEST");
}
```

- `@WithCustomMockUser`의 role 인자 값을 `GUEST`으로 전달한다

![spring_security_3_9.png](/assets/img/chapter1/auth_server/spring_security_3_9.png)

- 권한이 false를 반환하면서 Forbidden 접근 거부 예외가 발생하면서 테스트에 성공한다

![spring_security_3_10.png](/assets/img/chapter1/auth_server/spring_security_3_10.png)