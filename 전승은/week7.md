## 6.4 스프링의 프록시 팩토리 빈
### 6.4.1 ProxyFactoryBean
- 스프링은 일관된 방법으로 프록시를 만들 수 있게 도와주는 추상 레이어를 제공
- 스프링의 ProxyFactoryBean은 프록시를 생성해서 빈 오브젝트로 등록하게 해주는 팩토리 빈
- InvocationHandler와의 차이 : MethodIntercepter의 invoke() 메소드는 ProxyFactoryBean으로부터 타깃 오브젝트에 대한 정보까지도 함께 제공 받음 
-> 독립적으로 만들어짐 
-> 타깃이 다른 여러 프록시에서 함께 사용할 수 있고, 싱글톤 빈으로 등록 가능

```java
public class DynamicProxyTest {
	@Test
	public void simpleProxy() {
		// JDK 다이내믹 프록시 생성
		Hello proxiedHello = (Hello)Proxy.newProxyInstance(
				getClass().getClassLoader(), 
				new Class[] { Hello.class},
				new UppercaseHandler(new HelloTarget()));
		...
	}
	@Test
	public void proxyFactoryBean() {
		ProxyFactoryBean pfBean = new ProxyFactoryBean();
		pfBean.setTarget(new HelloTarget()); // 타깃 설정
		pfBean.addAdvice(new UppercaseAdvice()); // 부가 기능을 담은 어드바이스를 추가한다. 여러 개 추가 가능

		Hello proxiedHello = (Hello) pfBean.getObject(); // FractoryBean이므로 getObject로 생성된 프록시를 가져옴
		
		assertThat(proxiedHello.sayHello("Toby"), is("HELLO TOBY"));
		assertThat(proxiedHello.sayHi("Toby"), is("HI TOBY"));
		assertThat(proxiedHello.sayThankYou("Toby"), is("THANK YOU TOBY"));
	}
	
	static class UppercaseAdvice implements MethodInterceptor {
		public Object invoke(MethodInvocation invocation) throws Throwable {
			String ret = (String)invocation.proceed(); // 리플렉샨의 Method와 달리 메소드 실행 시 타깃 오브젝트를 전달할 필요가 없다. MethodInvocation은 메소드 정보와 함께 타깃 오브젝트를 알고 있기 때문
			return ret.toUpperCase();	// 부가 기능 적용
		}
	}
	static interface Hello {
		String sayHello(String name);
		String sayHi(String name);
		String sayThankYou(String name);
	}
	
	static class HelloTarget implements Hello {
		public String sayHello(String name) {
			return "Hello " + name;
		}

		public String sayHi(String name) {
			return "Hi " + name;
		}

		public String sayThankYou(String name) {
			return "Thank You " + name;
		}
	}
}
```
- 어드바이스 : 타깃이 필요없는 순수한 부가 기능
	- MethodInvocation은 일종의 콜백 오브젝트로, proceed() 메소드를 실행하면 타깃 오브젝트의 메소드를 내부적으로 실행해주는 기능. 공유 가능한 템플릿처럼 동작.
	- MethodInvocation을 싱글톤으로 두고 공유
	- ProxyFactoryBean에 이 MethodInterceptor를 설정해줄 때는 일반적인 DI 경우처럼 수정자 메소드를 사용하는 대신 addAdvice()라는 메소드를 사용한다.
	- 새로운 부가기능을 추가할 때마다 프록시와 프록시 팩토리 빈도 추가해줘야 한다는 문제를 해결
	> MethodInterceptor처럼 타깃 오브젝트에 적용하는 부가기능을 담은 오브젝트를 스프링에서는 어드바이스라고 부른다.

	- ProxyFactoryBean가 인터페이스 타입을 제공 받지 않고 작동할 수 있는 이유 : ProxyFactoryBean에 있는 인터페이스 자동검출 기능을 사용해 타깃오브젝트가 구현하고 있는 인터페이스 정보를 알아내서 인터페이스를 모두 구햔하는 프록시를 만들어줌.

