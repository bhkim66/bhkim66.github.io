# @Transactional 옵션 readOnly의 이점과 주의점
## 이점

### 성능 최적화

- 트랜잭션을 읽기 전용으로 설정하면 해당 메서드가 데이터를 읽기만 한다는 것을 DB에 알려줌으로써 쿼리 및 캐싱을 최적화 할 수 있다
- 그리고 읽기 전용을 설정하여 데이터 변경이 일어나지 않기 때문에 변경 감지를 위한 스냅샷을 저장하는 동작 또한 하지 않아 성능이 향상되는 것을 기대할 수 있다

### 데이터 일관성

- 일반적으로 트랜잭션을 사용해서 DB에 데이터의 일관성과 무결성을 보장하기 위해 사용하는데 트랜잭션을 읽기 전용으로 설정하면 실수로 데이터를 수정해서 일관성을 위반할 가능성이 낮아진다

### 가독성 향상

- 코드를 작성하는 개발자는 `@Transactional(readOnly=true)`이 설정된 메서드가 DB에서 데이터를 읽기만 한다는 것을 명확하게 확인할 수 있다. 이로 인해 코드의 가독성이 향상이 된다.

### 부하 분산

- 실제 운용되는 서비스에서는 데이터베이스의 장애를 빠르게 복구하고, 트래픽을 분산하기 위해 실시간 복제본 데이터베이스를 운용하는 `레플리케이션(Replication)` 방식을 사용할 수 있다

    ![spring_data_jpa_2_3](/assets/img/chapter2/jpa/jpa_2_3.png)

- 레플리케이션은 `Master-Slave` 구조로 복제본 DB를 함께 운용함으로써, `Master DB`의 장애 발생 시 `Slave DB`를 `Master DB`로 승력시켜 장애를 빠르게 복구할 수 있으며, 조회 작업은 `Slave DB`에서 수행하고 수정 작업은 `Master DB`에서 수행함으로써 트래픽을 분산할 수 있다는 장점이 있다
- 이러한 데이터베이스 구조를 가져갈 때, readOnly = true가 설정되어있는 메서드의 경우 `Slave DB`에서 데이터를 가져오도록 동작한다. 이를 통해 레플리케이션의 목적에 맞게 트래픽 분산을 온전하게 적용할 수 있다는 추가적인 이점이 존재한다.

## 주의 점

- 읽기 작업만 하는 모든 메서드에 `@Transactional(readOnly=true)`을 무지성을 사용해서는 안된다
- readOnly = true를 적용할 때 Optimistic Lock의 동작에 영향을 미칠 수 있다는 것을 고려해야한다
- JPA의 동시성 제어에 접근 방식이 두 가지 존재하는데 그중 하나가 `Optimistic Lock`이고 또 다른 하나가 `Pessimistic Lock`이다

### Optimistic Lock

- 낙관적 락은 두 개의 트랜잭션이 동시에 동일한 데이터를 수정하려고 시도할 때 발생하는 충돌을 방지하는 데 사용되는 메커니즘이다
    - 트랜잭션의 대부분은 충돌이 발생하지 않는다고 낙관적으로 가정하는 방법이다
- 낙관적 락을 사용하려면 Entity에 `@Version` 어노테이션을 추가해서 사용할 수 있다

```java
@Entity
public class Post {
@GeneratedValue
@Id
private Long id;

    @Version
    private Long version;
    
    // Constructor, Getter ...
}
```

- `@Version` 어노테이션을 추가하면 트랜잭션이 엔티티를 수정할 때마다, 현재 버전 번호가 자동으로 업데이트되면 기록이 된다.
    - 다른 트랜잭션이 동일한 엔티티를 수정하려고 하면 버전 번호를 확인하는데 이때 첫 번째 트랜잭션이 수정을 하고 두 번째 트랜잭션의 버전 번호는 조회한 시점의 버전과 수정한 시점의 버전이 다르다는 것을 확인한다
    - 이때 충돌이 발생했다는 것을 두번째 트랜잭션에 알리게 된다
    - 즉 데이터를 조회한 시점의 버전과 수정하려고 할 때 버전이 일치하지 않을경우 충돌 발생이라고 간주하고 예외가 발생한다
    - *@Transactional(readOnly=true)를 사용하면 이러한 낙관적 락 동작에 영향을 미치게 된다*
        - 만일 `@Transactional(readOnly=true)`로 설정한 메서드에 엔티티를 수정하는 로직이 있을 경우, 해당 트랜잭션이 엔티티를 **수정하는 것이 아니라 읽기 전용으로 설정했기 때문에 버전 번호를 확인하지 못할 수 있다**. 이때 충돌을 감지하지 못하고 동시에 발생한 트랜잭션의 변경 사항을 덮어쓰게 되어 데이터 불일치 문제가 발생할 수 있다
        - 그래서 낙관적 락이 활성화된 엔티티는 @Transactional(readOnly=true)로 설정된 메서드에서 **엔티티를 읽기 작업만 하도록 하고, 수정하지 않도록 조심해야 한다**.