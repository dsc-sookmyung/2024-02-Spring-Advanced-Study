# 6장 AOP
스프링에 적용된 가장 인기 있는 AOP의 적용 대상은 바로 선언적 트랜잭션 기능이다. 서비스 추상화를 통해 많은 근본적인 문제를 해결했던 트랜잭션 경계설정기능을 AOP를 이용해 더욱 세련되고 깔끔한 방식으로 바꿔보자.

## 6.1 트랜잭션 코드의 분리
### 6.1.1 메소드 분리
	public void upgradeLevels() {
		TransactionStatus status = this.transactionManager
				.getTransaction(new DefaultTransactionDefinition());
		try {

			List<User> users = userDao.getAll();
			for (User user : users) {
				if (canUpgradeLevel(user)) {
					upgradeLevel(user);
				}
			}

			this.transactionManager.commit(status);
		} catch (RuntimeException e) {
			this.transactionManager.rollback(status);
			throw e;
		}
	}


- 트랜잭션 경계설정의 코드와 비즈니스 로직 코드 간에 서로 주고받는 정보가 없다. 
- 성격이 다를 뿐 아니라 서로 주고받는 것도 없는, 완벽하게 독립적인 코드

비즈니스 로직을 담당하는 코드를 메소드로 추출해서 독립
```java
public void upgradeLevels() {
	TransactionStatus status = this.transactionManager
		.getTransaction(new DefaultTransactionDefinition());
	try {
		upgradeLevelsInternal();
		this.transactionManager.commit(status);
	} catch (RuntimeException e) {
		this.transactionManager.rollback(status);
		throw e;
	}
}

private void upgradeLevelsInternal() {
	List<User> users = userDao.getAll();
	for (User user : users) {
		if (canUpgradeLevel(user)) {
			upgradeLevel(user);
		}
	}
}
```

### 6.1.2 DI를 이용한 클래스의 분리
- DI 적용을 이용한 트랜잭션 분리
UserService는 현재 클래스로 되어 있으니 다른 코드에서 사용한다면 UserService 클래스를 직접 참조하게 된다. 그렇다면 트랜잭션 코드를 어떻게든 해서 UserService 밖으로 빼번리면 UserService 클래스를 직접 사용하는 클라이언트 코드에서는 트랜잭션 기능이 빠진 UserService를 사용하게 될 것이다.

	UserService를 인터페이스로 만들고 기존 코드는 UserService 인터페이스의 구현 클래스를 만들어넣도록 한다.

	UserService를 구현한 또 다른 구현 클래스를 만든다 -> 트랜잭션의 경계설정이라는 책임을 맡고 있음 

	UserService의 구현 클래스에 실제적인 로직 처리 작업은 위임

- UserService 인터페이스 도입
```java
public interface UserService {
	void add(User user);
	void upgradeLevels();
}
```
UserServiceImpl은 PlatformTransactionManager 인스턴스 변수와 수정자 메소드, 또 upgradeLevels()에 남겨뒀던 트랜잭션 관련 코드도 모두 제거

- 분리된 트랜잭션 기능

UserService에서 트랜잭션 코드와 비즈니스 로직을 분리하기 위해 메일 발송 기능도 DI를 통해 구현했던 것처럼 트랜잭션 처리 기능을 도입. 
기존에는 코드에서 직접 구현했지만, 이제는 스프링의 트랜잭션 API를 도입해 처리한다.

 - 트랜잭션 기능 구현의 UserServiceTx 클래스

UserServiceTx는 다음과 같이 기본적으로 UserService 인터페이스를 구현하게 만듦
```java
package springbook.user.service;

public class UserServiceTx implements UserService {
   UserService userService;
	PlatformTransactionManager transactionManager;

	public void setTransactionManager(
			PlatformTransactionManager transactionManager) {
		this.transactionManager = transactionManager;
	}

	public void setUserService(UserService userService) {
		this.userService = userService;
	}

	public void add(User user) {
		this.userService.add(user);
	}

	public void upgradeLevels() {
		TransactionStatus status = this.transactionManager
				.getTransaction(new DefaultTransactionDefinition());
		try {

			userService.upgradeLevels();

			this.transactionManager.commit(status);
		} catch (RuntimeException e) {
			this.transactionManager.rollback(status);
			throw e;
		}
	}
}
```
UserServiceTx에 트랜잭션의 경계설정이라는 부가적인 작업을 부여