- 포인트컷 : 부가기능 적용 대상 메소드 선정 방법
	- InvocationHandler : 메소드의 이름을 가지고 부가기능을 적용 대상 메소드를 선정했었음
	- 트랜잭션 적용 메소드 패턴은 프록시마다 다를 수 있기 때문에 여러 프록시가 공유하는 MethodInterceptor에 특정 프록시에만 적용되는 패턴을 넣으면 문제가 됨. -> 분리
	> 포인트컷 : 메소드 선정 알고리즘을 담은 오브젝트 
	
	- 프록시 - 클라이언트로 요청받음 -> 포인트컷에게 부가기능을 부여할 메소드인지를 확인해달라고 요청 -> MethodInterceptor 타입의 어드바이스를 호출 - 타깃 메소드의 호출이 필요하면 프록시로부터 전달받은 MethodInvocation 타입 콜백 오브젝트의 proceed() 메소드를 호출
	- 어드바이스 - 템플릿, MethodInvocation 오브젝트 - 콜백
```java
@Test
	public void pointcutAdvisor() {
		ProxyFactoryBean pfBean = new ProxyFactoryBean();
		pfBean.setTarget(new HelloTarget());
		
		NameMatchMethodPointcut pointcut = new NameMatchMethodPointcut(); // 메소드 이름을 비교해서 대상을 선정하는 알고리즘을 제공하는 포인트컷 생성
		pointcut.setMappedName("sayH*");  // 이름 비교조건 설정
		
		pfBean.addAdvisor(new DefaultPointcutAdvisor(pointcut, new UppercaseAdvice())); // 포인트컷과 어드바이스를 Advisor로 묶어서 한 번에 추가
		
		Hello proxiedHello = (Hello) pfBean.getObject();
		
		assertThat(proxiedHello.sayHello("Toby"), is("HELLO TOBY"));
		assertThat(proxiedHello.sayHi("Toby"), is("HI TOBY"));
		assertThat(proxiedHello.sayThankYou("Toby"), is("Thank You Toby")); 
		// 메소드 이름이 포인트컷의 선정조건에 맞지 않으므로, 부가기능이 적용되지 않음.
	}
```	
포인트컷과 어드바이스를 따로 등록하면 어떤 어드바이스에 대해 어떤 포인트컷을 적용할지 애매해지기 때문에 Advisor 타입의 오브젝트에 담아서 조합을 만들어 등록.

### 6.4.2 ProxyFactoryBean 적용
- TransactionAdvice
```java
						// 스프링의 어드바이스 인터페이스 구현
public class TransactionAdvice implements MethodInterceptor {
	PlatformTransactionManager transactionManager;

	public void setTransactionManager(PlatformTransactionManager transactionManager) {
		this.transactionManager = transactionManager;
	}

	// 타겟을 호출하는 기능을 가진 콜백 오브젝트를 프록시로부터 받는다. 덕분에 어드바이스는 특정 타깃에 의존하지 않고 재사용 가능하다.
	public Object invoke(MethodInvocation invocation) throws Throwable {
		TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
		try {
		// 콜백을 호출해서 타깃의 메소드를 실행한다. 타깃 메소드 호출 전후로 필요한 부가기능을 넣을 수 있다. 경우에 따라서 타깃이 아예 호출되지 않게 하거나 재시도를 위한 반복적인 호출도 가능하다.
			Object ret = invocation.proceed();
			this.transactionManager.commit(status);
			return ret;
		} catch (RuntimeException e) {
			this.transactionManager.rollback(status);
			throw e;
		}
	}
}
```
- 스프링의 XML 설정파일
```java
<bean id="transactionAdvice" class="springbook.user.service.TransactionAdvice">
		<property name="transactionManager" ref="transactionManager" />
	</bean>
	
	<bean id="transactionPointcut" class="org.springframework.aop.support.NameMatchMethodPointcut">
		<property name="mappedName" value="upgrade*" />
	</bean>
	
	<bean id="transactionAdvisor" class="org.springframework.aop.support.DefaultPointcutAdvisor">
		<property name="advice" ref="transactionAdvice" />
		<property name="pointcut" ref="transactionPointcut" />
	</bean>
	<bean id="userService" class="org.springframework.aop.framework.ProxyFactoryBean">
		<property name="target" ref="userServiceImpl" />
		<property name="interceptorNames">
			<list>
				<value>transactionAdvisor</value>
			</list>
		</property>
	</bean>
	
```

