# QueryDSL

### JPAQueryFactory

- QueryDSL을 사용하기위해선 JPAQueryFactory가 필요하다

    ```java
    @Autowired
    EntityManager em;
    
    JPAQueryFactory queryFactory;
    
    public void init() {
      queryFactory = new JPAQueryFactory(em);
    }
    ```

    - JPAQueryFactory 객체를 EntityManager 객체로 생성

    ```java
    @Configuration
    public class QuerydslConfiguration {
    
        @PersistenceContext
        private EntityManager entityManager;
    
        @Bean
        public JPAQueryFactory jpaQueryFactory() {
            return new JPAQueryFactory(entityManager);
        }
    }
    ```

    - JPAQueryFactory Bean을 생성자 인젝션으로 주입해 사용
    - 요즘은 `@PersistenceContext` 대신 `@Autowired`로 선언해도 된다

    ```java
    @Configuration
    public class JpaQueryConfig {
        @Bean
        public JPAQueryFactory jpaQueryFactory(EntityManager em) {
            return new JPAQueryFactory(em);
        }
    }
    ```

- 대부분의 쿼리 작성 키워드는 모두 사용 가능하다. (join, on, groupBy, orderBy 등)

## Q-Type 활용

### Q클래스 인스턴스를 사용하는 방법

```java
QMember qMember = new QMember("m"); //별칭 직접 지정
QMember qMember = QMember.member; //기본 인스턴스 사용
```

**기본 인스턴스를 static import와 함께 사용**

```java
import static study.querydsl.entity.QMember.*;

@Test
public void startQuerydsl3() {
//member1을 찾아라.
	Member findMember = queryFactory
		.select(member)
		.from(member)
		.where(member.username.eq("member1"))
		.fetchOne();
	
	assertThat(findMember.getUsername()).isEqualTo("member1");
}
```

- 같은 테이블을 조인해야 하는 경우가 아니면 기본 인스턴스를 사용하자
- 같은 테이블을 조인하게 되면 alias를 따로 지정해줘야 한다
    - `QMember qMember = new QMember("m")`

## 검색 조건 쿼리

### JPQL이 제공하는 모든 검색 조건 제공

```java
member.username.eq("member1") // username = 'member1'
member.username.ne("member1") //username != 'member1'
member.username.eq("member1").not() // username != 'member1'
member.username.isNotNull() //이름이 is not null

member.age.in(10, 20) // age in (10,20)
member.age.notIn(10, 20) // age not in (10, 20)
member.age.between(10,30) //between 10, 30
member.age.goe(30) // age >= 30
member.age.gt(30) // age > 30
member.age.loe(30) // age <= 30
member.age.lt(30) // age < 30

member.username.like("member%") //like 검색
member.username.contains("member") // like ‘%member%’ 검색
member.username.containsIgnoreCase("member") // 대소문자 무시 like ‘%member%’ 검색
member.username.startsWith("member") //like ‘member%’ 검색
```

### 결과 조회

- `fetch()` : 리스트 조회, 데이터가 없으면 빈 리스트 반환
- `fetchOne()` : 단 건 조회
    - 결과가 없으면 : null
    - 결과가 둘 이상이면 : `com.querydsl.core.NonUniqueResultException`
- `fetchFirst()` : `limit(1).fetchOne()`
- `~~fetchResults()`: 페이징 정보 포함, total count 쿼리 추가 실행~~
    - 데이터가 많고 복잡한 쿼리 같은 경우 성능을 위해서 `content` 쿼리와 `count`쿼리를 분리하는 것이 유리하다
    - Querydsl 5.0 *deprecated 되어 사용할 수 없다*
        - fetch()로 대체, content와 totalCount를 각각 조회해야 한다
- `~~fetchCount()` : count 쿼리로 변경해서 count 수 조회~~
    - Querydsl 5.0 *deprecated 되어 사용할 수 없다*
        - fetchOne()으로 대체

### 정렬

