# 6장. AOP - 1

배울 것 
> AOP의 등장 배경
<br>
> 스프링이 AOP를 도입한 이유
<br>
> AOP를 적용하여 얻는 장점

## 6.1 트랜잭션 코드의 분리 

```java
public void upgradeLevels() throws Exception {
    TransactionStatus status = this.transactionManager 
        .getTransaction(new DefaultTransactionDefinition());
        // 트랜잭션 경계
    try {
        List<User> users = userDao.getAll();
        for (User user : users) {
            if (canUpgradeLevel(user)) {
                upgradeLevel(user);
            }
        }
        // 트랜잭션 경계
        this.transactionManager.commit(status);
    } catch (Exception e) {
        this.transactionManager.rollback(status);
        throw e;
    }
}
```

- 비즈니스 로직만 들어가야할 메소드 안에 트랜잭션 코드의 비중이 너무 크다.
- 비즈니스 로직 전후로 설정되는 트랜잭션 경계설정 코드는 서로 구분되어 있으며 주고받는 정보가 없다. 

```java
public void upgradeLevels() throws Exception {
    TransactionStatus status = this.transactionManager
        .getTransaction(new DefaultTransactionDefinition());
    try {
        upgradeLevelsInternal();
        this.transactionManager.commit(status);
    } catch (Exception e) {
        this.transactionManager.rollback(status);
        throw e;
    }
}

// 분리된 비즈니스 로직 코드는 이전과 동일. 
private void upgradeLevelsInternal() {
    List<User> users = userDao.getAll();
    for (User user : users) {
        if (canUpgradeLevel(user)) {
            upgradeLevel(user);
        }
    }
}
```

- 트랜잭션 코드가 UserService에 남아있는 문제가 있음
- 해결을 위해 DI 적용이 필요

- UserService를 인터페이스로 변경하고 구현 클래스를 주입하는 방식으로 개선
- 이를 통해 두 개의 구현 클래스를 동시에 사용 가능:
  1. UserServiceImpl: 순수 비즈니스 로직 담당
  2. UserServiceTx: 트랜잭션 처리 담당, UserServiceImpl에 위임

- 클라이언트는 트랜잭션이 적용된 비즈니스 로직을 사용하게 됨

```java
public interface UserService {
    void add(User user);
    void upgradeLevels();
}
```

