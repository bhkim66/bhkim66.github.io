# Spring Data JPA
## 스프링 데이터 JPA

스프링 데이터 JPA는 스프링 프레임워크에서 JPA를 편리하게 사용할 수 있도록 지원하는 프로젝트다.

데이터 접근 계층을 개발할 때 구현 클래스 없이 **인터페이스만 작성해도 개발을 완료**할 수 있다

- CRUD를 처리하기 위한 공통 메소드는 스프링 데이터 JPA가 제공하는 `JpaRepository` 인터페이스에 있다
- 어플리케이션 실행 시점에 스프링 데이터 JPA가 **생성해서 주입해주므로**, 개발자는 직접 구현체를 개발하지 않아도 된다

## 스프링 데이터 프로젝트

`스프링 데이터 JPA`는 스프링 데이터 프로젝트의 하위 프로젝트 중 하나이다

이 외에 스프링 데이터 프로젝트는 JPA, 몽고 DB, NEO4J, REDIS, HADOOP 등 다양한 데이터 저장소에 대한 접근을 추상화해서 개발자 편의를 제공한다

![spring_data_jpa_2_1](/assets/img/chpater2/jpa/jpa_2_1.png)

### JpaRepository

```java
public interface AuthRepository extends JpaRepository<User, Long> {
	...
}
```

```java
@RequiredArgsConstructor
public class AuthServiceImpl implements AuthService {
	private final AuthRepository authRepository
}
```

- JpaRepository 상속하는 경우 Spring에서 proxy 객체로 생성 후 `Injection`을 한다
    - @Repository 애노테이션 생략 가능하다
    - JPA 예외를 Spring 예외로 변환하는 과정도 자동으로 처리한다
    - authRepository.getClass() → class com.sun.proxy.$ProxyXXX

**주요 메서드**

참고 : (T = 엔티티), (ID = 엔티티의 식별자 타입), (S = 엔티티와 그 자식 타입)

- **save(S) :** 새로운 엔티티를 저장하고 이미 있는 엔티티는 수정한다
    - 엔티티에 식별자 값이 없으면 **새로운 엔티티**로 판단해서 `EntityManager.persist()` 를 호출하고 식별자 값이 있으면 이미 있는 엔티티로 판단하여 `EntityManager.merge()` (준영속 → 영속화)를 호출한다
- **delete(T) :** 엔티티 하나를 삭제한다. 내부에서 EntityManager.remove() 를 호출한다
- **findOne(ID)** : 엔티티 하나를 조회한다. 내부에서 EntityManager.find() 를 호출한다
- **getOne(ID)** : 엔티티를 프록시로 조회한다. 내부에서 EntityManager.getReference() 를 호출한다
- **findAll(…)** : 모든 엔티티를 조회힌다. 정렬(Sort)이나 페이징(Pageable) 조건을 파라미터로 제공할 수 있다

## 쿼리 메소드

스프링 데이터 JPA가 제공하는 쿼리 메소드 기능은 크게 3가지가 있다

- 메소드 이름으로 쿼리 생성
- 메소드 이름으로 JPA NamedQuery 호출
- `@Query` 어노테이션을 사용해서 리포지토리 인터페이스에 쿼리 직접 정의

### **메소드 이름으로 쿼리 생성**

정해진 규칙에 따라서 메소드 이름을 지을 경우 해당하는 SQL문을 생성되어 실행된다

ex) findByEmailAndName = select m from Member m where m.email = ?1 and m.name

- Spring 공식 사이트에서 제공하는 method 이다
- https://docs.spring.io/spring-data/jpa/reference/jpa/query-methods.html

### **JPA NamedQuery**

메소드 이름으로 `JPA Named` 쿼리를 호출하는 기능을 제공한다

`JPA Named` 쿼리는 이름 그대로 쿼리에 이름을 부여해서 사용하는 방법인데, 어노테이션이나 XML에 쿼리를 정의할 수 있다