```java
List<Member> result = queryFactory
	.selectFrom(member)
	.where(member.age.eq(100))
	.orderBy(member.age.desc(), member.username.asc().nullsLast())
	.fetch();
```

- `desc()`, `asc()` : 일반 정렬
- `nullsLast()`, `nullsFirst()` : null 데이터 순서 부여

### 페이징

**조건 건수 제한**

```java
@Test
public void paging1() {
	List<Member> result = queryFactory
		.selectFrom(member)
		.orderBy(member.username.desc())
		.offset(1) //0부터 시작(zero index)
		.limit(2) //최대 2건 조회
		.fetch();

	assertThat(result.size()).isEqualTo(2);
}
```

**전체 조회 건수**

```java
@Test
public void paging2() {
	QueryResults<Member> queryResults = queryFactory
		.selectFrom(member)
		.orderBy(member.username.desc())
		.offset(1)
		.limit(2)
		.fetchResults();
	
	assertThat(queryResults.getTotal()).isEqualTo(4);
	assertThat(queryResults.getLimit()).isEqualTo(2);
	assertThat(queryResults.getOffset()).isEqualTo(1);
	assertThat(queryResults.getResults().size()).isEqualTo(2);
}
```

- *count 쿼리가 실행되니 성능상 주의*
- 실무에서 페이징 쿼리를 작성할 때, 데이터를 조회하는 쿼리는 여러 테이블을 조인해야 하지만, count 쿼리는 조인이 필요 없는 경우도 있다
    - 그런데 이렇게 자동화된 count 쿼리는 원본 쿼리와 같이 모두 조인을 해버리기 때문에 성능이 안 나올 수  있다
    - count 쿼리에 조인이 필요없는 성능 최적화가 필요하다면, count 전용 쿼리를 별도로 작성해야 한다

### 집합 함수

```java
List<Tuple> result = queryFactory
	.select(member.count(),
		member.age.sum(),
		member.age.avg(),
		member.age.max(),
		member.age.min())
	.from(member)
	.fetch();
```

### GroupBy 사용

```java
List<Tuple> result = queryFactory
	.select(team.name, member.age.avg())
	.from(member)
	.join(member.team, team)
	.groupBy(team.name)
	.fetch();
```

- GroupBy에서 결과를 제한하려면 `having`

```java
.groupBy(item.price)
.having(item.price.gt(1000))
```

## 조인

### 기본 조인

- 조인의 기본 문법은 첫 번째 파라미터에 조인 대상을 지정하고, 두 번째 파라미터에 별칭(alias)으로 사용할 Q 타입을 지정하면 된다
    - **join(조인 대상, 별칭으로 사용할 Q타입)**

```java
List<Member> result = queryFactory
	.selectFrom(member)
	.join(member.team, team)
	.where(team.name.eq("teamA"))
	.fetch();
```

- `join()`, `innerJoin()` : 내부 조인(inner join)
- `leftJoin()` : left 외부 조인(left outer join)
- `rightJoin()`: right 외부 조인(right outer join)
- JPQL의 on과 성능 최적화를 위한 fetch 조인 제공

### 세타 조인

```java
List<Member> result = queryFactory
	.select(member)
	.from(member, team)
	.where(member.username.eq(team.name))
	.fetch();
```

- from 절에 여러 엔티티를 선택해서 세타 조인
- 외부 조인 불가능

### 조인 on 절

- ON절을 활용한 조인
    - 조인 대상 필터링
    - 연관관계 없는 엔티티 외부 조인

### 조인 대상 필터링

```java
* 예) 회원과 팀을 조인하면서, 팀 이름이 teamA인 팀만 조인, 회원은 모두 조회
* JPQL: SELECT m, t FROM Member m LEFT JOIN m.team t on t.name = 'teamA'
* SQL: SELECT m.*, t.* FROM Member m LEFT JOIN Team t ON m.TEAM_ID=t.id and t.name='teamA'

List<Tuple> result = queryFactory
	.select(member, team)
	.from(member)
	.leftJoin(member.team, team).on(team.name.eq("teamA"))
	.fetch();
```