- 트랜잭션 적용을 위한 DI 설정
빈 오브젝트와 의존관계
> Client (UserServiceTest) -> UserServiceTx -> UserServiceImpl
스프링 설정 파일에서 다음과 같이 빈을 설정한다.
```xml
<bean id="userService" class="springbook.user.service.UserServiceTx">
    <property name="transactionManager" ref="transactionManager" />
    <property name="userService" ref="userServiceImpl" />
</bean>

<bean id="userServiceImpl" class="springbook.user.service.UserServiceImpl">
    <property name="userDao" ref="userDao" />
    <property name="mailSender" ref="mailSender" />
</bean>
```
- 트랜잭션 분리에 따른 테스트 수정
UserService 기능 테스트시 UserServiceTx가 트랜잭션이 제대로 적용되는지 확인
	-   MockMailSender를 통해 메일 발송 로직 검증 - UserServiceImpl을 DI 받아야함.

	-   @Autowired를 사용해 스프링 DI 설정을 테스트
기존의 테스트 코드를 수정하여 트랜잭션이 포함된 새로운 구조에 맞게 변경
```java
@Test
public void upgradeLevels() throws Exception {
    MockMailSender mockMailSender = new MockMailSender();
    userServiceImpl.setMailSender(mockMailSender);
}
```
트랜잭션 롤백 상황에서의 테스트도 가능하도록 구성
```java
// 리스트 6-9에서 보여준 UserService의 테스트 코드
public void upgradeAllOrNothing() throws Exception {
    TestUserService testUserService = new TestUserService(users.get(3).getId());
    testUserService.setUserDao(userDao);
    testUserService.setMailSender(mailSender);
    
    UserServiceTx txUserService = new UserServiceTx(); 
    txUserService.setTransactionManager(transactionManager);
    txUserService.setUserService(testUserService);
    
    userDao.deleteAll();
    for(User user : users) userDao.add(user);
    
    try {
        txUserService.upgradeLevels();
        fail("TestUserServiceException expected");
    }
    ...
}
```
TestUserService를 UserServiceImpl을 상속하도록 바꿈

- 트랜잭션 경계설정 코드 분리에 따른 장점

1.  비즈니스 로직 코드를 작성할 때는 트랜잭션과 같은 기술적인 내용은 신경 쓰지 않아도 됨
	- 트랜잭션 적용이 필요한 곳에서는 UserServiceTx를 통해 간단히 트랜잭션 기능을 적용 가능
	- DI를 통해 트랜잭션 기능을 자유롭게 확장하거나 변경 가능
2. 비즈니스 로직에 대한 테스트를 손쉽게 만들어낼 수 있다.

이러한 구조를 통해 관심사의 분리가 잘 이루어지고, 각 컴포넌트의 독립성이 보장되며, 테스트가 용이한 구조를 만들 수 있게 됨

## 6.2 고립된 단위 테스트

가장 편하고 좋은 테스트 방법은 가능한 한 작은 단위로 쪼개서 테스트하는 것이다. 작은 단위의 테스트가 좋은 이유는 테스트가 실패했을 때 그 원인을 찾기 쉽기 때문이다.

### 6.2.1 복잡한 의존관계 속의 테스트
UserService는 엔터프라이즈 시스템의 복잡한 모듈과는 비교할 수 없을 만큼 간단한 기능만을 갖고 있다. 그럼에도 UserService의 구현 클래스들이 동작하려면 세 가지 타입의 의존 오브젝트가 필요하다.

1.  UserDao를 통해 DB와 데이터를 주고받아야 하고
2.  TransactionManager를 구현한 PlatformTransactionManager와
3.  MailSender를 구현한 JavaMailSenderImpl이 필요하다

- 테스트 실행의 문제점

1.  DB가 항상 테스트와 함께 동작해야 하는 테스트는 작성하기 힘든 경우가 많다
2.  예를 들어 UserDao에서 사용하는 SQL이 여러 개의 테이블을 조인하고, 복잡한 조건을 갖고 있고, 통계처리를 해서 가져오는 경우는 더 많은 테스트 데이터 준비가 필요하다
3.  이런 작업이 모든 테스트에 필요하다면, 배보다 배꼽이 더 큰 상황이 될지도 모른다

### 6.2.2 테스트 대상 오브젝트 고립시키기
- 테스트를 위한 UserServiceImpl 고립
UserDao는 단지 테스트 대상의 코드가 정상적으로 수행되도록 도와주기만 하는 스텁이 아니라, 부가적인 검증 기능까지 갖진 목 오브젝트로 만들었다. 그 이유는 고립된 환경에서 동작하는 `upgradeLevels()`의 테스트 결과를 검증할 방법이 필요하기 때문이다.