- 테스트
팩토리 빈을 직접 가져올 때 캐스팅할 타입만 ProxyFactoryBean으로 간단히 변경
```java
@Test 
	@DirtiesContext -> 컨택스트 설정을 변경하기 때문에 필요
	public void upgradeAllOrNothing() {
		TestUserService testUserService = new TestUserService(users.get(3).getId());
		testUserService.setUserDao(userDao);
		testUserService.setMailSender(mailSender);
		
		ProxyFactoryBean txProxyFactoryBean = 
			context.getBean("&userService", ProxyFactoryBean.class); // 변경
		txProxyFactoryBean.setTarget(testUserService);
		UserService txUserService = (UserService) txProxyFactoryBean.getObject();			 
```
- 어드바이스와 포인트컷의 재사용
트랜잭션 부가기능을 담은 TransactionAdvice는 하나만 만들어서 싱글톤 빈으로 등록해주면, DI 설정을 통해 모든 서비스에 적용이 가능하다. 메소드 선정 방식이 달라지는 경우만 포인트컷의 설정을 따로 등록하고 어드바이저로 조합해서 적용해주면 된다.


## 6.5 스프링의 AOP
### 6.5.1 자동 프록시 생성
스프링 AOP의 핵심은 비즈니스 로직과 공통 관심사를 분리하는 것이다. 이를 위해 자동 프록시 생성 기능이 필요하다. `ProxyFactoryBean`을 통해 프록시 객체를 생성하며, 이 과정에서 특정 메서드에 대한 부가 기능을 적용할 수 있다.

- 중복 문제의 접근 방법
중복된 코드 문제를 해결하기 위해 JDBC API를 사용하는 DAO 메서드에서 `try/catch/finally` 블록을 활용한다. 이를 통해 코드의 중복을 줄이고, 객체 간의 의존성을 낮출 수 있다.
다이내믹 프록시처럼 코드 자동생성 기법을 이용해 해결할 수 있다.

- 빈 후처리기를 이용한 자동 프록시 생성기
스프링에서는 빈 후처리기를 통해 자동 프록시를 생성한다. `DefaultAdvisorAutoProxyCreator`가 이를 담당하며, 빈 오브젝트의 생성과 관련된 전반적인 설정을 관리한다. 빈 후처리기를 사용하여 필요에 따라 다양한 프록시를 생성할 수 있다.
빈 후처리기를 빈에 등록하면 스프링은 빈 오브젝트를 만들 때마다 후처리기에 빈을 보냄. -> DefaultAdvisorAutoProxyCreator는 빈이 프록시 적용 대상인지 확인

- 확장된 포인트컷
`Pointcut` 인터페이스는 두 가지 기능인 `MethodMatcher`와 `ClassFilter`를 정의한다. `MethodMatcher`는 어드바이스를 적용할 메서드인지를 검사하며, `ClassFilter`는 프록시를 적용할 클래스를 필터링하는 역할을 한다. `ProxyFactoryBean`에서 이들 기능을 활용하여 프록시를 적용한다.

- 포인트컷 테스트
포인트컷의 기능을 확인하기 위해 `NameMatchMethodPointcut`을 사용하는 테스트를 진행한다. 이 테스트는 특정 조건을 만족하는 메서드에 대해 프록시가 적용되는지를 검사한다. `HelloTarget` 클래스를 대상으로 하여 여러 메서드에 대해 조건을 설정하고, 프록시가 제대로 작동하는지 확인한다.
```java
@Test
	public void classNamePointcutAdvisor() {
		NameMatchMethodPointcut classMethodPointcut = new NameMatchMethodPointcut() {  
			public ClassFilter getClassFilter() {
				return new ClassFilter() {
					public boolean matches(Class<?> clazz) {
						return clazz.getSimpleName().startsWith("HelloT");
					}
				};
			}
		};
		classMethodPointcut.setMappedName("sayH*");

		checkAdviced(new HelloTarget(), classMethodPointcut, true);  

		class HelloWorld extends HelloTarget {};
		checkAdviced(new HelloWorld(), classMethodPointcut, false);  
		
		class HelloToby extends HelloTarget {};
		checkAdviced(new HelloToby(), classMethodPointcut, true);
	}


	private void checkAdviced(Object target, Pointcut pointcut, boolean adviced) { 
		ProxyFactoryBean pfBean = new ProxyFactoryBean();
		pfBean.setTarget(target);
		pfBean.addAdvisor(new DefaultPointcutAdvisor(pointcut, new UppercaseAdvice()));
		Hello proxiedHello = (Hello) pfBean.getObject();
		
		if (adviced) {
			assertThat(proxiedHello.sayHello("Toby"), is("HELLO TOBY"));
			assertThat(proxiedHello.sayHi("Toby"), is("HI TOBY"));
			assertThat(proxiedHello.sayThankYou("Toby"), is("Thank You Toby"));
		}
		else {
			assertThat(proxiedHello.sayHello("Toby"), is("Hello Toby"));
			assertThat(proxiedHello.sayHi("Toby"), is("Hi Toby"));
			assertThat(proxiedHello.sayThankYou("Toby"), is("Thank You Toby"));
		}
	}
	
```

