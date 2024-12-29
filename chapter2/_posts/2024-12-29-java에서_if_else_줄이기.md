# Java에서 if-else문 줄이기

If-else 문은 흔하지만 남용되면 복잡하고 유지보수가 어려운 코드로 이어질 수 있다!

상황마다 if-else의 사용을 줄일 수 있는 좋은 전략들이 있다

- 전략 패턴
- Enum 사용
- 다형성
- 람다 표현식과 함수형 인터페이스
- 명령 패턴
- 가드 절

### 전략 패턴

전략 패턴은 알고리즘의 가족을 정의하고, 각각을 캡슐화하며, 이들이 상호 교환 가능하게 만든다. 여러가지 방법으로 특정 작업을 수행해야 할 때 유용하다

```java
public interface PaymentStrategy {
    void pay(double amount);
}
```

- 먼저 PaymentStrategy 인터페이스를 정의한다

```java
@Component
public class CreditCardPayment implements PaymentStrategy {
    @Override
    public void pay(double amount) {
        // 신용 카드 결제 처리 로직
        System.out.println("Paid " + amount + " using Credit Card.");
    }
}

@Component
public class PaypalPayment implements PaymentStrategy {
    @Override
    public void pay(double amount) {
        // PayPal 결제 처리 로직
        System.out.println("Paid " + amount + " using PayPal.");
    }
}
```

- PaymentStrategy를 구현한다.
- 그리고 정적의 context에 strategy를 삽입하여 변경이 필요한 곳에 주입하면 된다

```java
public class Context {
	private PaymentStrategy strategy;
	
	public ContextV1(PaymentStrategy strategy) {
		this.strategy = strategy;
	}
	
	public void execute() {
		//비즈니스 로직 실행
		strategy.pay(); //위임
		//비즈니스 로직 종료
	}
}

public class ContextV2 {
	public void execute(PaymentStrategy strategy) {
		//비즈니스 로직 실행
		strategy.pay(); //위임
		//비즈니스 로직 종료
	}
}
```

- 혹은 파라미터로 보내서 전략의 위임할 수 있다

### Enum 사용

Enum은 미리 정의된 상수 집합과 그와 관련된 동작을 나타내는 데 사용할 수 있다

```java
public enum OrderStatus {
    NEW {
        @Override
        public void handle() {
            System.out.println("Processing new order.");
        }
    },
    SHIPPED {
        @Override
        public void handle() {
            System.out.println("Order shipped.");
        }
    },
    DELIVERED {
        @Override
        public void handle() {
            System.out.println("Order delivered.");
        }
    };
    public abstract void handle();
}
```

### 다형성

다형성은 객체를 실제 클래스가 아닌 부모 클래스의 인스턴스로 취급할 수 있게 해준다. 이를 통해 부모 클래스의 참조를 통해 파생 클래스의 재정의된 메서드를 호출할 수 있다

**예제: 알림 시스템**

```java
public interface Notification {
    void send(String message);
}

public class EmailNotification implements Notification {
    @Override
    public void send(String message) {
        // 이메일 전송 로직
        System.out.println("Sending email: " + message);
    }
}

public class SmsNotification implements Notification {
    @Override
    public void send(String message) {
        // SMS 전송 로직
        System.out.println("Sending SMS: " + message);
    }
}
```

```java
public class Context {
	private Notification notification;
	
	public ContextV1(Notification notification) {
		this.notification = notification;
	}
	
	public void excute() {
			//비즈니스 로직 실행
			notification.send()
			//비즈니스 로직 종료
	}
}
```

```java
public class Main() {
	public void run() {
		Notification notification = new EmailNotification();
		Context context = new Context(notification);
	}
	
	public void run2() {
		Notification notification = new SmsNotification();
		Context context = new Context(notification);
	}
}
```

### 람다 표현식과 함수형 인터페이스

람다 표현식은 특히 단일 메서드 인터페이스를 다룰 때 코드를 간소화할 수 있다

**예제: 할인 서비스**

```java
public class DiscountService {
    private Map<String, Function<Double, Double>> discountStrategies = new HashMap<>();

    public DiscountService() {
        discountStrategies.put("SUMMER_SALE", price -> price * 0.9);
        discountStrategies.put("WINTER_SALE", price -> price * 0.8);
    }

    public double applyDiscount(String discountCode, double price) {
        return discountStrategies.getOrDefault(discountCode, Function.identity()).apply(price);
    }
}
```

### 명령 패턴

명령 패턴은 요청을 객체로 캡슐화하여 클라이언트를 큐, 요청, 작업으로 매개변수화할 수 있게한다

```java
public interface Command {
    void execute();
}

public class OpenFileCommand implements Command {
    private FileSystemReceiver fileSystem;

    public OpenFileCommand(FileSystemReceiver fs) {
        this.fileSystem = fs;
    }

    @Override
    public void execute() {
        this.fileSystem.openFile();
    }
}

public class CloseFileCommand implements Command {
    private FileSystemReceiver fileSystem;

    public CloseFileCommand(FileSystemReceiver fs) {
        this.fileSystem = fs;
    }

    @Override
    public void execute() {
        this.fileSystem.closeFile();
    }
}
```

```java
public class CommandExecutor implements CommandManager {
    private final Map<String, Command> commands = new HashMap<>();
    private final DefaultCommand defaultCommand = new DefaultCommand();

    public CommandExecutor(SessionManager sessionManager) {
        commands.put("OpenFile", new OpenFileCommand(sessionManager));
        commands.put("CloseFile", new CloseFileCommand(sessionManager));
    }

    @Override
    public void execute(String c) throws IOException {
        Command command = commands.getOrDefault(c, defaultCommand); // 키 값이 없을 경우 어떻게 처리할 것인가 -> default 값을 리턴한다
        command.execute();
    }
}
```

참조 - https://careerly.co.kr/comments/107907