```java
@Entity
@NamedQuery (
	name = "Member.findByUsername"
	query="select m from Member m where m.username = :username")
public class Member {
	...
}
```

메소드의 이름만으로 Named 쿼리를 호출할 수 있다

```java
public interface MemberRepository extends JpaRepository(Member, Long> {
	List<Member> findByUseranme(@Param("username") String username);
}
```

### @Query, 리포지토리 메소드에 쿼리 정의

리포지토리 메소드에 직접 쿼리를 정의하려면 `@Query` 어노테이션을 사용한다. 이 방법은 실행할 메소드에 정적 쿼리를 직접 작성하므로 이름 없는 Named 쿼리라 할 수 있다. **어플리케이션 실행 시점**에 **문법 오류를 발견할 수 있는 장점이 있다.**

```java
public interface MemberRepository extends JpaRepository<Member, Long> {
	@Query("select m from Member m where m.username = :username")
	List<Member> findByUseranme(@Param("username") String username);
}
```

### **벌크성 수정쿼리**

스프링 데이터 JPA에서 벌크성 수정, 삭제 쿼리는 `@Modifying` 어노테이션을 사용하면 된다

벌크성 쿼리를 실행 후 영속성 컨텍스트를 초기화하고 싶으면 `@Modifing(clearAutomatically = true)` 로 설정해주면 된다

```java
@modifying
@Query("update Prodect p set p.price = p.price * 1.1 where p.stockAmount < :stockAmount")
int bulkPriceUp(@Parma("stockAmount") String stockAmount);
```

## 페이징

쿼리 메소드에 페이징 정렬 기능을 사용할 수 있도록 2가지 파라미터를 제공한다

- **Sort** : 정렬 기능
- **Pageable** : 페이징 기능(내부에 Sort 포함)

파라미터에 Pageable를 사용하면 반환 타입으로 List나 Page를 사용할 수 있다

```java
//count 쿼리 사용
Page<Member> findByName(String name, Pageable pageable);

//count 쿼리 사용 안함
List<Member> findByName(String name, Pageable pageable);

List<Member> findByName(String name, Sort sort)
```

### 스프링 데이터 JPA가 사용하는 구현체

```java
@Repository
@Transcational(readOnly = true)
public class SimpleJpaRepositiory<T, ID extends Serializable> 
							   implements JpaRepository<T, ID>, JpaSpecificationExecutor<T> {
							   
		   @Transactional
		   public <S extends T> S savne(S entity) {
					  ...
		   }
}

```

- **@Repository** : JPA 예외를 스프링이 추상화한 예외로 변환한다
- **@Transactional 트랜잭션 적용** : JPA의 모든 변경은 트랜잭션 안에서 이루어져야 한다. 스프링 데이터의 인터페이스를 사용하면 데이터를 변경(등록, 수정, 삭제)하는 메소드에 `@Transactional` 로 트랜잭션 처리가 되어 있다. 따라서 *서비스 계층에서 트랜잭션을 시작하지 않으면* **리포지토리에서 트랜잭션을 시작한다**. 서비스 계층에서 트랜잭션을 시작했으면 리포지토리도 해당 트랜잭션을 전파받아서 그대로 사용한다
- **@Transactional(readOnly = true) :** 데이터를 변경하지 않는 트랜잭션에서 readOnly = true 옵션을 사용하면 플러시를 생략해서 **약간의 성능 향상을 얻을 수 있다**

## 트랜잭션 범위의 영속성 컨텍스트

### 스프링 컨테이너의 기본 전략

스프링 컨테이너는 **트랜잭션 범위의 영속성 컨텍스트** 전략을 기본으로 사용한다. 이 전략은 이름 그대로 **트랜젹션의 범위와 영속성 컨텍스트의 생존 범위가 같다는 뜻**이다. **같은 트랜잭션 안에서는 항상 같은 영속성 컨텍스트에 접근한다**.

