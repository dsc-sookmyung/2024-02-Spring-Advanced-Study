#Chapter6. AOP
AOP 는 IoC/DI, 서비스 추상화와 더불어 스프링의 3대 기반기술의 하나다.

스프링에 적용된 가장 인기 있는  AOP 의 적용 대상은 바로 선언적 트랜잭션 기능이다.
서비스 추상화를 통해 많은 근본적 문제를 해결했던 트랜잭션 경계설정 기능을 AOP 를 이용해 더욱 세련되고 깔끔한 방식으로 바꾸자.

# 6.1 트랜잭션 코드의 분리
## 6.1.1 메소드 분리
```java
public void upgradeLevels() throws Exception {  
    TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());  
      
    try {  
        List<User> users = userDao.getAll();  
        for (User user : users) {  
            if (canUpgradeLevel(user)) {  
                upgradeLevel(user);  
            }  
        }  
          
        this.transactionManager.commit(status);  
    } catch (Exception e) {  
        this.transactionManager.rollback(status);  
        throw e;  
    }  
}
```
비지니스 로직 코드를 사이에 두고 트랜잭션의 시작과 종료를 담당하는 코드가 앞뒤에 위치하고 있다.
두 코드는 서로 주고 받는 것이 없는 독립적인 코드이다.
다만 ***비지니스 로직을 담당하는 코드가 트랜잭션의 시작과 종료 작업 사이에서 수행돼야 한다는 것을 잘 지켜야 한다.***

`비지니스 로직과 트랜잭션 경계설정의 분리`
```java
public void upgradeLevels() throws Exception {  
    TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());  
      
    try {  
	    upgradeLevelsInternal();
        this.transactionManager.commit(status);  
    } catch (Exception e) {  
        this.transactionManager.rollback(status);  
        throw e;  
    }  
}
private void upgradeLevelsInternal(){
	List<User> users = userDao.getAll();  
        for (User user : users) {  
            if (canUpgradeLevel(user)) {  
                upgradeLevel(user);  
            }  
        }  
}
```
이렇게 비지니스 로직 코드만 메소드에 담아서 따로 분리하니 훨씬 깔끔해보인다.

## 6.1.2 DI를 이용한 클래스의 분리
### DI 적용을 이용한 트랜잭션 분리
`DI의 기본 아이디어`
실제 사용할 오브젝트의 클래스 정체는 감춘 채 인터페이스를 통해 간접으로 접근하는 것

`현재 구조`
![img.png](img.png)
현재 구조는 강한 결합도로 고정돼 있다.

`바꾸고자 하는 구조`
![img_1.png](img_1.png)
인터페이스를 이용해 구현 클래스를 클라이언트에 노출하지 않고 런타임 시에  DI를 통해 적용하는 방법을 사용하는 것으로 바꾸고자 한다.

이걸 쓰는 이유는 일반적으로 구현 클래스를 바꿔가면서 사용하기 위해서이다.

지금 해결하고자 하는 문제는 UserService에는 순수하게 비지니스 로직을 담는 코드만 놔두고 트랜잭션 경계설정을 담당하는 코드를 외부로 빼려는 것이다.

그러므로 UserService를 구현한 또 다른 구현 클래스를 만드는 방법이 있다.

![img_2.png](img_2.png)
UserServiceTx는 트랜잭션 경게설정을 맡고 있다.
### UserService 도입
1. UserService 클래스를 UserServiceImpl로 이름 변경
2. 클라이언트가 사용할 로직을 담은 핵심 메소드만 UserService 인터페이스로 만든 후 UserServiceImpl 이 구현하도록 만든다.
3. UserServiceImpl은 UserService의 클래스 내용을 대부분 그대로 유지하면 되지만 트랜잭션과 관련된 코드는 모두 제거하면 된다.

```java
public interface UserService{
	void add(User user);
	void upgradeLevels();
}
```
```java
public class UserServiceImpl implements userService {  
    UserDao userDao;  
    MailSender mailSender;  
      
    public void upgradeLevels() {  
        List<User> users = userDao.getAll();  
        for (User user : users) {  
            if (canUpgradeLevel(user)) {  
                upgradeLevel(user);  
            }  
        }  
    }  
   
    ...  
}
```
### 분리된 트랜잭션 기능
> 비지니스 트랜잭션 처리를 담은 UserServiceTx를 만들어보자.

UserServiceTx는 기본적으로 UserService를 구현하게 만든다.
비지니스 로직에 관해서는 UserServiceTx가 아무런 관여도 하지 않는다.
```java
public class UserServiceTx implements UserService {  
    UserService userService;  
      
    // UserService를 구현한 다른 오브젝트를 DI 받는다.  
    public void setUserService(UserService userService) {  
        this.userService = userService;  
    }  
      
    // DI 받은 UserService 오브젝트에 모든 기능을 위임한다.  
    public void add(User user) {  
        userService.add(user);  
    }  
      
    public void upgradeLevels() {  
        userService.upgradeLevels();  
    }  
}
```
UserServiceTx에 트랜잭션의 경계설정이라는 부가적 작업을 부여해준다.

