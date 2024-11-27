# 6장. AOP

✨ AOP?

- 스프링의 3대 기반 기술 중 하나

- AOP를 바르게 이용하려면…
    - 필연적인 등장 배경
    - 도입 이유
    - 적용을 통한 장점에 대한 충분한 이해가 필요하다

→ 그래야 AOP의 가치를 이해하고 효과적으로 사용할 방법을 찾을 수 있다.

- 스프링에서 가장 인기 있는 AOP 적용 대상: 선언적 트랜잭션 기능
    - 서비스 추상화를 통해 해결했던 트랜잭션 경계설정 기능을 AOP를 이용해 세련되고 깔끔하게 바꿔보자!
    - 그리고 그 과정에서 AOP의 도입 이유에 대해서 알아보자

## 6.1 트랜잭션 코드의 분리

🤔 비즈니스 로직이 주가 되어야할 메소드 안에 트랜잭션 코드가 더 많은 자리를 차지하고 있어 못마땅하다!

### 6.1.1 메소드 분리

- 트랜잭션이 적용된 코드에서 성격이 다른 코드를 두 개의 메소드로 분리해보자
    
    ```java
    public void upgradeLevels() {
        TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
        try{
            upgradeLevelsInternal();
            this.transactionManager.commit(status);
        } catch (Exception e) {
            this.transactionManager.rollback(status);
            throw e;
        }
    }
        
    private void upgradeLevelsInternal() {
        List<User> users = userDao.getAll();
        for(User user : users) {
            if(canChangedLevel(user)) {
                upgradeLevel(user);
            }
        }
    }
    ```
    
    - 비즈니스 로직 코드만 독립적인 메소드에 담겨 있으니 이해하기도 편하고 수정하기에도 부담이 없어졌다.

### 6.1.2 DI를 이용한 클래스 분리

📌 트랜잭션 코드를 클래스 밖으로 분리하자!

#### DI 적용을 이용한 트랜잭션 분리

- 트랜잭션 코드를 `UserService` 밖으로 빼면 `UserService` 클래스를 직접 사용하는 클라이언트 코드에서는 트랜잭션 기능이 빠진 `UserService`를 사용하게 된다.
- 직접 사용하는 것이 문제가 된다면 간접적으로 사용하게 만들면 된다.
    - DI를 이용하자!

- 현재 구조
    - UserService 클래스와 사용 클라이언트 간의 관계가 강한 결합도로 고정되어 있다.