- on 절을 활용해 조인 대상을 필터링 할 때, 외부조인이 아니라 내부조인을 사용하면, where 절에서 필터링 하는 것과 동일하다
    - 따라서 on 절을 활용한 조인 대상 필터링을 사용할 때, 내부조인 이면 익숙한 where을 쓰고 외부조인이 필요한 경우에만 on절을 사용해야 한다

### 연관 관계가 없는 엔티티 외부 조인

```java
* 예) 회원의 이름과 팀의 이름이 같은 대상 외부 조인
* JPQL: SELECT m, t FROM Member m LEFT JOIN Team t on m.username = t.name
* SQL: SELECT m.*, t.* FROM Member m LEFT JOIN Team t ON m.username = t.name

List<Tuple> result = queryFactory
	.select(member, team)
	.from(member)
	.leftJoin(team).on(member.username.eq(team.name))
	.fetch();
```

- 주의! 문법을 잘 봐야 한다. **leftJoin()** 부분에 일반 조인과 다르게 엔티티 하나만 들어간다.
    - 일반조인: `leftJoin(member.team, team)`
    - on조인: `from(member).leftJoin(team).on(xxx)`

### fetch 조인

```java
Member findMember = queryFactory
	.selectFrom(member)
	.join(member.team, team).fetchJoin()
	.where(member.username.eq("member1"))
	.fetchOne();
```

- 즉시 조인으로 Member, Team SQL 쿼리 조인으로 한번에 조회

### 서브 쿼리

`com.querydsl.jpa.JPAExpressions` 사용

```java
List<Member> result = queryFactory
	.selectFrom(member)
	.where(member.age.eq(
		JPAExpressions
			.select(memberSub.age.max())
			.from(memberSub)
		))
	.fetch();
	
List<Member> result = queryFactory
	.selectFrom(member)
	.where(member.age.goe(
		JPAExpressions
			.select(memberSub.age.avg())
			.from(memberSub)
		))
	.fetch();
```

**select 절에 subQuery**

```java
List<Tuple> fetch = queryFactory
	.select(member.username,
		JPAExpressions
			.select(memberSub.age.avg())
			.from(memberSub)
	).from(member)
	.fetch();
```

**from 절의 서브쿼리 한계**

JPA JPQL 서브쿼리의 한계점으로 *from 절의 서브쿼리(인라인 뷰)는 지원하지 않는다*. 당연히 QueryDsl도 지원하지 않는다

**from 절의 서브쿼리 해결방안**

1. 서브쿼리를 join으로 변경한다.
2. 애플리케이션에서 쿼리를 2번 분리해서 실행한다
3. natvieeSQL을 사용한다

### case 문

select, 조건절(where), order by에서 사용 가능

```java
List<String> result = queryFactory
	.select(member.age
		.when(10).then("열살")
		.when(20).then("스무살")
		.otherwise("기타"))
	.from(member)
	.fetch();
	
List<String> result = queryFactory
	.select(new CaseBuilder()
		.when(member.age.between(0, 20)).then("0~20살")
		.when(member.age.between(21, 30)).then("21~30살")
		.otherwise("기타"))
	.from(member)
	.fetch();
```

```java
NumberExpression<Integer> rankPath = new CaseBuilder()
	.when(member.age.between(0, 20)).then(2)
	.when(member.age.between(21, 30)).then(1)
	.otherwise(3);
	
List<Tuple> result = queryFactory
	.select(member.username, member.age, rankPath)
	.from(member)
	.orderBy(rankPath.desc())
	.fetch();
```

- 복잡한 조건을 변수로 선언해서 `select`절, `orderBy`절에서 함께 사용할 수 있다
- 쿼리에서 값을 변경해서 넘겨주기보단 raw값을 넘겨서 비지니스단에서 처리하는 것이 나은 방법이다

### DTO 조회

결과를 DTO로 변환할 때 사용

- 프로퍼티 접근 - Setter 필요
    - DTO에 기본 생성자 필요

    ```java
    List<MemberDto> result = queryFactory
    	.select(Projections.bean(MemberDto.class,
    		member.username,
    		member.age))
    	.from(member)
    	.fetch();
    ```