### 6.5.2 DefaultAdvisorAutoProxyCreator의 적용
- 클래스 필터를 적용한 포인트컷 작성
클래스 필터를 적용하여 특정 클래스에 대해서만 프록시를 생성하도록 설정할 수 있다. `NameMatchMethodPointcut`과 `SimpleClassFilter`를 조합하여 필터링을 구현한다.

```java
static class SimpleClassFilter implements ClassFilter {
    String mappedName;
	private SimpleClassFilter(String mappedName {
	     this.mappedName = mappedName;
	}
    public boolean matches(Class<?> clazz) {
        return PatternMatchUtils.simpleMatch(mappedName, clazz.getSimpleName());
    }
}
```
- 어드바이저를 이요아는 자동 프록시 생성기 등록
DefaultAdvisorAutoProxyCreator 등록

- 포인트컷 등록
`transactionPointCut`이라는 이름의 포인트컷 빈을 등록한다. 이 빈은 `NameMatchClassMethodPointcut` 클래스를 사용하여 `ServiceImpl` 클래스를 대상으로 하며, 메서드는 `upgrade*`로 지정된다.

```xml
<bean id="transactionPointCut" 
      class="springbook.service.NameMatchClassMethodPointcut">
    <property name="mappedClassName" value="*ServiceImpl" />
    <property name="mappedName" value="upgrade*" />
</bean>
```

- 어드바이저와 어드바이저
`transactionAdvice` 빈을 통해 트랜잭션 관리 기능을 제공한다. `ProxyFactoryBean`을 통해 `transactionAdvisor` 는 DefaultAdvisorAutoProxyCreator에 자동 수집되고 프록시 선정 과정에 참여하며, 자동 생성된 프록시에 다이내믹하게 DI 돼서 동작하는 어드바이저가 됨

- ProxyFactoryBean 제어와 서비스 빈의 원상복구
 `UserServiceImpl` 빈의 아이디를 UserService로 바꿈
```xml
<bean id="userService" class="springbook.service.UserServiceImpl">
    <property name="userDao" ref="userDao" />
    <property name="mailSender" ref="mailSender" />
</bean>
```

- 자동 프록시 생성기를 사용하는 테스트
문제점 
1. TestUserService가 UserServiceTest 클래스 내부에 정의된 스태틱 클래스이다.
2. 포인트 컷이 트랜잭션 어드바이스를 적용해주는 대상 클래스의 이름 패턴이 *ServiceImpl이라고 되어 있음 
-> 이름을 TestUserServiceImpl로 변경

테스트 클래스
```java
public class UserServiceTest {
    @Autowired
    UserService userService;

    @Test
    public void upgradeAllOrNothing() {
        // 테스트 코드
    }
}
```
테스트에서는 `@Autowired`를 활용하여 `UserService` 타입의 오브젝트를 주입받는다. 이때 프록시가 아닌 실제 구현체가 아닌 `ProxyFactoryBean`을 통해 생성된 프록시가 사용되도록 설정한다. 

`upgradeAllOrNothing()` 테스트를 추가하여, 특정 조건에서 예외가 발생하는지를 확인한다. 이 과정에서 서비스 빈의 DI 정보가 제대로 반영되는지를 검증한다.
자동 프록시 생성 방식과 DI 정보를 활용하여 설정된 테스트 코드가 정상적으로 동작하는지 확인하며, 스프링 AOP의 유연성을 확인할 수 있다.


