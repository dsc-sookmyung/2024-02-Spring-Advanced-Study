# 2주차

- 스프링 테스트 컨텍스트 프레임워크
    - 용어가 너무 헷갈린다
    - 테스트 컨텍스트 프레임워크 : 스프링이 제공하는, 테스트에 사용하는 애플리케이션 컨텍스트를 생성하고 관리해주는 프레임워크
        - 동일한 구성의 애플리케이션 컨텍스트는 공유하도록 관리하는 역할
    - 테스트 컨텍스트 : 애플리케이션 컨텍스트를 생성하고 관리하기 위한 컨텍스트
    - 애플리케이션 컨텍스트 : 테스트에서 사용되는 컨테이너, 애플리케이션 컨텍스트
- @Configuration @Bean @Component @Autowired
    - @Bean - @Component
        - @Bean : 메소드 레벨의 애노테이션
            - return 객체를 빈으로 등록
            - 메소드 내부에서 객체를 생성해서 반환하는 형식
            - 개발자가 직접 제어가 불가능한 라이브러리를 사용하는 경우
        - @Component : 클래스 레벨의 애노테이션
            - 클래스가 런타임에 빈 객체로 등록
            - 개발자가 직접 생성한 클래스를 빈으로 등록하는 경우
    - @Configuration : 빈을 등록하고 있는 클래스를 알리는 것
        - 내부에 @Component를 포함하고 있는 컴포넌트의 일종
    - @ComponentScan
        - @Component를 가진 모든 클래스를 빈으로 등록
    - @Autowired
        - type → name에 따라 일치하는 빈을 자동으로 주입

- 스프링의 중요한 가치

> 1. 확장과 변화를 고려한 **객체지향**적 설계 - **IoC/DI**
2. 변화에 빠르고 유연하게 대처하기 위한 **테스트**
> 

---

### 웹을 통한 DAO 테스트의 문제점

DAO를 만든 뒤 바로 테스트 하는 것이 아니라, 모든 레이어의 기능(서비스, MVC 프레젠테이션 계층, 입출력, …)을 완성한 후 직접 웹 화면을 통해 < 값 입력 → 기능수행 → 결과확인 > 순서로 수행한다

: **문제점 : 참여하는 클래스, 코드가 너무 많다**

> → 테스트 결과에 영향을 줄 수 있는 요소가 너무 많다 (다른 계층의 코드, 컴포넌트, 서버 상태, URL 매핑, … )
→ 오류가 있을 때 빠르고 정확한 대응이 힘들다
> 

## 효율적인 테스트의 활용

: UserDaoTest를 통해 확인할 수 있는 효율적인 테스트 방법들

### 1. 작은 단위의 테스트

 **: 관심사의 분리** 

: 간단한 테스트 수행과정, 빠른 오류 원인 파악을 위해 관심사에 따라 테스트할 대상을 분리해야 한다

- **단위테스트** : 한 가지 관심에 집중할 수 있도록 작은 단위로 만들어진 테스트
    
    > - 단위를 넘어서는 **다른 코드는 참여하지 않아도 테스트가 동작**할 수 있으면 좋다
    - **통제할 수 없는 외부의 리소스**에 의존하지 않는 것이 좋다
    > 
    > 
    >   : 통제할 수 없는 DB의 상태에 의존하는 경우, MVC, 웹화면, 서버 등이 동원되는 경우, ..
    > 

- 단위테스트를 하는 이유
    - 코드가 의도대로 동작하는지 **코드 작성 직후에 스스로 확인**하기 위하여
    - 통합 테스트 이전에 각 **단위 별로 충분한 검증**을 마치는 것이 문제 원인 파악에 유리함

### 2. 자동 수행 테스트 코드

- 테스트할 **데이터가 코드를 통해 제공**되며
- 테스트 작업이 **코드를 통해 자동으로 수행**된다
    - UserDaoTest : 직접 웹 화면을 띄워 입력하는 방식이 아니라, main() 메소드 실행으로 테스트 가능
    - 클래스 안에 테스트코드를 포함하는 것 보다는 테스트용 클래스를 만드는 것이 낫다

