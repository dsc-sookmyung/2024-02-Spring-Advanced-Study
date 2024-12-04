## Previously on Tobby's Spring 6 

**AOP**(비즈니스 로직과 관련 없는 공통 관심사를 모듈화 -> 코드 중복 간소화 프로그래밍 패러다임)에서 트랜잭션 관리는 대표적인 활용 사례이다. 
트랜잭션이란 DB에서 하나의 논리적 작업 단위로 <u>더 이상 쪼갤 수 없는</u> 최소 단위를 의미한다. 
개념을 토대로 트랜잭션은 **ACID** 원칙이 있는데, 이것은 다음과 같다.

1. **Atomicity** - 트랜잭션 내의 작업은 모두 commit 되거나 rollback 된다.
2. **Consistency** - 트랜잭션 성공 시, 데이터는 항상 일관된 상태이다.
3. **Isolation** - 서로간의 작업은 간섭받지 않는다.
4. **Durability(지속성)** - 완료 후, 데이터는 영구 저장된다. 

한편 AOP는 Before, After, Around 와 같이 메서드 실행 전후 작업을 처리하는데, AOP를 활용한 트랜잭션의 경우, 선언적 트랜잭션이라고 한다...
</br>
## 선언적 트랜잭션

### TransactionAspect
```
@Aspect
@Component
public class TransactionAspect {

    @Autowired
    private DataSource dataSource;

    private ThreadLocal<Connection> connectionHolder = new ThreadLocal<>();

    // 트랜잭션 시작 (Before)
    @Before("execution(* com.example.dao.*.*(..))")
    public void startTransaction() throws SQLException {
        Connection connection = dataSource.getConnection();
        connection.setAutoCommit(false); // 트랜잭션 시작
        connectionHolder.set(connection);
        System.out.println("트랜잭션 시작");
    }

    // 정상적으로 메서드가 실행된 경우 트랜잭션 커밋 (AfterReturning)
    @AfterReturning("execution(* com.example.dao.*.*(..))")
    public void commitTransaction() throws SQLException {
        Connection connection = connectionHolder.get();
        if (connection != null) {
            connection.commit(); // 커밋
            connection.close();
            connectionHolder.remove();
            System.out.println("트랜잭션 커밋");
        }
    }

    // 메서드 실행 중 예외 발생 시 롤백 (AfterThrowing)
    @AfterThrowing("execution(* com.example.dao.*.*(..))")
    public void rollbackTransaction() throws SQLException {
        Connection connection = connectionHolder.get();
        if (connection != null) {
            connection.rollback(); // 롤백
            connection.close();
            connectionHolder.remove();
            System.out.println("트랜잭션 롤백");
        }
    }

    // 메서드 실행 전후를 모두 관리 (Around)
    @Around("execution(* com.example.dao.*.*(..))")
    public Object manageTransaction(ProceedingJoinPoint joinPoint) throws Throwable {
        Connection connection = dataSource.getConnection();
        try {
            connection.setAutoCommit(false); // 트랜잭션 시작
            connectionHolder.set(connection);
            System.out.println("트랜잭션 시작 (Around)");

            Object result = joinPoint.proceed(); // 실제 메서드 실행

            connection.commit(); // 커밋
            System.out.println("트랜잭션 커밋 (Around)");
            return result;

        } catch (Exception e) {
            connection.rollback(); // 롤백
            System.out.println("트랜잭션 롤백 (Around)");
            throw e;

        } finally {
            connection.close();
            connectionHolder.remove();
            System.out.println("트랜잭션 종료 (Around)");
        }
    }
}
```
### In DAO..
```
@Repository
public class UserDao {

    @Autowired
    private DataSource dataSource;

    // 공통적으로 PreparedStatement를 처리하는 유틸리티 메서드
    private PreparedStatement prepareStatement(Connection connection, String sql, Object... params) throws SQLException {
        PreparedStatement ps = connection.prepareStatement(sql);
        for (int i = 0; i < params.length; i++) {
            ps.setObject(i + 1, params[i]);
        }
        return ps;
    }

    public void saveUser(User user) throws SQLException {
        try (Connection connection = dataSource.getConnection()) {
            PreparedStatement ps = prepareStatement(connection, "INSERT INTO users (name, age) VALUES (?, ?)",
                    user.getName(), user.getAge());
            ps.executeUpdate();
        }
    }
}
```
AOP를 적용했으므로 DAO 클래스에서는 트랜잭션 코드 없이 비즈니스 로직에만 집중할 수 있게 된다.