- UserService를 인터페이스로 만들고 기존 코드는 UserService 인터페이스의 구현 클래스로 만들어 넣자
    - 이때, 한 번에 두 개의 UserService 인터페이스 구현 클래스를 동시에 이용하면 어떨까?
        
        ![6 1](https://github.com/user-attachments/assets/8297f6de-385b-45f9-95dc-c8453a96a0bf)

        
        - UserService를 구현한 또 다른 구현 클래스를 만들어 트랜잭션의 경계설정이라는 책임을 맡게 한다.

→  이 방법을 이용해 트랜잭션 경계설정 코드를 분리해낸 결과를 살펴보자.

#### UserService 인터페이스 도입

1. 기존의 UserService 클래스를 UserServiceImpl로 이름을 변경한다.
2. 클라이언트가 사용할 로직을 담은 핵심 메소드만 UserService 인터페이스로 만든 후 UserServiceImpl이 구현하도록 만든다

- UserService 인터페이스
    
    ```java
    public interface UserService {
        void add(User user);
        void upgradeLevels();
    }
    ```
    

- 트랜잭션 코드를 제거한 UserService 구현 클래스
    
    ```java
    public class UserServiceImpl implements UserService {
        UserDao userDao;
    
        ...
    
        public void upgradeLevels() {
            List<User> users = userDao.getAll();
            for(User user : users) {
                if(canChangedLevel(user)) {
                    upgradeLevel(user);
                }
            }
        }
    
        ...
    }
    ```
    
    - 인터페이스를 이용하고 비즈니스 로직에만 충실한 깔끔한 코드가 됐다.

#### 분리된 트랜잭션 기능

✨ 비즈니스 트랜잭션 처리를 담은 `UserServiceTx`를 만들어보자

```java
public class UserServiceTx implements UserService {
    UserService userService;

    public void setUserService(UserService userService) {
        this.userService = userService;
    }
    
    @Override
    public void add(User user) {
        userService.add(user);
    }

    @Override
    public void upgradeLevels() {
        userService.upgradeLevels();
    }
}
```

- `UserServiceTx` 는 사용자 관리라는 비즈니스 로직을 전혀 갖지 않고
- 다른 UserService 구현 오브젝트에 기능을 위임한다.
- 여기에 트랜잭션의 경계설정이라는 부가적인 작업을 부여해보자.

- 트랜잭션이 적용된 `UserServiceTx`
    
    ```java
    public class UserServiceTx implements UserService {
        UserService userService;
        private PlatformTransactionManager transactionManager;
    
        public void setUserService(UserService userService) {
            this.userService = userService;
        }
    
        public void setTransactionManager(PlatformTransactionManager transactionManager) {
            this.transactionManager = transactionManager;
        }
    
        @Override
        public void add(User user) {
            userService.add(user);
        }
    
        @Override
        public void upgradeLevels() {
            TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
            try{
                userService.upgradeLevels();
                this.transactionManager.commit(status);
            } catch (Exception e) {
                this.transactionManager.rollback(status);
                throw e;
            }
        }
    }
    ```
    

#### 트랜잭션 적용을 위한 DI 설정

✨ 설정파일을 수정해보자!

- 기존에 userService 빈이 의존하고 있던 transactionManager는 `UserServiceTx`의 빈이
- userDao와 mailSender는 `UserServiceImpl` 빈이 각각 의존하도록 프로퍼티 정보를 분리하자.

- 트랜잭션 오브젝트가 추가된 설정파일
    
    ```java
    <bean id="userService" class="springbook.user.service.UserServiceTx">
        <property name="transactionManager" ref="transactionManager"/>
        <property name="userService" ref="userServiceImpl"/>
    </bean>
    
    <bean id="userServiceImpl" class="springbook.user.service.UserServiceImpl">
        <property name="userDao" ref="userDao"/>
        <property name="mailSender" ref="mailSender"/>
    </bean>
    ```
    

→ 이제 userService 빈은 `UserServiceImpl` 클래스로 정의되는, 아이디가 userServiceImpl인 빈을 DI하게 만든다.

#### 트랜잭션 분리에 따른 테스트 수정

📌 기존의 `UserService` 클래스가 인터페이스와 두 개의 클래스로 분리된 만큼 테스트도 수정해야 한다.

- 먼저, 스프링의 테스트용 컨텍스트에서 가져올 빈들을 생각해보자.
    - 기존: `UserService` 클래스 타입의 빈을 `@Autowired`로 가져다가 사용
    - 수정한 스프링의 설정파일에는 `UserService` 라는 인터페이스 타입을 가진 두 개의 빈이 존재하므로 문제 발생
        - userService 변수를 설정해두자!
            
            `@Autowired UserService userService;`
            

- 그런데 `UserServiceTest`는 `UserServiceImpl`클래스로 정의된 빈을 더 가져와야 한다.
    - MailSender를 DI 해줄 대상을 구체적으로 알고 있어야 하기 때문에 `UserServiceImpl` 클래스의 오브젝트를 가져와야 함.
        
        `@Autowired UserServiceImpl userServiceImpl;`
        
- userServiceImpl 빈에 MailSender의 목 오브젝트를 설정해주어야 한다.
    
    ```java
    @Test
    public void upgradeLevels() throws Exception {
        ...
        MockMailSender mockMailSender = new MockMailSender();
        userServiceImpl.setMailSender(mockMailSender);
    ```
    

- `upgradeAllorNothing()` 테스트 수정
    - 바뀐 구조를 모두 반영해야 함.
    - TestUserService 오브젝트를 `UserServiceTx`  오브젝트에 수동 DI 시킨 후에 트랜잭션 기능까지 포함된 `UserServiceTx`의 메소드를 호출하면서 테스트를 수행하도록 해야 한다.
        
        ```java
        @Test
        public void upgradeAllOrNothing() {
            UserServiceImpl testUserService = new TestUserService(users.get(3).getId());
            testUserService.setUserDao(this.userDao);
            testUserService.setMailSender(mailSender);
        
        		//트랜잭션 기술 적용이 되었는지 확인을 위한 테스트 코드로 testUserService를 수동 DI 하여 사용하기 위해
            //UserServiceTx 오브젝트를 생성하도록 한다
            UserServiceTx userServiceTx = new UserServiceTx();
            userServiceTx.setTransactionManager(transactionManager);
            userServiceTx.setUserService(testUserService);
            
            userDao.deleteAll();
            for(User user:users) {
                userDao.add(user);
            }
        
            try {
                userServiceTx.upgradeLevels();
                fail("TestUserServiceException");
            } 
            ...
        ```
        

#### 트랜잭션 경계설정 코드 분리의 장점

🤔 트랜잭션 경계설정 코드의 분리와 DI를 통한 연결을 통해 얻을 수 있는 장점이 무엇일까?

1. 비즈니스 로직을 담당하고 있는 코드를 작성할 때는 기술적인 내용에는 신경쓰지 않아도 된다.
2. 비즈니스 로직에 대한 테스트를 손쉽게 만들어낼 수 있다
    
    → 이에 대해 다음 장에서 좀 더 자세히 알아보자!


## 6.2 고립된 단위 테스트

✨ 가장 편하고 좋은 테스트 방법 → 작은 단위로 쪼개서 테스트하는 것

- 테스트 실패시 원인을 찾기 쉽기 때문
- 하지만
    - 테스트 대상이 다른 오브젝트와 환경에 의존하고 있다면 작은 단위의 테스트가 주는 장점을 얻기 힘들다!

### 6.2.1 복잡한 의존관계 속의 테스트

- UserService는 간단한 기능만을 가지고 있지만
- 동작하기 위해서 세 가지 타입의 의존 오브젝트가 필요하다.

- UserService 를 분리하기 전의 테스트가 동작하는 모습
    
    ![6 2-1](https://github.com/user-attachments/assets/2ab1118a-61b1-4d7e-9e56-d8e44aaa58e3)

    
    - `UserDao`, `TransactionManager`, `MailSender` 라는 세 가지 의존 관계를 갖고 있음.
    - 세 가지 의존관계를 가지는 오브젝트들이 테스트와 함께 실행
    - 문제는…
        - 의존 오브젝트들도 자신의 코드만 실행하지 않는다!
    - UserService를 테스트 하는데 오브젝트, 환경, 서비스, 서버까지 함께 테스트하는 셈이다.

### 6.2.2 테스트 대상 오브젝트 고립시키기

📌 따라서 테스트의 대상이 다른 것에 종속되고 영향을 받지 않도록 고립시킬 필요가 있다.

- 테스트를 위한 대역을 사용해보자!

#### 테스트를 위한 UserServiceImpl 고립

- 고립된 테스트가 가능하도록 `UserService`를 재구성
    
    ![6 2-2](https://github.com/user-attachments/assets/a315c72c-463d-425e-9a73-803597592441)

    
    - UserServiceImpl에 대한 테스트가 진행될 때 두 개의 목 오브젝트에만 의존

- `UserDao`, `UserServiceImpl`의 `upgradeLevels()` 메소드
    
    
    | `UserDao` | `UserServiceImpl` |
    | --- | --- |
    | 스텁이 아니라 부가적인 검증기능까지 가진 목 오브젝트 | 리턴 값이 없는 void형
    → 메소드 실행 후 결과를 받아 검증 불가
     |
    | 고립된 환경에서 동작하는 upgradelevels()의 테스트 결과를 검증 가능 | 검증을 위해서는 결과가 남아 있는 DB를 직접확인해야 함. |
    - 고립된 테스트 방식인 `UserServiceImpl`은 DB 확인을 통해 작업 결과를 검증하기 힘들다.
        
        → 테스트 대상인 `UserServiceImpl`과 그 협력 오브젝트인 `UserDao`에게 어떤 요청을 했는지 확인하는 작업이 필요, 목 오브젝트를 만들 필요가 있음.
        

#### 고립된 단위 테스트 활용

🤔 고립된 단위 테스트 방법을 `UserServiceTest`의 `upgradeLevels()`테스트에 적용해보자.

- 기존 `upgradeLevels()` 테스트 코드의 구성
    1. 테스트 실행 중에 `UserDao`를 통해 가져올 테스트용 정보를 DB에 넣는다.
    2. 메일 발송 여부를 확인하기 위해 `MailSender` 목 오브젝트를 DI 해준다.
    3. 실제 테스트 대상인 `userService`의 메소드를 실행한다.
    4. 결과가 DB에 반영됐는지 확인하기 위해서 `UserDao`를 이용해 DB에서 데이터를 가져와 결과를 확인한다.
    5. 목 오브젝트를 통해 `UserService`에 의한 메일 발송이 있었는지를 확인한다.

#### `UserDao` 목 오브젝트

🤔 `UserDao` 와 DB까지 직접 의존하고 있는 첫 번째와 네 번째의 테스트 방식도 목 오브젝트로 만들어서 적용해보자.

- 목 오브젝트는 기본적으로 스텁과 같은 방식으로 테스트 대상을 통해 사용될 때 필요한 기능을 지원해줘야 함.

- `UserServiceImpl` 의  `upgradeLevels()` 메소드에서 `UserDao` 를 사용하는 경우
    - 업그레이드 될 후보 사용자의 목록을 받아올 때
    - 수정된 사용자 정보를 DB에 반영할 때

1️⃣ 업그레이드 될 후보 사용자의 목록을 받아오는 경우

- 스텁으로서 `update()` 에 대해서는 목 오브젝트로 동작하는 `UserDao` 타입의 테스트 대역이 필요
    - `MockUserDao`
        
        ```java
        public class UserServiceImpl implements UserService {
            public static class MockUserDao implements UserDao {
                //레벨 업그레이드 후보 User 오브젝트 목록
                private List<User> users; 
                //업그레이드 대상 오브젝트를 저장해둘 목록
                private List<User> updated = new ArrayList<>();
        
                private MockUserDao(List<User> users) {
                    this.users = users;
                }
        
                public List<User> getUpdated() {
                    return this.updated;
                }
        
                @Override
                public List<User> getAll() {
                    return this.users;
                }
        
                @Override
                public void update(User user) {
                    updated.add(user);
                }
        
                @Override
                public void add(User user) {throw new UnsupportedOperationException();}
        
                @Override
                public User get(String id) {throw new UnsupportedOperationException();}
        
                @Override
                public void deleteAll() {throw new UnsupportedOperationException();}
        
                @Override
                public int getCount() {throw new UnsupportedOperationException();}
            }
        }
        ```
        
        - 두 개의 `User`  타입 리스트를 정의
            1. 생성자를 통해 전달받은 사용자 목록을 저장해뒀다가, `getAll()` 메소드가 호출되면 DBㅇ서 가져온 것처럼 돌려주는 용도
            2. `update()` 메소드를 실행하면서 넘겨준 `upgradeLevels()` 메소드가 실행되는 동안 업그레이드 대상으로 선정된 사용자가 어떤 것인지 확인하는 용도

- `MockUserDao` 를 사용해서 만든 고립된 테스트
    
    ```java
    public class UserServiceTest {
        ...
        @Test
        public void upgradeLevels() throws Exception {
            UserServiceImpl userServiceImpl = new UserServiceImpl();
    
            UserServiceImpl.MockUserDao mockUserDao =
                    new UserServiceImpl.MockUserDao(this.users);
            userServiceImpl.setUserDao(mockUserDao);
    
            MockMailSender mockMailSender = new MockMailSender();
            userServiceImpl.setMailSender(mockMailSender);
    
            userServiceImpl.upgradeLevels();
    
            List<User> updated = mockUserDao.getUpdated();
            assertThat(updated.size(), is(2));
            checkUserAndLevel(updated.get(0), "joytouch", Level.SILVER);
            checkUserAndLevel(updated.get(1), "madnite1", Level.GOLD);
    
            List<String> request = mockMailSender.getRequests();
            assertThat(request.size(), is(2));
            assertThat(request.get(0), is(users.get(1).getEmail()));
            assertThat(request.get(1), is(users.get(3).getEmail()));
        }
        private void checkUserAndLevel(User updated, String expectedId,
                                       Level expectedLevel) {
            assertThat(updated.getId(), is(expectedId));
            assertThat(updated.getLevel(), is(expectedLevel));
        }
    }
    ```
    
    1. 테스트하고 싶은 로직을 담은 `UserServiceImpl`  클래스의 오브젝트를 직접 생성
    2. 준비해둔 `MockUserDao`  오브젝트를 사용하도록 수동 DI
    3. 메소드 실행
    4. 미리 준비해둔 사용자 목록 리턴, 레벨 변경 후 `MockDaoUser` 의 `update()` 메소드를 호출
    5. `UserServiceImpl` 은 `UserDao` 의 `update()` 를 이용해 목록의 수와 업그레이드 된 사용자의 아이디, 바뀐 레벨을 확인해서 검증한다.

#### 테스트 수행 성능의 향상

✨ `UserServiceTest` 의 `upgradeLevels()`  테스트를 실행해보자.

- 테스트가 성공하고 수행시간이 빨라졌다!

### 6.2.3 단위 테스트와 통합 테스트

✨ 단위 테스트

- 테스트 대상 클래스를 테스트 대역을 이용해 의존 오브젝트나 외부의 리소스를 사용하지 않도록 고립시켜서 테스트하는 것

✨통합 테스트

- 두 개 이상의, 성격이나 계층이 다른 오브젝트가 연동하도록 만들어 테스트하거나
- 외부의 리소스가 참여하는 테스트

- 단위 테스트와 통합 테스트를 선택하는 가이드라인
    - 항상 단위 테스트를 먼저 고려한다.
    - 하나의 클래스나 긴밀한 클래스 몇 개를 모아서 외부와의 의존관계를 모두 차단하고 필요에 따라 테스트 대역을 이용하도록 테스트를 만든다.
    - 외부 리소스를 사용해야만 가능한 테스트는 통합 테스트로 만든다.
    - 애초에 단위 테스트로 만들기 어려운 코드도 있다.
    - DAO 테스트는 DB라는 외부 리소스를 사용하기 때문에 통합 테스트로 분류되지만, 코드에서 보자면 하나의 기능 단위를 테스트하는 것이기도 하다.
    - 여러 개의 단위가 의존관계를 가지고 동작할 때를 위한 통합 테스트는 필요하다.
    - 단위 테스트를 만들기가 너무 복잡하다고 판단되는 코드는 처음부터 통합 테스트를 고려해 본다.
    - 스프링 테스트 컨텍스트 프레임워크를 이용하는 테스트는 통합 테스트다.

### 6.2.4 목 프레임워크

🤔 단위 테스트가 가장 우선시해야 할 테스트 방법이지만 목 오브젝트를 만드는 일이 가장 큰 걸림돌이다.

→ 이처럼 번거로운 목 오브젝트를 편리하게 작성하도록 도와주는 목 오브젝트 지원 프레임워크가 있다.

#### Mockito 프레임워크

🤔 직접 만든 목 오브젝트를 사용했던 테스트를 이 Mockito를 이용하도록 바꿔보자!

- 목 프레임워크의 특징은 목 클래스를 일일이 준비할 필요가 없다는 점이다.
- 간단한 메소드 호출만으로 특정 인터페이스를 구현한 테스트용 목 오브젝트 생성이 가능하다.
    - `UserDao mockUserDao = mock(UserDao.class);`
- 여기에 스텁 기능을 추가한다.
    - `when(mockUserDao.getAll()).thenReturn(this.users);`
- `update()` 호출이 있었는지를 검증한다.
    - `verify(mockUserDao, times(2)).update(any(User.class));`

- Mockito를 이용해 만든 테스트 코드
    
    ```java
    @Test
    public void mockUpgradeLevels() throws Exception{
        UserServiceImpl userServiceImpl = new UserServiceImpl();
    
        // 다이나믹한 목 오브젝트 생성과 메소드의 리턴 값 설정, 그리고 DI까지 세 줄이면 충분하다.
        UserDAO mockUserDao = mock(UserDAO.class);
        when(mockUserDao.getAll()).thenReturn(this.users);
        userServiceImpl.setUserDao(mockUserDao);
    
        // 리턴 값이 없는 메소드를 가진 목 오브젝트는 더욱 간단하게 만들 수 있다.
        MailSender mockMailSender = mock(MailSender.class);
        userServiceImpl.setMailSender(mockMailSender);
    
        userServiceImpl.upgradeLevels();
    
        // 목 오브젝트가 제공하는 검증 기능을 통해서 어떤 메소드가 몇 번 호출 됐는지, 파라미터는 무엇인지 확인할 수 있다.
        verify(mockUserDao, times(2)).update(any(User.class));
        verify(mockUserDao).update(users.get(1));
        assertThat(users.get(1).getLevel(), is(Level.SILVER));
        verify(mockUserDao).update(users.get(3));
        assertThat(users.get(3).getLevel(), is(Level.GOLD));
    
        // Mockito 프레임워크의 툴 사용법으로써 배우지 않은 부분이니 그냥 보내질 메일 주소를 확인한다고 알고 있자.
        ArgumentCaptor<SimpleMailMessage> mailMessageArg = ArgumentCaptor.forClass(SimpleMailMessage.class);
        verify(mockMailSender, times(2)).send(mailMessageArg.capture());
        List<SimpleMailMessage> mailMessages = mailMessageArg.getAllValues();
        assertThat(mailMessages.get(0).getTo()[0], is(users.get(1).getEmail()));
        assertThat(mailMessages.get(1).getTo()[0], is(users.get(3).getEmail()));
        }


## 6.3 다이내믹 프록시와 팩토리 빈

### 6.3.1 프록시와 프록시 패턴, 데코레이터 패턴

🤔 트랜잭션 경계설정 코드를 비즈니스 로직 코드에서 분리해낼 때 적용했던 기법을 다시 검토해보자.

- 부가적인 기능을 위임을 통해 외부로 분리했을 때의 결과
    
    ![6 3-1](https://github.com/user-attachments/assets/84c329ac-1b58-45ae-b71d-0683c64e4389)

    
    - 구체적인 구현 코드는 제거했을지라도 위임을 통해 기능을 사용하는 코드는 핵심 코드와 함께 남아 있음.
    - 트랜잭션은 아예 그 적용 사실 자체를 밖으로 분리 가능

- 부가기능 전부를 핵심 코드가 담긴 클래스에서 분리했을 때의 결과
    
    ![6 3-2](https://github.com/user-attachments/assets/1eceff0c-5619-4b8c-bd80-56067c0d535a)

    
    - 부가기능 외의 나머지 모든 기능은 원래 핵심기능을 가진 클래스로 위임

- 하지만 이렇게 구성했더라도 클라이언트가 핵심기능을 가진 클래스를 직접 사용하면 부가 기능이 적용될 기회가 없음. → 부가기능은 자신이 핵심 기능을 가진 클래스인 것처럼 꾸며서 클라이언트가 자신을 거쳐 핵심 기능을 사용하도록 만들어야 함.
    
    ![6 3-3](https://github.com/user-attachments/assets/0685583d-644a-42dc-af60-f92145b99dd2)

    

✨ **프록시**

- 이처럼 자신이 클라이언트가 사용하려고 하는 실제 대상인 것처럼 위장해서 클라이언트의 요청을 받아주는 것
    - 클라이언트가 타깃에 접근하는 방법을 제어
    - 타깃에 부가적인 기능을 부여

✨ **타깃, 실체**

- 프록시를 통해 최종적으로 요청을 위임받아 처리하는 실제 오브젝트

#### 데코레이터 패턴

✨ **데코레이터 패턴**

- 타깃에 부가적인 기능을 런타임 시 다이내믹하게 부여해주기 위해 프록시를 사용하는 패턴
- 실제 내용물은 동일하지만 부가적인 효과를 부여해줄 수 있기 때문에 프록시가 꼭 한 개로 제한되지 않는다.
- 또, 프록시가 직접 타깃을 사용하도록 고정시킬 필요도 없다.

- 데코레이터 패턴 적용의 예시
    
    ![6 3-4](https://github.com/user-attachments/assets/255fb3c1-1e71-420e-b6ff-1372c4ff1f8c)

    
    - 부가적인 기능을 각각 프록시로 만들어두고 런타임 시에 이를 적절한 순서로 조합해서 사용하면 된다.

- 프록시로서 동작하는 각 데코레이터는 위임하는 대상에도 인터페이스로 접근하기 때문에 자신이 최종 타깃으로 위임하는지, 다음 단계의 데코레이터 프록시로 위임하는지 알지 못한다.
- 데코레이터의 다음 위임 대상은 인터페이스로 선언하고 생성자나 수정자 메소드를 통해 위임 대상을 외부에서 런타임 시에 주입받을 수 있도록 만들어야 한다.

- 데코레이터 패턴을 위한 DI 설정
    
    ```java
    <beans 
    		.../>
        <!--데코레이터-->
        <bean id="userService" class="springbook.user.service.UserServiceTx">
            <property name="transactionManager" ref="transactionManager"/>
            <property name="userService" ref="userServiceImpl"/>
        </bean>
    		
        <!--타깃-->
        <bean id="userServiceImpl" class="com.ksb.spring.UserServiceImpl">
            <property name="userDao" ref="userDao"/>
            <property name="mailSender" ref="mailSender"/>
        </bean>
    <beans/>
    ```
    
    - 다이내믹한 부가기능의 부여라는 데코레이터 패턴의 전형적인 적용 예

- 데코레이터 패턴은 인터페이스를 통해 위임하는 방식이기 때문에 어느 데코레이터에서 타깃으로 연결될지 코드 레벨에선 미리 알 수 없다. 구성하기에 따라서 여러 개의 데코레이터를 적용할 수 있다.
- 데코레이터 패턴은 타깃의 코드를 손대지 않고, 클라이언트가 호출하는 방법도 변경하지 않은 채로 새로운 기능을 추가할 때 유용한 방법이다.

#### 프록시 패턴

✨ **프록시 패턴**

- 프록시를 사용하는 방법 중, 타깃에 대한 접근 방법을 제어하려는 목적을 가진 경우
- 프록시 패턴의 프록시
    - 타깃의 기능을 확장하거나 추가하지 않음.
    - 클라이언트가 타깃에 접근하는 방식을 변경해줌.

- 프록시 패턴의 용도
1. 타깃 오브젝트에 대한 레퍼런스가 미리 필요할 때
    - 실제 타깃 오브젝트를 만드는 대신 프록시를 넘겨주는 것
2. 원격 오브젝트를 이용하는 경우
    - 클라이언트로 하여금 원격 오브젝트에 대한 접근 방법을 제공
3. 특별한 상황에서 타깃에 대한 접근 권한을 제어해야 할 때

⇒ 이렇게 프록시 패턴은 타깃의 기능 자체에는 관여하지 않으면서 접근하는 방법을 제어해주는 프록시를 이용하는 것

### 6.3.2 다이내믹 프록시

🤔 많은 개발자는 번거롭게 프록시를 만들지 않겠다고 생각하는데, 프록시도 목 오브젝트처럼 편리하게 만들어서 사용하는 방법이 있을까?

→ java.long.reflect 패키지 안에 프록시를 손쉽게 만들 수 있도록 지원해주는 클래스들이 있다.

#### 프록시의 구성과 프록시 작성의 문제점

📌프록시는 다음의 두 가지 기능으로 구성된다

- 타깃과 같은 메소드를 구현하고 있다가 메소드가 호출되면 타깃 오브젝트로 위임한다.
- 지정된 요청에 대해서는 부가기능을 수행한다.

- 프록시를 만들기 번거로운 이유
    1. 타깃의 인터페이스를 구현하고 위임하는 코드를 작성하기 번거롭다.
    2. 부가기능 코드가 중복될 가능성이 많다.
    
    ⇒ 두 번째 문제는 중복되는 코드를 분리해서 어떻게든 해결하면 되겠지만 첫 번째 문제는 어떻게 해결해야 할까?
    
    - JDK의 다이내믹 프록시를 이용하자.

#### 리플렉션

- 다이내믹 프록시는 리플렉션 기능을 이용해서 프록시를 만들어준다.
- 리플렉션: 자바의 코드 자체를 추상화해서 접근하도록 만든 것

📌 리플렉션 API 중에서 메소드에 대한 정의를 담은 Method라는 인터페이스를 이용해 메소드 호출하는 방법을 살펴보자.

- `Method lengthMethod = String.class.getMethod("length");`
    - 스트링이 가진 메소드 중에서 “length”라는 이름을 갖고 있고, 파라미터는 없는 메소드의 정보를 가져오는 것
- `Method`  인터페이스에 정의된 `invoke()` 메소드를 사용해서 실행
    - `public Object invoke(Object obj, Object... args)`
- 이를 이용해 `length()` 메소드 실행
    - `int length = lengthMethod.invoke(name);`

- 리플렉션 학습 테스트
    
    ```java
    public class ReflectionTest {
        @Test
        public void invokeMethod() throws Exception {
            String name = "Spring";
    
            //length()
            assertThat(name.length(), is(6));
    
            Method lengthMethod = String.class.getMethod("length");
            assertThat((Integer)lengthMethod.invoke(name), is(6));
    
            //charAt()
            assertThat(name.charAt(0), is('S'));
    
            Method charAtMethod = String.class.getMethod("charAt", int.class);
            assertThat((Character)charAtMethod.invoke(name, 0), is('S'));
        }
    }
    ```
    

#### 프록시 클래스

📌 다이내믹 프록시를 이용한 프록시를 만들어보자

- 구현할 인터페이스
    
    ```java
    public interface Hello {
        String sayHello(String name);
        String sayHi(String name);
        String sayThankYou(String name);
    }
    ```
    

- 이를 구현한 타깃 클래스
    
    ```java
    public class HelloTarget implements Hello{
        @Override
        public String sayHello(String name) {
            return "Hello "+name;
        }
    
        @Override
        public String sayHi(String name) {
            return "Hi "+name;
        }
    
        @Override
        public String sayThankYou(String name) {
            return "Thank You "+name;
        }
    }
    ```
    

- 인터페이스를 통해 HelloTarget오브젝트를 사용하는 클라이언트 역할의 테스트
    
    ```java
    public class ReflectionTest {
        ...
        @Test
        public void simpleProxy(){
            Hello hello = new HelloTarget();
            assertThat(hello.sayHello("Toby"), is("Hello Toby"));
            assertThat(hello.sayHi("Toby"), is("Hi Toby"));
            assertThat(hello.sayThankYou("Toby"), is("Thank You Toby"));
        }
    }
    ```
    

- Hello 인터페이스를 구현한 프록시 생성
    
    ```java
    public class HelloUppercase implements Hello {
        Hello hello;
    
        public HelloUppercase(Hello hello) {
            this.hello = hello;
        }
    
        @Override
        public String sayHello(String name) {
            return this.hello.sayHello(name).toUpperCase();
        }
    
        @Override
        public String sayHi(String name) {
            return this.hello.sayHi(name).toUpperCase();
        }
    
        @Override
        public String sayThankYou(String name) {
            return this.hello.sayThankYou(name).toUpperCase();
        }
    }
    ```
    
    - 프록시에는 데코레이터 패턴을 적용해서 타깃인 HelloTarget에 부가기능을 추가
        - 추가할 기능: 리턴하는 문자를 모두 대문자로 바꿔주는 것
    - HelloUppercase 프록시 → Hello 인터페이스를 구현하고 Hello 타입의 타깃 오브젝트를 받아서 저장
    - Hello 인터페이스 구현 메소드 → 타깃 오브젝트의 메소드를 호출한 뒤에 결과를 대문자로 바꿔주는 부가기능을 적용하고 리턴
    - 위임과 기능 부가라는 두 가지 프록시의 기능을 모두 처리

- 테스트 코드 추가
    
    ```java
    public class ReflectionTest {
        ...
        @Test
        public void simpleProxy(){
            Hello hello = new HelloTarget();
            assertThat(hello.sayHello("Toby"), is("Hello Toby"));
            assertThat(hello.sayHi("Toby"), is("Hi Toby"));
            assertThat(hello.sayThankYou("Toby"), is("Thank You Toby"));
            
            //프록시를 통한 타깃 접근
            Hello proxiedHello = new HelloUppercase(new HelloTarget());
            assertThat(proxiedHello.sayHello("Toby"), is("HELLO TOBY"));
            assertThat(proxiedHello.sayHi("Toby"), is("HI TOBY"));
            assertThat(proxiedHello.sayThankYou("Toby"), is("THANK YOU TOBY"));
        }
    }
    ```
    

‼️ 이 프록시의 문제점

1. 인터페이스의 모든 메소드를 구현해 위임하도록 코드를 만들어야 한다.
2. 부가기능인 리턴 값을 대문자로 바꾸는 기능이 모든 메소드에 중복돼서 나타난다.

#### 다이내믹 프록시 적용

🤔 클래스로 만든 프록시인 HelloUppercase를 다이내믹 프록시를 이용해 만들어보자!

- 다이내믹 프록시의 동작 방식
    
    ![6 3-5](https://github.com/user-attachments/assets/35a30e2a-019d-4495-9a6a-4788e53db71c)

    
- 다이내믹 프록시
    - 프록시 팩토리에 의해 런타임 시 다이내믹하게 만들어지는 오브젝트
    - 타깃의 인터페이스와 같은 타입

- 클라이언트는 다이내믹 프록시 오브젝트를 타깃 인터페이스를 통해 사용할 수 있음
    
    → 프록시를 만들 때 인터페이스를 모두 구현해가면서 클래스를 정의하는 수고를 덜 수 있지만, 부가기능을 추가하기 위해서는 직접 작성해야 함. 
    
    → 여기서 부가기능은 독립적으로 InvocationHandler를 구현한 오브젝트에 담는다. 
    
    - `public Object invoke(Object proxy, Method method, Object[] args)`

- 메소드 요청은 어떻게 처리할까?
    - 타깃 오브젝트의 메소드를 호출하게 할 수도 있다.
    - `InvocationHandler` 구현 오브젝트가 타깃 오브젝트 레퍼런스를 갖고 있다면 리플렉션을 이용해 간단히 위임 코드를 만들어 낼 수 있다.

- 모든 요청을 타깃에 위임하면서 리턴 값을 대문자로 바꿔주는 부가 기능을 가진 `InvocationHandler` 클래스
    
    ```java
    public class UppercaseHandler implements InvocationHandler {
    
        Hello target;
        // 다이내믹 프록시로부터 전달받은 요청을 다시 타깃 오브젝트에 위임해야 하기 때문에
        // 타깃 오브젝트를 주입받아 둔다.
        public UppercaseHandler(Hello target) {
            this.target = target;
        }
    
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            String ret = (String)method.invoke(target, args); // 타깃으로 위임, 인터페이스의 메소드 호출에 모두 적용된다.
            return ret.toUpperCase(); // 부가기능 제공
        }
    }
    ```
    

- `InvocationHandler` 를 사용하고 `Hello` 인터페이스를 구현하는 프록시
    
    ```java
    @Test
    public void InvocationTest(){
           
        Hello proxiedHello = (Hello) Proxy.newProxyInstance(
                getClass().getClassLoader(),
                new Class[] { Hello.class },
                new UppercaseHandler(new HelloTarget())
        );
        assertThat(proxiedHello.sayHello("Toby"), is("HELLO TOBY"));
        assertThat(proxiedHello.sayHi("Toby"), is("HI TOBY"));
        assertThat(proxiedHello.sayThankYou("Toby"), is("THANK YOU TOBY"));
    }
    ```
    
    - 사용 방법
        - 첫 번째 파라미터는 클래스 로더를 제공
            - 다이내믹 프록시가 정의되는 클래스 로더를 지정
        - 두 번째 파라미터는 다이내믹 프록시가 구현해야 할 인터페이스
            - 다이내믹 프록시는 한 번에 하나 이상의 인터페이스를 구현할 수도 있다.
            - 따라서 인터페이스의 배열을 사용
        - 마지막 파라미터로는 부가기능과 위임 관련 코드를 담고 있는 InvocationHandler 구현 오브젝트를 제공해야 함.
            - Hello 타입의 타깁 오브젝트를 생성자로 받고, 모든 메소드 호출의 리턴 값을 대문자로 바꿔주는 UppercaseHandler 오브젝트를 전달.

🤔 복잡한 다이내믹 프록시 생성 방법을 적용했는데도 코드의 양은 줄어들지 않은 것 같고 작성 역시 까다로워 진 것 같은데…

**다이내믹 프록시를 적용했을 때 장점이 있는 걸까?**

#### 다이내믹 프록시의 확장

‼️ 다이내믹 프록시 방식이 직접 정의해서 만든 프록시보다 훨씬 유연하고 많은 장점이 있다.

- Hello 인터페이스의 메소드가 3개가 아니라 30개로 늘어나면…
    - 클래스로 직접 구현한 프록시는 매번 코드를 추가해야 한다.
    - 하지만 UppercaseHandler와 다이내믹 프록시를 생성해서 사용하는 코드는 전혀 손댈 게 없다.
        - 다이내믹 프록시가 만들어질 때 추가된 메소드가 자동으로 포함될 것이고, 부가기능은 invoke() 메소드에서 처리되기 때문

- UppercaseHandler의 확장
    - 모든 메소드의 리턴 타입이 스트링이라고 가정하는데, 그 외의 리턴 타입을 갖는 메소드가 추가되면 어떨까?
    - 런타임 오류가 발생할 것이다.
    
    → 메소드를 이용한 타깃 오브젝트의 메소드 호출 후 리턴 타입을 확인해서 스트링인 경우만 대문자로 바꾸고 나머지는 그대로 넘겨주는 방식으로 수정하자!
    
    ```java
    public class UppercaseHandler implements InvocationHandler {
    
        Object target;
        public UppercaseHandler(Hello target) {
            this.target = target;
        }
    
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    
            // 호출한 메소드의 리턴 타입이 String인 경우만 대문자 변경 기능을 적용하도록 수정.
            Object ret = method.invoke(target, args);
            if(ret instanceof String) { // String으로 캐스팅이 가능하면 대문자열로 모두 치환한다.
                return ((String) ret).toUpperCase();
            } else { // 그렇지 않으면 Object 그대로 돌려준다.
                return ret;
            }
        }
    ```
    

- 메소드를 선별해서 부가기능을 적용하는 `invoke()`
    - `InvocationHandler` 는 단일 메소드에서 모든 요청을 처리하기 때문에 어떤 메소드에 어떤 기능을 적용할지 선택하는 과정이 필요할 수 있다.
    - 리턴 타입뿐 아니라 메소드의 이름도 조건으로 걸 수 있다.
        
        ```java
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        
            Object ret = method.invoke(target, args);
            // 리턴 타입과 메소드 이름이 일치할 때만 부가기능을 적용
            if(ret instanceof String && method.getName().startsWith("say")) {
                return ((String) ret).toUpperCase();
            } else { // 그렇지 않으면 Object 그대로 돌려준다.
                return ret;
            }
        }
        ```
        

### 6.3.3 다이내믹 프록시를 이용한 트랜잭션 부가기능

✨ `UserServiceTx` 를 다이내믹 프록시 방식으로 변경해보자

- 트랜잭션이 필요한 클래스와 메소드가 증가하면 프록시 클래스를 일일이 구현하는 것은 큰 부담이기 때문
- 트랜잭션 부가기능을 제공하는 다이내믹 프록시를 만들어 적용하는 방법이 효율적임

#### 트랜잭션 InvocationHandler

- 트랜잭션 부가기능을 가진 핸들러의 코드
    
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
    
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            if(method.getName().startsWith(pattern)){
                return invokeInTransaction(method, args);
            } else {
                return method.invoke(target, args);
            }
        }
    
        public Object invokeInTransaction(Method method, Object[] args) throws Throwable {
            TransactionStatus status = this.transactionManager.getTransaction(new DefaultTransactionDefinition());
            try {
                Object ret = method.invoke(target, args);
                this.transactionManager.commit(status);
                return ret;
            } catch(InvocationTargetException e) {
                this.transactionManager.rollback(status);
                throw e.getTargetException();
            }
        }
    }
    ```
    
    - 요청을 위임할 타깃을 DI로 제공받도록 한다.
    - 변수는 Object로 선언하여 `UserServiceImpl` 외에 트랜잭션 적용이 필요한 어떤 타깃 오브젝트에도 적용할 수 있다.
    - 트랜잭션 추상화 인터페이스인 `PlatformTransactionManager` 를 DI 받도록 한다.
    - 타깃 오브젝트의 모든 메소드에 무조건 트랜잭션이 적용되지 않도록 적용할 메소드 이름의 패턴을 DI 받는다.
    - `InvocationHandler` 의 `invoke()` 메소드를 구현하는 방법
        - 트랜잭션을 적용할 대상을 선별
        - DI 받은 이름 패턴으로 시작되는 이름을 가진 메소드인지 확인
        - 그렇다면 트랜잭션을 적용하는 메소드 호출, 아니라면 부가기능 없이 타깃 오브젝트의 메소드를 호출해서 결과 리턴

⇒ 이제 `UserServiceTx` 보다 코드는 복잡하지 않고 모든 트랜잭션이 필요한 오브젝트에 적용 가능한 트랜잭션 프록시 핸들러가 만들어졌다!

#### TransactionHandler와 다이내믹 프록시를 이용하는 테스트

📌 앞에서 만든 다이내믹 프록시에 사용되는 `TransactionHandler`가 `UserServiceTx`를 대신할 수 있는지 확인하기 위해 `UserServiceTest`에 적용해보자!

- 다이내믹 프록시를 이용한 트랜잭션 테스트
    
    ```java
    @Test
    public void upgradeAllorNothing() throws Exception {
        ...
        TransactionHandler txHandler = new TransactionHandler();
        txHandler.setTarget(testUserService);
        txHandler.setTransactionManager(transactionManager);
        txHandler.setPattern("upgradeLevels");
    
        UserService txUserService = (UserService) Proxy.newProxyInstance(getClass().getClassLoader(), new Class[]{UserService.class}, txHandler);
    }
    ```
    
    - `UserServiceTx` 오브젝트 대신 `TransactionHandler`를 만들고 타깃 오브젝트와 트랜잭션 매니저, 메소드 패턴을 주입
    - 준비된 `TransactionHandler` 오브젝트를 이용해 `UserService` 타입의 다이내믹 프록시를 생성하면 된다.

### 6.3.4 다이내믹 프록시를 위한 팩토리 빈

🤔 이제 `TransactionHandler`와 다이내믹 프록시를 스프링의 DI를 통해 사용할 수 있도록 해야 한다. 

→ 그런데 DI의 대상이 되는 다이내믹 프록시 오브젝트는 일반적인 스프링의 빈으로는 등록할 방법이 없다. 

→ 따라서 사전에 프록시 오브젝트의 클래스 정보를 미리 알아내서 스프링의 빈에 정의해야 한다.

- 다이내믹 프록시는 Proxy 클래스의 `newProxyInstance()`라는 스태틱 팩토리 메소드를 통해서만 만들 수 있다.

#### 팩토리 빈

- 스프링은 클래스 정보를 가지고 디폴트 생성자를 통해 오브젝트를 만드는 방법 외에도 빈을 만들 수 있는 여러 가지 방법을 제공한다.
- 대표적으로 팩토리 빈을 이용한 빈 생성 방법을 들 수 있다.

✨ 팩토리 빈

- 스프링을 대신해서 오브젝트의 생성 로직을 담당하도록 만들어진 특별한 빈
- 팩토리 빈을 만드는 방법 중 가장 간단한 방법은  `FactoryBean`이라는 인터페이스를 구현하는 것이다.

- `FactoryBean`  인터페이스
    
    ```java
    package org.springframework.beans.factory;
    
    public interface FactoryBean<T> {
     // 빈 오브젝트를 생성해서 돌려준다.
        T getObject() throws Exception;
     // 생성되는 오브젝트의 타입을 알려준다.
        Class<?> getObjectType();
     // getObject()가 돌려주는 오브젝트가 항상 같은 싱글톤 오브젝트인지 알려준다.
        boolean isSingleton();
    }
    ```
    
    - `FactoryBean` 인터페이스를 구현한 클래스를 스프링의 빈으로 등록하면 팩토리 빈으로 동작한다.

✨ 팩토리 빈의 동작원리를 확인할 수 있도록 만들어진 학습 테스트를 살펴보자

- 생성자를 제공하지 않는 클래스
    
    ```java
    public class Message {
      String text;
    
      // 생성자가 private으로 선언되어 외부에서 생성자를 통해 오브젝트를 만들 수 없다.
      private Message(String text) {
          this.text = text;
      }
    
      public String getText() {
          return text;
      }
    
      // 생성자 대신 사용할 수 있는 스태틱 팩토리 메소드 제공
      public static Message newMessage(String text) {
          return new Message(text);
      }
    }
    ```
    
    - `Message` 클래스의 오브젝트를 만들려면 `newMessage()`라는 스태틱 메소드를 사용.

- `Message` 클래스의 오브젝트를 생성해주는 팩토리 빈 클래스
    
    ```java
    public class MessageFactoryBean implements FactoryBean<Message> {
      String text;
    
      /*
       * 오브젝트를 생성할 때 필요한 정보를 팩토리 빈의 프로퍼티로 설정하여 대신 DI
       * 주입된 정보는 오브젝트 생성 중 사용됨
      */
      public void setText(String text) {
          this.text = text;
      }
    
      /*
       * 실제 빈으로 사용될 오브젝트를 직접 생성
       * 코드를 이용하므로 복잡한 방식의 오브젝트 생성과 초기화 작업도 가능
      */
      @Override
      public Message getObject() throws Exception {
          return Message.newMessage(this.text);
      }
    
      @Override
      public Class<?> getObjectType() {
          return Message.class;
      }
    
      /*
       * getObject()가 돌려주는 오브젝트가 싱글톤인지 알려준다.
       * 이 팩토리 빈은 요청할 때마다 새로운 오브젝트를 만들어주므로 false
       * 이것은 팩토리 빈의 동작방식에 관한 설정이고, 
       * 만들어진 빈 오브젝트는 싱글톤으로 스프링이 관리해줄 수 있다.
      */
      @Override
      public boolean isSingleton() {
          return false;
      }
    }
    ```
    
    - 팩토리 빈은 팩토리 메소드를 가진 오브젝트임.
    - FactoryBean 인터페이스를 구현한 클래스가 빈의 클래스로 지정되면, 팩토리 빈 클래스의 오브젝트를 `getObject()`를 이용해 가져오고, 이를 빈 오브젝트로 사용

#### 팩토리 빈의 설정 방법

- 펙토리 빈의 설정은 일반 빈과 다르지 않다.
- 다른 점은 message 빈 오브젝트 타입이 `MessageFactoryBean`이 아니라 `Message` 타입이라는 것
    - Message 빈의 타입은 `getObjectType()` 메소드가 돌려주는 타입으로 결정됨.
    - `getObject()` 메소드가 생성해주는 오브젝트가 message 빈의 오브젝트가 됨.
    
    ⇒ 정말 그런지 학습 테스트를 만들어서 확인해보자!
    
- 팩토리 빈 테스트
    
    ```java
    @RunWith(SpringJUnit4ClassRunner.class)
    // 설정파일 이름을 지정하지 않으면 클래스이름 + "-context.xml" 파일을 찾아서 사용함
    @ContextConfiguration
    public class FactoryBeanTest {
        @Autowired
        ApplicationContext context;
    
        @Test
        public void getMessageFromFactoryBean() {
            Object message = context.getBean("message");
            // 타입 확인
            // is(class) Deprecated -> is(instanceOf(class))
            assertThat(message, is(instanceOf(Message.class)));
            // 설정과 기능 확인
            assertThat(((Message)message).getText(), is("Factory Bean"));
        }
    }
    ```
    
    - message 빈의 타입이 정확히 무엇인지 확실하지 않으므로 ApllicationContext를 이용해 getBean()을 사용
        - 타입을 지정하지 않으면 Object 타입으로 리턴
    - message 빈의 타입 확인
        - getBean()이 리턴한 오브젝트는 Message 타입이어야 함
    - 기능 확인
        - MessageFactoryBean을 통해 text 프로퍼티의 값이 바르게 주입됐는지 점검

#### 다이내믹 프록시를 만들어주는 팩토리 빈

✨ 팩토리 빈을 사용하면 다이내믹 프록시 오브젝트를 스프링의 빈으로 만들어줄 수 있다.

- `getObject()` 메소드에 다이내믹 프록시 오브젝트를 만들어주는 코드를 넣으면 되기 때문

- 스프링 빈에는 팩토리 빈과 `UserServiceImpl`만 등록
- 다이내믹 프록시가 위임할 타깃 오브젝트에 대한 레퍼런스를 프로퍼티를 통해 DI 받아둬야 함.
    - 다이내믹 프록시와 함께 생설할 `TransactionHandler`에게 타깃 오브젝트를 전달해줘야 하기 때문
- 다이내믹 프록시를 직접 만들어서 `upgradeAllOrNothing()` 테스트 드를 팩토리 빈을 만들어서 `getObject()` 안에 넣어주기만 하면 됨.

#### 트랜잭션 프록시 팩토리 빈

- `TransactionHandler` 를 이용하는 다이내믹 프록시를 생성하는 팩토리 빈
    
    ```java
    public class TxProxyFactoryBean implements FactoryBean<Object> {
        // TransactionHandler 생성 시 필요한 프로퍼티
        Object target;
        PlatformTransactionManager transactionManager;
        String pattern;
        // 다이내믹 프록시를 생성할 때 필요
        // UserService 외 인터페이스를 가진 타겟에도 적용 가능
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
    
        // DI 받은 정보를 이용하여 TransactionHandelr를 사용하는 다이내믹 프록시를 생성
        @Override
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
    
        @Override
        public Class<?> getObjectType() {
            // 팩토리 빈이 생성하는 오브젝트의 타입은 DI 받은 인터페이스 타입에 따라 달라진다.
            // 다양한 타입의 프록시 오브젝트 생성에 재사용 가능
            return serviceInterface;
        }
    
        @Override
        public boolean isSingleton() {
            // 싱글톤 빈이 아니라는 뜻이 아니라 getObject()가 매번 같은 오브젝트를 리턴하지 않는다는 의미
            return false;
        }
    }
    ```
    
    - 팩토리 빈이 만드는 다이내믹 프록시는 구현 인터페이스나 타깃의 종류에 제한이 없다.
    
    → 재사용이 가능하다. (설정이 다른 여러개의 빈을 등록하면 된다.)
    

- `UserService` 에 대한 트랜잭션 프록시 팩토리 빈
    
    ```java
    <bean id="userService" class="springbook.user.service.TxProxyFactoryBean">
    	<property name="target" ref="userServiceImpl" />
    	<property name="transactionManager" ref="transactionManager" />
    	<property name="pattern" value="upgradeLevels" />
    	<property name="serviceInterface" value="springbook.user.service.UserService" />
    </bean>
    ```
    

#### 트랜잭션 프록시 팩토리 빈 테스트

🤔 기존의 `upgradeAllOrNothing()` 테스트가 문제이다.

- 스프링 빈에서 생성되는 프록시 오브젝트에 대해 테스트를 해야 하는데
- 타깃 오브젝트에 대한 레퍼런스를 `TransactionHandler` 오브젝트가 가지고 있는데 별도로 참조할 방법이 없다.

→ 그러면 어떻게 해야 할까?

- 가장 단순한 방법
    - 빈으로 등록된 `TxProxyFactoryBean`을 직접 가져와서 프록시를 만들어보면 된다.
    - 스프링 빈으로 등록된 `TxProxyFactoryBean` 을 가져와서 타깃 프로퍼티를 재구성해준 뒤에 다시 프록시 오브젝트를 생성하도록 요청한다.
    
    ```java
    public class UserServiceTest {
        // 팩토리 빈을 가져오기 위해서는 애플리케이션 컨텍스트가 필요
        @Autowired ApplicationContext context;
    
        @Test
        // 다이내믹 프록시 팩토리 빈을 직접 만들어 사용할 때는 없앴다가 다시 등잘
        @DirtiesContext
        public void upgradeAllOrNothing() throws Exception {
            TestUserService testUserService = new TestUserService(users.get(3).getId());
            testUserService.setUserDao(userDao);
            testUserService.setMailSender(mailSender);
    
            // 팩토리 빈 잧를 가져와야 하므로 '&' 필요
            // 테스트용 타깃 주입
            TxProxyFactoryBean txProxyFactoryBean = context.getBean("&userService", TxProxyFactoryBean.class);
            txProxyFactoryBean.setTarget(testUserService);
            // 변경된 타깃 설정을 이용해 트랜잭션 다이내믹 프록시를 다시 생성
            UserService txUserService = (UserService)txProxyFactoryBean.getObject();
    
            userDao.deleteAll();
            for (User user : users) {
                userDao.add(user);
            }
    
            try {
                txUserService.upgradeLvls();
                fail("TestUserServiceException expected");
            } catch (TestUserServiceException e) {
            }
    
            checkLvlUpgraded(users.get(1), false);
        }
    }
    ```
    

### 6.3.5 프록시 팩토리 빈 방식의 장점과 한계

#### 프록시 팩토리 빈의 재사용

- `TransactionHandler`를 이용하는 다이내믹 프록시를 생성해주는 `TxProxyFactoryBean`은 코드 수정 없이 다양한 클래스에 적용할 수 있다.
    - 타깃 오브젝트에 맞는 프로퍼티 정보를 설정해서 빈으로 등록하기만 하면 됨.

✨ 프록시 팩토리 빈을 이용하면 프록시 기법을 아주 빠르고 효과적으로 적용할 수 있다.

#### 프록시 팩토리 빈 방식의 장점

- 데코레이터 패턴 적용의 문제점을 해결
    1. 타깃 인터페이스를 구현하는 클래스를 일일이 만드는 번거로움 제거
    2. 부가 기능 코드의 중복 문제 해결

#### 프록시 팩토리 빈의 한계

📌 중복 없는 최적화된 코드와 설정만을 이용해 이런 기능을 적용하려고 한다면 한계에 부딪힐 것이다.

- 프록시를 통해 부가기능을 제공하는 것은 메소드 단위로 일어난다.
    1. 한 번에 여러 개의 클래스에 공통적인 부가기능 제공이 불가
        - 거의 비슷한 프록시 팩토리 빈의 설정이 중복되는 것을 방지할 수 없다.
    2. 하나의 타깃에 여러 개의 부가 기능을 적용할 때 문제
        - 비슷한 설정이 자꾸 반복되게 된다.
    3. TransactionHandler 오브젝트가 프록시 팩토리 빈 개수만큼 만들어짐
    ```