→ **장점 : 자주 반복할 수 있다** → 코드 수정 후 빠르게 단위 테스트를 수행해 기능을 확인할 수 있다

---

### UserDaoTest의 문제점

1. **수동확인** 작업의 번거로움
    - 테스트 수행은 코드에 의해 자동 실행 되지만
    - 테스트 **결과확인은 수동**이다
2. **실행작업**의 번거로움
    - main() 메소드를 실행하는 것보다 체계적인 테스트 실행 방법이 필요하다

## UserDaoTest 개선

### 1. 테스트 검증의 자동화

- 테스트의 **결과**
    
    > **1. 성공
    2. 실패**
        2 - 1. ****테스트 **에러 :** 테스트 동안 **에러** 발생
    ****    2 - 2. 테스트 **실패 : 예상과 다른 결과** 발생
    > 

- 수정 전 코드 : 사람 눈으로 결과를 확인하도록 직접 콘솔에 출력
    - 실패한 경우 자동으로 확인할 수 있도록 자동화 필요

```java
System.out.println(user2.getName());
System.out.println(user2.getPassword());
System.out.println(user2.getUd()+" 조회 성공 ");
```

- 수정 후 코드 : 테스트 **실패 확인**의 자동화

```java
if(!user.getName().equals(user2.getName()) {
	System.out.println("테스트 실패 (name)");
}
else if(!user.getPassword().equals(user2.getPassword()) {
	System.out.println("테스트 실패 (password)");
}
else {
	System.out.println("조회 테스트 성공");
}
```

### 2. 테스트의 효율적인 수행과 결과 관리
- JUnit 테스트로의 전환

- main() 메소드를 개발자가 직접 수행하는 것보다 효율적인 수행, 결과관리가 가능
- JUnit **프레임워크**는 개발자가 만든 클래스에 대한 제어권한을 넘겨받아 **주도적으로 애플리케이션의 흐름을 제어**한다

**< JUnit 프레임워크를 적용하기 위한 메소드 조건 >**

> **1. public 메소드**
**2. @Test 애노테이션 붙이기**
> 

**<  검증코드의 전환 >**

**if/else → JUnit**의 제공방식으로 전환하기

- 수정 전 코드 : 조건에 대한 if/else

```java
if(!user.getName().equals(user2.getName()) {
	System.out.println("테스트 실패 (name)");
}
else if(!user.getPassword().equals(user2.getPassword()) {
	System.out.println("테스트 실패 (password)");
}
else {
	System.out.println("조회 테스트 성공");
}
```

- 수정 후 코드 : assertThat 이용

```java
public class UserDaoTest {
	@Test
	public void addAndGet() {
		//xml 방식
		ApplicationContext ac = new GenericXmlApplicationContext("applicationContext.xml");
		UserDao dao = ac.getBean("userDao", UserDao.class);
		User user = new User();
		//테스트 데이터
		uesr.setId("jieun");
		user.setName("김지은");
		user.setPassword("1234");

		dao.add(user);

		User user2 = dao.get(user.getId());
		//assertThat
		assertThat(user2.getName(), is(user.getName()));
		assertThat(user2.getPassword(), is(user.getPassword()));
	}
```

- **assertThat : JUnit이 제공하는 스태틱 메소드**
    - **첫 번째 파라미터** 값 - 두 번째 파라미터의 **matcher 조건**과 비교 : 일치하면 넘기고, 아니면 테스트 실패
    - **is() : equals()로 값을 비교**하는 매처의 일종
    - **assertThat**(user2.getPassword(), is(user.getPassword()))
        - user2.getPassword()의 값 - user.getPassword()의 값 equals()로 비교\

---

### 3. 테스트 결과의 일관성

<aside>
💡

- 단위 테스트는 코드에 변경사항이 없다면 **항상 동일한 결과**를 내야 한다
반복적으로 테스트 했을 때 **외부 상태에 따라 실패하기도 성공하기도** 한다면 좋은 테스트라고 할 수 없다
- 테스트를 **실행하는 순서를 바꿔도 동일한 결과**가 보장되어야 한다
</aside>

