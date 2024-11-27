# 8주차 - AOP3

노션 링크 : [8주차 - AOP3](https://www.notion.so/8-AOP3-14b6873728d0801286f3f5e64084ba32?pvs=21) 

## 트랜잭션 정의 : TransactionDefinition

TransacitonDefinition 인터페이스는 트랜잭션 동작방식에 영향을 줄 수 있는 네 가지 속성을 정의한다

1. **트랜잭션 전파** : 이미 진행 중인 트랜잭션이 있는 / 없는 경우 동작방식을 결정하는 방식
    - **PROPAGATION_REQUIRED**
        - 진행 중인 트랜잭션이 **있으면 → 참여**
            - 중간에 예외가 발생하면 참여 중인 코드가 전부 롤백
        - 진행 중인 트랜잭션이 **없으면 → 새로 시작**
        - **Default**TransactionDefinition의 전파속성
    - **PROPAGATION_REQUIRES_NEW**
        - **항상 새로운 트랜잭션을 시작**
        - **독립적인 트랜잭션**이 보장되어야 하는 경우에 사용
    - **PROPAGATION_NOT_SUPPORTED**
        - **트랜잭션 없이 동작**
        - 특정한 메소드만 트랜잭션 적용에서 제외해야 하는 경우에 사용 : 모든 메소드에 AOP가 적용되게 설정 + 트랜잭션이 없어야 하는 부분에만 전파속성을 다르게 설정하는 방식
2. **제한시간 :** 트랜잭션 수행 제한시간 설정
    - **Default**TransactionDefinition : **제한시간 없음**
    - 트랜잭션을 직접 시작할 수 있는 경우(PROPAGATION_REQUIRED,  PROPAGATION_REQUIRES_NEW)에만 의미가 있다
3. **읽기전용** : 트랜잭션 내에서 데이터를 조작하는 시도를 막을 수 있다
    - 데이터 액세스 기술에 따라 성능향상이 있을 수 있다
4. **격리수준 :** 성능 향상을 위해 여러 개의 트랜잭션이 동시에 실행되게 하면서도 적절한 격리수준을 조정하여 문제가 발생하지 않게 제어하는 것이 필요하다
    - **Default**TransactionDefinition의 격리수준 = **ISOLATION_DEFAULT** : DB의 디폴트를 따른다

---

## TransactionInterceptor

트랜잭션 경계설정 어드바이스로 사용할 수 있으며, **트랜잭션 정의를 메소드 이름 패턴을 이용해 다르게 정의할 수 있다**

인터셉터는 두 가지 프로퍼티를 갖는다

- PlatformTransacitonManager
- **Properties - transactionAttributes**
    - **트랜잭션 네 가지 기본항목(전파, 격리, 제한시간, 읽기전용) + rollbackOn 메소드 (롤백 발생시킬 예외 정의)**

- 스프링이 제공하는 TransacitonInterceptor의 예외처리 방식
    1. **런타임**예외 → 트랜잭션 **롤백** / **체크예외** → **커밋** (예외사항이 아닌 비즈니스 로직의 한 부분으로 해석)
    2. rollbackOn()에 직접 원하는 방식으로 정의

### 메소드이름패턴 이용한 트랜잭션 속성 지정

**transactionAttributes : 키-값의 컬렉션 : 메소드패턴 - 트랜잭션속성**

<aside>
💡

**PROPAGATION_NAME, ISOLATION_NAME, READ_ONLY, TIMEOUT_NNNN, -Exception1, +Exception2**

- **필수항목 : 전파속성**
- 생략 시 DefaultTransactionDefinition의 디폴트 속성이 부여된다
- -Exception : 체크예외지만 롤백 대상으로 추가할 항목
- +Exception : 런타임예외지만 롤백 대상에서 삭제할 항목
</aside>

---

### 포인트것 - 트랜잭션속성 적용 전략

1. **트랜잭션 포인트컷 표현식은 타입패턴, 빈 이름을 사용한다**
    - 타깃 클래스의 **메소드는 전부 트랜잭션 적용 후보**로 사용하자
    - 비즈니스 로직의 클래스라면 메소드 단위로 세밀하게 포인트컷을 정의해 주는 것 보다는
    - **트랜잭션 경계로 삼을 패키지를 통째로 선택하거나, 클래스의 패턴을 표현식으로 설정**하자
        - 클래스보다는 인터페이스 타입을 기준으로 타입 패턴을 적용하자
2. **공통된 메소드 이름 규칙을 통해 최소한의 트랜잭션 어드바이스 속성을 정의한다**
    - 너무 많은 세부적인 트랜잭션을 정의하는 것보다는
    - **기준이 되는 몇 가지 트랜잭션 속성을 정의하고 + 적절한 메소드 명명규칙을 사용해서 : 하나의 어드바이스로 대부분의 서비스 빈 트랜잭션 속성을 지정**하자
        - 예외적인 오브젝트의 경우에만 어드바이스, 포인트컷을 새롭게 추가하자
3. **프록시 방식의 AOP는 같은 타깃 오브젝트 내의 메소드를 호출할 때는 적용되지 않는다**
    - 타깃 오브젝트 내부에서의 메소드 호출은 프록시를 거치지 않기 때문에 부가기능이 적용되지 않는다
        - 새로운 트래잭션 속성을 부여하지 못한다
    - 프록시가 아닌 AspectJ와 같은 타깃의 바이트코드를 직접 조작하는 방식의 AOP 기술을 적용하자
4. **트랜잭션 경계설정의 일원화**
    - **특정 계층의 경계 - 트랜잭션 경계를 일치**시키는 것이 바람직하다
        - **서비스 계층**의 오브젝트가 트랜잭션 경계를 부여하기 가장 적절하다
    - 서브시 계층을 경계로 설정했다면
        - 다른 계층, 모듈에서 DAO에 직접 접근하는 것을 차단해야 한다
        - DAO가 제공하는 **주요 기능은 서비스 계층에 위임 메소드**를 만들자
        - **DAO에 접근할 때는 서비스 계층을 거치도록** 하는 것이 바람직하다

- UserService에 직접 적용

```java
public interface UserService {
    void upgradeLevels(); 
    User get(String id);
    List<User> getAll();
    void deleteAll(); 
    void add(User user);
    void update(User user);
}

// ...
// 구현체
// DAO로 위임하도록 만든다

@Override
public User get(String id) { return userDao.get(id); }
@Override
public List<User> getAll() { return userDao.getAll(); }

@Override
public void deleteAll() { userDao.deleteAll(); }

@Override
public void update(User user) { userDao.update(user); }
```

```java
// 서비스 빈에 적용되는 포인트컷 표현식 등록
<aop:config>
  <aop:advisor advice-ref="transactionAdvice" pointcut="bean(*Service)" />
</aop:config>
```

---

## 애노테이션 트랜잭션 속성과 포인트컷 : @Transactional

- 직접 타깃에 트랜잭션 속성정보를 가진 애노테이션을 지정하여 더 세밀하게 트랜잭션 속성을 부여할 수 있다

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Transactional {
	@AliasFor("transactionManager")
	String value() default "";
	@AliasFor("value")
	String transactionManager() default "";
	String[] label() default {};
	Propagation propagation() default Propagation.REQUIRED;
	Isolation isolation() default Isolation.DEFAULT;
	int timeout() default TransactionDefinition.TIMEOUT_DEFAULT;
	String timeoutString() default "";
	boolean readOnly() default false;
	Class<? extends Throwable>[] rollbackFor() default {};
	String[] rollbackForClassName() default {};
	Class<? extends Throwable>[] noRollbackFor() default {};
	String[] noRollbackForClassName() default {};

}
```

- @Target({ElementType.TYPE, ElementType.METHOD}) : **애노테이션 사용 대상** 지정
    - **메소드, 타입(클래스, 인터페이스)**에 지정 가능
- @Retention(RetentionPolicy.RUNTIME) : 애노테이션 정보 유지 지정
    - 런타임에도 애노테이션 정보를 얻을 수 있다
- value, propagation, isolation, timeout~ : 트랜잭션 속성의 모든 항목을 엘리먼트로 지정할 수 있다
- 메소드, 클래스, 인터페이스에 사용할 수 있으며, 애노테이션을 트랜잭션 속성정보로 사용하면 스프링은 **@Transactional이 부여된 모든 오브젝트를 자동으로 타깃 오브젝트로 인식**한다
→ 사용 포인트컷 : TransactionAttributeSourcePointcut

### 트랜잭션 속성을 이요하는 포인트컷

애노테이션을 사용했을 때의 어드바이저의 동작방식

- **어드바이스** : @Transactional 애노테이션의 엘리멘트에서 트랜잭션 속성을 가져오는 AnnotationTransactionAttributeSource를 사용
    - 즉 애노테이션에 담겨있는 트랜잭션 속성정보를 사용한다
- **포인트컷** : TransactionAttributeSourcePointcut : 속성이 부여된 대상을 확인한다

**⇒ 트랜잭션 부가기능 적용 단위 : 메소드**

- 메소드마다 애노테이션을 부여하여 속성을 지정하면
    - 세분화된 유연한 속성제어가 가능하지만
    - 코드는 지저분해지고, 동일한 속성정보를 가진 애노테이션을 반복적으로 메소드마다 부여하게 된다

### 대체 정책

- @Transactional 애노테이션의 4단계 대체정책
- 메소드 적용 속성 순서 : **타깃메소드 → 타깃클래스 → 선언메소드 → 선언타입**(클래스, 인터페이스)
    - 순서대로 애노테이션 적용 여부를 확인하고, **가장 먼저 발견되는 속성정보를 사용**한다
    - 반대로 공통으로 **적용되는 범위가 가장 큰 순서로는 : 선언 > 선언메소드 > 타깃클래스 > 타깃클래스**
    - 즉 큰 범위에 애노테이션을 부여하면, 그 하위에는 해당 속성이 공통적으로 적용된다
- 따라서 **더 큰 범위에 @Transactional을 부여하고, 공통 속성을 따르지 않는 특정 메소드에만 추가로 트랜잭션을 부여**하여 중복을 방지하자
⇒ **타깃 클래스에 @Transactional을 두고, 공통 속성을 따르지 않는 메소드에 대해서만 메소드 레벨에 다시 @Transactional을 붙이는 방법을 권장**한다
- 프록시 방식의 AOP가 아닌 방식으로 트랜잭션을 적용하면, 인터페이스에 둔 @Transactional은 무시되기 때문에 클래스에 두는 것을 권장한다
    - 인터페이스 레벨에 두면 구현 클래스가 바뀌어도 속성을 유지할 수 있다는 장점이 있다

- 적용 코드

```java
@Transactional
public interface UserService {
		// 메소드 레벨에 부여된 애노테이션이 없으므로
		// 타입에 부여된 애노테이션의 속성이 적용된다
    void upgradeLevels();
    void add(User user);
    void deleteAll();
    void update(User user);
    
    // 메소드 레벨에서 설정된 속성을 따른다
    @Transactional(readOnly = true)
    User get(String id);
    @Transactional(readOnly = true)
    List<User> getAll();
}
```

---

## 선언적 트랜잭션과 트랜잭션 전파 속성

- 프로그램에 의한 트랜잭션 : 개별 데이터 기술의 트랜잭션 API를 사용해 직접 코드 안에서 사용하는 방법
- **선언적 트랜잭션** : AOP를 이용해 코드 외부에서 트랜잭션의 기능을 부여해주고 속성을 지정할 수 있게 하는 방법
- 선언적 트랜잭션의 장점
    - **독자적으로 트랜잭션 단위**가 될 수도 있고, **다른 트랜잭션의 일부**로 참여할 수도 있다
        - 즉 **스스로 트랜잭션 경계를 설정**할 수도 있으며, 다른 메소드에서 만들어진 **트랜잭션의 경계 안에 포함**될 수도 있다
    - 따라서 다른 메소드에서 **호출되어 사용**될 수 있으며
    - 다른 비즈니스 트랜잭션에서 사용되더라도 **불필요한 코드중복이 발생하지 않는다**

---

## 트랜잭션 동기화와 테스트

트랜잭션 전파의 기술적 기반 : **AOP + 트랜잭션 추상화**

→ 트랜잭션 추상화의 기술적 기반 : **트랜잭션 매니저 + 트랜잭션 동기화**

- 구체적인 트랜잭션 기술과 상관없이 일관된 트랜잭션 제어가 가능해진다
- 트랜잭션 정보를 보관했다가 DAO에서 공유하여 사용할 수 있게 된다

### **< 프로그램에 의한 트랜잭션 방식**을 이용한 트랜잭션 동기화 >

모두 같은 트랜잭션 안에서 돌아가게 만드는 예시

```java
@Test(expected = UncategorizedSQLException.class)
public void transactionSync() {
    DefaultTransactionDefinition transactionDefinition = new DefaultTransactionDefinition();
    transactionDefinition.setReadOnly(true);
    TransactionStatus txStatus = transactionManager.getTransaction(transactionDefinition);

		// 앞에서 만들어진 트랜잭션에 참여한다
		
		// 트랜잭션의 readOnly를 위반했으므로 에러 발생
    userService.deleteAll();

    userService.add(users.get(0));
    userService.add(users.get(1));

    transactionManager.commit(txStatus);
}
```

- 모두 같은 트랜잭션으로 통합하는 방법 : **메소드가 호출되기 전에 트랜잭션이 시작**되게 한다
    - 전부 REQUIRED이므로, 이미 시작된 트랜잭션이 있다면 통합되기 때문
- transactionManager.**getTransaction**(transactionDefinition)
    - **새로운 트랜잭션 시작**
- **읽기전용, 제한시간은 처음 시작된 트랜잭션의 속성으로 결정**된다
    - **이후에 참여한 트랜잭션의 속성은 무시된다**
    
    → transactionDefinition.setReadOnly(true); 이므로 **userService.deleteAll(); 또한 읽기전용 트랜잭션이 적용된 상태에서 진행**된다
    
    → 따라서 deleteAll()에서 에러가 발생한다 : 메소드들이 같은 트랜잭션에서 동작하고 있다는 테스트 검증 완료
    

### **< 롤백 테스트 >**

- 테스트 내의 모든 DB 작업을 **하나의 트랜잭션 안에서 동작하게 하고, 테스트가 끝나면 무조건 롤백**
- 복잡한 데이터를 사용하는 테스트에서는 DB 데이터, 상태가 매우 중요하다
    - 따라서 **테스트의 순서는 보장할 수 없으며, 앞서의 테스트에 의해 준비한 테스트 데이터가 바뀌는 상황**이라면 하나의 테스트에서 **조작한 데이터를 시작하기 전 상태로 만들고 마무리 할 수 있는 롤백 테스트가 매우 유용**하다
- 또한 여러 개발자가 하나의 공용 테스트 DB를 사용할 수도 있다

```java
@Test
public void transactionSync() {
    DefaultTransactionDefinition transactionDefinition = new DefaultTransactionDefinition();
    TransactionStatus txStatus = transactionManager.getTransaction(transactionDefinition);

		try {
			userService.add(users.get(0));
	    userService.add(users.get(1));
	    Assertions.assertEquals(2, userDao.getCount());
		} finally {
			transactionManager.rollback(txStatus);
		}
    
}
```

### < 트랜잭션 애노테이션 : 테스트용 >

**@Transactional**

- 테스트 코드에도 @Transactional 적용으로 트랜잭션 경계를 자동으로 설정할 수 있다
- 클래스, 메소드 레벨에 애노테이션을 부여할 수 있다
- **자동으로 롤백**된다
- 롤백에 관련된 설정을 담을 수 없다

**@Rollback**

- Transactional처럼 **트랜잭션 경계설정이 가능하면서 + 롤백 관련 설정**을 담을 수 있다
- 롤백 미적용 : @Rollback(false)
- **메소드 레벨**에만 적용할 수 있다

**@TransactionConfiguration**

- **클래스 레벨에 트랜잭션과 관련된 설정을 부여**할 수 있다
- 따라서 테스트 클래스의 모든 메소드에 트랜잭션을 적용하면서 + 롤백되지 않게 하려면 
클래스 레벨에 @TransactionConfiguration(defaultRollback=false) + 메소드 레벨에 @Rollback