- 고립된 단위 테스트 활용

```java
public void upgradeLevels() throws Exception {
    userDao.deleteAll();
    for (User user : users) userDao.add(user);
    
    MockMailSender mockMailSender = new MockMailSender();
    UserServiceImpl userService = new UserServiceImpl();
    userService.setMailSender(mockMailSender);
    
    userService.upgradeLevels();
    
    checkLevelUpgraded(users.get(0), false);
    checkLevelUpgraded(users.get(1), true);
    checkLevelUpgraded(users.get(2), false);
    checkLevelUpgraded(users.get(3), true);
    
    List<String> request = mockMailSender.getRequests();
    assertThat(request.size(), is(2));
    assertThat(request.get(0), is(users.get(1).getEmail()));
    assertThat(request.get(1), is(users.get(3).getEmail()));
}
```
이 테스트는 UserServiceTest의 upgradeLevels() 메서드에 적용된다. 이 메서드는 사용자 목록을 초기화하고 각 사용자의 레벨을 업그레이드하는 기능을 검증한다. MockMailSender를 사용하여 이메일 전송 요청을 시뮬레이션하고, 결과적으로 업그레이드된 사용자에게 이메일이 전송되었는지 확인한다.

```java
protected void upgradeLevel(User user) {
    user.upgradeLevel();
    userDao.update(user);
    sendUpgradeEmail(user);
}
```
upgradeLevel(User user) 메서드는 사용자의 레벨을 업그레이드하고, 이를 데이터베이스에 반영하며 이메일 전송을 처리한다. 이 과정에서 UserDao를 통해 데이터베이스 업데이트를 수행하고, 이메일 전송 요청을 발생시킨다.

UserDao 목 오브젝트
```java
static class MockUserDao implements UserDao {
    private List<User> users;
    private List<User> updated = new ArrayList<>();

    private MockUserDao(List<User> users) {
        this.users = users;
    }

    public List<User> getUpdated() {
        return this.updated;
    }
    
    // UserDao 메서드 구현
	public List<User> getAll() {	// 스텁 기능 제공
	    return this.users;
	}

	public void update(User user) {	// 목 오브젝트 기능 제공
	    updated.add(user);
	}

	// UnsupportedOperationException을 던지는 메서드들, 테스트에 사용되지 않음
	public void add(User user) { throw new UnsupportedOperationException(); }
	public void deleteAll() { throw new UnsupportedOperationException(); }
	public User get(String id) { throw new UnsupportedOperationException(); }
	public int getCount() { throw new UnsupportedOperationException(); }
	}
```
MockUserDao 클래스는 테스트에서 사용될 사용자 목록을 관리한다. 이 클래스를 통해 실제 데이터베이스와의 연결 없이 사용자의 상태를 조작하고, 업그레이드가 이루어진 사용자를 추적할 수 있다. getUpdated() 메서드는 업그레이드된 사용자 목록을 반환한다.

이러한 구성은 단위 테스트의 고립성을 유지하면서도, 실제 사용자의 상태를 변경하고 그 결과를 검증하는 데 중점을 둔다.

MockUserDao는 UserDao 인터페이스를 구현하여 테스트에 사용된다. 일부 메서드는 지원되지 않는 기능으로 설정되어, 실제 데이터베이스와의 상호작용 없이 테스트를 수행할 수 있게 한다. 이 클래스는 사용자 객체를 저장하고, 업그레이드된 사용자 목록을 관리하는 데 중점을 둔다.

```java
@Test
public void upgradeLevels() throws Exception {
    UserServiceImpl userServiceImpl = new UserServiceImpl();
    MockUserDao mockUserDao = new MockUserDao(this.users);
    userServiceImpl.setUserDao(mockUserDao);
    
    MockMailSender mockMailSender = new MockMailSender();
    userServiceImpl.setMailSender(mockMailSender);
    
    userServiceImpl.upgradeLevels();
    
    List<User> updated = mockUserDao.getUpdated();
    assertThat(updated.size(), is(2));
    checkUserAndLevel(updated.get(0), "joytouch", Level.SILVER);
    checkUserAndLevel(updated.get(1), "madnitel", Level.GOLD);
	
	List<String> request = mockMailSender.getRequests();
	assertThat(request.size(), is(2));
	assertThat(request.get(0), is(users.get(1).getEmail()));
	assertThat(request.get(1), is(users.get(3).getEmail()));
}

private void checkUserAndLevel(User updated, String expectedId, Level expectedLevel) {
	assertThat(updated.getId(), is(expectedId));
	assertThat(updated.getLevel(), is(expectedLevel));
}
```
이 테스트는 MockUserDao를 사용하여 upgradeLevels() 메서드를 검증한다. 업그레이드된 사용자 목록을 확인하고, 각 사용자의 ID와 레벨이 올바르게 설정되었는지 검증한다. MockMailSender를 통해 이메일 발송도 검증한다.