```java
public class UserServiceTx implements UserService {  
    UserService userService;  
 	PlatformTransactionManager transactionManager;  
      
    public void setTransactionManager(PlatformTransactionManager transactionManager) {  
    	this.transactionManager = transactionManager;      
    }  
      
    public void setUserService(UserService userService) {  
        this.userService = userService;  
    }  
      
    public void add(User user) {  
        userService.add(user);  
    }  
      
    public void upgradeLevels() {  
        TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());  
          
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
추상화된 트랜잭션 구현 오브젝트를 DI 받을 수 있도록 PlatformTransactionManager 타입의 프로퍼티도 추가했다.

### 트랜잭션 적용을 위한 DI 설정
![img_3.png](img_3.png)

트랜잭션 오브젝트를 추가해서 설정 파일을 변경해준다.(자세한 건 책을 확인)

### 트랜잭션 분리에 따른 테스트 수정

> Q. 현재 UserService가 인터페이스로 되어 있다. 그렇기 때문에 @Autowired는 기본적으로 타입이 일치하는 빈을 찾아주는데 같은 타입의 빈이 두 개가 존재한다 그럼 어떻게 찾나?
>> A. 필드 이름을 이용해서 빈을 찾는다. (id가 userService인 것을 찾음)

### 트랜잭션 경계설정 코드 분리의 장점
1. 비지니스 로직을 담당하는 UserServiceImpl의 코드를 작성할 때는 트랜잭션과 같은 기술적 내용에 전혀 신경 쓰지 않아도 된다.
2. 비지니스 로직에 대한 테스트를 손쉽게 만들 수 있다.
# 6.2 고립된 단위 테스트
테스트 대상이 다른 오브젝트와 환경에 의존하고 있다면 작은 단위 테스트가 주는 장점을 얻기 힘들겠지만 테스트는 작은 단위로 하는 것이 좋다.

## 6.2.1 복잡한 의존관계 속 테스트
UserService 구현 클래스들이 동작하려면 세 가지 타입의 의존 오브젝트가 필요하다.
![img_4.png](img_4.png)
- UserDao 타입의 오브젝트를 통해 DB와 데이터를 주고 받아야 한다.
- MailSender를 구현한 오브젝트를 이용해 메일을 발송해야 한다.
- 트랜잭션 처리를 위해 PlatformTransactionManager와 커뮤니케이션이 필요하다.

UserService 라는 테스트 대상이 테스트 단위인 것처럼 보이지만 사실 그 뒤 의존관계에 따라 등장하는 오브젝트와 서비스, 환경 등이 모두 합쳐져서 테스트 대상이 되는 것이다.

그렇기 때문에 이런 경우의 테스트는 준비하기 힘들고, 환경이 조금이라도 달라지면 동일한 테스트 결과를 내지 못 할 수 있으며, 수행 속도도 느리다.

## 6.2.2 테스트 대상 오브젝트 고립시키기
### 테스트를 위한 UserServiceImpl 고립
![img_5.png](img_5.png)
UserServiceImpl 은 PlatformtransactionManager에 의존하지 않는다.

UserDao는 단지 테스트 대상의 코드가 정상적으로 수행되도록 도와주기만 하는 스텁이 아니라, 부가적인 검증 기능까지 가진 목 오브젝트로 만들었다.
**왜?** 고립된 환경에서 동작하는 upgradeLevels() 테스트 결과를 검증할 방법이 필요해서
DB에 결과가 반영되게 해서 확인하려고

### 고립된 단위 테스트 활용
> UserServiceTest에 upgradeLevels() 테스트를 적용해보자.

```java
@Test
public void upgradeLevels() throws Exception {
	// 1. DB 테스트 데이터 준비
    userDao.deleteAll();
    for (User user : users) {
        userDao.add(user);
    }

    //2. 메일 발송 여부 확인을 위해 목 오브젝트 DI
    MockMailSender mockMailSender = new MockMailSender();
    userServiceImpl.setMailSender(mockMailSender);
    
    // 3. 테스트 대상 실행
    userService.upgradeLevels();

    // 4. DB에 저장된 결과 확인
    checkLevelUpgraded(users.get(0), false);
    checkLevelUpgraded(users.get(1), true);
    checkLevelUpgraded(users.get(2), false);
    checkLevelUpgraded(users.get(3), true);
    checkLevelUpgraded(users.get(4), false);

// 5. 목 오브젝트를 이용한 결과 확인
    List<String> request = mockMailSender.getRequests();
    assertThat(request.size(), is(2));
    assertThat(request.get(0), is(users.get(1).getEmail()));
    assertThat(request.get(1), is(users.get(3).getEmail()));
}


private void checkLevelUpgraded(User user, boolean upgraded) {
    User userUpdate = userDao.get(user.getId());
    ...
}