- 스프링 프레임워크에서 `@Transactinal` 어노테이션을 선언해서 트랜잭션을 시작하면 호출한 메소드를 실행하기 직전에 스프링의 **트랜잭션 AOP**가 먼저 동작한다
- 트랜잭션을 커밋하면 **JPA는 먼저 영속성 컨텍스트를 플러시**해서 변경 내용을 **데이터베이스에 반영한 후**에 데이터베이스 트랜잭션을 커밋한다
    - 트랜잭션 commit() → EntitiyManager.flush() → DB 트랜잭션 COMMIT

```java
@Controller
class testController {
	
	@Autowired TestService testServcie;
	
	public void test() {
		// Test는 준영속 상태이다 ...4
		Test test = testService.test();
	}
}

@Service
class testService {

	// 트랜잭션 시작 ... 1
	@Transactional
	public Test test() {
		repository.test();
		// Test는 영속 상태이다 ...2
		Test test = repository.findTest();
		
		return test
	}
	// 트랜잭션 종료 ...3
}
```

1. TestService.test() 메소드의 `@Transactional`을 선언해서 메소드를 호출할 때 트랜잭션을 먼저 시작한다
2. repository.findTest()를 통해 조회한 Test 엔티티는 트랜잭션 범위 안에 있으므로 영속성 컨텍스의 관리를 받는다. **따라서 지금은 영속 상태이다**
3. `@Transactional`을 선언한 메소드가 정상 종료되면 **트랜잭션을 커밋**하는데, 이때 **영속성 컨텍스트가 종료**된다. 영속성 컨텍스트가 사라졌으므로 조회한 엔티티는 이제부터 **준영속 상태가 된다**
4. 서비스 메소드가 끝나면서 트랜잭션과 영속성 컨텍스트가 종료되었다. 따라서 컨트롤러에 반환된 Test 엔티티는 준영속 상태이다

**트랜잭션이 같으면 같은 영속성 컨텍스트를 사용한다**

- 영속성 컨텍스트 전략은 다양한 위치에서 엔티티 매니저를 주입받아 사용해도 **트랜잭션이 같으면 항상 같은 영속성 컨텍스트를 사용한다.**
- 같은 엔티티 매니저를 사용해도 **트랜잭션에 따라 접근하는 영속성 컨텍스트가 다르다**
- 스프링이나 J2EE 컨테이너의 가장 큰 장점은 트랜잭션과 복잡한 멀티 스레드 상황을 컨테이너가 처리해준다

### 준영속 상태와 지연 로딩

트랜잭션 범위의 영속성 컨텍스트 전략을 사용하면 트랜잭션이 없는 프리젠테이션 계층에서 엔티티는 준영속 상태다. 따라서 변경감지와 지연 로딩이 동작하지 않는다.

- 준영속 상태는 *영속성 컨텍스트가 없으므로 지연 로딩을 할 수 없다*

준영속 상태의 지연 로딩 문제를 해결하는 방법에는 2가지가 있다

- 뷰가 필요한 엔티티를 미리 로딩해두는 방법
    - 글로벌 페치 전략 수정
    - `JPQL` 페치 조인
    - 강제로 초기화
- `OSIV`를 사용해서 엔티티를 항상 영속 상태로 유지하는 방법

### **글로벌 페치 전략 수정**

글로벌 페치 전략을 지연 로딩에서 즉시 로딩으로 변경하면 된다. 하지만 단점이 존재한다

**글로벌 페치 전략에 즉시 로딩 사용 시 단점**

- **사용하지 않는 엔티티를 로딩한다**
    - 다른 화면에서 필요 없는 엔티티 이지만 즉시 로딩으로 인해 필요 없는 엔티티를 조회하게 된다