- 필드 직접 접근

    ```java
    List<MemberDto> result = queryFactory
    	.select(Projections.fields(MemberDto.class,
    		member.username,
    		member.age))
    	.from(member)
    	.fetch();
    ```

- 생성자 사용

    ```java
    List<MemberDto> result = queryFactory
    	.select(Projections.constructor(MemberDto.class,
    		member.username,
    		member.age))
    	.from(member)
    	.fetch();
    }
    ```


**별칭이 다를 때**

```java
List<UserDto> fetch = queryFactory
	.select(Projections.fields(UserDto.class, 
		member.username.as("name"), // 별칭 사용
		ExpressionUtils.as(         // ExpressionUtils 사용
			JPAExpressions
				.select(memberSub.age.max())
				.from(memberSub), "age")
		)
	).from(member)
	.fetch();
```

- 프로퍼티나, 필드 접근 생성 방식에서 이름이 다를 때 해결 방안
- `ExpressionUtils.as(source, alias)` : 필드나, 서브 쿼리에 별칭 적용
- `username.as("memberName")` : 필드에 별칭 사용

### @QueryProjection

**생성자 + @QueryProjection**

```java
@Data
public class MemberDto {
	private String username;
	private int age;
	public MemberDto() {}
	
	@QueryProjection
	public MemberDto(String username, int age) {
		this.username = username;
		this.age = age;
	}
}
```

```java
List<MemberDto> result = queryFactory
	.select(new QMemberDto(member.username, member.age))
	.from(member)
	.fetch();
```

- 이 방법은 컴파일러로 타입을 체크할 수 있으므로 가장 안전한 방법이다. 하지만 DTO에서 QueryDSL 어노테이션을 유지해야 하므로 의존성 문제가 발생한다
- DTO까지 Q파일을 생성해야 하는 단점이 있다

### distinct

```java
List<String> result = queryFactory
	.select(member.username).distinct()
	.from(member)
	.fetch();
```

### **동적 쿼리 작성**

QueryDSL에서는 `BooleanBuilder`와 `where절 다중 파라미터`를 이용해 동적 쿼리를 처리할 수 있다

**BooleanBuilder**

```java
@Test
public void 동적쿼리_BooleanBuilder() throws Exception {
	String usernameParam = "member1";
	Integer ageParam = 10;
	List<Member> result = searchMember1(usernameParam, ageParam);
	Assertions.assertThat(result.size()).isEqualTo(1);
}

public void searchMember1(String usernameParam, Integer ageCond) {
	BooleanBuilder builder = new BooleanBuilder();
	// BooleanBuilder builder = new BooleanBuilder(member.username.eq(user)); 초기값 셋팅가
	if (usernameCond != null) {
		builder.and(member.username.eq(usernameCond));
	}
	if (ageCond != null) {
		builder.and(member.age.eq(ageCond));
	}
	return queryFactory
					.selectFrom(member)
					.where(builder)
					.fetch();
}
```

```java
void dynamicQuery_WhereParam() {

    String usernameParam = "member1";
    Integer ageParam = 10;

    List<Member> result = searchMember(usernameParam, ageParam);
    assertThat(result.size()).isEqualTo(1);
}

private List<Member> searchMember(String usernameCond, Integer ageCond) {

    return queryFactory
            .selectFrom(member)
            .where(usernameEq(usernameCond), ageEq(ageCond))
            .fetch();
}

private BooleanExpression usernameEq(String usernameCond) {

    return usernameCond != null ? member.username.eq(usernameCond) : null ;
}

private BooleanExpression ageEq(Integer ageCond) {

    return ageCond != null ? member.age.eq(ageCond) : null;
}

private BooleanExpression allEq(String usernameCond, Integer ageCond) {

    return usernameEq(usernameCond).and(ageEq(ageCond));
}

```