```
`위 테스트의 단계 설명`
1. 테스트 실행 중 UserDao를 통해 가져올 테스트용 정보를 DB에 넣는다. UserDao는 결국 DB를 이용해 정보를 가져오기 때문에 최후의 의존 대상인 DB에 직접 정보를 넣어줘야 한다.
2. 메일 발송 여부를 확인하기 위해 MailSender 목 오브젝트를 DI 해준다.
3. 실제 테스트 대상인 userService의 메소드를 실행한다.
4. 결과가 DB에 반영됐는지 확인하기 위해 UserDao를 이용해 DB에서 데이터를 가져와 결과를 확인한다.
5.  목 오브젝트를 통해 UserService에 의한 메일 발송이 있었는지 확인한다.
### UserDao 목 오브젝트
목 오브젝트는 기본적으로 스텁과 같은 방식으로 테스트 대상을 통해 사용될 때 필요한 기능을 지원해줘야 한다.
MockUserDao는 UserDao 구현 클래스를 대신해야 하니 당연히 UserDao 인터페이스를 구현해야 한다.
인터페이스를 구현하려면 인터페이스 내의 모든 메소드를 만들어줘야 한다.
우리가 사용하는 건 getAll() 과 update() 뿐인데도 말이다.

```java
package springbook.user.dao;
 
import java.util.ArrayList;
import java.util.List;
 
import springbook.user.domain.User;
 
public class MockUserDao implements UserDao{
	
	// 레벨 업그레이드 후보 목록 User 오브젝트
	private List<User> users; 
	
	// 업그레이드 대상 오브젝트를 저장해둘 목록 
	private List<User> updated = new ArrayList<User>();
 
	public MockUserDao(List<User> users) {
		this.users = users;
	}
	
	public List<User> getUpdated(){
		return this.updated;
	}
	
	// 스텁 기능 제공
	@Override
	public List<User> getAll(){
		return this.users;
	}
	
	// 목 오브젝트 기능 제공 
	@Override
	public void update(User user) {
		updated.add(user);
	}
 
	
	// 사용하지 않는 메소드는 UnsupportedOperationException 예외를 던진다.
	@Override
	public void add(User user) {throw new UnsupportedOperationException(); }
 
	@Override
	public User get(String id) {throw new UnsupportedOperationException();}
 
	@Override
	public void deleteAll() {throw new UnsupportedOperationException();}
 
	@Override
	public int getCount() {throw new UnsupportedOperationException();}
	
}
```
사용하지 않을 메소드도 구현해줘야 한다면 UnsupportedOperationException을 던지도록 만드는 것이 좋다.

위의 코드에서 두 개의 User 타입 리스트를 정의한다.
- **생성자를 통해 전달받은 사용자 목록을 저장해뒀다가, getAll() 메소드가 호출되면 DB에서 가져온 것처럼 돌려주는 용도** (목 오브젝트를 사용하지 않을 땐 DB에 일일이 저장했다가 가져와야 된다. MockUserDao 는 미리 준비된 테스트용 리스트를 메모리에 갖고 있다가 돌려주면 된다.)
- update() 메소드를 실행하면서 넘겨준 업그레이드 대상 User 오브젝트를 저장해뒀다가 검증을 위해 돌려주기 위한 것이다.
```java
@Test
public void upgradeLevels() throws Exception {
	//고립된 테스트에서는 테스트 대상 오브젝트를 직접 생성하면된다.
	UserServiceImpl userServiceImpl = new UserServiceImpl();
	
	//목 오브젝트로 만든 UserDao를 직접 DI해준다.
	MockUserDao mockUserDao = new MockUserDao(this.users);
	userServiceImpl.setUserDao(mockUserDao);
	
	MockMailSender mockMailSender = new MockMailSender();
	userServiceImpl.setMailSender(mockMailSender);
	
	userServiceImpl.upgradeLevels();
	
	//MockUserDao로부터 업데이트된 결과를 가져온다. 
	List<User> updated = mockUserDao.getUpdated();
	assertEquals(updated.size(), 2);
	checkUserAndLevel(updated.get(0), "joytouch", Level.SILVER);
	checkUserAndLevel(updated.get(1), "madnite1", Level.GOLD);
	
	
	List<String> request = mockMailSender.getReuests();
	assertEquals(request.size(), 2);
	assertEquals(request.get(0), users.get(1).getEmail());
	assertEquals(request.get(1), users.get(3).getEmail());
}
 