> UserDaoTest에서 **일관성과 관련된 문제점**
> 

이전 테스트 때문에 **DB에 중복 데이터**가 있을 수 있다
→ 이로 인해 동일한 테스트 결과가 나타나지 않을 수 있다
→ **테스트를 실행하기 전, 이전의 테스트가 등록한 사용자 정보를 삭제** 해서 테스트 수행에 문제되지 않는 상태로 만들어 주어야 한다

→ DB 테이블의 모든 레코드를 삭제하는 **deleteAll()**, 테이블의 레코드 개수를 반환하는 **getCount()** 메소드를 추가하고, 추가된 기능도 테스트 해보자

- **테스트 시나리오** : count()
    - **모든 레코드를 삭제하고 getCount() 결과가 0**임을 확인 : deleteAll()도 검증
    - 3개의 사용자 정보를 **하나씩 추가하며 매 번 getCount()의 호출 결과가 1씩 증가**함을 확인 : getCount의 0~3의 결과 검증
    - 추가 기능의 검증을 addAndGet()안에서 테스트 하는 것은 바람직하지 않다 : **테스트 메소드는 한 번에 한 가지 검증 목적에만 충실**한 것이 좋다

```java
@Test
public void count() throws SQLException {
	ApplicationContext ac = new GenericXmlApplicationContext("applicationContext.xml");
	
	UserDao dao = ac.getBean("userDao", UserDao.class);
	User user1 = new User("kje","김지은","spring1");
	User user2 = new User("lkw","이길원","spring2");
	User user3 = new User("pbj","박범진","spring3");
	
	//모든 레코드 삭제 후 결과 확인
	dao.deleteAll();
	assertThat(dao.getCount(), is(0));
	//하나씩 추가하며 결과 확인
	dao.add(user1);
	assertThat(dao.getCount(), is(1));
	dao.add(user2);
	assertThat(dao.getCount(), is(2));
	dao.add(user3);
	assertThat(dao.getCount(), is(3));
}
```

<aside>
💡

**주의할 점**

JUnit은 테스트 메소드의 **실행순서를 보장해주지 않는다**

→ 테스트의 **실행순서에 상관없이 독립적으로 항상 동일한 결과**를 낼 수 있도록 하자

: 다른 테스트 메소드에서의 결과, 진행과정에 의존하는 테스트 메소드를 작성하면 안 됨

</aside>

### 4. 예외조건에 대한 테스트

**>** get()메소드에 전달된 id값에 해당하는 **사용자 정보가 없는 경우**

1. **null**과 같은 특별한 값 리턴
2. id에 해당하는 값이 없음을 알리는 **예외 던지기**
    - 테스트 중 **특정한 예외가 던져지면 테스트 성공**
    - 예외 없이 **정상적으로 작업을 마치면 테스트 실패**

```java
@Test(expected=EmptyResultDataAccessException.class)
public void getUserFailure() throws SQLException {
	ApplicaitonContext ac = new GenericXmlApplicationContext("applicationContext.xml");
	UserDao dao = ac.getBean("userDao",UserDao.class);
	
	dao.deleteAll();
	assertThat(dao.getCount(), is(0));
	
	//예외발생 케이스 : 예외가 발생하지 않으면 실패
	dao.get("unknown_id");
}		
```

- **예외 발생 여부를 검증**하는 JUnit의 테스트 방식 : **expected**
    - assertThat으로는 불가능 : 메소드를 실행한 리턴 값을 비교하는 방법으로는 불가능
    - **@Test 애노테이션의 expected 엘리먼트 사용**
        - **실행 중에 발생하리라 기대하는 예외 클래스**를 넣어주면 된다
        - expected에서 지정한 **예외가 던져지면 테스트가 성공**한다

### 포괄적인 테스트