- 테스트 수행 성능의 향상
UserServiceTest의 upgradeLevels() 테스트를 실행하여 결과를 확인한다. 테스트는 다른 테스트와 독립적으로 실행되며, 성능을 비교하는 데 유용하다. 예를 들어, add() 메서드는 0.703초, upgradeAllOrNothing()는 0.250초가 걸리는 등 결과를 기록하여 성능을 분석할 수 있다.

이러한 과정은 고립된 단위 테스트를 통해 코드의 품질을 높이고, 실제 데이터베이스와의 상호작용 없이도 효과적으로 기능을 검증할 수 있도록 돕는다.

### 6.2.3 단위 테스트와 통합 테스트
단위 테스트는 사용자 관리 기능과 같은 특정 기능을 검증하는 데 중점을 둔다. JUnit의 테스트는 최소 0.001초의 실행 시간을 요구하며, 모든 테스트가 성공적으로 수행되기를 바란다. 그러나 데이터베이스를 사용하는 경우, 성능에 영향을 미칠 수 있다. 통합 테스트는 여러 단위를 결합하여 시스템 전체의 동작을 검증한다.

- 단위 테스트의 특징
단위 테스트는 개별 클래스나 메서드의 동작을 검증할 수 있다.
통합 테스트는 여러 단위를 결합하여 전체 시스템의 동작을 확인한다.
DAO와 같은 데이터베이스 연동 기능을 포함한 테스트는 신중하게 설계해야 한다.
코드 작성 및 TDD
테스트 주도 개발(TDD)을 통해 코드를 작성하는 경우, 먼저 테스트 케이스를 만들고 그에 맞춰 코드를 작성해야 한다. 이를 통해 코드의 품질을 높이고, 유지보수성을 향상시킬 수 있다.

### 6.2.4 목 프레임워크
단위 테스트를 효과적으로 수행하기 위해 목 오브젝트를 활용하는 것이 중요하다. MockUserDao와 같은 목 오브젝트를 사용하여, 실제 데이터베이스와의 상호작용 없이도 테스트를 수행할 수 있다. 이를 통해 테스트의 독립성을 유지하고, 성능을 향상시킬 수 있다.

이러한 원칙을 통해 단위 테스트와 통합 테스트가 상호 보완적으로 작용하며, 소프트웨어의 품질을 높이는 데 기여한다.

- Mockito 프레임워크
Mockito는 목 오브젝트를 쉽게 생성하고 관리할 수 있도록 도와주는 프레임워크이다. 이를 통해 코드의 직관성을 높이고, 테스트를 간편하게 작성할 수 있다. Mockito를 사용하면 특정 메서드 호출 시 반환값을 설정하거나, 메서드 호출 횟수를 검증할 수 있다.
```java
UserDao mockUserDao = mock(UserDao.class);
when(mockUserDao.getAll()).thenReturn(this.users);
verify(mockUserDao, times(2)).update(any(User.class));
이 코드는 mockUserDao의 getAll() 메서드가 호출될 때 this.users를 반환하도록 설정한다. 이를 통해 실제 데이터베이스와의 상호작용 없이도 테스트를 수행할 수 있다.

```java
@Test
	public void mockUpgradeLevels() throws Exception {
		UserServiceImpl userServiceImpl = new UserServiceImpl();

		UserDao mockUserDao = mock(UserDao.class);	    
		when(mockUserDao.getAll()).thenReturn(this.users);
		userServiceImpl.setUserDao(mockUserDao);

		MailSender mockMailSender = mock(MailSender.class);  
		userServiceImpl.setMailSender(mockMailSender);

		userServiceImpl.upgradeLevels();

		verify(mockUserDao, times(2)).update(any(User.class));				  
		verify(mockUserDao, times(2)).update(any(User.class));
		verify(mockUserDao).update(users.get(1));
		assertThat(users.get(1).getLevel(), is(Level.SILVER));
		verify(mockUserDao).update(users.get(3));
		assertThat(users.get(3).getLevel(), is(Level.GOLD));

		ArgumentCaptor<SimpleMailMessage> mailMessageArg = ArgumentCaptor.forClass(SimpleMailMessage.class);  
		verify(mockMailSender, times(2)).send(mailMessageArg.capture());
		List<SimpleMailMessage> mailMessages = mailMessageArg.getAllValues();
		assertThat(mailMessages.get(0).getTo()[0], is(users.get(1).getEmail()));
		assertThat(mailMessages.get(1).getTo()[0], is(users.get(3).getEmail()));
    }