// id와 level을 확인하는 메소드 
private void checkUserAndLevel(User updated, String expectedId, Level expectedLevel) {
	assertEquals(updated.getId(), expectedId);
	assertEquals(updated.getLevel(), expectedLevel);
}
```
### 테스트 수행 성능 향상
사용자 관리 로직을 검증하는 데 직접적으로 필요하지 않은 의존 오브젝트와 서비스를 모두 제거한다.
고립된 테스트를 하면 테스트가 다른 의존 대상에 영향을 받는 경우를 대비해 복잡하게 준비할 필요가 없을 뿐만 아니라, 테스트 수행 성능도 크게 향상된다.

## 6.2.3 단위 테스트와 통합 테스트
`단위 테스트`
테스트 대상 클래스를 목 오브젝트 등의 테스트 대역을 이용해 의존 오브젝트나 외부 리소스를 사용하지 않도록 고립시켜서 하는 테스트

`통합 테스트`
두 개 이상의, 성격이나 계층이 다른 오브젝트가 연동하도록 만들어 테스트하거나, 또는 외부 DB 나 파일, 서비스 등의 리소스가 참여하는 테스트

> Q. 단위 테스트와 통합 테스트 중에서 어떤 방법을 쓸지 어떻게 결정하나?
>> A. 가이드라인을 보자.

`가이드라인`
1. 항상 단위 테스트를 먼저 고려한다.
2. 하나의 클래스나 성격과 목적이 같은 긴밀한 클래스 몇 개를 모아서 외부와의 의존관계를 모두 차단하고 필요에 따라 스텁이나 목 오브젝트 등의 테스트 대역을 이용하도록 테스트를 만든다.
3. 외부 리소스를 사용해야만 가능한 테스트는 통합 테스트로 만든다.
4. 단위 테스트로 만들기 어려운 코드가 있다. 대표적으로 DAO이다.
5. DAO 테스트는 DB라는 외부 리소스를 사용하기 때문에 통합 테스트로 분류된다. 하지만 코드에서 보자면 하나의 기능 단위를 테스트하는 것이기도 하다.
6. 여러 개의 단위가 의존관계를 가지고 동작할 때를 위한 통합 테스트는 필요하다. 다만, 단위 테스트를 충분히 거쳤다면 통합 테스트의 부담은 상대적으로 줄어든다.
7. 단위 테스트를 만들기 너무 복합하다고 판단되는 코드는 처음부터 통합 테스트를 고려해본다.
8. 스프링 테스트 컨텍스트 프레임워크를 이용하는 테스트는 통합 테스트다.

전문 테스터나 고객에 의한 기능 테스트는 다른 관점에서 생각해야 한다.
위에는 개발자 테스트이다.

## 6.2.4 목 프레임워크
단위 테스트를 만들기 위해서 스텁이나 목 오브젝트의 사용이 필수적이다.

단위 테스트를 가장 우선시해야 되는 건 사실이지만 **번거롭다**.
테스트에서 사용하지 않는 인터페이스도 **모두** 일일이 **구현**해줘야 하며 테스트 메소드별로 다른 검증 기능이 필요하면, 같은 의존 인터페이스를 구현한 여러 개의 목  클래스를 선언해야 한다.

#### Mockito 프레임워크
목 오브젝트를 편리하게 작성하도록 도와주는 목 오브젝트 지원 프레임워크

목 클래스를 일일이 준비해둘 필요가 없다.
간단한 메소드 호출만으로 다이나믹하게 특정 인터페이스를 구현한 테스트용 목 오브젝트를 만들 수 있다.

```java
UserDao mockUserDao = mock(UserDao.class);
```
UserDao 인터페이스를 구현한 테스트용 목 오브젝트는 위와 같이 Mockito 의 스태틱 메소드를 한 번 호출해주면 만들어진다.

```java
when(mockUserDao.getAll()).thenReturn(this.users);
```
mockUserDao.getAll()을 호출했을 때 , users 리스트를 리턴해줘라
이런 선언이다.

```java
verify(mockUserDao, times(2)).update(any(User.class));
```
update() 호출이 있었는지를 검증하는 부분으로,
테스트를 진행하는 동안 mockUserDao의 update() 메소드가 두 번 호출됐는지 확인하는 코드이다.

> Mockito 목 오브젝트 사용 방법
>> 1. 인터페이스를 이용해 목 오브젝트를 만든다.
>> 2. 목 오브젝트가 리턴할 값이 있으면 이를 지정해준다. 메소드가 호출되면 예외를 강제로 던지게 만들 수 있다.
>> 3. 테스트 대상 오브젝트 DI 해서 목 오브젝트가 테스트 중에 사용되도록 만든다.
>> 4. 테스트 대상 오브젝트를 사용한 후에 목 오브젝트의 특정 메소드가 호출됐는지, 어떤 값을 가지고 몇 번 호출됐는지 검증한다.

`Mockito를 적용한 테스트 코드`
```java
import static org.mockito.Matchers.any;
import static org.mockito.Mockito.mock;
import static org.mockito.Mockito.times;
import static org.mockito.Mockito.verify;
import static org.mockito.Mockito.when;
 
//...
 