- 자신이 작성한 코드에서 발생할 수 있는 **다양한 상황, 입력 값을 고려하는 포괄적인 테스트**를 만들어야 한다
- 성공하는 테스트 뿐만 아니라, **부정적인 케이스를 먼저** 만드는 습관을 들이자
    - 존재하지 않는 데이터가 주어진 경우 : 어떻게 반응할지
    - **예외적인 상황을 빠뜨리지 않는** 꼼꼼한 개발이 가능하다

---

## 테스트 주도 개발 : TDD (Test Driven Development)

**테스트 코드를 먼저 작성하고 → 테스트를 성공하게 해주는 코드를 작성**하는 방식의 개발 방법

- **테스트코드** : 일종의 **기능 정의서**
    - **기능의 내용**을 담고 있으면서
        1. **조건 (given)** : 가져올 사용자 정보가 존재하지 않는 경우에 : dao.deleteAll()
        2. **행위 (when)** : 존재하지 않는 id로 get을 수행하면 : dao.get(”unknown_id”)
        3. **결과 (then)** : 특별한 예외가 던져진다 : @Test(expected=EmptyResultDataAccessException.class)
    - **코드를 검증**할 수 있다
    - 따라서 테스트 코드는 **구현하고 싶은 기능을 일반 언어가 아닌 코드로 표현**한 것이라고 할 수 있다
    - 테스트 코드에 맞는 기능구현 → 테스트 수행을 마치면 **코드구현+테스트검증의 작업이 동시에 끝난다**

> **장점**
> 
> - 테스트를 빼먹지 않고 꼼꼼하게 작성할 수 있다
> - 빠른 피드백을 받을 수 있다
>     - 코드를 만들고 → 테스트를 실행하는 간격이 거의 없다
>     - 따라서 **오류를 빨리 발견하고, 원인을 찾기 더 쉬워진다**
>     - 이를 위해선 **빠르게 자동으로 실행할 수 있는 단위 테스트**를 설계해야 한다
> - 빠른 오류 해결로 전체적인 개발속도는 오히려 빨라진다

---

## 추가적인 DaoTest의 개선 - 스프링 테스트 적용

### 1. @Before 적용

테스트 실행마다 반복되는 작업을 별도의 메소드로 분리

> **JUnit 프레임워크가 테스트 클래스를 가져와 테스트를 수행하는 과정**
> 
> 1. 테스트 **메소드 찾기** : **@Test가 붙은 + public이고 + void형이며 + 파라미터가 없는** 테스트 메소드 찾기
> 2. **테스트 클래스의 오브젝트 생성**
> : **각 메소드의 실행마다 테스트 클래스의 오브젝트를 새로** 만든다
> 3. **@Before** 메소드 실행
> 4. **@Test** 메소드 호출 → 결과 저장
> 5. **@After** 메소드 실행
> 6. 모든 테스트 **메소드에 대해 2~5번 반복**
> 7. 결과 종합해 반환

<aside>
💡

**중요한 점**

1. 각 **테스트 메소드를 실행할 때마다 테스트 클래스 오브젝트를 새로 만든다**
    - 각 테스트가 서로 영향을 주지 않고 **독립적으로 실행됨**을 보장한다
    - 테스트마다 **인스턴스 변수**를 부담없이 사용할 수 있다
2. **픽스처**는 **@Before 메소드**를 이용해 생성해두면 편리하다
    - **픽스처 (fixture)** : 테스트를 **수행하는 데 필요한 정보나 오브젝트**, 일반적으로 여러 테스트에서 **반복적으로 사용**된다
    - @Before, @After는 메소드에서 직접 호출하는 것이 아니므로 주고받을 정보, 오브젝트는 **인스턴스 변수를 이용**해야 한다
    - UserDaoTest의 경우에는 dao, user 등이 해당한다
</aside>

```java
public class UserDaoTest {
	// 픽스처 : 인스턴스 변수
	private UserDao dao;
	private User user1;
	private User user2;
	private User user3;
	
	@Before
	public void setUp() {
		ApplicaitonContext ac = new GenericXmlApplicationContext("applicationContext.xml");
		this.dao = ac.getBean("userDao", UserDao.class);
		
		this.user1 = new User("kje","김지은","spring1");
		this.user2 = new User("lkw","이길원","spring2");
		this.user3 = new User("pbj","박범진","spring3");
	}
}
```