- **N+1 문제가 발생한다**
    - JPA를 사용하면서 성능상 가장 조심해야 하는 것이 바로 `N+1` 문제이다
    - **JPA가 JPQL을 분석해서 SQL을 생성할 때는 글로벌 페치 전략을 참고하지 않고 오직 JPQL 자체만 사용한다**. 따라서 즉시 로딩이든 지연 로딩이든 구분하지 않고 JPQL 쿼리 자체에 충실하게 SQL을 만든다
        1. select o from Order o JPQL을 분석해서 SELECT * FROM Order SQL을 생성한다
        2. 데이터베이스에서 결과를 받아 order 엔티티 인스턴스들을 생성한다
        3. Order.member의 글로벌 페치 전략이 즉시 로딩이므로 order를 로딩하는 즉시 연관된 member도 로딩해야 한다
        4. 연관된 member를 영속성 컨텍스트에서 찾는다
        5. 만약 영속성 컨텍스트에 없으면 SELECT * FROM MEMBER WHERE id = ? SQL을 **조회한 order 엔티티 수만큼 실행한다**
    - 만약 조회한 order 수가 10개라면 member를 조회하는 SQL도 10번 실행한다. 이는 JPA를 사용하면서 매우 치명적인 문제를 야기한다. 이런 **N+1 문제는 JPQL 페치 조인으로 해결할 수 있다**

### **JPQL 페치 조인**

글로벌 페치 전략을 즉시 로딩으로 설정하면 어플리케이션 전체에 영향을 주므로 너무 비효율적이다

하지만 페치 조인을 사용하면 SQL JOIN을 사용해서 페치 조인 대상까지 함께 조회한다. 따라서 N+1 문제가 발생하지 않는다.

페치 조인은 N+1 문제를 해결하면서 **화면에 필요한 엔티티를 미리 로딩하는 현실적인 방법이다**

단점 : 각각 화면마다 다른 데이터가 필요할 수 있기 때문에 어떤 화면은 페치 조인을 사용하고 어떤 화면은 페치 조인을 사용 안 할 수 있다. 그러다보니 **뷰와 리포지토리 간에 논리적인 의존관계가 발생한다.**

- 해결방안으로 한 메소드를 만들어 놓고 페치 조인을 사용해서 쿼리를 한번 만으로 모두 조회하게 한다. 물론 조인이 필요없는 화면에서는 **약간의 로딩 시간이 증가**하겠지만 **성능에 미치는 영향이 미비하다**

## OSIV (Opne Session in View)

엔티티가 프리젠테이션 계층에서 준영속 상태에서 발생하는 문제들을 해결하기 위해 뷰(View) 단 까지 영속성 컨텍스트를 열어두는 `OSIV` 가 존재한다. 이를 통해 **뷰에서도 지연 로딩을 사용할 수 있다**

**스프링 OSIV**: 비지니스 계층 트랜잭션

스프링 OSIV는 기존 OSIV의 문제점이던 **뷰 단에서 데이터 변경에 대한 문제**를 해결하고자 나온 라이브러리이다.

- 트랜잭션을 비즈니스 계층에서만 사용하고 뷰나 컨트롤러 단에서는 사용이 불가능하다

![spring_data_jpa_2_2](/assets/img/chpater2/jpa/jpa_2_2.png)

1. 클라이언트 요청이 들어오면 서블릿 필터나, 스프링 인터셉터에서 영속성 컨텍스트를 생성한다. **이때 트랜잭션은 시작하지 않는다**
2. 서비스 계층에서 `@Transcational`로 트랜잭션을 시작할 때 1번에서 미리 생성해둔 영속성 컨텍스트를 찾아와서 트랜잭션을 시작한다
3. 서비스 계층이 끝나면 트랜잭션을 커밋하고, 영속성 컨텍스트를 플러시한다. 이때 트랜잭션은 끝내지만 영속성 컨텍스트는 종료하지 않는다
4. 조회한 엔티티는 영속 상태를 유지한다
5. 서블릿 필터나, 스프링 인터셉터로 요청이 들어오면 영속성 컨텍스트를 종료한다
- 스프링 OSIV를 사용하면 프리젠테이션 계층에서는 **트랜잭션이 없으므로 엔티티를 수정할 수 없다**
- 사실 엔티티는 자주 변경된다. 엔티티가 변경되면 노출하는 JSON API도 함께 변경된다. 따라서 외부 API는 엔티티를 직접 노출하기보다는 엔티티를 변경해도 **완충 역활을 할 수 있는 DTO로 변환해서 노출하는 것이 안전하다**
- Spring Boot에서는 기본값이 true로 되어 있다