### JPA에서의 ...
**jpa는 @Transactional 만으로 선언적 트랜잭션을 사용하게 하는 ORM 기술이다. **

</br>
</br>

## 개요

6.6 단원에서는 트랜잭션의 속성을 살펴본다. 
6.7 단원에서는 애노테이션을 사용하는 트랜잭션을 살펴보고 세밀한 조정이 필요할 때는 매번 포인트컷과 어드바이스를 추가해야 하는데, 이 때의 지저분해지는 포인트컷을 조정하기 위한 스프링 도구를 살핀다. 
그 후 6.8에서는 트랜잭션을 활용하는 클래스의 테스트를 어떻게 할 것인지 살펴본다.

## 트랜잭션의 속성
트랜잭션에는 전파 개념이 있다. 트랜잭션이 이미 존재하는 상황에서 새로 호출된 메서드가 해당 트랜잭션에 끼워져서 활동하는지, 또다른 독립적인 트랜잭션을 생성할지 결정하는 것이다. 즉, 활동 중인 트랜잭션이 있을 때 추가 트랜잭션을 어떻게 처리하는지에 관한 것이다. 
![](https://velog.velcdn.com/images/ykky2115/post/16f69fab-e8d5-4c5e-a8e0-0added819c7a/image.png)


보이는 것과 같이 A 트랜잭션을 활동 중 B 트랜잭션이 호출된다면? A 트랜잭션과 B 트랜잭션의 독립성 여부는 **전파 속성**에 따라 다르다.
## 트랜잭션 전파

### 1. PROPAGATION_REQUIRED
기본적인 속성이다. A가 활동 중이라면 B가 참여하고 역전의 경우에도 가능한
참여하는 속성이다.
![](https://velog.velcdn.com/images/ykky2115/post/e3548c0f-c5fd-40a4-ae71-a8b626bd82b7/image.png)

### 2. PROPAGATION_REQUIRES_NEW
매 새로운 트랜잭션을 시작한다. 독립적인 트랜잭션을 보장하기 위함인데 아래 그림을 예시로 든다면, 
중심이 되는 _영화 서비스와 영화 예매_ 두 논리 트랜잭션은 같은 물리 트랜잭션으로 묶일 수 있으나 서비스 이용자인 '김유빈'이 영화 예매 로그가 저장되는 것은 1도 궁금치 않으니 _로그 저장_이 rollback되어도 영화 예매에는 문제가 없도록 설계할 때 REQUIRES_NEW를 로그 저장 트랜잭션에 붙이는 것이다. 
![](https://velog.velcdn.com/images/ykky2115/post/e4f2ae3b-8c95-4a15-870c-5bbb81065381/image.png)
총 두 개의 물리 단위를 갖게 되는 것이다. 

### 3. PROPAGATION_NOT_SUPPORTED
트랜잭션이 없이 동작하도록 만든다. 진행 중인 트랜잭션이 있어도 무시한다. 
![](https://velog.velcdn.com/images/ykky2115/post/cb091382-ab76-43b2-aa22-a0f79c196d3e/image.png)
무시하는 속성을 두는 이유에는 AOP 비적용에 있다. 
트랜잭션 경계설정은 보통 AOP를 이용해 한 번에 많은 메소드를 동시 적용한다. 
그러나 그 중 특정 메소드만 트랜잭션 적용 제외가 필요하다면? 
이 때 특정 메소드에게는 NOT_SUPPORTED를 적용하는 것이다. 
앞서 개요에서 말했듯 포인트컷을 잘 만드는 방식도 있겠으나, 자칫 포인트컷이 get dirty해질수도 있으므로 해당 기능을 활용하도록 한다. 


#### 이 외에도...
![](https://velog.velcdn.com/images/ykky2115/post/74ce9841-089f-48c5-9e2b-1779bc34b823/image.png)



[10분 테코톡](https://www.youtube.com/watch?v=b0s9RzKyHN0) 키아라의 스프링 트랜잭션 전파 참조.

## 격리수준 
DB 트랜잭션은 보통 동시에 진행된다. 적절히 격리 수준을 조정해서 가능한 한 많은 트랜잭션을 동시에 진행시키면서도 문제가 발생하지 않도록 하는 제어가 필요하다. 기본적으로 격리수준은 DB에 설정되어 있으나 JDBC 드라이버나 DataSource에서(혹은 트랜잭션 단위로도) 재설정할 수 있다. 

## 제한시간
트랜잭션을 수행하는 제한시간을 설정할 수 있다. 제한시간은 REQUIRED/REQUIRES_NEW와 함께 사용해야 의미가 있다.

## 읽기전용 
트랜잭션 내에서 데이터를 조작할 수 없게 하는 것인데 데이터 액세스 기술에 따라 <u>성능이 향상</u>될 수 있다.

## TransactionInterceptor
기본적으로 두 가지 종류의 예외 처리 방식을 제공한다.

1. <u>런타임 예외</u>  발생 시, 트랜잭션은 <u>롤백</u>된다. 타깃 메소드가 <u>체크 예외</u> 를 던질 때는 예외상황으로 해석하지 않아 트랜잭션은 <u>커밋</u> 된다. [기본 원칙]
2. TransactionAttribute에서 rollbackOn()이라는 속성을 둬서 <u> 특정 체크 예외</u> 에는 <u>롤백</u> 을 하도록 하고, <u> 특정 런타임</u> 에서는 <u>커밋</u>을 하게끔 만드는 것이다. 

인터셉터는 애튜리뷰트를 Properties라는 컬렉션 타입의 맵 오브젝트로 전달받는다.

### 메서드 이름 패턴을 이용한 트랜잭션 속성 지정
위에서의 TransactionAttribute는 다음과 같은 형식을 갖는다. 

> PROPAGATION_NAME, ISOLATION_NAME, readOnly, timeout_NNNN, -Exception1, +Exception2

트랜잭션 전파 항목만 필수이다!

#### 포인트컷과 트랜잭션 속성의 적용 전략

1. 트랜잭션 포인트컷 표현식은 타입 패턴이나 빈 이름을 이용한다.

 트랜잭션을 적용할 타깃 클래스의 메서드는 모두 트랜잭션 적용 후보가 되는 것이 바람직하다.
add() 메서드도 다른 트랜잭션에 참여할 가능성이 높기 때문에 트랜잭션 적용 대상이어야 한다.
조회의 경우, 읽기전용으로 트랜잭션 속성을 설정해두면 그만큼 성능의 향상을 가져올 수 있다. 또는 복잡한 조회의 경우 제한시간을 지정하거나 격리 수준에 따라 조회도 반드시 트랜잭션 안에서 진행해야할 필요가 발생하기도 한다.
따라서 트랜잭션용 포인트컷 표현식에는 메서드나 파라미터, 예외에 대한 패턴을 정의하지 않는게 바람직하다.
가능하면 클래스보다 인터페이스 타입을 기준으로 타입 패턴을 적용하는 것이 좋다.
메서드의 execution() 방식의 포인트컷 표현식 대신, 스프링의 빈 이름을 이용하는 bean() 표현식을 사용하는 방법도 고려해볼만하다.
 

2. 공통된 메서드 이름 규칙을 통해 최소한의 트랜잭션 어드바이스와 속성을 정의한다. </br>
기준의 되는 몇 가지 트랜잭션 속성을 정의하고, 그에 따라 적절한 메서드 명명 규칙을 만들어 두면 하나의 어드바이스만으로 어플리케이션의 모든 서비스 빈에 트랜잭션 속성을 지정할 수 있다.
가끔 예외적인 경우는 트랜잭션 어드바이스와 포인트컷을 새롭게 추가할 필요가 있다.
트랜잭션 적용 대상 클래스의 메서드는 일정한 명명 규칙을 따르게 해야 한다.


## 트랜잭션 속성 적용 

#### 1. 트랜잭션 경계설정의 일원화
서비스 계층 클래스가 트랜잭션 경계를 부여하기에 적절하다. 
또한 DAO에 접근하는 것은 해당 서비스 클래스여야 하며 테스트 같은 특별 케이스는 제외한다. 
DAO 메소드는 서비스 클래스에도 추가한다.
```
public interface UserService {
	void add();
    List<User> get();
    void deleteAll();
    void update();
}
```

#### 2. 서비스 빈에 적용되는 포인트컷 표현식, 어드바이스 등록
XML -> AppConfig.class로 변환해보았다.

```
@Configuration
@EnableTransactionManagement
public class AppConfig {

    @Bean
    public Advisor transactionAdvisor(PlatformTransactionManager transactionManager) {
        AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();
        pointcut.setExpression("bean(*Service)"); // 빈 이름 기준 포인트컷

        TransactionInterceptor interceptor = new TransactionInterceptor();
        interceptor.setTransactionManager(transactionManager);
        interceptor.setTransactionAttributeSource(transactionAttributeSource());

        return new DefaultPointcutAdvisor(pointcut, interceptor);
    }

    @Bean
    public TransactionAttributeSource transactionAttributeSource() {
        NameMatchTransactionAttributeSource source = new NameMatchTransactionAttributeSource();
        RuleBasedTransactionAttribute txAttribute = new RuleBasedTransactionAttribute();
        txAttribute.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
        source.addTransactionalMethod("*", txAttribute);
        return source;
    }
}
```

## 애노테이션 트랜잭션 속성과 포인트컷
다음은 @Transactional을 정의한 코드이다. ion을 무시해주도록 하자. 
![](https://velog.velcdn.com/images/ykky2115/post/c58e44db-2237-4031-9df6-fa697dc6b964/image.png)
메타 애노테이션을 보면
- 클래스나 메서드 수준에 트랜잭션이 적용될 수 있고
- 런타임에도 유지되어야 하며, 리플렉션으로 접근 가능하다
- 또한 하위 클래스 상속 가능하 

주요 속성은
- value, transactionManager: 트랜잭션 매니저 지정
- propagation: 전파 수준 설정
- isolation: 격리수준 지정
- timeout, timeoutString: 최대 실행 시간. 기본값은 -1(무한대기)
- readOnly: 읽기 전용인가? 기본값은 false
- rollbackFor, rollbackForClassName: 특정 예외시 롤백 설정
- noRollbackFor, noRollbackForClassName: 특정 예외시에도 롤백 X



![](https://velog.velcdn.com/images/ykky2115/post/cf951e1e-7419-498d-83df-d32776732010/image.png)
트랜잭션을 사용했을 때의 어드바이저의 동작방식.


## 트랜잭션 테스트 하는 법

#### @Transactional/@Rollback

1.	@Transactional
 • 테스트 메서드에 트랜잭션을 적용.
	•	테스트가 끝나면 트랜잭션은 자동으로 롤백.
	•	트랜잭션 범위 내에서 DB 변경 작업을 수행.
2.	@Rollback
	•	@Transactional과 함께 사용하여 테스트 후 데이터를 롤백.
	•	@Rollback(false)를 사용하면 트랜잭션이 커밋되어 데이터베이스에 반영.

테스트 클래스와 메소드에도 위 애노테이션을 사용할 수 있다. 

```
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.transaction.annotation.Transactional;

import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest
@Transactional // 자동으로 롤백 처리
public class UserServiceTest {

    @Autowired
    private UserService userService;

    @Autowired
    private UserRepository userRepository;

    @Test
    public void testAddUserWithRollback() {
        // Given
        User user = new User("test@example.com", "TestUser", "password123");

        // When
        userService.add(user);

        // Then
        User savedUser = userRepository.findByEmail("test@example.com");
        assertNotNull(savedUser); // 데이터가 정상적으로 저장됨
        assertEquals("TestUser", savedUser.getNickname());
    }
}
```

```
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.test.annotation.Rollback;

import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest
@Transactional
public class UserServiceCommitTest {

    @Autowired
    private UserService userService;

    @Autowired
    private UserRepository userRepository;

    @Test
    @Rollback(false) // 롤백하지 않고 커밋
    public void testAddUserWithoutRollback() {
        // Given
        User user = new User("commit@example.com", "CommitUser", "password123");

        // When
        userService.add(user);

        // Then
        User savedUser = userRepository.findByEmail("commit@example.com");
        assertNotNull(savedUser); // 데이터가 저장되고 커밋됨
    }
}
```
