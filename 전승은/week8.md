## 목차
>[6.6 트랜잭션 속성](#66-트랜잭션-속성)
> [6.7 애노테이션 트랜잭션 속성과 포인트컷](#67-애노테이션-트랜잭션-속성과-포인트컷)
> [6.8 트랜잭션 지원 테스트](#68-트랜잭션-지원-테스트)


## 6.6 트랜잭션 속성
트랜잭션을 가져올 때 파라미터로 트랜잭션 매니저에게 전달하는 DefaultTransactionDefinition의 용도가 무엇인지 알아보자.

### 6.1.1 트랜잭션 정의
트랜잭션의 동작방식에 영향을 줄 수 있는 네 가지 속성의 정의
- 트랜잭션 전파
: 트랜잭션의 경계에서 이미 진행 중인 트랜잭션이 있을 때 또는 없을 때 어떻게 동작할 것인가를 경정하는 방식을 말한다.
	- PROPAGATION_REQUIRED
진행 중인 트랜잭션이 없으면 새로 시작하고, 이미 시작된 트랜잭션이 있으면 이에 참여한다.
DefaultTransactionDefinition의 트랜잭션 전파 속성
	- PROPAGATION_REQUIRES_NEW
항상 샐운 트랜잭션을 시작
	- PROPAGATION_NOT_SUPPORTED
트랜잭션 없이 동작
모든 메소드에 트랜잭션 AOP가 적용되게 하고, 특정 메소드의 트랜잭션 전파 속성만 PROPAGATION_NOT_SUPPORTED로 설정해서 트랜잭션 없이 동작하게 만듦.
트랜잭션을 시작하려고 할 때 getTransaction()이라는 메소드를 사용하는 이유 : 항상 트랜잭션을 새로 시작하는 것이 아니기 때문. 트랜잭션 전파 속성에 의해 정해진다.

- 격리수준
모든 db 트랜잭션은 격리수준을 적절하게 조정해서 가능한 한 많은 트랜잭션을 동시에 진행시키면서도 문제가 발생하지 않게 하는 제어가 필요하다. 
DefaultTransactionDefinition에 설정된 격리수준은 ISOLATION_DEFAULT (DataSource에 설정된 디폴트 격리수준을 그대로 따름)

- 제한시간
트랜잭션을 수행하는 제한시간 설정
DefaultTransactionDefinition의 기본 설정은 제한시간이 없음

- 읽기전용
트랜잭션 내에서 데이터를 조작하는 시도를 막라줌. 성능 향상

트랜잭션 정의를 수정하려면 TransactionDefinition 오브젝트를 DI 받아서 사용
-> TransactionAdvice를 사용하는 모든 트랜잭션의 속성이 한꺼번에 바뀐다는 문제 발생

### 6.6.2 트랜잭션 인터셉터와 트랜잭션 속성
메소드 이름에 따라 다른 트랜잭션 정의가 적용되도록 함
- TransactionInterceptor
트랜잭션 정의를 메소드 이름 패턴을 이용해서 다르게 지정할 수 있는 방법을 추가로 제공.
프로퍼티
	- PlatformTransactionManager 타입
	- Properties 타입의 transactionAttributes 프로퍼티 : 트랜잭션 속성을 정의 (기본 4가지 + rollBackOn())
	- rollBackOn() : 어떤 예외가 발생하면 롤백할 것인가를 결정
		- 기본 원칙과 다른 예외처리가 가능하게 해줌
		- 기본 원칙 : 런타임예외 -> 롤백, 체크예외 -> 커밋 (스프링은 비즈니스적인 의미가 있는 예외상황에서만 체크 예외 사용하기 때문)
- 메소드 이름 패턴을 이용한 트랜잭션 속성 지정
Properties 타입의 transactionAttributes 프로퍼티는 메소드 패턴과 트랜잭션 속성을 키와 값으로 갖는 컬렉션임
	- PROPAGATION_NAME : 트랜잭션 전파 방식, 필수
	- ISOLATION_NAME : 격리수준
	- readOnly : 읽기전용 항목
	- timeout_NNNN : 제한시간
	- -Exception1 : 체크 예외 중에서 롤백 대상으로 추가할 것을 넣음
	- +Exception2 : 런타임 예외지만 롤백시키지 않을 예외들을 넣음
```xml
<bean id="transactionAdvice" class="springbook.user.service.TransactionAdvice">
		<property name="transactionManager" ref="transactionManager" />
		<property name="transactionAttributes">
			<props>
				<prop key="get*">PROPAGATION_REQUIRED,readOnly,timeout_30</prop>
				<!-- readOnly와 timeout은 트랜잭션이 처음 시작될 때가 아니라면 적용되지 않음.-->
				<prop key="upgrade*">PROPAGATION_REQUIRED_NEW, ISOLATION_SERIALIZABLE</prop>
				<prop key="*">PROPAGATION_REQUIRED</prop>
			</props>
		</property>
	</bean>
```

- tx 네임스페이스를 이용한 설정 방법
```xml
<?xml version=*1.0" encoding="UTF-8"?>
<beans xmlns=*http://www.springframework.org/schema/beans"
	xmlns:xsi=“http://www.w3.org/2001/XMLSchema-instance”
	xmlns:aop= “http://www.springframework.org/schema/aop”
	xmlns:tx = “http://ww.springfranewock.org/schema/tx"> -> tx 네임스페이스 선언
	xsi:schemaLocation=“http://.springfranework.org/schema/beans
					http://www.springframework.org/schena/beans/spring-beans-3.0.xsd
					http://www.springframework.org/schema/aop
					http://www.springframework.org/schema/aop/spring-aop-3.0.xsd
tx 스키마 위치 지정	<-	http://www.springframework.org/schema/tx
					http://www.springframework.org/schema/tx/spring-tx-2.5.xsd”>

…

이 태그에 의해 Transacienhtercepkor 빈이 등록된다. 트랜잭션 매니저의 빈 아이디가 TransactorManager라면 생각 가능
＜tx:advice id="transactionAdvice" transaction-manager="transactionManager">
	<tx:attributes>
		<tx:method name="get*" propagation="REQUIRED" read-only="true"timeout=*30"/>
		<tx:method name="upgrade*" propagation="REQUIRES NEW" isolation="SERIALIZABLE" />
																🔼
		〈tx:method name="*" propagation="REQUIRED" /〉  Enumeration으로 스키마에 값이 정의되어 있으므로 오타가 있으면 XML 유효성 검사만으로 확인 가능
							🔼
		디폴트 값이 스키마에 정의되어 있으므로 RECURED라면 아예 생각도 가능하다.
	</tx:attributes>
</tx:advice>
```


트랜잭션 속성이 개별 애트리뷰트를 통해 지정될 수 있으므로 설정 내용을 읽기가 쉽고 편함

### 6.6.3 포인트컷과 트랜잭션 속성의 적용 전략

포인트컷 표현식과 트랜잭션 속성을 정의할 때 따르면 좋은 전략

- 트랜잭션 포인트컷 표현식은 타입 패턴이나 빈 이름을 이용한다
트랜잭션용 포인트컷 표현식에 메소드나 파라미터, 예외에 대한 패턴을 정의하는게 아니라, 트랜잭션의 경계로 삼을 클래스들이 선정됐다면, 그 클래 스들이 모여 있는 패키지를 통째로 선택하거나 클래스 이름에서 일정한 패턴을 찾아서 표현식으로 만든다.
가능하다면 클래스보단 변경 빈도가 적은 인터페이스 타입을 기준으로 적용.
bean() 표현식을 사용하는 것도 좋음

- 공통된 메소드 이름 규칙을 통해 최소한의 트랜잭션 어드바이스와 속성을 정의한다
기준이 되는 몇 가지 트랜잭션 속성을 정의하고 그에 따라 적절한 메소드 명명 규칙을 만들어 두면 하나의 어드바이스만으로 애플리케이션의 모든 서비스 빈에 트랜젝션 속성을 지정할 수 있다.
	- 트랜잭션 속성의 적용 패턴이 일반적인 경우와 크게 다른 오브젝트
	: 모든 메소드에 대해 디폴트 속성(*)을 지정
	
- 프록시 방식 AOP는 같은 타깃 오브젝트 내의 메소드를 호출할 때는 적용되지 않는다
타깃 오브젝트 내로 들어와서 타깃 오브젝트의 다른 메소드를 호출하는 경우에는 프록시를 거치지 않고 직접 타깃의 메소드가 호출된다. 
만약 update() 메소드에 대해 트랜잭션 전파 속성을 REQUIRES_NEW라고 해놨더라도 같은 타깃 오브젝트에 있는 delete() 메소드를 통해 update()가 호출되면 트랜잭 션 전파 속성이 적용되지 않으므로 REQUIRES_NEW는 무시되고 프록시의 delete() 메소 드에서 시작한 트랜잭션에 단순하게 참여하게 될 뿐이다.

### 6.6.4 트랜잭션 속성 적용

- 트랜잭션 경계설정의 일원화
DAO가 제공하는 주요 기능은 서비스 계층에 위임 메소드를 만들어둬서 가능하면 다른 모듈의 DAO에 접근할 때는 서비스 계층을 거치도록 하는 게 바람직하다. 그래야만 UseSerice의 add()처럼 부가 로직을 적용할 수도 있고, 트랜잭션 속성도 제어할 수 있기 때문이다. 
```java
public interface UserService { 
	// DAO 메소드와 1:1 대응되는 crud메소드이지만 add()처럼 단순 위임 이상의 로직을 가질 수 있다.
	void add (User user);
	
	// 신규 추가 메소드 (getCount() 제외)
	User get(String id):
	List<User> getAll():
	void deleteAll();
	void update(User user);
	
	void upgradeLevels();
}
```

UserServiceImpl 클래스에 추가된 메소드 구현 코드를 넣음
```java
public void deleteAll() { userDao.deleteAll(); }
public User get(String id) { return userDao.get(id); }
public List<User> getAIl() { return userDao.getAll(); }
public void update(User user) { userDao.update(user); }
```
- 서비스 빈에 적용되는 포인트컷 표현식 등록
upgrade() 에만 트랜잭션이 적용되게 했던 기존 포인트컷 표현식을 모든 비즈니스 로직의 서비스 빈에 적용되도록 수정
```xml
<aop:config>
	<aop:advisor advice-ref="transactionAdvice" pointcut="bean (*Service) />
</aop:config>
```

- 트랜잭션 속성을 가진 트랜잭션 어드바이스 등록
TransactionAdvice 클래스로 정의했던 어드바이스 빈을 스프링의 TransactionInterceptor를 이용하도록 변경
get으로 시작 -> 읽기전용
나머지 -> 디폴트
```xml
<?xml version="1.0" encoding=“UTF-8”?>
<beans xmIns="http://www.springframework.org/schema/beans"
	xmlns:xsi=http://www.w3.org/2001/XMLSchena-instance"
	xmlns:aop-http://www.springframework.org/schema/aop"
	xmlns:tx="http://www.springframework.org/schema/tx*
	xsi:schemaLocation= http://www.springframework.org/schema/beans
						http://www.springframework.org/schema/beans/spring-beans-3.0.xsd
						http://www.springfranework.org/schema/aop
						http://www.springfranework.org/schena/aop/spring-aop=3.0.xsd
						http://www.springfranework.org/schema/tx
						http://www.springfranework.org/schema/tx/spring-tx-3.0.xsd*>
<tx:advice id="transactionAdvice">
	<tx:attributes>
		<tx:method name="get*" read-only='true"/>
		<tx:method name="*" />
	</tx:attributes>
</tx:advice>
```

- 트랜잭션 속성 테스트
TestUserService에서 읽기전용 트랜색션의 대상인 get으로 시작하는 메소드를 오버라이드한 후에 강제로 쓰기 시도를 한다. 여기서 읽기전용 속성으로 인한 예외가 발생해야 한다.

	- UserServiceTest에 조작된 getAll()을 호출하는 테스트를 만든다.

		TestUserService를 testUserService 빈으로 등록해뒀으니 DI 받은 testUserService 변수를 사용해서 getAll() 메소드를 호출하면 된다

		일단은 어떤 예외가 던져질지 모르기 때문에 expected 없이 테스트를 작성
		->TransientDataAccessResourCeEXception이라는 예외 발생
-> getAlI()의 userDao.update()에 의해 일어나는 DB 쓰기 작업은 원래 정상적으로 처리돼야 함에도 일시적인 제약조건 때문에 예외를 발생시켰다는 뜻.

## 6.7 애노테이션 트랜잭션 속성과 포인트컷
세밀한 트랜잭션 속성의 제어가 필요한 경우를 위해 직접 타깃에 트랜잭션 속성 정보를 가진 애노테이션을 지정하는 방법

### 6.7.1 트랜잭션 애노테이션
- @Transactional
```java
package org.springframework.transaction.annotation;

@Target({ElementType.METHOD, ElementType.TYPE}) // 애노테이션을 사용할 대상을 지정한다. 여기에 사용된 '메소드와 타입(클래스, 인터페이스)'처럼 한 개 이상의 대상을 지정할 수 있다.
@Retention(RetentionPolicy.RUNTIME)	// 애노테이션 정보가 언제까지 유지되는지를 지정한다. 이렇게 설정하면 런타임 때도 애노테이션 정보를 리플렉션을 통해 얻을 수 있다.
@Inherited	// 상속을 통해서도 애노테이션 정보를 얻을 수 있게 한다.
@Documented
// 트랜잭션 속성의 모든 항목을 엘리먼트로 지정할 수 있다. 디폴트 값이 설정되어 있으므로 모두 생략 가능하다.
public @interface Transactional {
    String value() default "";
    Propagation propagation() default Propagation.REQUIRED;
    Isolation isolation() default Isolation.DEFAULT;
    int timeout() default TransactionDefinition.TIMEOUT_DEFAULT;
    boolean readOnly() default false;
    Class<? extends Throwable>[] rollbackFor() default {};
    String[] rollbackForClassName() default {};
    Class<? extends Throwable>[] noRollbackFor() default {};
    String[] noRollbackForClassName() default {};
}
```

스프링은 @Transactional이 부여된 모든 오브젝트를 자동으로 타깃 오브젝트로 인식한다. TransactionAttributeSourcePointcut이 사용되는데, 이것은 @Transactional이 타입 레벨이든 메소드 레벨이든 상관없이 부여된 빈 오브젝트를 모두 찾아서 포인트컷의 선정 결과로 돌려준다.

- 트랜잭션 속성을 이용하는 포인트컷
@Transactional은 메소드마다 다르게 설정할 수도 있으므로 매우 유연하고 세분화된 트랜잭션 설정이 가능하다. 동시에 포인트컷도 @Transactional을 통한 트랜잭션 속성정보를 참조하도록 만든다.

- 대체 정책
1. 타깃의 메소드에 @Transactional이 있는지 확인
2. 있으면 이를 속성으로 사용
	없으면 타깃 클래스에 부여된 @Transactional를 찾음
3. 있으면 트랜잭션 속성으로 사용
4. 이런 식으로 메소드가 선언된 타입까지 단계적으로 확인해서 없으면 해당 메소드는 트랜잭션 적용 대상이 아니라고 판단

```java
[1]
public interface Service {
	[2]
    void method1();
	[3]
    void method2();
}
[4]
public class ServiceImpl implements Service {
    [5]
    public void method1() {
    }
	[6]
    public void method2() {
    }
}
```
순서 : [5], [6] -> [4] -> [2], [3] -> [1] (이 순서대로 확인하여 애노테이션 적용)

메소드에 부여된 @Transactional이 가장 우선이기 때문에 @Transactional이 붙은 메소드는 클래스 레벨의 속성을 무시하고 메소드 레벨의 속성을 사용할 것이다.

코드를 짤 땐, 먼저 타입 레벨에 정의되고 공통 속성을 따르지 않는 메소드에 대해서만 메소드 레벨에 다시 @Transactional을 부여해주는 식으로 사용해야 한다.

- 트랜잭션 애노테이션 사용을 위한 설정
``<tx:annotaion-driven /> `` 사용

### 6.7.2 트랜잭션 애노테이션 적용
클래스, 빈, 메소드의 이름에 일관된 패턴을 만들어 적용하고 이를 활용해 포인트컷과 트랜잭션 속성을 지정하는 것보다는 단순하게 트랜잭션이 필요한 타입 또는 메소드에 직접 애노테이션을 부여하는 것이 훨씬 편리하고 코드를 이해하기도 좋음. 다만, 빼먹지 않도록 주의.

앞에서 <tx:attributes> 태그를 이용해 설정했던 트랜잭션 속성을 그대로 애노테이션으로 바꿔보기
```java
@Transactional // 디폴트 속성. 인터페이스 방식의 프록시를 사용하는 경우에는 인터페이스에 @Transactional 적용 가능.
public interface UserService {
	// 메소드 레벨 @Transactional 애노테이션이 없으므로 대체 정책에 따라 타입 레벨에 부여된 디폴트 속성이 적용됨
    void add(User user);
    void deleteAll();
    void update(User user);
    void upgradeLevels();

    @Transactional(readOnly = true)	// get에 대한 읽기 전용 속성. 우선적으로 적용됨.
    User get(String id);

    @Transactional(readOnly = true) // 같은 속성을 가졌어도 메소드 레벨에 부여될 때는 메소드마다 반복되야함.
    List<User> getAll();
}
```

## 6.8 트랜잭션 지원 테스트
### 6.8.1 선언적 트랜잭션과 트랜잭션 전파 속성
트랜잭션 전파라는 기법을 사용했기 때문에 UserService의 add()는 독자적인 트랜잭션 단위가 될 수도 있고, 다른 트랜잭션의 일부로 참여할 수도 있다.

- ex) UserDao의 add() 메소드. 트랜잭션 전파 방식 : REQUIRED
processDailyEventRegistration() 메소드를 구현하고, 이 메소드를 비즈니스 트랜잭션의 경계로 설정
	
	이 안에서 UserService의 add() 메소드를 이용해 사용자 등록 중 처리해야 할 로직을 적용해야 함.
	
	이 때 add() 메소드는 독자적인 트랜잭션을 시작하는 대신 processDailyEventRegistration() 메소드에서 시작된 트랜잭션의 일부로 참여하게 됨.
-> 불필요한 코드 중복 없음

> 선언적 트랜잭션 : AOP를 이용해 코드 외부에서 트랜잭션의 기능을 부여해주고 속성을 지정할 수 있게 하는 방법
> 프로그램에 의한 트랜잭션 : TransactionTemplate이나 개별 데이터 기술의 트랜잭션 API를 사용해 직접 코드 안에서 사용하는 방법

💡 선언적 트랜잭션과 프로그램에 의한 트랜잭션 자세히 살펴보기
1. **선언적 트랜잭션**

-   트랜잭션 관리가 코드 외부에서 이루어짐
-   AOP를 활용하여 메서드의 실행에 따라 트랜잭션이 자동으로 시작되고 종료됩니다.
-   Spring에서 `@Transactional`을 사용하여 구현됩니다.

예시 코드 (선언적 트랜잭션):
```java
import org.springframework.transaction.annotation.Transactional;

public class UserServiceImpl implements UserService {

    private final UserRepository userRepository;

    public UserServiceImpl(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Transactional
    public void addUser(User user) {
        // 트랜잭션 시작
        userRepository.save(user); // 데이터 저장
        // 트랜잭션 자동 종료 (성공 시 커밋, 실패 시 롤백)
    }

    @Transactional(readOnly = true)
    public User getUser(String id) {
        return userRepository.findById(id); // 데이터 조회만 수행 (읽기 전용)
    }
}
```
-   `@Transactional`을 사용하면 메서드 실행 전에 **트랜잭션 시작**, 실행 후 **커밋/롤백**을 처리합니다.
-   개발자는 트랜잭션 관리 코드를 작성할 필요 없이 비즈니스 로직에만 집중할 수 있습니다.

### 2. **프로그램에 의한 트랜잭션**

-   개발자가 직접 코드로 트랜잭션의 시작, 커밋, 롤백을 관리합니다.
-   `TransactionTemplate` 또는 데이터베이스 API를 사용하여 구현됩니다.

예시 코드 (프로그램에 의한 트랜잭션):

```java
import org.springframework.transaction.PlatformTransactionManager;
import org.springframework.transaction.TransactionStatus;
import org.springframework.transaction.support.DefaultTransactionDefinition;

public class UserServiceImpl implements UserService {

    private final UserRepository userRepository;
    private final PlatformTransactionManager transactionManager;

    public UserServiceImpl(UserRepository userRepository, PlatformTransactionManager transactionManager) {
        this.userRepository = userRepository;
        this.transactionManager = transactionManager;
    }

    public void addUser(User user) {
        // 트랜잭션 시작
        TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());
        try {
            userRepository.save(user); // 데이터 저장
            transactionManager.commit(status); // 트랜잭션 커밋
        } catch (Exception e) {
            transactionManager.rollback(status); // 트랜잭션 롤백
            throw e; // 예외 다시 던지기
        }
    }

    public User getUser(String id) {
        return userRepository.findById(id); // 읽기 전용 트랜잭션이 따로 필요하지 않음
    }
}
```
-   `PlatformTransactionManager`를 사용하여 직접 트랜잭션을 시작(`getTransaction`), 커밋(`commit`), 롤백(`rollback`)합니다.
-   트랜잭션 관리 로직이 직접 코드에 작성되어야 하므로 **복잡성**이 증가할 수 있습니다.

### 차이점

| **특징**             | **선언적 트랜잭션**                          | **프로그램에 의한 트랜잭션**               |
|----------------------|---------------------------------------------|-------------------------------------------|
| **트랜잭션 제어 위치** | AOP 및 `@Transactional`으로 코드 외부에서 관리 | 개발자가 코드 내부에서 직접 관리           |
| **코드 복잡성**      | 간단하고 깔끔                                | 복잡도가 높아짐                            |
| **적용 대상**        | Spring 프레임워크 기반 애플리케이션에 적합    | 특별한 트랜잭션 관리 로직이 필요한 경우 사용 |


### 6.8.2 트랜잭션 동기화와 테스트
트랜잭션의 자유로운 전파가 가능했던 이유
	- AOP 
	- 스프링의 트랜잭션 추상화 
- 트랜잭션 매니저와 트랜잭션 동기화
트랜잭션 동기화 기술이 있어서 시작된 트랜잭션의 정보를 저장소에 보관해뒀다가 DAO에서 공유할 수 있고, 진행 중인 트랜잭션이 있는지 확인하고, 트랜잭션 전파 속성에 따라서 이에 참여할 수 있도록 만들어줌

트랜잭션 매니저를 이용해 트랜잭션에 참여하거나 트랜잭션을 제어하는 방법
: @Autowired로 트랜잭션 메니저 빈을 가져와 테스트
```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations = "/test-applicationContext, xml")
public class UserServiceTest {
	@Autowired
	PlatformTransactionManager transactionManager;
…	
	@Test
	public void transactionSync() {
		userService.deleteAll();

		userService.add (users.get(0));
		userService.add(users.get (1));
	}
}
```
3개의 트랜잭션 실행됨

- 트랜잭션 매니저를 이용한 테스트용 트랜잭션 제어
3개의 메소드를 하나의 트랜잭션 안에서 실행하게 하고 싶음
-> 테스트 메소드에서 UserService의 메소드를 호출하기 전에 트랜젝 션을 미리 시작해주면 됨
테스트에서 트랜잭션 매니저를 이용해 트랜잭션을 시작시키고 이를 동기화해줌

```java
@Test
public void transactionSync() {
	DefaultTransactionDefinition txDefinition = new DefaultTransactionDefinition);
	TransactionStatus txStatus = transactionManager.getTransaction(txDefinition); // 트랜잭션 매니저에게 트랜잭션을 요청한다. 기존에 시작된 트랜잭션이 없으니 새로운 트랜잭션을 시작시키고 트랜잭션 정보를 돌려줌. 동시에 만들어진 트랜잭션을 다른 곳에서도 사용할 수 있도록 동기화한다.
	
	userService.deleteAll();

	userService.add (users.get(0));
	userService.add(users.get (1));
	
	transactionManager.commit(txStatus); // 앞에서 시작한 트랜잭션을 커밋한다.
}
```

- 트랜잭션 동기화 검증
	- 트랜잭션 속성 중에서 읽기 전용과 제한시간 등은 처음 트랜잭션이 시작할 때만 적 용되고 그 이후에 참여하는 메소드의 속성은 무시된다.
	- 트랜잭션 속성을 강제로 읽기전용으로 바꾸고 테스트 실행. 
	- 만약 deleteAll()이 테스트 코드에서 시작된 트랜잭션에 참여한다면 읽기전용 속성을 위반했으니 예외가 발생

이런 동기화는 선언적 트랜잭션이 적용된 서비스 메소드에만 적용되는 것이 아니다. JdbcTemplate과 같이 스프링이 제공하는 데이터 액세스 추상화를 적용한 DA0에도 동일한 영향을 미친다.

- 롤백 테스트
롤백 테스트는 테스트 내의 모든 DB 작업을 하나의 트랜잭션 안에서 동작하게 하고 테스트가 끝나면 무조건 롤백해버리는 테스트를 말한다.
```xml
@Test
public void transactionSync() {
	DefaultTransactionDefinition tDefinition = new DefaultTransactionDefinition();
	TransactionStatus txStatus = transactionManager.getTransaction(txDefinition);

	try { 
	userService.deleteAll(); //테스트 안의 모든 작업을 하나의 트랜잭션으로 통합한다.
	userService.add(users.get (0));
	userService.add(users.get(1));
	}
	finally {
	transactionManager.rollback(txStatus); // 테스트 결과가 어떻든 상관없이 테스트가 끝나면 무조건 롤백한다. 테스트 중에 발생했던 DB의 변경 사항은 모두 이전 상태로 복구된다.
	}
}
```
롤백 테스트의 이점
- 테스트용 데이터를 DB에 잘 준비해놓더라도 앞에서 실행된 테스트 에서 DB의 데이터를 바꿔버리면 이후에 실행되는 테스트에 영향을 미칠 수 있는데 이를 해결해줌
- 전체 테스트를 수행하기 전에 여러 테스트에서 공통적 으로필요한 사용자 정보를 테스트 데이터로 DB에 넣어뒀다면, 롤백 테스트 덕분에 매테스트마다 처음과 동일한 User 테이블의 테스트 데이터로 테스트를 수행할 수 있다.
- 성능이 더 향상되기도 함.

### 6.8.3 테스트를 위한 트랜잭션 애노테이션
스프링 컨텍스트 테스트에서 쓸 수 있는 유용한 애노테이션
- @Transactional
앞에서 봤던 것과 동일. 테스트 메소드에 트랜잭션 경계가 자동으로 설정돼서 테스트 내에서 진행하는 모든 트랜잭션 관련 작업을 하나로 묶어줄 수 있다.
AOP를 위한 것은 아님.
```xml
@Test
@Transactional
public void transactionSync() {
	userService.deleteAll();
	userservice.add(users.get(0));
	userservice.add(users.get(1));
}
```
테스트 메소드 실행 전에 새로운 트랜잭션을 만들어주고 메소드가 종료되면 트랜잭션을 종료해준다. 메소드 안에서 실행되는 deleteAll(), add() 등은 테스트 메소드의 트랜잭션에 참여해서 하나의 트랜잭션으로 실행됨.

테스트 클래스 레벨에 부여할 수도 있음. 대체 정책 같음

- @Rollback
테스트용 트랜잭션은 테스트가 끝나면 자동으로 롤백된다.
트랜잭션을 커밋시켜서 테스트에서 진행한 작업을 그대로 DB에 반영하고 싶다면 @Rollback(false) 이용

- @TransactionConfiguration
@Rollback 애노테이션은 메소드 레벨에만 적용할 수 있다.
테스트 클래스의 모든 메소드에 트랜잭션을 적용하면서 모든 트랜잭션 이 롤백되지 않고 커밋되게 하려면 @TransactionConfiguration 애노테이션을 이용

``@TransactionConfiguration(defaultRollback=false)``

- NotTransactional과 Propagation.NEVER
	- NotTransactional을 테스트 메소드에 부여하면 클래스 레벨의 OTransactional 설정을 무시하고 트랜잭션을 시작하지 않은 채로 테스트를 진행
	- 스프링 3.0에서 제거 대상이 됐기 때문에 사용하지 않고 트랜잭션 테스트와 비 트랜잭션 테스트를 아예 클래스를 구분해서 만들도록 권장한다.
	- @Transactional을 다음과 같이 NEVER 전파 속성으로 지정해주면 (NotTransactional과 마찬가지로 트랜잭션이 시작되지 않는다.

- 효과적인 DB 테스트
1. 의존, 협력 오브젝트를 사용하지 않고 고립된 상태에서 테스트를 진행하는 단위 테스트와, DB 같은 외부의 리소스나 여러 계층의 클래스가 참여하는 통합 테스트는 아예 클래스를 구분해서 따로 만듦
2. 클래스 레벨에 @Transactional 사용
3. 통합테스트는 롤백 테스트로 만듦
4. 테스트는 어떤 경우에도 서로 의존하면 안됨