### 스프링 데이터 JPA가 제공하는 쿼리 메소드 기능

- **조회**: find…By ,read…By ,query…By get…By,
    - 예:) findHelloBy 처럼 ...에 식별하기 위한 내용(설명)이 들어가도 된다.
- **COUNT**: count…By 반환타입 `long`
- **EXISTS**: exists…By 반환타입 `boolean`
- **삭제**: delete…By, remove…By 반환타입 `long`
- **DISTINCT**: findDistinct, findMemberDistinctBy
- **LIMIT**: findFirst3, findFirst, findTop, findTop3

### @MappedSuperclass

- 공통 매핑 정보가 필요할 때, 부모 클래스에 선언하고 속성만 상속 받아서 사용하고 싶을 때 사용한다

```java
@MappedSuperclass
public abstract class BaseEntity {
	private String createdBy;    
	private LocalDateTime createdDate;    
	private String lastModifiedBy;    
	private LocalDateTime lastModifiedDate;
}
```

### **JPA Auditing**

- Java에서 ORM 기술인 `JPA`를 사용해서 도메인을 관계형 데이터베이스 테이블에 매핑할 때 공통적으로 도메인들이 가지고 있는 필드나 컬럼들이 존재한다
    - ex) 생성일자, 수정일자, 식별자
- `JPA`에서는 `Audit`이라는 기능을 제공하고 있다. `Spring Data JPA`에서 시간에 대해서 자동으로 값을 넣어주는 기능이다

```java
@Getter
@MappedSuperclass 
@EntityListeners(AuditingEntityListener.class) 
public abstract class BaseTimeEntity{
    // Entity가 생성되어 저장될 때 시간이 자동 저장됩니다.
    @CreatedDate
    private LocalDateTime createdDate;

    // 조회한 Entity 값을 변경할 때 시간이 자동 저장됩니다.
    @LastModifiedDate
    private LocalDateTime modifiedDate;
}
```

**JPA Auditing 활성화**