- 자동 생성 프록시 확인
테스트를 통해 자동 프록시 생성이 제대로 이루어졌는지를 확인한다. 여러 가지 빈 등록과 포인트컷 설정을 통해 프록시가 올바르게 작동하는지 점검한다. 특히, 트랜잭션 관련 테스트에서는 트랜잭션 어드바이저가 제대로 적용되었는지를 확인해야 한다.

트랜잭션 포인트컷을 설정하기 위해 `mappedClassName` 속성을 변경한다. 테스트 대상이 되는 `TestUserServiceImpl` 클래스가 올바르게 설정되도록 조정한다.
```xml
<bean id="transactionPointCut"
      class="springbook.user.service.NameMatchClassMethodPointcut">
    <property name="mappedClassName" value="NotServiceImpl" />
    <property name="mappedName" value="upgrade*" />
</bean>
```

자동 프록시 생성 테스트
`upgradeAllOrNothing()` 테스트를 통해 자동 프록시 생성이 올바르게 작동하는지 확인한다. 이 과정에서 `userService` 빈이 올바르게 프록시화되었는지를 점검하고, JDK 다이내믹 프록시가 제대로 생성되었는지를 확인한다.

```java
@Test
public void advisorAutoProxyCreator() {
    assertThat(testUserService, is(java.lang.reflect.Proxy.class)); // 프록시 타입 확인
}
```

### 6.5.3 포인트컷 표현식을 이용한 포인트컷
- 포인트컷 표현식
`AspectJExpressionPointcut`을 사용하여 포인트컷을 정의하는 방법을 설명한다. 이 방식은 프로그래밍적으로 메서드를 선택할 수 있는 유연성을 제공한다.

포인트컷 테스트 클래스
```java
package springbook.learningtest.spring.pointcut;

public class Target implements TargetInterface {
    public void hello() {}
    public void hello(String a) {}
    public int minus(int a, int b) { return 0; }
    public int plus(int a, int b) { return 0; }
    public void method() {}
}
```

- 포인트컷 표현식 문법
포인트컷 표현식은 메서드를 필터링하는 강력한 도구로, `execution` 표현식을 사용하여 메서드를 정의한다. 예를 들어, `execution(int minus(int, int))`는 `minus`라는 메서드가 두 개의 `int` 매개변수를 가질 때 선택되는 포인트컷이다.

- public: 접근 제어자
- int: 반환 타입
- minus: 메서드 이름
- (int, int): 매개변수 타입

이러한 방식으로 메서드 시그니처를 정의하면, 특정 메서드를 선택하는 데 유용하다.

포인트컷 표현식을 사용한 테스트 코드의 예시
```java
@Test
public void methodSignaturePointcut() throws SecurityException, NoSuchMethodException {
    AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();
    pointcut.setExpression("execution(public int springbook.learningtest.spring.pointcut.Target.minus(int, int))");

    assertThat(pointcut.getClassFilter().matches(Target.class) && 	
		    pointcut.getMethodMatcher().matches(Target.class.getMethod("minus", int.class, int.class), null), is(true));

    // Target.plus()
    assertThat(pointcut.getMethodMatcher().matches(Target.class) && pointcut.getMethodMacher().matches(Target.class.getMethod("plus", int.class, int.class), null), is(false));
}
```

- 포인트컷 표현식 테스트
포인트컷 표현식을 테스트하기 위해 `methodSignatureMatches` 메서드를 사용하여 다양한 메서드와의 일치 여부를 확인한다. 이 메서드는 메서드 이름과 타입, 매개변수를 비교하여 포인트컷이 올바르게 작동하는지를 검증한다.

```java
public void pointcutMatches(String expression, Boolean expected, Class<?> clazz, String methodName, Class<?>... args) throws Exception {
    AspectJExpressionPointcut pointcut = new AspectJExpressionPointcut();
    pointcut.setExpression(expression);
    assertThat(pointcut.getClassFilter().matches(clazz));
    assertThat(pointcut.getMethodMatcher().matches(clazz.getMethod(methodName, args), null), is(expected));
}
```
포인트컷 표현식을 사용하면 메서드를 세밀하게 선택할 수 있으며, 이를 통해 AOP의 유연성을 극대화할 수 있다. 다양한 메서드에 대해 포인트컷을 적용하고 테스트함으로써, AOP의 적용 범위를 확장하고 비즈니스 로직과 공통 관심사를 효과적으로 분리하는 데 기여할 수 있다.

