# Index Explain
쿼리를 작성했을 때 Explain 실행을 통해 INDEX 확인 절차를 통해서 성능을 최적화 할 수 있다

```java
explain (format = json) 쿼리문
```

- `format = json` : json 형태로 확인 가능

### Query Plan

Query Plan이란, SQL 관계형 데이터베이스 관리 시스템의 데이터 접근에 사용되는 순서 집합이다

```bash
id|select_type|table|type |possible_keys                                              |key                                                        |key_len|ref        |rows|Extra                             |
--+-----------+-----+-----+-----------------------------------------------------------+-----------------------------------------------------------+-------+-----------+----+----------------------------------+
 1|SIMPLE     |c    |const|PRIMARY                                                    |PRIMARY                                                    |8      |const      |1   |Using index                       |
 1|SIMPLE     |p    |ref  |paticipant_competiton_seq_competition_code_used_code_idx_01|paticipant_competiton_seq_competition_code_used_code_idx_01|19     |const,const|43  |Using index condition; Using where|
```

- 이와 같은 결과를 얻을 수 있다

### 항목

**id**: 쿼리 안에 있는 SELECT 문에 대한 순차적인 식별자

**`select_type`**: select 문의 유형

| SIMPLE | 단순 SELECT |
| --- | --- |
| PRIMAIRY | Sub Query를 사용할 경우 Sub Query의 외부에 있는 쿼리, UNION의 첫 번째 SELECT 쿼리 |
| SUBQUERY | Sub Query와 동일하나, Sub Query를 구성하는 여러 쿼리 중 첫 번째 SELECT |
| UNION | UNION 쿼리에서 Primary를 제외한 나머지 SELECT |

**table**: 참조되는, 참조 하고있는 table

**`type`**: 어떤 식으로 join 되는지 설명해줌, 아래로 갈수록 안좋은 형태이다

| 구분 | 설명 |
| --- | --- |
| system | 테이블에 단 한개의 데이터만 있는 경우 |
| const | SELECT에서 Primary Key 혹은 Unique Key를 조회하는 경우로 많아야 한 건의 데이터만 있음 |
| eq_ref | 조인수행을 위해 각 테이블에서 하나의 행만이 읽혀지는 형태. const 타입 외에 가장 훌륭한 조인타입이다 |
| ref | 조인을 할 때 Primary Key 혹은 Unique Key가 아닌 Key로 매칭하는 경우,  |
| **ref_or_null** | **ref 와 같지만 null 이 추가되어 검색되는 경우** |
| index_merge | 두 개의 인덱스가 병합되어 검색이 이루어지는 경우 |
| unique_subquery | 다음과 같이 IN 절 안의 서브쿼리에서 Primary Key가 오는 특수한 경우SELECT * FROM tab01 WHERE col01 IN (SELECT Primary Key FROM tab01); |
| index_subquery | unique_subquery와 비슷하나 Primary Key가 아닌 인덱스인 경우SELECT * FROM tab01 WHERE col01 IN (SELECT key01 FROM tab02); |
| range | 특정 범위 내에서 인덱스를 사용하여 원하는 데이터를 추출하는 경우로,데이터가 방대하지 않다면 단순 SELECT 에서는 나쁘지 않음 |
| **index** | **인덱스를 처음부터 끝까지 찾아서 검색하는 경우로, 일반적으로 인덱스 풀 스캔이라고 함** |
| **all** | **테이블을 처음부터 끝까지 검색하는 경우로, 일반적으로 테이블 풀 스캔이라고 함** |

**possible_keys:** 이용 가능성 있는 인덱스 목록

**`key:` possible_keys** 목록 중 옵티마이저가 선택한 인덱스

**key_len**: 키의 길이

**ref:** 키 컬럼에 나와 있는 인덱스에서 값을 찾기 위해 선행 테이블의 어떤 컬럼이 사용 되었는지를 나타냄

**`rows:`** 원하는 행을 가져오기 위해 얼마나 많은 행을 읽을지 예측치

**extra:** 실행계획에 있어서 SQL이 해석되어지는 부가적인 정보

| using index | 커버링 인덱스라고 하며 인덱스 자료 구조를 이용해서 데이터를 추출 |
| --- | --- |
| using where | where 조건으로 데이터를 추출. type이 ALL 혹은 Indx 타입과 함께 표현되면 성능이 좋지 않다는 의미 |
| using filesort | 데이터 정렬이 필요한 경우로 메모리 혹은 디스크상에서의 정렬을 모두 포함. 결과 데이터가 많은 경우 성능에 직접적인 영향을 줌 |
| using temporary | 쿼리 처리 시 내부적으로 temporary table이 사용되는 경우를 의미함 |