```java
@EnableJpaAuditing 
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

- `@MappedSuperclass`가 적용된 BaseTimeEntity 추상 클래스를 상속하기 때문에 JPA가 생성 일자, 수정 일자 컬럼을 인식하게 된다. 그리고 영속성 컨텍스트에 저장 후 BaseTimeEntity 클래스의 Auditing 기능으로 인해서 트랜잭션 커밋 시점에 플러시가 호출할 때 하이버네이트가 자동으로 시간 값을 채워주는것을 확인 할 수 있다
- 스프링부트의 Entry 포인트인 실행 클래스에 **@EnableJpaAuditing 어노테이션을 적용하여 JPA Auditing을 활성화 해야하는 것을 잊지 말아야 한다**

### @GeneratedValue

@GeneratedValue은 식별자 값을 자동 생성 시켜줄 수 있다

**strategy = GenerationType.IDENTITY**

- 기본 키 생성을 데이터베이스에 위임한다
- 주로 Mysql, PostgreSql, Sql Server, DB2에서 사용한다 (예: MySQL의 AUTO_ INCREMENT)
- IDENTITY 전략은, em.persist()로 객체를 영속화 시키는 시점에 곧바로 insert 쿼리가 db로 전송되고, 거기서 변환받은 식별자 값을 가지고 1차 캐시에 엔티티를 등록시켜 관리한다
- 그러므로 쓰기 지연의 성능적 이득을 취할 순 없다

**strategy = GenerationType.SEQUENCE**

- DB의 시퀀스를 활용하여 ID값을 증가시킨다
- 데이터베이스 시퀀스는 유일한 값을 순서대로 생성하는 특별한 데이터베이스 오브젝트(예: 오라클 시퀀스)
- 오라클, PostSql, DB, H2 데이터베이스에서 사용한다

**strategy = GenerationType.Table**

- 키 생성 전용 테이블을 하나 만들어서 데이터베이스 스퀀스를 흉내내는 전략이다
- 모든 데이터베이스에 적용 가능하나, 성능적 손해가 있어서 잘 사용하지 않는다

### @MappedSuperclass

- 부모 클래스는 테이블과 매핑하지 않고 부모 클래스를 상속받는 자식 클래스에게 **매핑 정보만 제공하고싶을 때 사용한다**
- 단순히 매핑 정보를 상속할 목적으로만 사용한다
- 상속 받은 컬럼을 재정의 하려면 `@AttributeOverride` 를 사용하고 둘 이상일 경우 `@AttributeOverrides` 를 사용한다
- 이 클래스 같은 경우 직접 생성해서 사용할 일은 거의 없으므로 **추상 클래스로 만드는 것을 권장한다**
- 엔티티는 엔티티 이거나 @MappedSuperclass로 지정한 클래스만 상속받을 수 있다

### @ManyToOne

- optional = false : NOT NULL 제약 옵션 가능
- fetch :
    - FetchType.LAZY : 지연로딩 // OneTomany 기본값
    - FetchType.EAGER : 즉시로딩 // ManyToOne 기본값

## 스프링 데이터 JPA 페이징과 정렬

**페이징과 정렬 파라미터**

- `org.springframework.data.domain.Sort` : 정렬 기능
- `org.springframework.data.domain.Pageable` : 페이징 기능 (내부에 `Sort` 포함)

**특별한 반환 타입**

- `org.springframework.data.domain.Page` : 추가 count 쿼리 결과를 포함하는 페이징
- `org.springframework.data.domain.Slice` : 추가 count 쿼리 없이 다음 페이지만 확인 가능(내부적으로 limit + 1조회)
- `List` (자바 컬렉션): 추가 count 쿼리 없이 결과만 반환

```java
public interface MemberRepository extends Repository<Member, Long> {
	Page<Member> findByAge(int age, Pageable pageable);
}
```

```java
PageRequest pageRequest = PageRequest.of(0, 3, Sort.by(Sort.Direction.DESC,"username"));
Page<Member> page = memberRepository.findByAge(10, pageRequest);
```

- 두 번째 파라미터로 받은 `Pageable` 은 인터페이스다. 따라서 실제 사용할 때는 해당 인터페이스를 구현한 `org.springframework.data.domain.PageRequest` 객체를 사용한다
- `PageRequest` 생성자의 첫 번째 파라미터에는 현재 페이지를, 두 번째 파라미터에는 조회할 데이터 수를 입력한다. 여기에 추가로 정렬 정보도 파라미터로 사용할 수 있다. 참고로 페이지는 0부터 시작(Spring 페이징)

참고 : count 쿼리를 다음과 같이 분리할 수 있다

```java
@Query(value = "select m from Member m", countQuery = "select count(m.username) from Member m")
Page<Member> findMemberAllCountBy(Pageable pageable);
```

- count 쿼리를 분리함으로써 join 같은 쿼리에서 성능을 최적화 할 수 있다
    - 이건 복잡한 sql에서 사용, 데이터는 left join, 카운트는 left join 안해도 됨
- 스프링부트 3 이상에서는 하이버네이트가 6이 적용된다

```java
@Query(value = "select m from Member m left join m.team t")
Page<Member> findByAge(int age, Pageable pageable);
```

- 다음과 같은 쿼리를 실행하면

```sql
select
	m1_0.member_id,
	m1_0.age,
	m1_0.team_id,
	m1_0.username
from
	member m1_