`pointcutMatches()` 메서드를 활용하여 타깃 클래스의 모든 메서드에 대해 포인트컷 선택 여부를 확인하는 헬퍼 메서드를 작성한다. 이를 통해 다양한 메서드에 대한 포인트컷의 적합성을 검증한다.
```java
public void targetClassPointcutMatches(String expression, boolean... expected) throws Exception {
    pointcutMatches(expression, expected[0], Target.class, "hello");
    pointcutMatches(expression, expected[1], Target.class, "hello", String.class);
    pointcutMatches(expression, expected[2], Target.class, "plus", int.class, int.class);
    pointcutMatches(expression, expected[3], Target.class, "minus", int.class, int.class);
    pointcutMatches(expression, expected[4], Target.class, "method");
}
```
테스트 결과
포인트컷 표현식의 테스트 결과는 다음과 같다. 다양한 메서드에 대해 포인트컷이 예상대로 작동하는지를 확인한다. 예를 들어, `execution("hello(..)")`는 `hello()`와 일치하고, `execution("hello(String)")`는 `hello(String)`와 일치한다.


포인트컷 표현식을 이용하는 포인트컷 적용
AspectJ 포인트컷 표현식을 사용하여 메서드를 선택할 때, `execution` 표현식을 활용한다. 예를 들어, `execution(*.ServiceImpl.upgrade(..))`와 같은 표현식을 사용하여 특정 메서드에 대해 포인트컷을 설정할 수 있다.

```xml
<property name="mappedClassName" value="*ServiceImpl" />
<property name="mappedName" value="upgrade*" />
```
포인트컷 표현식을 사용하여 메서드를 선택하는 강력한 방법을 제공한다. 예를 들어, `execution(..*ServiceImpl.upgrade(..))`와 같은 표현식은 특정 메서드에 대해 포인트컷을 설정할 수 있다.
```xml
<bean id="transactionPointCut"
      class="org.springframework.aop.aspectj.AspectJExpressionPointcut">
    <property name="expression" value="execution(*.*ServiceImpl.upgrade(..))" />
</bean>
```

- 타입 패턴과 클래스 이름 패턴
포인트컷 표현식은 메서드의 이름 패턴과 클래스 이름 패턴을 조합하여 사용할 수 있다. 예를 들어, `UserService`를 구현한 `UserServiceImpl` 클래스의 메서드를 선택하는 데 유용하다.

### 6.5.4 AOP란 무엇인가?
AOP(Aspect-Oriented Programming)는 비즈니스 로직을 분리하여 공통 관심사를 처리하는 프로그래밍 기법이다. 이를 통해 코드의 중복을 줄이고, 모듈화를 촉진할 수 있다.

- 트랜잭션 서비스 추상화
트랜잭션 관리 기능을 제공하여 비즈니스 로직을 간소화하고, 트랜잭션의 일관성을 보장한다. 이를 위해 JTA와 JDBC를 활용하여 트랜잭션을 관리하는 방법을 제공한다.

- 프록시 및 데코레이터 패턴
AOP를 통해 비즈니스 로직을 감싸는 데코레이터 패턴을 활용하여, 트랜잭션 관리 또는 로깅과 같은 기능을 추가할 수 있다. 이를 통해 코드의 가독성을 높이고, 유지보수를 용이하게 한다.

- 자동 프록시 생성 방법과 포인트컷
자동 프록시 생성 시 포인트컷을 설정하여 특정 메서드에 대한 프록시를 적용할 수 있다. 이를 통해 비즈니스 로직을 간소화하고, 공통 관심사를 효과적으로 관리할 수 있다.

- 부가기능의 모듈화
AOP(Aspect-Oriented Programming)는 공통 관심사를 모듈화하여 코드의 재사용성을 높이고, 비즈니스 로직과 부가기능을 분리하는 방법을 제공한다. 이를 통해 코드를 간결하고 유지 보수하기 쉽게 만들 수 있다.

