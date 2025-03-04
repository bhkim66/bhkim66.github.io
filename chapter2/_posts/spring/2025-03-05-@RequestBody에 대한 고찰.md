# DTO에 @Getter, @NoArgsConstructor를 사용하는 이유

### Spring에서 재공하는 Binding

스프링은 외부에서 전달한 Request Body 값을 서버 내 클래스 객체로 받을 수 있다

- @RequestParam
    - Http 요청의 쿼리 파라미터나 폼 데이터를 받을 때 사용한다
- @RequestBody
    - HTTP 요청의 본문(body) 데이터를 자바 객체로 변환할 때 사용한다

이렇게 외부 json 데이터를 자바의 객체로 변환하는 과정을 `역직렬화`라고 한다

@RequestBody는 spring의 HttpMessageConverter를 통해 데이터를 바인딩한다

### **HttpMessageConverter**

![image.png](/assets/img/chapter2/spring/spring_1_1.png)

- 역직렬화와 직렬화 과정은 `HttpMessageConverter`를 이용해 변환된다
1. Dispatcher Servlet에서 요청을 Handler Adapter로 전달한다
2. Handler Adapter은 ArgumentResolver를 호출하여 요청을 컨트롤러에 전달한다. 이때 ArgumentResolver에서 `HttpMessageConveter`를 사용하여 요청(json)을 객체(dto)로 만든다
3. 만들어진 객체를 컨트롤러의 매개변수에 전달하고 핸들러가 동작한다
4. 만약 핸들러가 반환할 값이 있으면 ReturnValueHandler에서 `HttpMessageConverter`를 이용해 객체를 응답(json)으로 만들어 반환한다 (직렬화)

![image.png](/assets/img/chapter2/spring/spring_1_2.png)

- Jackson에서는 JSON 필드의 이름을 자바 Object의 getter, setter 메소드를 통해 일치시켜 JSON 필드를 자바 Object 필드와 매치시켜 값을 주입한다
    - 이 과정을 위해 dto로 사용되는 객체에 `@Getter`를 사용한다
- 또한 `@RequestBody`에서 Binding할 때 `ObjectMapper`가 생성되는데 기본 생성자로 DTO를 생성하기 때문에 기본 생성자는 일반적인 경우 생성해야한다

DTO 객체에 `@Getter`, `@NoArgsConstructor`가 사용되는 이유이다

출처 - https://blogshine.tistory.com/445