```

- 다음과 같은 쿼리가 실행된다
- `Team`을 조인하지만 `Team`을 전혀 사용하지 않고 있기 때문에 최적화를 통해 left join을 실행하지 않는다

**기본값**

- 글로벌 설정: 스프링부트

```
spring.data.web.pageable.default-page-size=20 /# 기본 페이지 사이즈/
spring.data.web.pageable.max-page-size=2000 /# 최대 페이지 사이즈/
```

**`@PageableDefault` 어노테이션을 사용**

```java
@RequestMapping(value = "/members_page", method = RequestMethod.GET)
public String list(@PageableDefault(size = 12, sort = "username", 
					direction = Sort.Direction.DESC) Pageable pageable) {
	...
}
```

## 벌크성 수정 쿼리

**JPA를 이용한 벌크성 수정 쿼리**

```java
public int bulkAgePlus(int age) {
	int resultCount = em.createQuery(
		"update Member m set m.age = m.age + 1" +	"where m.age >= :age")
		.setParameter("age", age)
		.executeUpdate();
	return resultCount;
}
```

```java
@Test
public void bulkUpdate() throws Exception {
	//given
	memberJpaRepository.save(new Member("member1", 10));
	memberJpaRepository.save(new Member("member2", 19));
	memberJpaRepository.save(new Member("member3", 20));
	memberJpaRepository.save(new Member("member4", 21));
	memberJpaRepository.save(new Member("member5", 40));
	//when
	int resultCount = memberJpaRepository.bulkAgePlus(20);
	//then
	assertThat(resultCount).isEqualTo(3);
}
```

**스프링 데이터 JPA를 사용한 벌크성 수정 쿼리**

```java
@Modifying
@Query("update Member m set m.age = m.age + 1 where m.age >= :age")
int bulkAgePlus(@Param("age") int age);
```

- 벌크성 수정, 삭제 쿼리는 `@Modifying` 어노테이션을 사용
    - 사용하지 않으면 오류가 발생한다
- 벌크성 쿼리를 실행하고 나서 영속성  컨텍스트 초기화 : `@Modifying(clearAutomatically = true)` (이 옵션의 기본값은 `false` )
    - 벌크성 쿼리는 영속성 컨텍스트를 초기화 하지 않고 바로 flush()를 호출해 DB에 저장한다
    - 이 옵션 없이 회원을 다시 조회하면 영속성 컨텍스트에 과거 값이 남아서 문제가 될 수 있다
    - 다시 조회해야 한다면 영속성 컨텍스트를 초기화 해야한다

> 참고: 벌크 연산은 영속성 컨텍스트를 무시하고 실행하기 때문에, 영속성 컨텍스트에 있는 엔티티의 상태와 DB에 엔티티 상태가 달라질 수 있다.
>
>
> **권장하는 방안**
>
> 1. 영속성 컨텍스트에 엔티티가 없는 상태에서 벌크 연산을 먼저 실행한다.
> 2. 부득이하게 영속성 컨텍스트에 엔티티가 있으면 벌크 연산 직후 영속성 컨텍스트를 초기화 한다.

## @EntityGraph

- 연관된 엔티티를 한번에 조회하려면 페치 조인이 필요하다
- 스프링 데이터 JPA는 JPA가 제공하는 엔티티 그래프 기능을 편리하게 사용하게 도와준다. 이 기능을 사용하면 JPQL 없이 페치 조인을 사용할 수 있다

```java
//공통 메서드 오버라이드
@Override
@EntityGraph(attributePaths = {"team"})
List<Member> findAll();

//JPQL + 엔티티 그래프
@EntityGraph(attributePaths = {"team"})
@Query("select m from Member m")
List<Member> findMemberEntityGraph();