@Test
public void upgradeLevels() throws Exception {
	UserServiceImpl userServiceImpl = new UserServiceImpl();
	
	UserDao mockUserDao = mock(UserDao.class);
	when(mockUserDao.getAll()).thenReturn(this.users);
	userServiceImpl.setUserDao(mockUserDao);
	
	MailSender mockMailSender = mock(MailSender.class);
	userServiceImpl.setMailSender(mockMailSender);
	
	userServiceImpl.upgradeLevels();
	
	//레벨 검증 
	verify(mockUserDao, times(2)).update(any(User.class));
	verify(mockUserDao).update(users.get(1));
	assertEquals(users.get(1).getLevel(), Level.SILVER);
	verify(mockUserDao).update(users.get(3));
	assertEquals(users.get(3).getLevel(), Level.GOLD);
	
	ArgumentCaptor<SimpleMailMessage> mailMessageArg = 
			ArgumentCaptor.forClass(SimpleMailMessage.class);
	verify(mockMailSender, times(2)).send(mailMessageArg.capture());
	List<SimpleMailMessage> mailMessages = mailMessageArg.getAllValues();
	assertEquals(mailMessages.get(0).getTo()[0], users.get(1).getEmail());
	assertEquals(mailMessages.get(1).getTo()[0], users.get(3).getEmail());
}
```
# 6.3 다이나믹 프록시와 팩토리 빈
## 6.3.1 프록시와 프록시 패턴, 데코레이터 패턴
![img_6.png](img_6.png)
부가기능 외의 나머지 모든 기능은 원래 핵심기능을 가진 클래스로 위임해줘야 한다.
핵심기능은 부가기능을 가진 클래스의 존재 자체를 모른다. 따라서 부가기능이 핵심기능을 사용하는 구조가 된다.

문제는 이렇게 구성했더라도 클라이언트가 핵심기능을 가진 클래스를 직접 사용해 버리면 부가기능이 적용될 기회가 없다.

![img_7.png](img_7.png)

그래서 부가기능은 마치 자신이 핵심 기능을 가진 클래스인 것처럼 꾸며서, 클라이언트가 자신을 거쳐서 핵심기능을 사용하도록 만들어야 한다.
그러기 위해서는 클라이언트는 인터페이스를 통해서만 핵심기능을 사용하게 하고, 부가기능 자신도 같은 인터페이스를 구현한 뒤에 자신이 그 사이에 끼어들어야 한다.

>`프록시(proxy)`
>> 자신이 클라이언트가 사용하려고 하는 실제 대상인 것처럼 위장해서 클라이언트의 요청을 받아주는 것
>> 대리자, 대리인과 같은 역할은 한다고 생가하면 된다.

프록시를 통해 최종적으로 요청을 위임받아 처리하는 실제 오브젝트를 타깃(target) 또는 실체(real subject)라고 부른다.

![img_8.png](img_8.png)

프록시의 특징은 타깃과 같은 인터페이스를 구현했다는 것과 프록시가 타깃을 제어할 수 있는 위치에 있다는 것이다.

프록시는 사용 목적에 따라 두 가지로 구분 가능
-  클라이언트가 타깃에 접근하는 방법을 제어하기 위해서다.
-  타깃에 부가적인 기능을 부여해주기 위해서다.
   두 가지 모두 대리 오브젝트라는 개념의 프록시를 두고 사용한다는 점은 동일하지만, 목적에 따라서 디자인 패턴에서는 다른 패턴으로 구분한다.
### 데코레이션 패턴
**데코레이터 패턴은 타깃에 부가적인 기능을 런타임 시에 다이내믹하게 부여해주기 위해 프록시를 사용하는 패턴을 말한다.**
-  코드상에서는 어떤 방법과 순서로 프록시와 타깃이 연결되어 사용되는지 정해져 있지 않다는 뜻이다.
-  프록시가 꼭 한 개로 제한되지 않는다.
-  같은 인터페이스를 구현한 타겟과 여러 개의 프록시를 사용할 수 있다.

![img_9.png](img_9.png)

프록시로서 동작하는 각 데코레이터는 위임하는 대상에도 인터페이스로 접근하기 때문에 자신이 최종 타깃으로 위임하는지, 아니면 다음 단계의 <u>데코레이터 프록시로 위임하는지 알지 못한다</u>. 그래서 ***데코레이터의 다음 위임 대상은 인터페이스로 선언하고 생성자나 수정자 메소드를 통해 위임 대상을 외부에서 런타임 시에 주입받을 수 있도록 만들어야 한다***.

인터페이스를 통한 데코레이터 정의와 런타임 시의 다이내믹한 구성 방법은 스프링의 DI를 이용하면 편하다. 데코레이터 빈의 프로퍼티로 같은 인터페이스를 구현한 다른 데코레이터 또는 타깃 빈을 설정하면 된다.

> 데코레이터 패턴은 타깃의 코드를 손대지 않고, 클라이언트가 호출하는 방법도 변경하지 않은 채로 새로운 기능을 추가할 때 유용한 방법이다.

### 프록시 패턴
일반적으로 사용하는 프록시라는 용어와 디자인 패턴에서 말하는 프록시 패턴은 구분할 필요가 있다.
<mark>프록시</mark>
클라이언트와 사용 대상 사이에 대리 역할을 맡은 오브젝트를 두는 방법
<mark>프록시 패턴</mark>
프록시를 사용하는 방법 중에 타깃에 대한 접근 방법을 제어하려는 목적을 가진 경우 말한다.

프록시 패턴의 프록시는 타깃의 기능을 확장하거나 추가하지 않는다. 대신 클라이언트가 타깃에 접근하는 방식을 변경해준다.

프록시 패턴은 **타깃의 기능 자체에는 관여하지 않으면서 접근하는 방법을 제어해주는 프록시**를 **이용**하는 것이다.

![img_10.png](img_10.png)

구조적으로 보자면 프록시와 데코레이터는 유사하다.
다만 프록시는 <u>코드에서 자신이 만들거나 접근할 타깃 클래스 정보를 알고 있는 경우가 많다. </u>
생성을 지연하는 프록시라면 구체적인 생성 방법을 알아야 하기 때문에 타깃 클래스에 대한 직접적인 정보를 알아야 한다.
물론 프록시 패턴이라고 하더라도 인터페이스를 통해 위임하도록 만들 수도 있다. 인터페이스를 통해 다음 호출 대상으로 접근하게 되면 그 사이에 다른 프록시나 데코레이터가 계속 추가될 수 있기 때문이다.

앞으로는 타깃과 동일한 인터페이스를 구현하고 클라이언트와 타깃 사이에 존재하면서 기능의 부가 또는 접근 제어를 담당하는 오브젝트를 모두 프록시라고 부를 것이다.
### 다이내믹 프록시
목 오브젝트를 만드는 불편함을 목 프레임워크를 사용해 편리하게 바꿨던 것처럼 프록시도 일일이 모든 인터페이스를 구현해서 클래스를 새로 정의하지 않고도 편리하게 만들어서 사용할 방법은 없을까?

있다. 자바에는 java.lang.reflect 패키지 안에 프록시를 손쉽게 만들 수 있도록 지원해주는 클래스들이 있다. 일일이 프록시 클래스를 정의하지 않고도 몇 가지 API를 이용해 프록시처럼 동작하는 오브젝트를 다이내믹하게 생성하는 것이다.

`프록시의 구성과 프록시 작성의 문제점`
1. 타깃과 같은 메소드를 구현하고 있다가 메소드가 호출되면 타깃 오브젝트로 위임한다.
2. 지정된 요청에 대해 부가기능을 수행한다.
```java
public class UserServiceTx implements UserService {
	UserService userService; // 타깃 오브젝트
	PlatformTransactionManager transactionManager; // 부가기능
 