```
이 테스트는 UserServiceImpl의 upgradeLevels() 메서드를 호출하고, mockUserDao의 update() 메서드가 두 번 호출되었는지 검증한다.

ArgumentCaptor<SimpleMailMessage> mailMessageArg = ArgumentCaptor.forClass(SimpleMailMessage.class);
이 코드는 메일 메시지를 캡처하여, 실제로 전송된 메일의 내용을 검증할 수 있게 해준다.

Mockito를 활용하면 단위 테스트를 간편하게 작성할 수 있으며, 테스트의 정확성을 높이는 데 기여한다. 이를 통해 개발자는 코드의 품질과 유지보수성을 향상시킬 수 있다.

## 6.3 다이내믹 프록시와 팩토리 빈
### 6.3.1 프록시와 프록시 패턴, 데코레이터 패턴
- 부가기능과 핵심기능의 분리
부가기능과 핵심기능을 분리하여 클래스를 설계하는 것이 중요하다.
핵심기능은 비즈니스 로직을 포함하고, 부가기능은 로깅, 트랜잭션 관리 등의 기능을 담당한다.
이로 인해 각 기능의 책임을 명확히 하고, 코드의 재사용성을 높일 수 있다.
- 데코레이터 패턴
데코레이터 패턴은 부가기능을 동적으로 추가하기 위해 사용된다.
여러 개의 데코레이터를 조합하여 다양한 기능을 제공할 수 있다.
예를 들어, 입력 스트림을 감싸는 데코레이터를 만들어 기능을 확장할 수 있다.
	- DI를 통한 데코레이터 패턴 설정
Spring의 DI(Dependency Injection)를 통해 데코레이터 패턴을 적용할 수 있다.
UserService와 같은 클래스에 데코레이터를 주입하여 부가기능을 동적으로 추가하는 구조를 만든다.
이 방식을 통해 코드의 유연성을 높이고, 테스트 용이성을 확보할 수 있다.

- 프록시 패턴
프록시 패턴은 객체에 대한 접근을 제어하는 방식으로, 실제 객체에 대한 참조를 대신하는 프록시 객체를 생성한다.
프록시는 원본 객체에 대한 접근을 제어하거나, 추가적인 행동을 수행할 수 있도록 한다.
이를 통해 원본 객체의 메서드를 호출하기 전에 부가적인 작업을 수행할 수 있다.
	- 데코레이터 패턴과 프로시 패턴의 결합
데코레이터 패턴은 여러 기능을 조합하여 사용할 수 있는 구조를 제공한다.
프로시 패턴과 함께 사용할 경우, 프록시가 데코레이터 역할을 하여 여러 기능을 동적으로 추가할 수 있다.
이 패턴을 통해 코드의 유연성을 높이고, 복잡한 기능을 보다 깔끔하게 구현할 수 있다.


### 6.3.2 다이내믹 프록시
다이내믹 프록시는 런타임 시에 객체를 생성하고 메서드 호출을 동적으로 처리할 수 있게 해준다.
Java의 리플렉션을 이용하여 특정 인터페이스를 구현한 프록시 객체를 생성할 수 있다.
이 방식은 코드의 재사용성과 유연성을 높이는 데 기여한다.
- 트랜잭션 관리
UserServiceTx 클래스는 UserService 인터페이스를 구현하며, 트랜잭션 관리를 위한 부가 기능을 포함한다.
트랜잭션 상태를 관리하여, upgradeLevels() 메서드에서 예외가 발생하면 롤백할 수 있도록 설정한다.
이 방식은 데이터베이스의 일관성을 유지하는 데 중요한 역할을 한다.
- 리플렉션
리플렉션은 런타임 시에 객체의 메타데이터를 검사하고 조작할 수 있는 기능을 제공한다.
Java의 Method 클래스를 사용하여 메서드를 동적으로 호출할 수 있으며, 이를 통해 코드의 유연성을 높일 수 있다.
getMethod() 메서드를 사용하여 특정 메서드를 가져오고, invoke() 메서드로 실행할 수 있다.
	- 리플렉션을 이용한 테스트
리플렉션을 활용하여 메서드를 테스트하는 방법을 보여준다.
예를 들어, String 클래스의 length() 및 charAt() 메서드를 리플렉션을 통해 호출하는 방법을 설명한다.
이 방법을 통해 코드의 유연성을 높이고, 메서드 호출을 동적으로 수행할 수 있다.

- 프록시 클래스
프로시 클래스는 특정 인터페이스를 구현하여 원본 객체에 대한 접근을 제어하는 역할을 한다.
예를 들어, Hello 인터페이스를 정의하고, 이를 구현한 HelloTarget 클래스를 생성한다.
HelloTarget 클래스는 sayHello(), sayHi(), sayThankYou() 메서드를 구현하며, 특정 기능을 수행한다.

HelloUppercase 프로시 클래스
```java
public class HelloUppercase implements Hello {
    private Hello hello;