```java
package springbook.user.service;
...

public class UserServiceImpl implements UserService {
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

- 트랜잭션, 스프링, 서버 환경 및 기술에 대한 코드 없이 User라는 도메인 정보를 가진 비즈니스 로직에만 충실한 코드. 

```java
public class UserServiceTx implements UserService {
    UserService userService;
    PlatformTransactionManager transactionManager; // 추상화된 트랜잭션 구현 오브젝트를 DI 받을 프로퍼티 

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
- UserServiceTx는 UserService 인터페이스를 구현하고, 같은 인터페이스를 구현한 다른 오브젝트에 작업을 위임하는 프록시 역할을 한다.


```xml
<!-- 리스트 6-7 트랜잭션 오브젝트가 추가된 설정파일 -->
<bean id="userService" class="springbook.user.service.UserServiceTx">
    <property name="transactionManager" ref="transactionManager" />
    <property name="userService" ref="userServiceImpl" />
</bean>

<bean id="userServiceImpl" class="springbook.user.service.UserServiceImpl">
    <property name="userDao" ref="userDao" />
    <property name="mailSender" ref="mailSender" />
</bean>
```
- 이제 클라이언트는 UserServiceTx 빈을 호출해서 사용해야 한다. 

```java

```
- @Autowired는 타입이 일치하는 빈을 찾는데, UserService 타입의 빈이 두 개(UserServiceTx, UserServiceImpl)라서 필드 이름으로 빈을 찾아 userService인 UserServiceTx가 주입된다.
- 테스트를 위해 UserServiceImpl 빈도 필요하므로 목 오브젝트와 수동 DI를 사용한다.

장점:
1. UserServiceImpl은 비즈니스 로직에만 집중할 수 있고 트랜잭션 등 기술적인 내용은 분리된다.
2. 비즈니스 로직 테스트가 용이하다.

## 6.2 고립된 단위 테스트 

### 작은 단위 테스트가 좋은 이유
1. 테스트가 실패했을 때 그 원인을 찾기 쉽다.
2. 테스트의 의도나 내용이 분명해진다.
3. 코드 규모가 커짐에 따라 작은 단위 테스트로 검증한 부분은 제외하고 디버깅 할 수 있다. 

### 작은 단위 테스트를 하기 어려운 이유
- 테스트 대상이 다른 오브젝트와 환경에 의존하고 있는 경우
    - 작은 단위의 테스트가 주는 장점을 얻기 힘들다. 


### UserServiceTest의 테스트 대상
  - UserService의 사용자 정보 관리 비즈니스 로직이 주 검증 대상
  - 의존관계에 따른 오브젝트, 서비스, 환경 등도 모두 테스트 대상이 됨

### 테스트 대상 오브젝트 고립시키기
  - MailSender처럼 테스트를 위한 대역 사용
  - DummyMailSender와 같은 테스트 스텁 적용
  - MockMailSender 같은 테스트 검증용 목 오브젝트 활용

### UserDao를 이용한 테스트 검증
  - 단순 스텁이 아닌 부가적인 검증 기능을 가진 목 오브젝트로 구현
  - 고립된 UserServiceImpl은 DB 등에 결과가 남지 않아 직접적인 결과 확인 불가
  - UserServiceImpl과 UserDao 간의 요청 내용을 확인하는 방식으로 검증

```java
@Test
public void upgradeLevels() throws Exception {
    // DB 테스트 데이터 준비
    userDao.deleteAll();
    for (User user : users) userDao.add(user);

    // 메일 발송 여부 확인을 위해 목 오브젝트 DI
    MockMailSender mockMailSender = new MockMailSender();
    userServiceImpl.setMailSender(mockMailSender);

    // 테스트 대상 실행
    userService.upgradeLevels();

    // DB에 저장된 결과 확인
    checkLevelUpgraded(users.get(0), false);
    checkLevelUpgraded(users.get(1), true);
    checkLevelUpgraded(users.get(2), false);
    checkLevelUpgraded(users.get(3), true);
    checkLevelUpgraded(users.get(4), false);

    // 목 오브젝트를 이용한 결과 확인
    List<String> request = mockMailSender.getRequests();
    assertThat(request.size(), is(2));
    assertThat(request.get(0), is(users.get(1).getEmail()));
    assertThat(request.get(1), is(users.get(3).getEmail()));
}

// 사용자 레벨 업그레이드 여부를 검증하는 헬퍼 메서드
private void checkLevelUpgraded(User user, boolean upgraded) {
    User userUpdate = userDao.get(user.getId());
    assertEquals(upgraded, userUpdate.getLevel().isUpgradedFrom(user.getLevel()));
}
```

```markdown
1. 테스트 실행 중에 UserDao를 통해 가져올 테스트용 정보를 DB에 넣는다. UserDao는 결국 DB를 이용해 정보를 가져오기 때문에 최후의 의존 대상인 DB에 직접 정보를 넣어줘야 한다.  
2. 메일 발송 여부를 확인하기 위해 MailSender 목 오브젝트를 DI 해준다. 
3. 실제 테스트 대상인 userService의 메소드를 실행한다.
4. 결과가 DB에 반영됐는지 확인하기 위해서 UserDao를 이용해 DB에서 데이터를 가져와 결과를 확인한다. 
5. 목 오브젝트를 통해 UserService에 의한 메일 발송이 있었는지를 확인하면 된다.
```

- UserDao와 같은 역할을 하면서, UserServiceImpl과 주고 받은 정보를 저장해뒀다가 테스트 검증에 사용할 수 있는 목 오브젝트를 만든다. 
- 현재 UserServiceImpl은 public void upgradeLevels()와 protected void upgradeLevel(User user)에서 UserDao를 사용한다. 

```java
public void upgradeLevels() {
    List<User> users = userDao.getAll(); // 테스트용 UserDao는 DB에서 읽어온 것처럼 미리 준비된 사용자 목록을 제공해야 한다 
    for (User user : users) {
        if (canUpgradeLevel(user)) { 
            upgradeLevel(user); 
        }
    }
}

protected void upgradeLevel(User user) {
    user.upgradeLevel(); 
    userDao.update(user); // 테스트용 UserDao는 업그레이드를 통해 레벨이 변경된 사용자는 DB에 반영된 것처럼 보이게 해야 한다. 
    sendUpgradeEMail(user); 
}
```
- 그래서 getAll()에 대해서는 스텁으로서, update()에 대해서는 목 오브젝트로서 동작하는 UserDao 타입의 테스트 대역이 필요하다.

```java
// UserDao 인터페이스를 구현한 MockUserDao 클래스
static class MockUserDao implements UserDao {
    private List<User> users;  // 레벨 업그레이드 후보 User 오브젝트 목록
    private List<User> updated = new ArrayList<>();  // 업그레이드 대상 오브젝트를 저장해둘 목록

    private MockUserDao(List<User> users) {
        this.users = users;
    }

    public List<User> getUpdated() {
        return this.updated;
    }

    public List<User> getAll() {  // 스텁 기능 제공
        return this.users;
    }

    public void update(User user) {  // 목 오브젝트 기능 제공
        updated.add(user);
    }

    // 테스트에 사용되지 않는 메소드
    public void add(User user) { throw new UnsupportedOperationException(); }
    public void deleteAll() { throw new UnsupportedOperationException(); }
    public User get(String id) { throw new UnsupportedOperationException(); }
    public int getCount() { throw new UnsupportedOperationException(); }
}
```
- 인터페이스를 구현한다. 사용하지 않을 메소드에 대해서는 UnsupportedOperationException을 던져서 예외가 발생시킨다. 

```java
@Test
public void upgradeLevels() throws Exception {
    UserServiceImpl userServiceImpl = new UserServiceImpl();  // 고립된 테스트에서는 테스트 대상 오브젝트를 직접 생성하면 된다.

    MockUserDao mockUserDao = new MockUserDao(this.users);  // 목 오브젝트로 만든 UserDao를 직접 DI 해준다.
    userServiceImpl.setUserDao(mockUserDao);

    MockMailSender mockMailSender = new MockMailSender();
    userServiceImpl.setMailSender(mockMailSender);

    userServiceImpl.upgradeLevels();  // 테스트 대상 실행

    List<User> updated = mockUserDao.getUpdated();  // MockUserDao로부터 업데이트 결과를 가져온다.
    assertThat(updated.size(), is(2));  // 업데이트 횟수와 정보를 확인한다.
    checkUserAndLevel(updated.get(0), "joytouch", Level.SILVER);
    checkUserAndLevel(updated.get(1), "madnite1", Level.GOLD);

    List<String> request = mockMailSender.getRequests();
    assertThat(request.size(), is(2));
    assertThat(request.get(0), is(users.get(1).getEmail()));
    assertThat(request.get(1), is(users.get(3).getEmail()));
}

// id와 level을 확인하는 간단한 헬퍼 메소드
private void checkUserAndLevel(User updated, String expectedId, Level expectedLevel) {
    assertThat(updated.getId(), is(expectedId));
    assertThat(updated.getLevel(), is(expectedLevel));
}
```
- 이제는 완전이 고립된 테스트 대상을 사용하기 때문에 스프링 컨테이너에서 빈을 가져올 필요가 없다. 
- 테스트할 대상이 담긴 클래스인 UserServiceImpl 오브젝트를 직접 생성한다. 
- mockMailSender는 mockUserDao를 받으니까 미리 준비해둔 사용자 목록을 받는다. 
- update()는 변경을 시도한 것만 확인하면 된다. 

테스트 성능 향상
- 고립된 테스트를 하면 테스트가 다른 의존 대상(의존 오브젝트, DB 등)의 영항을 받지 않아 테스트 성능이 향상된다. 
- 물론 목 오브젝트 작성이라는 비용이 들긴 한다. 

단위 테스트
- 단위는 정하기 나름이다. 중요한 것은 하나의 단위에 초점을 맞춘 테스트를 작성하는 것이다. 
- 테스트 대상 클래스를 목 오브젝트 등의 테스트 대역을 이용해 의존 오브젝트나 외부의 리소스를 사용하지 않도록 고립시켜서 테스트하는 것.

통합 테스트
- 두 개 이상의, 성격이나 계층이 다른 오브젝트가 연동하도록 만들어 테스트하거나, 또는 외부의 DB나 파일, 서비스 등의 리소스가 참여하는 테스트
- 스프링의 테스트 컨텍스트 프레임워크를 이용해 컨텍스트에서 생성되고 DI된 오브젝트를 테스트하는 것도 통합 테스트다. 

테스트 가이드라인 
- 단위 테스트를 우선적으로 고려
  - 외부 의존관계를 차단하고 테스트 대역 활용
  - 작성이 간단하고 실행 속도가 빠름
  - 외부 영향을 받지 않아 효과적

- 외부 리소스가 필요한 경우 통합 테스트로 작성
  - DAO는 DB 연동이 필요하므로 통합 테스트가 효과적
  - DB 테스트는 테스트 데이터 준비 등 부가 작업 필요
  - DAO 검증 후에는 목 오브젝트로 대체 가능

- 단위 테스트와 통합 테스트의 균형
  - 충분한 단위 테스트는 통합 테스트의 부담을 줄임
  - 복잡한 코드는 처음부터 통합 테스트 고려
  - 가능한 많은 부분을 단위 테스트로 검증

- 스프링 테스트 컨텍스트 프레임워크
  - 통합 테스트로 분류
  - 스프링 설정 자체가 테스트 대상일 때 사용
  - 더 추상적인 레벨의 테스트가 필요할 때 활용

### 개발자 테스트와 코드 품질의 관계

- 단위/통합 테스트 모두 개발자가 직접 작성하는 개발자 테스트임
  - 기능 테스트와는 다른 관점 필요

- 테스트 작성 시점의 중요성
  - 코드 작성 직후 빠른 테스트 작성이 효과적
  - TDD는 즉각적인 테스트가 가능한 장점
  - 시간이 지난 후 테스트 작성은 이해도 저하로 어려움

- 테스트와 코드 품질의 선순환
  - 테스트하기 좋은 코드는 좋은 설계일 가능성 높음
  - DI와 적절한 분리가 된 코드는 테스트 용이
  - 테스트 작성이 쉬운 코드가 유지보수성도 높음

- 스프링과 테스트의 시너지
  - 스프링이 권장하는 깔끔한 코드는 테스트 작성 용이
  - 좋은 테스트는 리팩토링 용이성 증가
  - 테스트 부재는 코드 품질 저하의 악순환 초래

목 프레임워크

- 단위 테스트를 만들기 위해서는 스텁이나 목 오브젝트 사용이 필수적
- 하지만 테스트에 사용하지 않을 메소드도 일일이 구현하고, 검증 기능이 필요하면 메소드 호출 내용을 저장해뒀다가 다시 불러와 검증해야한다. 
- 테스트 메소드별로 다른 검증 기능이 필요하면 같은 인터페이스에 대해 여러개의 목 클래스를 선언해야 한다. 

### Mockito 프레임워크

- 목 오브젝트 작성을 도와주는 지원 프레임워크
- 목 클래스를 일일이 작성하는게 아니라 메소드 호출만으로 다이내믹하게 특정 인터페이스를 구현한 테스트용 오브젝트를 만들 수 있다. 

Mockito 목 오브젝트 사용 방법
- 인터페이스를 이용해 목 오브젝트를 만든다.	
- 목 오브젝트가 리턴할 값이 있으면 이를 지정해준다. 메소드가 호출되면 예외를 강제로 던지게 만들 수도 있다.	
- 테스트 대상 오브젝트에 DI 해서 목 오브젝트가 테스트 중에 사용되도록 만든다.	
- 테스트 대상 오브젝트를 사용한 후에 목 오브젝트의 특정 메소드가 호출됐는지, 어떤 값을 가지고 몇 번 호출됐는지를 검증한다.

```java
@Test
public void mockUpgradeLevels() throws Exception {
    UserServiceImpl userServiceImpl = new UserServiceImpl();

    // 다이나믹한 목 오브젝트 생성과 메소드의 리턴 값 설정, 그리고 DI까지 세 줄이면 충분하다.
    UserDao mockUserDao = mock(UserDao.class);
    when(mockUserDao.getAll()).thenReturn(this.users);
    userServiceImpl.setUserDao(mockUserDao);

    // 리턴 값이 없는 메소드를 가진 목 오브젝트는 더욱 간단하게 만들 수 있다.
    MailSender mockMailSender = mock(MailSender.class);
    userServiceImpl.setMailSender(mockMailSender);

    userServiceImpl.upgradeLevels();

    // 목 오브젝트가 제공하는 검증 기능을 통해서 어떤 메소드가 몇 번 호출됐는지, 파라미터는 무엇인지 확인할 수 있다.
    verify(mockUserDao, times(2)).update(any(User.class));
    verify(mockUserDao).update(users.get(1));
    assertThat(users.get(1).getLevel(), is(Level.SILVER));
    verify(mockUserDao).update(users.get(3));
    assertThat(users.get(3).getLevel(), is(Level.GOLD));

    // 파라미터를 정밀하게 검사하기 위해 캡처할 수도 있다.
    ArgumentCaptor<SimpleMailMessage> mailMessageArg = ArgumentCaptor.forClass(SimpleMailMessage.class);
    verify(mockMailSender, times(2)).send(mailMessageArg.capture());
    List<SimpleMailMessage> mailMessages = mailMessageArg.getAllValues();
    assertThat(mailMessages.get(0).getTo()[0], is(users.get(1).getEmail()));
    assertThat(mailMessages.get(1).getTo()[0], is(users.get(3).getEmail()));
}
```
- ArgumentCaptor를 사용해 실제 MailSender 목 오브젝트에 전달된 파라미터를 가져와 내용을 검증했다. 이는 파라미터를 직접 비교하는게 아니라 파라미터 내부의 정보를 확인해야 하는 경우에 유용하다. 

## 6.3 다이내믹 프록시와 팩토리 빈

- 현재는 부가기능이 핵심기능을 사용하는 구조이다. 문제는 이렇게 구성했더라도 클라이언트가 핵심 기능을 가진 클래스를 직접 사용해버리면 부가기능이 적용될 기회가 없다. 
- 그래서 부가기능은 자신이 핵심기능을 가진 클래스인 것처럼 꾸며서, 클라이언트가 자신을 거쳐서 핵심기능을 사용하도록 만든다. 클라이언트는 인터페이스만 보고 사용하기 때문에 핵심기능을 가진 클래스를 사용하는 것으로 생각하지만, 사실은 부가기능을 통해 핵심기능을 이용하게 된다. 

프록시 
- 자신이 클라이언트가 사용하려는 실제 대상인 것처럼 위장하여 클라이언트의 요청을 받아주는 것을 대리자, 대리인과 같은 역할을 한다고 해서 프록시proxy라고 부른다. 

타깃target, 실체real object
- 그리고 프록시를 통해 최종적으로 요청을 위임받아 처리하는 실제 오브젝트 

프록시 특징

- 프록시는 타깃과 같은 인터페이스를 구현한다. 
- 프록시는 타깃을 제어할 수 있는 위치에 있다. 

프록시 사용 목적
1. 클라이언트가 타깃에 접근하는 방법 제어
2. 타깃에 부가적인 기능을 부여

> 두 목적은 다른 디자인 패턴으로 구현된다. 

데코레이터 패턴

- 부가적인 기능을 런타임 시 다이내믹하게 부여해주기 위해 프록시를 사용함.
- 포장지처럼 프록시는 꼭 한 개로 제한되지 않는다. 여러 개의 프록시가 순서를 정해서 단계적으로 위임하는 구조로 만들 수 있다. 
- 타깃의 코드에 손대지 않고, 클라이언트가 호출하는 방법도 변경하지 않은 채로 새로운 기능을 추가할 때 유용.

다이나믹함의 의미
- 컴파일 시점을 말한다. 코드상에는 어떤 방법과 순서로 프록시와 타깃이 연결되고 사용될지 정해져있지 않다. 

데코레이터 패턴 예시

자바 IO 패키지의 InputStream과 OutputStream 구현 클래스. 
InputStream이라는 인터페이스를 구현한 타깃인 FileInputStream에 버퍼 읽기 기능을 제공해주는 BufferedInputStream이라는 데코레이터를 적용한 예다.
```java
InputStream is = new BufferedInputStream(new FileInputStream("a.txt"));
```

UserServiceImpl이라는 타깃에 트랜잭션 부가기능을 제공해주는 UserServiceTx를 추가한 것도 데코레이터 패턴을 적용한 것으로 볼 수 있다. 
스프링 DI를 이용해 인터페이스를 통한 데코레이터를 정의하고 런타임 시에 다이내믹하게 타깃을 설정한다. 

프록시패턴

- 타깃에 대한 접근 방법을 제어하려는 목적을 갖고 있다. 
- 타깃 오브젝트를 생성하기 복잡하거나 당장 필요하지 않은 경우, 타깃 오브젝트에 대한 레퍼런스는 미리 필요하지만 실제 타깃 오브젝트는 필요한 시점에 생성해도 된다. 이때 실제 타깃 오브젝트를 생성하지 않고 대신 프록시를 클라이언트에 넘겨준다. 
- 이후 프록시의 메소드를 통해 타깃을 사용하려 하면 그때 프록시가 타깃 오브젝트를 생성하고 요청을 위임하는 식이다. 

사용 예시 
- 레퍼런스를 갖고 있지만 끝까지 사용하지 않거나, 많은 작업이 진행된 후에 사용되는 오브젝트인 경우. 이렇게 프록시를 통해 생성을 최대한 늦춰 자원을 절약할 수 있다. 
- 원격 오브젝트를 이용하는 경우. 어떠한 리모팅 기술을 이용해 다른 서버에 존재하는 오브젝트를 사용해야 하는 경우, 원격 오브젝트에 대한 프록시를 만들어두고, 클라이언트는 마치 로컬에 존재하는 오브젝트를 쓰듯이 프록시를 사용하게 할 수 있다. 
- 특별한 상황에서 타깃에 대한 접근권한을 제어하기 위해 사용. 수정 가능한 오브젝트지만 특정 레이어를 넘어가서는 읽기 전용으로 동작해야만 한다면, 프록시의 특정 메소드를 사용하려고 하면 접근이 불가능하도록 예외를 발생시켜 이를 강제할 수 있다. 
    - Collections의 unmodifiableCollection()을 통해 만들어지는 오브젝트가 전형적인 접근권한 제어용 프록시라고 볼 수 있다. 파라미터로 전달된 Collection 오브젝트의 프록시를 만들어서, add()나 remove() 같이 정보를 수정하는 메소드를 호출할 경우 UnsupportedOperationException 예외가 발생하게 해준다.

구조

- 구조 자체는 프록시 패턴과 데코레이터 패턴이 유사하다.
- 하지만 생성을 지연하는 프록시는 구체적인 생성 방법을 알아야 하기 때문에 타깃 클래스에 대한 직접적인 정보를 알아야 한다. 
- 이제부터는 접근제어/부가기능을 담당하는 오브젝트를 모두 프록시라고 부른다. 


프록시를 만들기 번거로운 이유

- 타깃의 인터페이스 구현과 위임 코드 작성이 번거로움
  - 부가기능이 필요없는 메소드도 일일이 구현 필요
  - 인터페이스 메소드가 많아지면 부담스러운 작업
  - 타깃 인터페이스 변경 시 프록시도 함께 수정 필요

- 부가기능 코드의 중복 가능성이 높음
  - 트랜잭션 등의 부가기능은 여러 메소드에 적용 필요
  - 메소드가 많아질수록 유사한 코드가 중복되어 나타남

리플랙션

다이내믹 프록시는 리플랙션 기능을 이용해 프록시를 만들어준다.
리플렉션은 클래스와 오브젝트의 메타정보를 사용하여 코드를 조작할 수 있게 해주는 기능이다.

- 모든 자바 클래스는 Class 타입의 오브젝트를 가짐
- 클래스이름.class나 getClass() 메소드로 Class 오브젝트 접근 가능
- Class 오브젝트로 할 수 있는 것들:
  - 클래스의 이름, 상속, 인터페이스 정보 조회
  - 필드와 메소드 정보(타입, 파라미터, 리턴타입 등) 조회
  - 오브젝트의 필드값 읽기/쓰기
  - 메소드 동적 호출 등...

```java
package springbook.learningtest.jdk;
...

public class ReflectionTest {
    @Test
    public void invokeMethod() throws Exception {
        String name = "Spring";

        // length()
        assertThat(name.length(), is(6));

        Method lengthMethod = String.class.getMethod("length");
        assertThat((Integer) lengthMethod.invoke(name), is(6));

        // charAt()
        assertThat(name.charAt(0), is('S'));

        Method charAtMethod = String.class.getMethod("charAt", int.class);
        assertThat((Character) charAtMethod.invoke(name, 0), is('S'));
    }
}
```

프록시 클래스 

```java
// Hello 인터페이스
interface Hello {
    String sayHello(String name);
    String sayHi(String name);
    String sayThankYou(String name);
}

// 타깃 클래스
public class HelloTarget implements Hello {
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

// 클라이언트 역할의 테스트
@Test
public void simpleProxy() {
    Hello hello = new HelloTarget(); // 타깃은 인터페이스를 통해 접근하는 습관을 들이자.
    assertThat(hello.sayHello("Toby"), is("Hello Toby"));
    assertThat(hello.sayHi("Toby"), is("Hi Toby"));
    assertThat(hello.sayThankYou("Toby"), is("Thank You Toby"));
}
```

Hello 인터페이스를 구현한 HelloUppercase 프록시는 데코레이터 패턴을 적용하여 HelloTarget에 대문자 변환이라는 부가기능을 추가한다. 프록시는 Hello 인터페이스를 구현하고 타깃 오브젝트를 참조하여, 메소드 호출을 위임하고 결과를 대문자로 변환하는 기능을 수행한다.

```java
// 프록시 클래스
public class HelloUppercase implements Hello {
    Hello hello; // 위임할 타깃 오브젝트. 여기서는 타깃 클래스의 오브젝트인 것은 알지만
                 // 다른 프록시를 추가할 수도 있으므로 인터페이스로 접근한다.

    public HelloUppercase(Hello hello) {
        this.hello = hello;
    }

    public String sayHello(String name) {
        return hello.sayHello(name).toUpperCase(); // 위임과 부가기능 적용
    }

    public String sayHi(String name) {
        return hello.sayHi(name).toUpperCase();
    }

    public String sayThankYou(String name) {
        return hello.sayThankYou(name).toUpperCase();
    }
}
```
```java
// HelloUppercase 프록시 테스트
Hello proxiedHello = new HelloUppercase(new HelloTarget()); // 프록시를 통해 타깃 오브젝트에 접근하도록 구성한다.
assertThat(proxiedHello.sayHello("Toby"), is("HELLO TOBY"));
assertThat(proxiedHello.sayHi("Toby"), is("HI TOBY"));
assertThat(proxiedHello.sayThankYou("Toby"), is("THANK YOU TOBY"));
```

문제점
- 인터페이스의 모든 메소드를 구현해서 위임해야 함
- 부가기능이 모든 메소드에 중복되어 나타남 

다이내믹 프록시 적용


### 다이내믹 프록시의 구조와 동작 원리

- 다이내믹 프록시는 런타임 시 프록시 팩토리가 동적으로 생성하는 오브젝트
- 타깃의 인터페이스와 동일한 타입으로 생성되어 클라이언트가 타깃 인터페이스로 사용 가능
- 프록시 팩토리에 인터페이스 정보만 제공하면 구현 클래스 자동 생성

#### InvocationHandler
- 프록시의 부가기능을 제공하는 코드를 담당
- 단일 메소드 `invoke(Object proxy, Method method, Object[] args)` 구현
- 클라이언트의 모든 요청을 리플렉션 정보로 변환하여 처리
- 중복되는 기능을 하나의 메소드로 집중하여 효과적으로 제공

#### 동작 과정
- 프록시 팩토리가 Hello 인터페이스 구현체 생성
- 다이내믹 프록시는 모든 요청을 InvocationHandler의 invoke()로 전달
- InvocationHandler는 리플렉션을 통해 타깃 오브젝트의 메소드 호출을 처리
- 타깃 메소드 호출 결과에 부가기능 적용 가능



```java
// InvocationHandler 구현 클래스
public class UppercaseHandler implements InvocationHandler {
    Hello target;

    public UppercaseHandler(Hello target) { // 다이내믹 프록시로부터 전달받은 요청을 다시 타깃 오브젝트에 위임해야 하기 때문에 타깃 오브젝트를 주입받아 둔다.
        this.target = target;
    }

    public Object invoke(Object proxy, Method method, Object[] args) 
        throws Throwable {
        String ret = (String)method.invoke(target, args); // 타깃으로 위임. 인터페이스의 메소드 호출에 모두 적용된다.
        return ret.toUpperCase(); // 부가기능 제공
    }
}
```

#### InvocationHandler 구현과 다이내믹 프록시 생성

- InvocationHandler 구현 필요
  - invoke() 메소드 하나만 구현
  - 다이내믹 프록시의 모든 요청이 invoke()로 전달됨

- 요청 처리 과정
  - 리플렉션 API로 타깃 오브젝트 메소드 호출
  - 타깃 오브젝트는 생성자로 미리 전달받음
  - Hello 인터페이스 메소드는 모두 String 반환
  - 부가기능(대문자 변환) 수행 후 결과 리턴
  - 다이내믹 프록시가 최종 결과를 클라이언트에게 전달

- 다이내믹 프록시 생성
  - Proxy.newProxyInstance() 스태틱 팩토리 메소드 사용
  - 파라미터가 많으므로 주의 필요

```java
// 프록시 생성
Hello proxiedHello = (Hello)Proxy.newProxyInstance(
    getClass().getClassLoader(), // 동적으로 생성되는 다이내믹 프록시 클래스의 로딩에 사용할 클래스 로더
    new Class[] { Hello.class }, // 구현할 인터페이스
    new UppercaseHandler(new HelloTarget()) // 부가기능과 위임 코드를 담은 InvocationHandler
);
```

- 첫 번째 파라미터 (클래스 로더)
  - 다이내믹 프록시가 정의되는 클래스 로더 지정
  - getClass().getClassLoader() 사용

- 두 번째 파라미터 (인터페이스)
  - 다이내믹 프록시가 구현할 인터페이스 배열
  - 여러 인터페이스 동시 구현 가능
  - 예제에서는 Hello 인터페이스만 구현

- 세 번째 파라미터 (InvocationHandler)
  - 부가기능과 위임 코드를 담은 구현체
  - 예제에서는 UppercaseHandler 사용

### 다이내믹 프록시의 확장

#### 다이내믹 프록시의 장점

- 유연성과 확장성
  - 인터페이스 메소드가 늘어나도 프록시 코드 수정 불필요
  - 자동으로 새로운 메소드 처리 가능

- 다양한 타입 처리
  - 리턴 타입 검사로 안전한 처리 가능
  - 타깃의 종류에 상관없이 적용 가능

- 메소드 선별 처리
  - 메소드 이름, 파라미터, 리턴 타입 등으로 선택적 기능 적용 가능
  - 예: say로 시작하는 메소드만 대문자 변환 적용


```java
// 확장된 UppercaseHandler
public class UppercaseHandler implements InvocationHandler {
    Object target; // 어떤 종류의 인터페이스를 구현한 타깃에도 적용 가능하도록 Object 타입으로 수정

    private UppercaseHandler(Object target) {
        this.target = target;
    }

    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        Object ret = method.invoke(target, args);
        if (ret instanceof String) { // 호출한 메소드의 리턴 타입이 String인 경우만 대문자 변경 기능을 적용하도록 수정
            return ((String)ret).toUpperCase();
        } else {
            return ret;
        }
    }
}
```

```java
// 메소드를 선별해서 부가기능을 적용하는 invoke()
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    Object ret = method.invoke(target, args);
    if (ret instanceof String && method.getName().startsWith("say")) // 리턴 타입과 메소드 이름이 일치하는 경우에만 부가기능을 적용한다.
        return ((String)ret).toUpperCase();
    else
        return ret; // 조건이 일치하지 않으면 타깃 오브젝트의 호출 결과를 그대로 리턴한다.
}
```

### UserServiceTx를 다이내믹 프록시 방식으로 변경

- 다이내믹 프록시와 연동해서 트랜잭션 기능을 부가해주는 InvocationHandler 하나 정의 

```java
public class TransactionHandler implements InvocationHandler {
    private Object target; // 부가기능을 제공할 타깃 오브젝트, 어떤 타입의 오브젝트에도 적용 가능하도록 Object 타입으로 수정
    private PlatformTransactionManager transactionManager; // 트랜잭션 기능을 제공하는 데 필요한 트랜잭션 매니저
    private String pattern; // 트랜잭션을 적용할 메소드 이름 패턴

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

    private Object invokeInTransaction(Method method, Object[] args) throws Throwable {
        TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
        try {
            Object ret = method.invoke(target, args); // 트랜잭션을 시작하고 타깃 오브젝트의 메소드를 호출한다. 예외가 발생하지 않았다면 커밋한다.
            this.transactionManager.commit(status);
            return ret;
        } catch (InvocationTargetException e) {
            this.transactionManager.rollback(status); // 예외가 발생하면 트랜잭션을 롤백한다.
            throw e.getTargetException();
        }
    }
}
```

#### TransactionHandler가 UserServiceTx를 대신할 수 있는지 테스트 

```java
@Test
public void upgradeAllOrNothing() throws Exception {
    // 트랜잭션 핸들러가 필요한 정보와 오브젝트를 DI 해준다.
    TransactionHandler txHandler = new TransactionHandler();
    txHandler.setTarget(testUserService);
    txHandler.setTransactionManager(transactionManager);
    txHandler.setPattern("upgradeLevels");

    // UserService 인터페이스 타입의 다이나믹 프록시 생성
    UserService txUserService = (UserService) Proxy.newProxyInstance(
        getClass().getClassLoader(), new Class[] { UserService.class }, txHandler
    );

    // 테스트 코드에서 txUserService를 사용하여 트랜잭션이 적용된 upgradeLevels 호출 등을 검증
    ...
}
``` 

### 다이내믹 프록시를 위한 팩토리 빈 

- 다이내믹 프록시는 일반적인 스프링 빈으로 등록할 수 없다.
  - 스프링은 클래스 이름으로 빈 오브젝트를 생성하는데, 다이내믹 프록시는 클래스를 알 수 없기 때문
  - Proxy.newProxyInstance() 스태틱 메소드로만 생성 가능

#### 팩토리 빈

- 스프링을 대신해서 오브젝트의 생성로직을 담당하는 특별한 빈
- FactoryBean 인터페이스를 구현한 클래스를 빈으로 등록하면 팩토리 빈으로 동작
- private 생성자를 가진 클래스는 팩토리 빈으로 등록하는 것이 권장됨
  - 리플렉션으로 강제 생성하면 위험할 수 있음


#### 팩토리 빈 설정 방법

```xml
<bean id="message"
      class="springbook.learningtest.spring.factorybean.MessageFactoryBean">
    <property name="text" value="Factory Bean" />
</bean>
```
```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration // 설정파일 이름을 지정하지 않으면 클래스이름+"-context.xml"이 디폴트로 사용된다.
public class FactoryBeanTest {
    @Autowired
    ApplicationContext context;

    @Test
    public void getMessageFromFactoryBean() {
        Object message = context.getBean("message"); // 빈의 타입을 지정하지 않으면 Object 타입 리턴
        assertThat(message, is(Message.class)); // 타입 확인
        assertThat(((Message)message).getText(), is("Factory Bean")); // 설정과 기능 확인
    }

    @Test
    public void getFactoryBean() throws Exception {
        Object factory = context.getBean("&message"); // &가 붙으면 팩토리 빈 자체를 가져온다.
        assertThat(factory, is(MessageFactoryBean.class)); // 팩토리 빈 클래스 확인
    }
}
```

#### 다이내믹 프록시를 만들어주는 팩토리 빈

- 다이내믹 프록시는 일반적인 방법으로는 스프링 빈으로 등록할 수 없지만, 팩토리 빈을 사용하면 가능하다. 팩토리 빈의 getObject() 메소드에 다이내믹 프록시 생성 코드를 넣으면 된다.
- 팩토리 빈은 다이내믹 프록시가 위임할 타깃 오브젝트를 DI 받고, 프록시와 TransactionHandler 생성에 필요한 정보를 프로퍼티로 설정해서 전달한다.

#### 트랜잭션 프록시 팩토리 빈 

```java
package springbook.user.service;

public class TxProxyFactoryBean implements FactoryBean<Object> {
    Object target; // 생성할 오브젝트 타입을 지정할 수도 있지만 범용적으로 사용하기 위해 Object로 했다.
    PlatformTransactionManager transactionManager; // TransactionHandler를 생성할 때 필요
    String pattern; // 트랜잭션을 적용할 메소드 이름 패턴
    Class<?> serviceInterface; // 다이내믹 프록시를 생성할 때 필요하다. UserService 외의 인터페이스를 가진 타깃에도 적용할 수 있다.

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

    // FactoryBean 인터페이스 구현 메소드
    public Object getObject() throws Exception { // DI 받은 정보를 이용해서 TransactionHandler를 사용하는 다이내믹 프록시를 생성한다.
        TransactionHandler txHandler = new TransactionHandler();
        txHandler.setTarget(target);
        txHandler.setTransactionManager(transactionManager);
        txHandler.setPattern(pattern);
        return Proxy.newProxyInstance(
            getClass().getClassLoader(), new Class[] { serviceInterface },
            txHandler);
    }

    public Class<?> getObjectType() { // 팩토리 빈이 생성하는 오브젝트의 타입은 DI 받은 인터페이스 타입에 따라 달라진다. 따라서 다양한 타입의 프록시 오브젝트 생성에 재사용할 수 있다.
        return serviceInterface;
    }

    public boolean isSingleton() {
        return false; // 싱글톤 빈이 아니라는 뜻이 아니라 getObject()가 매번 같은 오브젝트를 리턴하지 않는다는 의미다.
    }
}
```
```xml
<bean id="userService" class="springbook.user.service.TxProxyFactoryBean">
    <property name="target" ref="userServiceImpl" />
    <property name="transactionManager" ref="transactionManager" />
    <property name="pattern" value="upgradeLevels" />
    <property name="serviceInterface" value="springbook.user.service.UserService" />
</bean>
```
- 팩토리 빈이 만드는 다이내믹 프록시의 특징:
  - 구현 인터페이스나 타깃의 종류에 제한이 없음
  - UserService 외 다른 오브젝트의 트랜잭션 부가기능 프록시로 재사용 가능
  - 설정이 다른 여러 TxProxyFactoryBean 빈 등록 가능

- 빈 설정 변경사항:
  - UserServiceTx 빈 대신 TxProxyFactoryBean 팩토리 빈을 userService로 등록
  - UserServiceTx 클래스는 제거 가능

- 프로퍼티 설정 방법:
  - target, transactionManager: ref 애트리뷰트 사용 (다른 빈 참조)
  - pattern: value 애트리뷰트 사용 (문자열)
  - serviceInterface: value로 클래스/인터페이스 이름 지정 (Class 타입은 스프링이 자동 변환)

#### 트랜잭션 프록시 팩토리 빈 테스트 

```java
public class UserServiceTest {
    ...
    @Autowired ApplicationContext context; // 팩토리 빈을 가져오려면 애플리케이션 컨텍스트가 필요하다.

    @Test
    @DirtiesContext // 다이나믹 프록시 팩토리 빈을 직접 만들어 사용할 때는 없었던 다가 다시 등장한 컨텍스트 무효화 애노테이션
    public void upgradeAllOrNothing() throws Exception {
        TestUserService testUserService = new TestUserService(users.get(3).getId());
        testUserService.setUserDao(userDao);
        testUserService.setMailSender(mailSender);

        TxProxyFactoryBean txProxyFactoryBean = 
            (TxProxyFactoryBean) context.getBean("&userService"); // 팩토리 빈 자체를 가져와야 하므로 빈 이름에 &를 반드시 넣어야 한다.
        txProxyFactoryBean.setTarget(testUserService);
        UserService txUserService = (UserService) txProxyFactoryBean.getObject(); // 변경된 타깃 설정을 이용해서 트랜잭션 다이나믹 프록시 오브젝트를 다시 생성한다.

        userDao.deleteAll();
        for (User user : users) userDao.add(user);

        try {
            txUserService.upgradeLevels();
            fail("TestUserServiceException expected");
        } catch (TestUserServiceException e) {
        }

        checkLevelUpgraded(users.get(1), false);
    }
}
```
- UserServiceTest 테스트 
  - add() 테스트: 
    - @Autowired로 주입된 userService 빈 사용
    - TxProxyFactoryBean이 생성한 다이내믹 프록시 통해 실행
    - 트랜잭션 미적용 메소드라 단순 위임만 발생
  