//메서드 이름으로 쿼리에서 특히 편리하다.
@EntityGraph(attributePaths = {"team"})
List<Member> findByUsername(String username)
```

- 사실 복잡한 쿼리가 있는 경우 JPQL로 푸는 것이 제일 좋은 방법이다

## JPA Hint

### JPA Hint

```java
@QueryHints(value = @QueryHint(name = "org.hibernate.readOnly", value = "true"))
Member findReadOnlyByUsername(String username);
```

```java
@Test
public void queryHint() throws Exception {
	//given
	memberRepository.save(new Member("member1", 10));
	em.flush();
	em.clear();
	
	//when
	Member member = memberRepository.findReadOnlyByUsername("member1");
	member.setUsername("member2");
	em.flush(); //Update Query 실행X
}
```

- Spring JPA는 엔티티를 조회하면 변경 감지를 위해 두개의 객체를 생성한다
- 그리고 변경 감지를 위한 로직들이 내부적으로 돌아간다
- 정말로 조회만을 위한 쿼리가  필요하다면 불필요한 객체 생성 및 로직을을 생성하지 않아도 된다
- JPA Hint가 `readOnly` 를 통해 변경 불가능한 객체를 조회한다

### 스프링 데이터 JPA 구현체 분석

- 스프링 데이터 JPA가 제공하는 공통 인터페이스의 구현체
- `org.springframework.data.jpa.repository.support.SimpleJpaRepository`

```java
@Repository
@Transactional(readOnly = true)
public class SimpleJpaRepository<T, ID> ...{
	@Transactional
	public <S extends T> S save(S entity) {
		if (entityInformation.isNew(entity)) {
			em.persist(entity);
			return entity;
		} else {
			return em.merge(entity);
		}
	}
...
}
```

- `@Repository` 적용: JPA 예외를 스프링 추상화 한 예외로 변환
- `@Transcational` 트랜잭션 적용
    - JPA의 모든 변경은 트랜잭션 안에서 동작
    - 스프링 데이터 JPA는 변경(등록, 수정, 삭제) 메서드를 트랜잭션 처리
    - 서비스 계층에서 트랜잭션을 시작하지 않으면 리파지토리에서 트랜잭션 시작
    - 서비스 계층에서 트랜잭션을 시작하면 리파지토리는 해당 트랜잭션을 전파 받아서 사용
    - 그래서 스프링 데이터 JPA를 사용할 때 트랜잭션이 없어도 데이터 등록, 변경이 가능했음(사실은 트랜잭션이 리포지토리 계층에 걸려있는 것임)
- `@Transacational(readOnly = true)`
    - 데이터를 단순히 조회만 하고 변경하지 않는 트랜잭션에서 `readOnly = true` 옵션을 사용하면 플러시를 생략해서 약간의 성능 향샹을 얻을 수 있음

      [**@Transactional 옵션 readOnly의 이점과 주의점**](/chapter2/_posts/jpa/2024-11-28-Transactional_readOnly의_이점과_주의점.md)


### save() 메서드

- 새로운 엔티티면 저장(persist)
- 새로운 엔티티가 아니면 병합(merge)
- 엔티티가 식별자가 없다면 저장을 식별자를 가지고 있다면 병합을 실행한다
    - 병합은 엔티티를 DB에 조회 후에 기존 엔티티와 값을 비교, 변경 후 저장하기 때문에 성능 저하가 일어난다

**새로운 엔티티를 판단하는 기본 전략**

- 식별자가 객체일 때 null로 판단
- 식별자가 자바 기본 타입일 때 0으로 판단
- `Persistable` 인터페이스를 구현해서 판단 로직 변경 가능

```java
public interface Persistable<ID> {
	ID getId();
	boolean isNew();
}
```

**구현**

```java
@Entity
@EntityListeners(AuditingEntityListener.class)
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Item implements Persistable<String> {

	@Id
	private String id;
	
	@CreatedDate
	private LocalDateTime createdDate;
	
	public Item(String id) {
		this.id = id;
	}
	
	@Override
	public String getId() {
		return id;
	}
	
	@Override
	public boolean isNew() {
		return createdDate == null;
	}
}
```

- ID에 `@GeneratedValue` 을 사용할 수 없을 때 인터페이스를 상속 받아 구현 가능
- isNew() 메소드를 직접 구현하여 새로운 엔티티인지 구별해야함

참고 - 자바 ORM 표준 JPA 프로그래밍