### 2. 애플리케이션 컨텍스트 관리

> 테스트는 가급적 독립적으로 매번 새로운 오브젝트를 만드는 것이 원칙이지만,
> 
> 
> **ApplicationContext처럼 생성에 많은 시간, 자원이 소모되는 경우**에는 **테스트 전체가 공유하는 오브젝트**를 만들기도 한다
> 

- 문제점 : @Before 메소드에서 Application Context를 생성하므로, 메소드를 실행할 때마다 새로운 애플리케이션 컨텍스트가 생성된다
- 해결 : **여러 테스트가 함께 참조할 애플리케이션 컨텍스트**를 **스태틱** 필드에 저장하자
: **@BeforeClass** 활용 : **테스트 클래스 전체에 한 번만 실행되는 스태틱 메소드**

```java
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration(locations="/applicationContext.xml")
public class UserDaoTest {
	@Autowired
	private ApplicationContext ac;
	
	@Before
	public void setUp() {
		//...
	}
}
```

- **@RunWith** : 테스트 **실행방법을 확장**하는 애노테이션, **확장클래스를 지정**하자
    - **SpringJUnit4ClassRunner.class** : JUnit용 **테스트컨텍스트프레임워크** **확장클래스**
    : 테스트가 사용할 **애플리케이션 컨텍스트를 만들고 관리**하는 작업을 진행한다
- **@ContextConfiguration** : 자동으로 생성해줄 애플리케이션컨텍스트의 **설정파일 위치를 지정**

**< 결과 >**

1. **테스트 메소드**의 컨텍스트 공유
    - 같은 클래스 내부의 애플리케이션 컨텍스트가 모두 동일한 오브젝트이다
    - **클래스 안에서 애플리케이션 컨텍스트를 공유**한다
2. **테스트 클래스**의 컨텍스트 공유
    - **설정파일이 동일**하다면 클래스 간에도 애플리케이션 컨텍스트를 공유한다
    - 설정파일의 종류만큼 애플리케이션 컨텍스트를 만들고, 같은 설정파일을 지정한 테스트 클래스에서 이를 공유한다
        
        → 테스트 성능 향상
        

### 3. DI와 테스트

**@Autowired**

: 해당 애노테이션이 붙은 **인스턴스 변수**에 **테스트 컨텍스트 프레임워크가 애플리케이션 컨텍스트 내부의 빈을 찾아 주입**해준다(DI)

- **기준**
    1. **변수 타입**이 일치하는 빈
    2. 타입이 일치하는 빈이 중복되는 경우, **변수의 이름**이 같은 빈
    

```java
//...
public class UserDaoTest {
	@Autowired
	private ApplicationContext ac;
	@Autowired
	UserDao userdao;
	
	//..
}	
```

- **ApplicationContext의 DI**
    - 스프링은 **애플리케이션 컨텍스트를 초기화 할 때 자기자신(애플리케이션 컨텍스트)도 빈으로 등록**한다
    - 따라서 애플리케이션 컨텍스트 내부에는 **ApplicationContext 타입의 빈이 존재**한다
    → ApplicationContext와 타입이 일치하므로 빈 주입이 가능하다
- **UserDao의 DI**
    - 애플리케이션 컨텍스트 내부에 존재하는 빈을 자동으로 주입 받을 수 있다면
    **ac를 DI 받고 → ac.getBean을 통해 DL** 방식으로 의존성을 설정하는 것보다
    - **애플리케이션 컨텍스트 내부의 UserDao 빈을 직접 DI** 받자

---

## 테스트 코드에 의한 DI

애플리케이션 코드 - 테스트 코드가 서로 다른 오브젝트를 DI 받아야 하는 경우

테스트 코드에서 DI를 하는 방법을 생각해보자

### 1. 수동 DI - @DirtiesContext