- AOP: 애스펙트 지향 프로그래밍
트랜잭션 관리 기능을 추가하여 비즈니스 로직에서 발생할 수 있는 문제를 효과적으로 처리할 수 있다. DI(Dependency Injection)를 통해 부가기능을 구현하면, 코드의 중복을 줄이고, 독립적인 모듈로 관리할 수 있다.
트랜잭션 경계를 명확히 설정하여, 트랜잭션이 필요한 부분에서만 트랜잭션을 적용할 수 있도록 한다. 이를 통해 비즈니스 로직의 일관성을 유지하고, 트랜잭션의 수행 여부를 명확히 할 수 있다.
AOP는 애플리케이션 구조에서 부가기능을 효과적으로 관리하는 데 기여한다. 이를 통해 비즈니스 로직을 더 깔끔하게 유지하고, 부가기능이 원활하게 작동하도록 보장할 수 있다.

### 6.5.5 AOP 적용 기술
AOP를 적용하기 위해 다양한 기술을 사용할 수 있으며, 그 중에서도 프록시 패턴과 데코레이터 패턴이 있다. 이를 통해 비즈니스 로직에 부가기능을 추가하고, 코드의 가독성을 높일 수 있다.

AOP는 비즈니스 로직과 부가기능을 분리하여 코드의 가독성을 높이고 유지보수를 용이하게 만드는 기술이다. 스프링에서는 AOP를 적용하기 위해 다양한 방법을 제공하며, AspectJ를 활용한 포인트컷 표현식이 그 예이다.

부가기능은 코드의 중복을 줄이고, 독립적인 모듈로 관리할 수 있도록 도와준다. 이를 통해 비즈니스 로직을 더 깔끔하게 유지하고, 필요할 때마다 모듈을 쉽게 추가하거나 제거할 수 있다.

### 6.5.6 AOP의 용어
AOP에서 자주 사용되는 용어는 다음과 같다:

- 타깃: 부가기능이 적용될 대상인 클래스.
- 어드바이저: 특정 포인트컷에 부가기능을 적용하는 객체.
- 조인 포인트: 부가기능이 적용될 수 있는 지점.
- 포인트컷: 어떤 조인 포인트에 부가기능을 적용할지를 정의하는 표현식.
- 애스펙트: 비즈니스 로직과 부가기능을 결합하는 모듈.

### 6.5.7 AOP 네임스페이스
AOP를 적용하기 위해서는 최소한 네 가지 빈을 등록해야 한다:
1. 자동 프록시 생성기: `DefaultAdvisorAutoProxyCreator` 클래스를 빈으로 등록.
2. 어드바이저: `TransactionAdvice` 같은 부가기능을 담는 빈.
3. 포인트컷: `AspectJExpressionPointcut` 클래스를 빈으로 등록.
4. 타깃 클래스: 실제 비즈니스 로직을 담고 있는 클래스.

-  AOP 네임스페이스
스프링 AOP를 적용하기 위해 기계적으로 적용할 빈들을 간편하게 등록할 수 있다. AOP 관련 빈은 `aop` 네임스페이스를 사용하여 정의할 수 있다.
스프링 AOP를 사용하기 위해 XML 네임스페이스를 설정해야 한다. 이를 통해 AOP 관련 빈을 정의할 수 있으며, 다음과 같이 설정한다:
```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="...">
```

AOP 빈 설정
```xml
<aop:config>
    <aop:pointcut id="transactionPointCut"
                   expression="execution(*.*ServiceImpl.upgrade(..))" />
    <aop:advisor advice-ref="transactionAdvice"
                  pointcut-ref="transactionPointCut" />
</aop:config>
```

- 어드바이저 내장 포인트컷
AspectJ 포인트컷 표현식을 활용하여 포인트컷을 정의할 수 있으며, 어드바이저 태그와 결합하여 사용할 수 있다. 다음과 같이 설정할 수 있다:

```xml
<aop:config>
    <aop:advisor advice-ref="transactionAdvice"
                  pointcut="execution(*.*ServiceImpl.upgrade(..))" />
</aop:config>
```
이 설정을 통해 특정 메서드에 대해 트랜잭션 관리 기능을 쉽게 적용할 수 있다.