  - upgradeLevels(), mockUpgradeLevels() 테스트:
    - 목 오브젝트 활용한 단위 테스트
    - 트랜잭션과 무관
    
  - upgradeAllOrNothing() 테스트:
    - 수동 DI로 직접 다이내믹 프록시 생성
    - 팩토리 빈 미적용
    
- 트랜잭션 기능 테스트의 어려움:
  - TestUserService를 타깃으로 사용해야 함
  - 기존 설정의 UserServiceImpl을 대체해야 함
  - TransactionHandler가 팩토리 빈 내부에 있어 접근 불가
  
- 해결 방안:
  - 팩토리 빈 자체를 가져와 프록시 재생성
  - 과정:
    1. TxProxyFactoryBean을 컨텍스트에서 가져옴
    2. target 프로퍼티 재설정
    3. 새로운 프록시 오브젝트 생성
  - @DirtiesContext로 컨텍스트 설정 변경 처리

- 결과:
  - UserServiceTx 사용과 유사한 코드로 구현
  - 타깃 변경을 위해 팩토리 빈으로 프록시 재생성
  - 4개 테스트 모두 정상 동작 확인
  - TxProxyFactoryBean의 재사용성 입증

#### 프록시 팩토리 빈 방식의 장점

- 다이내믹 프록시 팩토리 빈 방식의 장점:
  - 부가기능을 가진 프록시 팩토리 빈의 재사용성이 높음
  - 타깃 타입에 상관없이 적용 가능
  - 코드 수정 없이 설정만으로 다양한 클래스에 적용 가능
  - 여러 개의 팩토리 빈을 동시에 등록/사용 가능

- TxProxyFactoryBean의 활용:
  - 다양한 서비스 클래스에 트랜잭션 기능 적용 가능
  - 설정 변경만으로 프록시 기능 추가 가능
  - 클라이언트 코드 수정 없이 트랜잭션 기능 부여 가능

- 기존 프록시 방식의 문제점 해결:
  - 인터페이스 구현 프록시 클래스를 매번 만들어야 하는 번거로움 제거
  - 부가기능 코드의 중복 문제 해결
  - 다이내믹 프록시 생성 코드 제거

- 스프링 DI의 중요성:
  - 프록시 사용을 위한 필수 요소
  - 다이내믹 프록시 활용을 위한 팩토리 빈 운영에 필수
  - 프록시 팩토리 빈의 효과적인 활용을 위한 기반

#### 프록시 팩토리 빈 방식의 한계

- 프록시 팩토리 빈의 한계점:
  - 여러 클래스에 공통 부가기능을 한 번에 제공하기 어려움
  - 여러 부가기능을 하나의 타깃에 적용할 때 설정이 복잡해짐
    - 서비스 클래스가 많을 경우 XML 설정이 기하급수적으로 증가
    - 설정 파일이 복잡해지면 실수하기 쉽고 관리가 어려워짐
  
- TransactionHandler 관련 문제:
  - 프록시 팩토리 빈 개수만큼 객체가 생성됨
  - 타깃 객체가 다르면 새로운 TransactionHandler 객체 필요
  - 동일한 코드임에도 중복 발생
  
- 개선이 필요한 부분:
  - 설정의 중복을 줄일 방법 필요
  - TransactionHandler를 모든 타깃에 적용 가능한 싱글톤 빈으로 만들 방법 모색
  - 스프링 DI를 활용한 더 나은 해결책 필요