- where절에서 BooleanExpression을 반환하는 메서드를 사용한다. 그리고 usernameCond, ageCond에서 값이 없다면 null을 반환하고, 아닐 경우 값을 반환한다. where절에서 null 값인 경우 조건을 무시한다.
- `usernameEq` 메서드와 `ageEq` 메서드를 조합하여 `allEq` 메서드를 생성할 수 있다. 이처럼 재활용 할 수 있고 객체지향적으로 관리할 수 있다는 점이 `where절 다중 파라미터` 방식의 장점이다

### 수정, 삭제 벌크 연산

- **execute**
    - 모든 종류의 쿼리를 실행하는 데 사용된다
        - Boolean 값을 반환한다
          SELECT 쿼리를 통해 ResultSet을 받으면 True를 반환한다
        - INSERT/UPDATE/DELETE 쿼리를 실행하면 별도의 ResultSet이 없으므로 False를 반환한다
- **executeQuery**
    - SELECT 쿼리를 실행하는 데 사용된다
    - ResultSet을 반환한다
        - 이때, 반환 값이 없더라도 ResultSet은 null이 아니다
    - INSERT/UPDATE/DELETE 쿼리를 실행하면 SQLException이 발생한다
- **executeUpdate**
    - 아무것도 반환하지 않는 DML(INSERT/UPDATE/DELETE)문 또는 DDL문을 실행하는 데 사용된다
    - Int 값을 반환한다
        - DML문을 실행할 땐 영향받은 행 수를 반환한다
        - DDL문을 실행할 땐 0을 반환한다 (MSSQL 드라이버 기준)

![jpa_3_1](/assets/img/chpater2/jpa/jpa_3_1.png)

- `MemberRepository`에 `JpaRepository`와 `MemberRepositoryCustom`을 다중 상속받아 SpringDataJPA 기능과 `MemberRepositoryCustom`의 구현체인 `MemberRepositoryCustomImpl`에서 구현한 메서드를 사용한다.
- 사용자 정의 클래스인 경우 파일명이 `사용자 정의 인터페이스명 + Impl`인 클래스를 찾아 삽입해준다. 따라서, 사용자 정의 레포지토리 구현체의 파일명은 `인터페이스명 + Impl` 이여야 한다.
- 특정 엔티티에 종속되는 것이 아닌 여러 테이블과 조인을 하거나 특정 화면을 위한 API가 필요하다면 `@Repository`를 직접 구현해도 된다

### 페이징

```java
@Override
public Page<MemberTeamDto> searchPageComplex(MemberSearchCondition condition, Pageable page) {
    List<MemberTeamDto> content = queryFactory
            .select(Projections.constructor(
                    MemberTeamDto.class,
                    member.id,
                    member.username,
                    member.age,
                    team.id,
                    team.name))
            .from(member)
            .leftJoin(member.team, team)
            .where(userNameEq(condition.getUsername()),
                    teamNameEq(condition.getTeamName()),
                    ageGoe(condition.getAgeGoe()),
                    ageLoe(condition.getAgeLoe()))
            .offset(page.getOffset())
            .limit(page.getPageSize())
            .fetch();

    JPAQuery<Long> countQuery = queryFactory
            .select(member.count())
            .from(member)
            .leftJoin(member.team, team)
            .where(userNameEq(condition.getUsername()),
                    teamNameEq(condition.getTeamName()),
                    ageGoe(condition.getAgeGoe()),
                    ageLoe(condition.getAgeLoe()));
    return PageableExecutionUtils.getPage(content, page, countQuery::fetchOne);
//        return new PageImpl<>(content, page, total);
}
```

- `fetchResults()`가 *deprecated* 되었기 때문에 content와 count를 나눠서 조회해야 한다
- `PageableExecutionUtils`를 사용하여 count 쿼리가 생략 가능할 경우 생략해서 처리한다
    - 페이지 시작이면서 컨텐츠 사이즈 페이지 사이즈보다 작을 때
    - 마지막 페이지 일 때 (offset + 컨텐츠 사이즈를 더해서 전체 사이즈를 구함)

참고
인프런 - 실전! Querydsl