```java
@DirtiesContext
public class UserDaoTest {
	@Autowired
	UserDao dao;
	
	@Before
	public void setUp() {
		//...
		DataSource dataSource = new 
		SingleConnectionDataSource("jdbc:mysql://localhost/testdb","spring","book",true);
		dao.setDataSource(dataSource);
	}
}		
```

- 이미 xml 설정파일에 맞게 설정된, 애플리케이션 컨텍스트에서 가져온 dao의 의존관계를 강제로 변경한다
- 직접 dataSource 오브젝트를 생성해서 **수정자 메소드를 통해 수동으로 DI** 한다
- 조심해야 할 점 : **클래스에서 공유**해서 사용할 **애플리케이션 컨텍스트에서 가져온 빈의 상태를 강제로 변경**한다
    
    = **애플리케이션 컨텍스트의 상태가 변경**된다
    
- **@DirtiesContext** - 클래스 레벨에 애노테이션을 붙이는 경우
    - 테스트 컨텍스트 프레임워크에게 해당 클래스에서 **애플리케이션 컨텍스트의 상태를 변경**한다고 알린다
    - 다른 테스트에서는 변경된 애플리케이션 컨텍스트가 사용되지 않도록, 이 애노테이션이 붙은 클래스에서는 애플리케이션 컨텍스틀을 공유하지 않는다
    - 테스트 메소드마다 새로운 애플리케이션 컨텍스트를 생성해서 사용한다
- 단점 : 매번 새로운 애플리케이션 컨텍스트를 만드는 비용이 든다

### 2. 테스트 전용 설정파일 사용하기

- 테스트 환경에 적합한 구성의 설정파일 생성
- 이용할 설정파일 경로만 설정해주면 됨
- 애플리케이션 컨텍스트도 한 개만 생성해 클래스 내에서 공유할 수 있다

### 3. 컨테이너 없는 DI 테스트

- 스프링 컨테이너를 활용한 DI에 의존하지 않고, 테스트 코드의 수동 DI만을 이용하기
- @RunWith, @Autowired를 사용하지 않고 UserDao와 DataSource 오브젝트를 직접 생성하고 연관관계를 설정하자

```java
public class UserDaoTest {
	UserDao dao;
	
	@Before
	public void setUp() {
		//...
		dao = new UserDao();
		DataSource dataSource = new 
		SingleConnectionDataSource("jdbc:mysql://localhost/testdb","spring","book",true);
		dao.setDataSource(dataSource);
	}
}
```

- 애플리케이션 컨텍스트를 생성하는 시간이 절약된다
- dao와 dataSource가 매 번 생성되긴 하지만, 비용이 큰 오브젝트가 아니기 때문에 문제 없다

### 이 모든 방식이 가능했던 이유

- 인터페이스를 두고 DI를 통해 주입받는 느슨한 연결관계를 설정했기 때문
- DI는 테스트가 작은 다위의 대상에 대해 독립적으로 만들어지고 실행되게 하는데 중요한 역할을 수행함
- 스프링 API에 의존하지 ㅇ낳고 자신의 관심에만 집중하여 만든 테스트이기 때문에

### DI 방식 선택하기

- 오브젝트 생성과 초기화가 **단순**하다면
    - **스프링 컨테이너 없이** 테스트 할 수 있는 방법을 우선적으로 고려
    - 속도가 빠르고 간결하기 때문
- **복잡한 의존관계**를 갖는 오브젝트를 테스트 해야한다면
    - **스프링 설정을 이용하는 DI방식**의 테스트 고려
    - 테스트 전용 환경에 맞게 설정된 **개별 설정파일**을 만들어 사용
- 테스트 설정을 따로 생성했더라도 **예외적인 의존관계를 구성**해야 하는 경우
    - 컨텍스트로 DI 받은 오브젝트에 **수동으로 DI**하는 방법 고려
    - **@DirtiesContext** 애노테이션을 붙여 애플리케이션 컨텍스트의 공유를 방지하자

RunWith로 사용하는 확장클래스 

매처 

dao - dto

Autowired의 빈 탐색기준 1. 타입 2. 변수이름

Autowired - Configuration+bean+applicationContext DI, DL

테스트 컨텍스트 프레임워크?