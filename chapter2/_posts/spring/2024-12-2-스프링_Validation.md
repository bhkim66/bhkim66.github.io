# 스프링 Validation

스프링 Validation을 통해 데이터에 대한 유효성 검증 처리를 수행할 수 있다

**build.gradle 추가**

```java
implementation 'org.springframework.boot:spring-boot-starter-validation'
```

### @Valid

- `Java`에서 유효성 검증으로 제공해주는 어노테이션이다
- `@RequestBody`와 함께 사용되며 JSON 형태로 전송 받은 데이터에 대해서 유효성 검증하기 위해 사용된다
- 해당 어노테이션을 선언한 곳에 유효성 검증을 실패했을 시 ***MethodArgumentNotValidException*** 로 에러가 발생합니다.

```java
@PostMapping("/test")
public ResponseEntity<ApiResponse<String>> test(@RequestBody @Valid TestVO testVO) {
    Integer result = codeService.insertCode(testVO);
    ApiResponse<Integer> res = new ApiResponse<>(result, SuccessCode.INSERT.getStatus(), SuccessCode.INSERT.getMessage());
    return new ResponseEntity<>(res, HttpStatus.OK);
}
```

### @Validated

- `Spring`에서 유효성 검증으로 제공하는 어노테이션이다
    - **`import org.springframework.validation.annotation.Validated`**
- 입력 파라미터의 유효성 검증은 컨트롤러에서 최대한 처리하고 넘겨주는 것이 좋다. 하지만 다른 곳에서도 파라미터를 검증해야 할 수 있다
    - 이를 위해 `Spring` 에서 `AOP` 기반으로 메소드의 요청을 가로채서 유효성 검증을 진행해준다
- 해당 어노테이션을 선언한 곳에 유효성 검증을 실패했을 시 ***MethodArgumentNotValidException*** 에러 발생

```java
@Service
@Validated
public class UserService {
	public void addUser(@Valid AddUserRequest addUserRequest) {
		...
	}
}
```

### 제약 조건 어노테이션

- @Null: Null 값만 입력 가능함
- @NotNull: 해당 값이 null이 아닌지 검증함
- @NotEmpty: 해당 값이 null이 아니고, 빈 스트링("") 아닌지 검증함(" "은 허용됨)
- @NotBlank: 해당 값이 null이 아니고, 공백(""과 " " 모두 포함)이 아닌지 검증함
    - @NotNull은 모든 타입 검사가 가능하지만 NotEmpty나 NotBlank는 String만 검사가 가능하다
- @AssertTrue: 해당 값이 true인지 검증함
- @Size: 해당 값의 문자열 길이를 검증함(String, Collection, Map, Array에도 적용 가능)
    - @Size(min=2, max=240)
      String message;
- @Min: 해당 값이 주어진 값보다 작지 않은지 검증함
- @Max: 해당 값이 주어진 값보다 크지 않은지 검증함
- @DecimalMax, @DecimalMin : 소수점이 있는 숫자에 대해 최대값 또는 최소값을 설정
- @Positive : 숫자가 양수 값인지 확인
    - @Negative : 음수
- @PositvieOrZero : 해당 값이 0 이상이여야
    - @NegativeOrZero : 음수 or 0
- @Pattern: 해당 값이 주어진 패턴과 일치하는지 검증함
    - @Pattern(regexp="\\(\\d{3}\\)\\d{3}-\\d{4}")
      String phoneNumber;