	...
 
	public void add(User user) {
		this.userService.add(user); // 메소드 구현과 위임임
	}
 
	public void upgradeLevels() { //메소드 구현
		TransactionStatus status = this.transactionManager
				.getTransaction(new DefaultTransactionDefinition()); //부가기능 수행
		try {
 
			userService.upgradeLevels(); //위임
 
			this.transactionManager.commit(status);//부가기능 수행
		} catch (RuntimeException e) {
			this.transactionManager.rollback(status);
			throw e;
		}
	}
}
```
`프록시를 만들기 번거로운 이유`
1. 타깃의 인터페이스를 구현하고 위임하는 코드를 작성하기가 번거롭다는 점이다. 부가기능이 필요 없는 메소드도 구현해서 타깃으로 위임하는 코드를 일일이 만들어줘야 한다. 또, 타깃 인터페이스의 메소드가 추가되거나 변경될 때마다 함께 수정해줘야 한다는 부담도 있다.
2. 부가기능 코드가 중복될 가능성이 많다는 점이다.

### 리플렉션
다이내믹 프록시는 리플렉션 기능을 이용해서 프록시를 만들어준다.
리플렉션은 자바의 코드 자체를 추상화해서 접근하도록 만든 것이다.

자바의 모든 클래스는 그 클래스 자체의 구성정보를 담은 Class 타입의 오브젝트를 하나씩 갖고 있다. ‘클래스이름.class’라고 하거나 오브젝트의 getClass() 메소드를 호출하면 클래스 정보를 담은 Class 타입의 오브젝트를 가져올 수 있다. 클래스 오브젝트를 이용하면 크래스 코드에 대한 메타정보를 가져오거나 오브젝트를 조작할 수 있다.
`리플렉션 학습 테스트`
```java
package springbook.learningtest.jdk;  
...  
public class ReflectionTest {  
	@Test  
    public void invokeMethod() throws Exception() {  
        String name = "Spring";  
          
        // length()  
        assertThat(name.length(), is(6));  
          
		Method lengthMethod = String.class.getMethod("length");  
        assertThat((Integer)lengthMethod.invoke(name), is(6));  
          
        // charAt()  
        assertThat(name.charAt(0), is('S'));  
          
        Method charAtMethod = String.class.getMethod("charAt", int.class);  
        assertThat((Character)charAtMethod.invoke(name, 0), is('S'));  
    }  
}
```
### 프록시 클래스
```java
public interface Hello {
    String sayHello(String name);
    String sayHi(String name);
    String sayThankYou(String name);
}
```

```java
public class HelloTarget implements Hello {
    @Override
    public String sayHello(String name) {
        return "Hello " + name;
    }

    @Override
    public String sayHi(String name) {
        return "Hi " + name;
    }

    @Override
    public String sayThankYou(String name) {
        return "Thank You " + name;
    }
}
```

```java
public class HelloUppercase implements Hello {
    private Hello hello;

    public HelloUppercase(Hello hello) {
        this.hello = hello;
    }

    @Override
    public String sayHello(String name) {
        return hello.sayHello(name).toUpperCase();
    }