    public HelloUppercase(Hello hello) {
        this.hello = hello;
    }

    public String sayHello(String name) {
        return hello.sayHello(name).toUpperCase(); // 부가기능 적용
    }

    public String sayHi(String name) {
        return hello.sayHi(name).toUpperCase();
    }

    public String sayThankYou(String name) {
        return hello.sayThankYou(name).toUpperCase();
    }
}
```
HelloUppercase는 Hello 인터페이스를 구현한 프로시 클래스로, 부가기능으로 문자열을 대문자로 변환하는 기능을 추가한다.
이를 통해 원본 HelloTarget 클래스의 메서드를 호출한 후 결과를 변환하여 반환한다.
- 다이내믹 프록시 적용
다이내믹 프록시는 인터페이스를 기반으로 런타임에 프록시 객체를 생성한다.
InvocationHandler를 사용하여 메서드 호출을 처리하며, 다이내믹 프록시를 통해 다양한 부가기능을 추가할 수 있다.

InvocationHandler의 역할
```java
public Object invoke(Object proxy, Method method, Object[] args) {
    // 메서드 호출 처리 로직
}
```
invoke() 메서드는 호출할 메서드와 인자를 받아서 처리하는 역할을 한다.
이 구조를 통해 Hello 인터페이스의 모든 메서드를 다이내믹 프록시로 처리할 수 있으며, 각 메서드에 따라 다양한 부가기능을 적용할 수 있다.

다이내믹 프록시 생성
```java
Hello proxiedHello = (Hello) Proxy.newProxyInstance(
    Hello.class.getClassLoader(),
    new Class[] { Hello.class },
    new UppercaseHandler(new HelloTarget())
);
```
Proxy.newProxyInstance() 메서드를 사용하여 다이내믹 프록시를 생성한다.
이 프록시는 HelloTarget 객체를 기본으로 설정하고, UppercaseHandler를 통해 부가기능을 추가한다.
- 다이내믹 프록시의 확장
다이내믹 프록시는 유연하게 다양한 부가기능을 추가할 수 있으며, 여러 인터페이스를 동시에 구현할 수 있다.
이는 복잡한 로직을 단순화하고, 코드의 재사용성을 높이는 데 기여한다.
UppercaseHandler는 메서드 호출 시 리턴 타입을 확인하여 적절한 처리를 할 수 있도록 설계된다.

### 6.3.3 다이내믹 프록시를 이용한 트랜잭션 부가기능
```java
public class TransactionHandler implements InvocationHandler {
    private Object target; // 부가기능을 적용할 타깃 오브젝트
    private PlatformTransactionManager transactionManager; // 트랜잭션 기능을 제공하는 객체
    private String pattern; // 트랜잭션을 적용할 메서드 이름 패턴

    public void setTransactionManager(PlatformTransactionManager transactionManager) {
        this.transactionManager = transactionManager;
    }

    public void setPattern(String pattern) {
        this.pattern = pattern;
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        if (method.getName().startsWith(pattern)) {
            return invokeInTransaction(method, args);
        } else {
            return method.invoke(target, args);
        }
    }

