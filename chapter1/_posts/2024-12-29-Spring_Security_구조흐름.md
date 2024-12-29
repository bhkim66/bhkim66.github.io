# Spring Security 구조 및 흐름

![image1.png](..%2F..%2Fassets%2Fimg%2Fspring_security%2Fimage1.png)
- 스프링 시큐리티는 Filter 기반으로 동작하기 때문에 Spring MVC와 분리되어 관리하고 동작할 수 있다
- HTTP 기본 인증을 요청하면 BasicAuthenticationFilter를 통과한다.
- HTTP Digest 인증을 요청하면 DigestAuthenticationFilter를 통과한다.
- 로그인 폼에 의해 요청된 인증은 UserPasswordAuthenticationFilter를 통과한다.
- x509 인증을 요청하면 X509AuthenticationFilter를 통과한다.
1. **Http Request 수신**
    - 사용자가 로그인 정보와 함께 인증 요청을 한다
2. 유저 자격을 기반으로 인증 토큰 생성
    - `AuthenticationFilter`가 요청을 가로채고 가로챈 정보를 통해 `UsernamePasswordAuthenticationToken`의 인증용 객체를 생성한다

      ![image2.png](..%2F..%2Fassets%2Fimg%2Fspring_security%2Fimage2.png)

    - `UserPasswordAuthenticationFilter` 내부에 가로챈 ID와 PassWord 기반으로 `UsernamePasswordAuthenticationToken` 객체가 생성되고 `AuthenticationManger`에게 인증요청을 위임하고 있다
3. Filter를 통해 `AuthenticationToken`을 `AuthenticationManger`로 위임
    - `AuthenticationManger`의 구현체인 `ProviderManger`에게 생성한 `UsernamePasswordToken` 객체를 전달한다
4. `AuthenticationProvider`의 목록으로 인증을 시도
    - `AuthenticationManger`는 등록된 `AuthenticationProvider`를 조회하며 인증을 요구한다

   ![image3.png](..%2F..%2Fassets%2Fimg%2Fspring_security%2Fimage3.png)

    - `AuthenticationProvider` 객체들을 리스트 형태로 가져온다
5. `UserDetailsSerivce`의 요구
    - 실제 데이터베이스에서 사용자 인증 정보를 가져오는 `UserDetailsSerivce`에 사용자 정보를 넘겨준다

      ![image4.png](..%2F..%2Fassets%2Fimg%2Fspring_security%2Fimage4.png)

        - `UserDetailsService`를 구현한 `CustomUserDetailService`가 전해 받는 유저 정보로 UserDetails 객체를 생성하고 있다
6. `UserDetails`를 이용해 User객체에 대한 정보 탐색
    - 넘겨받은 사용자 정보를  통해 데이터베이스에서 찾아낸 사용자 정보인 `UserDetails` 객체를 만든다
7. `User`객체의 정보들을 `UserDetails`가 `UserdetailsService`로 전달
    - `AuthenticationProvider`들은 `UserDetails`를 넘겨 받고 사용자 정보를 비교한다

     ![image5.png](..%2F..%2Fassets%2Fimg%2Fspring_security%2Fimage5.png) 

    - `AuthenticationProvider` 리스트를 순회하며 인증을 요청을 한다
8. 인증 객체 or exception
    - 인증이 완료되면 권한 등의 사용자 정보를 담은 `Authentication` 객체를 반환한다
      ![image6.png](..%2F..%2Fassets%2Fimg%2Fspring_security%2Fimage6.png)
        - 인증 요청한 provider의 리턴값 result를 보면 인증된`UsernamePasswordAuthenticationToken` 객체가 전달되고 있다
9. 인증 종료
    - 다시 최초의 `AuthenticationFilter`에 `Authentication`객체가 반환된다
10. SecurityContext에 인증 객체 설정
    - `Authentication` 객체를 `Security Context`에 저장한다