    @Override
    public String sayHi(String name) {
        return hello.sayHi(name).toUpperCase();
    }

    @Override
    public String sayThankYou(String name) {
        return hello.sayThankYou(name).toUpperCase();
    }
}
```

```java
@Test
public void manualProxy() {
    Hello hello = new HelloUppercase(new HelloTarget());
    Assertions.assertEquals("HELLO TOBY", hello.sayHello("Toby"));
    Assertions.assertEquals("HI TOBY", hello.sayHi("Toby"));
    Assertions.assertEquals("THANK YOU TOBY", hello.sayThankYou("Toby"));
}
```
이 프록시는 프록시 적용의 일반적 문제점 두 가지를 갖고 있다.
- 인터페이스의 모든 메소드를 구현해 위임하도록 코드를 만들어야 한다.
- 부가기능인 리턴 값을 대문자로 바꾸는 기능이 모든 메소드에 중복돼 나타난다.
### 다이내믹 프록시 적용
![img_11.png](img_11.png)
**다이내믹 프록시는 프록시 팩토리에 의해 런타임 시 다이내믹하게 만들어지는 오브젝트다. 다이내믹 프록시 오브젝트는 타깃의 인터페이스와 같은 타입으로 만들어진다.**

클라이언트는 다이내믹 프록시 오브젝트를 타깃 인터페이스를 통해 사용할 수 있다. 이 덕분에 프록시를 만들 때 인터페이스를 모두 구현해가면서 클래스를 정의하는 수고를 덜 수 있다. 프록시 팩토리에게 인터페이스 정보만 제공해주면 해당 인터페이스를 구현한 클래스의 오브젝트를 자동으로 만들어주기 때문이다.

다이내믹 프록시가 인터페이스 구현 클래스의 오브젝트는 만들어주지만, 프록시로서 필요한 부가기능 제공 코드는 직접 생성해야 한다.
부가기능은 프록시 오브젝트와 독립적으로 InvocationHandler를 구현한 오브젝트에 담는다. InvocationHandler 인터페이스는 다음과 같은 메소드 한 개만 가진 간단한 인터페이스다.

```java
public Object invoke(Object proxy, Method method, Object[] args)
```
invoke() 메소드는 리플렉션의 MEthod 인터페이스를 파라미터로 받는다. 메소드를 호출할 때 전달되는 파라미터도 args로 받는다.
다이내믹 프록시 오브젝트는 클라이언트의 모든 요청을 리플렉션 정보로 변환해서 InvocationHandler 구현 오브젝트의 invoke() 메소드로 넘기는 것이다.
타깃 인터페이스의 모든 메소드 요청이 하나의 메소드로 집중되기 때문에 중복되는 기능을 효과적으로 제공할 수 있다.

다이내믹 프록시로부터 요청을 전달받으려면 InvocationHandler를 구현해야 한다. 메소드는 invoke() 하나뿐이다. 다이내믹 프록시가 클라이언트로부터 받는 모든 요청은 incoke() 메소드로 전달된다. 다이내믹 프록시를 통해 요청이 전달되면 리플렉션 API를 이용해 타깃 오브젝트의 메소드를 호출한다. 타깃 오브젝트는 생성자를 통해 미리 전달받아 둔다.

![img_12.png](img_12.png)
### 다이내믹 프록시의 확장
```java
public class UppercaseHandler implements InvocationHandler {  
    // 어떤 종류의 인터페이스를 구현한 타깃에도 적용 가능하도록 Object 타입으로 수정  
    Object target;  
    private UppercaseHandler(Object target) {  
        this.target = target;  
    }  
      
    public Object invoke(Object proxy, Method method, Object[] args) thorws Throwable {  
    // 호출한 메소드의 리턴 타입이 String인 경우만 대문자 변경 기능을 적용하도록 수정.
        Object ret = method.invoke(target, args);  
        if (Ret instanceof String) {  
            return ((String)ret).toUpperCase();  
        } else {  
            return ret;  
        }  
    }  
}
```
## 6.3.3 다이내믹 프록시를 이용한 트랜잭션 부가기능
> UserServiceTx를 다이내믹 프록시 방식으로 변경해보자.

트랜잭션 부가기능을 제공하는 다이내믹 프록시를 만들어 적용하는 방법이 효율적이다.
다이내믹 프록시와 연동해서 트랜잭션 기능을 부가해주는 InvocationHandler는 한개만 정의해도 충분하기 때문이다.
```java
public class TransactionHandler implements InvocationHandler {  
    private Object target;  
    private PlatformTransactionManager transactionManager;  
    private String pattern;  
      
    public void setTarget(Object target) {  
        this.target = target;  
    }  
      
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
      