    private Object invokeInTransaction(Method method, Object[] args) throws Throwable {
        TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
        try {
            Object ret = method.invoke(target, args);
            this.transactionManager.commit(status);
            return ret;
        } catch (InvocationTargetException e) {
            this.transactionManager.rollback(status);
            throw e.getTargetException();
        }
    }
}
```
TransactionHandler는 트랜잭션 부가기능을 제공하기 위한 InvocationHandler를 구현한다.
특정 메서드가 호출될 때 트랜잭션을 시작하고, 메서드 실행 후 성공 시 커밋하고, 실패 시 롤백한다.
다이내믹 프록시를 이용한 트랜잭션 부가기능
UserServiceTx 클래스에서 다이내믹 프록시를 통해 트랜잭션 부가기능을 적용한다.
이 방식은 UserService의 모든 메서드에 트랜잭션을 적용할 수 있도록 하여, 코드의 재사용성을 높인다.
- 다이내믹 프록시를 이용한 테스트
```java
@Test
public void upgradeAllOrNothing() throws Exception {
    TransactionHandler txHandler = new TransactionHandler();
    txHandler.setTarget(testUserService);
    txHandler.setTransactionManager(transactionManager);
    txHandler.setPattern("upgradeLevels");

    UserService txUserService = (UserService) Proxy.newProxyInstance(
        getClass().getClassLoader(),
        new Class[] { UserService.class },
        txHandler
    );
    . . .
}
```
이 테스트는 TransactionHandler를 사용하여 트랜잭션 기능을 테스트한다.
upgradeLevels() 메서드가 호출될 때 트랜잭션이 적용되는지 확인한다.

### 6.3.4 다이내믹 프록시를 위한 팩토리 빈
다이내믹 프록시를 사용하여 트랜잭션 부가기능을 적용하기 위해 TransactionHandler를 만들고, 이를 UserService에 적용하는 방법을 설명한다.
이 과정에서 다이내믹 프록시와 TransactionHandler를 결합하여 트랜잭션 기능을 추가하는 방법을 제시한다.

FactoryBean 인터페이스
```java
public interface FactoryBean<T> {
    T getObject() throws Exception; // 빈 오브젝트를 생성하여 반환
    Class<? extends T> getObjectType(); // 생성될 오브젝트의 타입을 반환
    boolean isSingleton(); // 싱글톤 여부 확인
}
```
FactoryBean 인터페이스는 스프링에서 빈을 생성하는 방법을 정의한다.
이 인터페이스를 구현하여 특정 조건에 맞는 빈 오브젝트를 생성할 수 있다.

MessageFactoryBean 클래스
```java
public class MessageFactoryBean implements FactoryBean<Message> {
    String text;

    public void setText(String text) {
        this.text = text;
    }

    public Message getObject() throws Exception {
        return Message.newMessage(this.text); // Message 객체 생성
    }

    public Class<? extends Message> getObjectType() {
        return Message.class; // Message 클래스 반환
    }

    public boolean isSingleton() {
        return false; // 싱글톤이 아님
    }
}
```
MessageFactoryBean 클래스는 FactoryBean 인터페이스를 구현하여 Message 객체를 생성하는 팩토리 역할을 한다.
getObject() 메서드를 통해 새로운 Message 객체를 생성하고 반환한다.

- 팩토리 빈 설정 방법
```xml
<bean id="message" class="springbook.learningtest.spring.factorybean.MessageFactoryBean">
    <property name="text" value="Factory Bean" />
</bean>
```
팩토리 빈을 설정하기 위해 XML 설정 파일을 사용한다. id를 통해 빈을 식별하고, 빈의 타입을 명시한다.
getObject() 메서드를 호출하여 실제 객체를 생성하는 방식으로 팩토리 빈을 사용한다.

팩토리 빈 테스트
```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration("classpath:context.xml")
public class FactoryBeanTest {
    @Autowired
    ApplicationContext context;

    @Test
    public void getMessageFromFactoryBean() {
        Object message = context.getBean("message");
        assertThat(message, is(Message.class)); // 타입 확인
        assertThat(((Message) message).getText(), is("Factory Bean")); // 메시지 내용 확인
    }
}
```
팩토리 빈을 테스트하기 위해 JUnit을 사용하여 테스트 클래스를 작성한다.
context.getBean("message")를 호출하여 팩토리 빈에서 생성된 객체를 가져오고, 타입과 내용이 올바른지 확인한다.

트랜잭션 프록시 팩토리 빈
```java
public class TxProxyFactoryBean implements FactoryBean<Object> {
    Object target;
    PlatformTransactionManager transactionManager;
    String pattern;
    Class<?> serviceInterface;

    public void setTarget(Object target) {
        this.target = target;
    }

    public void setTransactionManager(PlatformTransactionManager transactionManager) {
        this.transactionManager = transactionManager;
    }
}
```
TxProxyFactoryBean 클래스는 트랜잭션 처리를 위한 다이내믹 프록시를 생성하는 팩토리 빈이다.
이 빈은 트랜잭션 관리 기능을 제공하고, UserService와 같은 특정 인터페이스를 타겟으로 설정할 수 있다.

- 트랜잭션 프록시 팩토리 
TxProxyFactoryBean 클래스
```java
public class TxProxyFactoryBean implements FactoryBean<Object> {
    Object target;
    PlatformTransactionManager transactionManager;
    String pattern;
    Class<?> serviceInterface;

    public void setTarget(Object target) {
        this.target = target;
    }

    public void setTransactionManager(PlatformTransactionManager transactionManager) {
        this.transactionManager = transactionManager;
    }

    public void setPattern(String pattern) {
        this.pattern = pattern;
    }

    public void setServiceInterface(Class<?> serviceInterface) {
        this.serviceInterface = serviceInterface;
    }

    public Object getObject() throws Exception {
        TransactionHandler txHandler = new TransactionHandler();
        txHandler.setTarget(target);
        txHandler.setTransactionManager(transactionManager);
        txHandler.setPattern(pattern);
        return Proxy.newProxyInstance(
            getClass().getClassLoader(),
            new Class[] { serviceInterface },
            txHandler
        );
    }

    public Class<?> getObjectType() {
        return serviceInterface;
    }

    public boolean isSingleton() {
        return false;
    }
}
```
TxProxyFactoryBean 클래스는 다이내믹 프록시를 생성하여 트랜잭션 기능을 적용하는 팩토리 빈이다.
getObject() 메서드는 TransactionHandler를 설정하고, 프록시 객체를 반환한다.

UserServiceImpl에 대한 트랜잭션 프록시 설정
```xml
<bean id="userService" class="springbook.user.service.TxProxyFactoryBean">
    <property name="target" ref="userServiceImpl" />
    <property name="transactionManager" ref="transactionManager" />
    <property name="pattern" value="upgradeLevels" />
    <property name="serviceInterface" value="springbook.user.service.UserService" />
</bean>
```
XML 설정 파일에서 TxProxyFactoryBean을 사용하여 UserService에 트랜잭션 기능을 적용한다.
target, transactionManager, pattern, serviceInterface 등의 속성을 설정하여 빈을 구성한다.

- 트랜잭션 프록시 팩토리 빈 테스트
```java
@Test
@DirtiesContext
public void upgradeAllOrNothing() throws Exception {
    TestUserService testUserService = new TestUserService();
    // UserService에 대한 트랜잭션 프록시 빈을 가져온다.
    TxProxyFactoryBean txProxyFactoryBean = (TxProxyFactoryBean) context.getBean("userService");
}
```
upgradeAllOrNothing() 메서드는 트랜잭션 프록시 빈을 테스트하는 메서드이다.
TxProxyFactoryBean을 통해 UserService의 트랜잭션 기능이 올바르게 작동하는지 확인한다.

### 6.3.5 프록시 팩토리 빈 방식의 장점과 한계

- 다이내믹 프록시를 사용하면 여러 개의 부가 기능을 쉽게 추가할 수 있다.
- 프록시 패턴을 사용하여 트랜잭션 관리 기능을 적용할 수 있다.

- 트랜잭션 프록시 빈의 재사용
```xml
<bean id="coreService" class="springbook.service.TxProxyFactoryBean">
    <property name="target" ref="coreServiceTarget" />
    <property name="transactionManager" ref="transactionManager" />
    <property name="pattern" value="upgradeLevels" />
    <property name="serviceInterface" value="complex.module.CoreService" />
</bean>
```
- 프록시 팩토리 빈 방식의 장점
타깃 인터페이스를 구현하는 클래스를 일일이 만드는 번거로움을 제거할 수 있다.


- 프록시 패턴의 한계
프록시 패턴은 부가 기능을 추가하는 데 유용하지만, 복잡성이 증가할 수 있다.
다이내믹 프록시는 특정 인터페이스에 대한 의존성을 가지므로 설계 시 주의해야 한다.