    private Object invokeTransaction(Method method, Object[] args) throws Throwable {  
        TransactionStatus = this.transactionManager.getTransaction(new DefaultTransactionDefinition());  
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
### TransactionHandler와 다이내믹 프록시를 이용하는 테스트
> TransactionHandler가 UserServiceTx를 대신 할 수 있는지 확인을 위해 UserServiceTest에 적용해보자.

```java
@Test  
public void upgradeAllOrNothing() throws Exception {  
    ...  
    TransactionHandler txHandler = new TransactionHandler();  
    txHandler.setTarget(testUserService);  
    txHandler.setTransactionManager(transactionManager);  
    txHandler.setPattern("upgradeLevels");  
    UserService txUSerService = (UserService)Proxy.newProxyInstance(getClass().getClassLoader(), new Class[] {UserService.class }, txHandler);  
    ...  
}
```
## 6.3.4 다이내믹 프록시를 위한 팩토리 빈
TransactionHandler와 다이내믹 프록시를 스프링의 DI를 통해 사용할 수 있도록 만들어야 할 차례다.
 스프링의 빈은 기본적으로 클래스 이름과 프로퍼티로 정의된다. 
 스프링은 지정된 클래스 이름을 가지고 리플렉션을 이용해서 해당 클래스의 오브젝트를 만든다. 클래스의 이름을 갖고 있다면 다음과 같은 방법으로 새로운 오브젝트를 생성할 수 있다. Class의 newInstance() 메소드는 해당 클래스의 파라미터가 없는 생성자를 호출하고, 그 결과 생성되는 오브젝트를 돌려주는 리플렉션 API다.
```java
Date now = (Date) Class.forName("java.util.Date").newInstance();
```
스프링은 내부적으로 리플렉션 API를 이용해서 빈 정의에 나오는 클래스 이름을 가지고 빈 오브젝트를 생성한다. 문제는 다이내믹 프록시 오브젝트는 이런 식으로 프록시 오브젝트가 생성되지 않는다는 점이다.

### 팩토리 빈
스프링을 대신해서 오브젝트의 생성 로직을 담당하도록 만들어진 특별한 빈을 말한다.

스프링을 대신해서 오브젝트의 생성로직을 담당하도록 만들어진 특별한 빈을 말한다.

```java
package org.springframework.beans.factory;  
  
public interface FactoryBean<T> {  
    T getObject() throws Exception; // 빈 오브젝트를 생성해서 돌려준다.  
    Class<? extends T> getOBjectType(); // 생성되는 오브젝트의 타입을 알려준다.  
    boolean isSingleton(); // getObject()가 돌려주는 오브젝트가 항상 같은 싱글톤 오브젝트인지 알려준다.  
}
```
### 다이내믹 프록시를 만들어주는 팩토리 빈
Proxy의 newPRoxyInstance() 메소드를 통해서만 생성이 가능한 다이내믹 프록시 오브젝트는 일반적인 방법으로는 스프링의 빈을 등록할 수 없다. 대신 팩토리 빈을 사용하면 다이내믹 프록시 오브젝트를 스프링의 빈으로 만들어줄 수가 있다. 팩토리 빈의 getObject() 메소드에 다이내믹 프록시 오브젝트를 만들어주는 코드를 넣으면 되게 때문이다.

![img_13.png](img_13.png)

스프링 빈에는 팩토리 빈과 UserServiceImpl만 빈으로 등록한다.
팩토리 빈은 다이내믹 프록시가 위임할 타깃 오브젝트인 UserServiceImpl에 대한 레퍼런스를 프로퍼티를 통해 DI 받아둬야 한다.
다이내믹 프록시와 함께 생성할 TransactionHandler에게 타깃 오브젝트를 전달해줘야 하기 때문이다.
그 외에도 다이내믹 프록시나 TransactionHandler를 만들 때 필요한 정보는 팩토리 빈의 프로퍼티로 설정해뒀다가 다이내믹 프록시를 만들면서 전달해줘야 한다.

## 6.3.5 프록시 팩토리 빈 방식의 장점과 한계
### 프록시 팩토리 빈 방식의 장점
다이내믹 프록시를 이용하면 타깃 인터페이스를 구현하는클래스를 일일이 만드는 번거로움을 제거할 수 있다.
하나의 핸들러 메소드를 구현하는 것만으로도 수많은 메소드에 부가기능을 부여해줄 수 있으니 부가기능 코드의 중복 문제도 사라진다. 다이내믹 프록시에 팩토리 빈을 이용한 DI까지 더해주면 번거로운 다이내믹 프록시 생성 코드도 제거할 수 있다. DI 설정만으로 다양한 타깃 오브젝트에 적용도 가능하다.

**스프링 DI는 매우 중요한 역할을 한다!!!**

### 프록시 팩토리 빈의 한계
프록시를 통해 타깃에 부가기능을 제공하는 것은 메소드 단위로 일어나는 일이다.
하나의 클래스 안에 존재하는 여러 개의 메소드에 부가기능을 한 번에 제공하는 건 어렵지 않게 가능했다.
하지만 한 번에 여러 개의 크래스에 공통적인 부가기능을 제공하는 일은 지금까지 살펴본 방법으로는 불가능하다.

하나의 타깃에 여러 개의 부가기능을 적용하려고 할 때도 문제다. 프록시 팩토리 빈 설정이 부가기능의 개수만큼 따라 붙어야 한다.**