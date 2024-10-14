# 2장 테스트


  * [2.1 UserDaoTest 다시 보기](#21-userdaotest-다시-보기)
  * [2.2 UserDaoTest 개선](#22-userdaotest-개선)
  * [2.3 개발자를 위한 테스팅 프레임워크 JUnit](#23-개발자를-위한-테스팅-프레임워크-junit)
  * [2.4 스프링 테스트 적용](#24-스프링-테스트-적용)
  * [2.5 학습테스트로 배우는 스프링](#25-학습테스트로-배우는-스프링)
  
---------------

## 💾**복습**

> 중요한 내용 생각나는 대로 정리
>  - 객체는 오브젝트, 스프링에서는 @Bean 어노테이션이 있어야 함. 이게 없는 객체도 객체긴하지만, 스프링의 IoC 원칙과 DI 따르지 않는다. (의존관계 검색은 가능)
>
>  - 관심사 분리
> 관심사가 다른 코드는 서로 다른 오브젝트로 분리해야 함.
> 확장은 잘되고, 변화는 없어야 함.
>  - 낮은 결합도 : 다른 기능을 담당하면 서로 영향을 주지 않음
>  - 높은 응집도 : 하나의 기능을 바꾸기 위해 관련있는 코드 전체를 바꿈 -> 어디를 변경해야하는지 찾지 않아도 됨
>
>  - IoC (제어의 역전)
> 오브젝트가 스스로 언제쓰일지, 언제 생성될지를 결정하지 못하고 알지도 못함. 스프링 같은 IoC 프레임워크가 결정
>  - 팩토리 : 코드의 설계도, 설정 정보를 담당하게 됨
>  - 애플리케이션 컨텍스트 : 빈 팩토리와 거의 유사, 애플리케이션 전체의 제어를 담당하는 IoC 오브젝트
>  - @Configuration
>
>  - 싱글톤 레지스트리
>  - 싱글톤 : 한 번에 하나의 오브젝트만 생성 -> 멀티 스레드 환경에선 위험
> 그래서 스프링에서 싱글톤 레지스트리라는 것을 만듦
> -> 스태틱 메소드와 private 생성자 사용 안해도 됨
> -> IoC 방식으로 컨테이너에게 제어권을 넘김
> -> 테스트에 유리
>
>  - DI (의존관계 주입)
> A가 B에 의존함 -> A는 B의 영향을 받지만, B는 A의 영향을 안 받음
> 주로 생성자를 통해 이뤄짐. setter 방식도 있음. 인터페이스 방식의 파라미터로 이루어짐
>  - 런타임 의존관계
> 코드 상에선 의존관계가 보이지 않지만, 런타임에서 의존관계가 나타남.
> 주로 제 3의 존재가 결정. (UserDao가 DConnectionFactory랑 런타임 의존관계를 가짐)
>  - 의존 관계 검색 : getBean() 같은 직접 오브젝트가 의존을 요청함. (생성과 결정은 외부 컨테이너가 함).
> 권장하지는 않지만 필수적으로 한 번 해야함
>

## 2.1 UserDaoTest 다시 보기

  

### 2.1.2 UserDaoTest의 특징

- 웹을 통한 Dao 테스트 방법의 문제점

Dao 기능 뿐만 아니라 모든 입출력 기능을 만든 후 웹 화면을 통해 값을 입력하고, 기능을 수행하고, 결과를 확인함

▶️ Dao에 대한 기능만 뿐만 아니라 다른 기능도 다 만들고 테스트를 진행해야하기 때문에 에러가 나도 어디서 문제가 발생했는지 찾기 어렵고 느림.

  

- 작은 단위의 테스트

💡 단위 테스트 : 작은 단위의 코드에 대해 테스트를 수행한 것

통제할 수 없는 외부의 리소스에 의존하면 안됨.

개발자가 설계하고 만든 코드가 원래 의도한 대로 동작하는지를 개발자 스스로 확인하기위해 사용

- 자동수행 테스트 코드

UserDaoTest 코드는 main()의 실행만으로 값 입력없이 자동으로 테스트가 수행됨.

테스트 진행 시간도 매우 짧음

▶️ 자주 반복할 수 있고, 수정에도 용이함

  

### 2.1.3 UserDaoTest의 문제점

- 수동 확인 작업의 번거로움

입력한 값과 가져온 값이 일치하는지를 사람의 눈으로 확인해야함.

- 실행 작업의 번거로움

main() 메소드가 너무 많아지면 부담.

  

## 2.2 UserDaoTest 개선

### 2.2.1 테스트 검증의 자동화

add()에 전달한 User 오브젝트에 담긴 사용자 정보와 get()을 통해 조회한 정보가 서로 일치하는가

  

``` if (!user.getName().equals(user2.getName())) ```

  

를 이용해 테스트를 실행하면 테스트 성공 혹은 실패를 띄움

  

### 2.2.2 테스트의 효율적인 수행과 결과 관리

  

- JUnit 테스트로 전환

JUnit은 프레임워크 -> 제어의 역전 -> main() 메소드나 오브젝트를 만들어서 실행시키는 코드 필요없음

- 테스트 메소드 전환

1. Mmain() 메소드는 제어권을 직접 갖는다는 의미 -> 일반 메소드로 옮김

2. JUnit 프레임워크가 요구하는 조건

1️⃣ public으로 선언

2️⃣ @Test 어노테이션

```java

@Test

public  void  andAndGet() throws SQLException {

ApplicationContext  context = new  GenericXmlApplicationContext("applicationContext.xml");

UserDao  dao = context.getBean("userDao", UserDao.class);

```

- 검증 코드 전환

- if 문장의 기능을 JUnit이 제공해주는 assertThat으로 변경

``assertThat(user2.getName(), is(user.getName()));``

is()는 매처의 일종

- JUnit 테스트 실행

- main() 메소드로 JUnit 프레임워크를 실행해줘야 함.

```

public static void main(String[] args) {

JUnitCore.main("springbook.user.dao.UserDaoTest");

}

```

위에 코드 실행하면 테스트 자동 실행해줌

## 2.3 개발자를 위한 테스팅 프레임워크 JUnit

### 2.3.1 JUnit 테스트 실행 방법

JUnitCore을 이용한 테스트는 테스트 수가 많아지면 관리하기 힘들다.

- IDE

이클립스(자바 IDE)에서 @Test가 있는 테스트 클래스나 패키지를 선택한 뒤 Run As->JUnit Test 실행하면 자동으로 실행하고 결과도 보여줌

- 빌드 툴 : 메이븐이나 ANT에서 제공하는 JUnit 플러그인/태스크 이용

### 2.3.2 테스트 결과의 일관성

문제점 : 테스트가 DB의 상태에 영향을 받는다.

UserDaoTest는 DB에 값이 이미 있다면 오류가 남 -> 테스트를 마치고 나면 사용자 정보를 삭제해준다.

- deleteAll()의 getCount() 추가

```java

public  void  deleteAll() throws SQLException {

Connection  c = dataSource.getConnection();

PreparedStatement  ps = c.prepareStatement("delete from users");

ps.executeUpdate();

  

ps.close();

c.close();

}

```

🔼user 테이블의 모든 레코드 삭제

```java

public  int  getCount() throws SQLException {

Connection  c = dataSource.getConnection();

PreparedStatement  ps = c.prepareStatement("select count(*) from users");

  

ResultSet  rs = ps.executeQuery();

rs.next();

int  count = rs.getInt(1);

  

rs.close();

ps.close();

c.close();

return count;

}

```

🔼user 테이블의 레코드 개수 반환

  

- deleteAll()의 getCount() 테스트

UserDaoTest를 확장시켜 여기에 추가해서 기능을 검증해봄

1. deleteAll()을 처음에 실행시켜 테이블의 모든 레코드 삭제

2. getCount()로 deleteAll()을 검증 -> 0이 나온다면 통과

3. add() 실행 후 getCount()를 한 번 더 실행해 getCount() 검증 -> 1이 나온다면 통과

  

```java

public  class  UserDaoTest {

@Test

public  void  andAndGet() throws  SQLException {

ApplicationContext  context = new  GenericXmlApplicationContext("applicationContext.xml");

UserDao  dao = context.getBean("userDao", UserDao.class);

dao.deleteAll();

assertThat(dao.getCount(), is(0)); // 검증

User  user = new  User();

user.setId("gyumee");

user.setName("박성철");

user.setPassword("springno1");

  

dao.add(user);

assertThat(dao.getCount(), is(1)); // 검증

User  user2 = dao.get(user.getId());

assertThat(user2.getName(), is(user.getName()));

assertThat(user2.getPassword(), is(user.getPassword()));

}

}

```

- 동일한 결과를 보장하는 테스트

단위 테스트는 코드가 바뀌지 않는다면 매번 실행할 때마다 동일한 테스트 결과를 얻을 수 있어야 한다.

  

### 2.3.3 포괄적인 테스트

getCount() 메소드가 진짜 정상적으로 동작하는지 알아보기 위해 추가적인 테스트 필요 (add()를 두 번 실행했을 때)

- getCount() 테스트 : UserDaoTest안에 @Test를 따로 만듦

1. User 테이블의 데이터를 모두 지움

2. GetCount()로 레코드 개수가 0임을 확인

3. 3개의 사용자 정보를 추가하면서 getCount()의 값이 하나씩 증가하는지 확인

  

📍User 생성자

  

```java

// 자바빈 규약 - 파라미터가 없는 생성자

public  User() {

}

public  User(String id, String name, String password) {

this.id = id;

this.name = name;

this.password = password;

}

```

  

이후 user 생성이 더욱 쉬워짐

📍getCount() 테스트

  

~~~ java

@Test

public  void  count() throws SQLException {

ApplicationContext  context = new  GenericXmlApplicationContext("applicationContext.xml");

UserDao  dao = context.getBean("userDao", UserDao.class);

User  user1 = new  User("gyumee", "박성철", "springno1");

User  user2 = new  User("leegw700", "이길원", "springno2");

User  user3 = new  User("bumjin", "박범진", "springno3");

dao.deleteAll();

assertThat(dao.getCount(), is(0));

dao.add(user1);

assertThat(dao.getCount(), is(1));

dao.add(user2);

assertThat(dao.getCount(), is(2));

dao.add(user3);

assertThat(dao.getCount(), is(3));

}

~~~

테스트 둘 중 어느 것이 먼저 실행될지 모름 -> 테스트 실행 순서에 영향을 받지 않음

  

- addAndGet() 테스트 보완

get()에 대한 검증 필요

▶️ 두 개의 user를 add()하고, 각 user의 id를 파라미터로 전달해서 get()을 실행

~~~java

@Test

public  void  andAndGet() throws SQLException {

ApplicationContext  context = new  GenericXmlApplicationContext("applicationContext.xml");

UserDao  dao = context.getBean("userDao", UserDao.class);

  

User  user1 = new  User("gyumee", "박성철", "springno1");

User  user2 = new  User("leegw700", "이길원", "springno2");

dao.deleteAll();

assertThat(dao.getCount(), is(0));

  

dao.add(user1);

dao.add(user2);

assertThat(dao.getCount(), is(2));

User  userget1 = dao.get(user1.getId());

assertThat(userget1.getName(), is(user1.getName()));

assertThat(userget1.getPassword(), is(user1.getPassword()));

User  userget2 = dao.get(user2.getId());

assertThat(userget2.getName(), is(user2.getName()));

assertThat(userget2.getPassword(), is(user2.getPassword()));

}

~~~

  

- get() 예외조건에 대한 테스트

Id에 해당하는 사용자 정보가 없다면?

▶️ 주어진 id에 해당하는 정보가 없다는 의미를 가진 예외 클래스를 던짐

스프링의 EmptyResultDataAccessException 사용

이런 경우엔 예외가 발생해야 테스트 성공 -> assertThat으로 검증을 못함

▶️ JUnit의 expected : 발생하길 기대하는 예외가 발생하면 테스트 성공

~~~ java

@Test(expected=EmptyResultDataAccessException.class)

public  void  getUserFailure() throws SQLException {

dao.deleteAll();

assertThat(dao.getCount(), is(0));

dao.get("unknown_id");

}

~~~

기존 UserDao 코드는 rs.next()에서 가져올 로우가 없다는 예외가 발생

~~~ java

User  user = null;

if (rs.next()) {

user = new  User();

user.setId(rs.getString("id"));

user.setName(rs.getString("name"));

user.setPassword(rs.getString("password"));

}

rs.close();

ps.close();

c.close();

// user값이 null이면 예외 발생

if (user == null) throw  new  EmptyResultDataAccessException(1);

  

return user;

~~~

- 포괄적인 테스트

네거티브 테스트를 피하지 말고 우선적으로 만들어야함

  

### 2.3.4 테스트가 이끄는 개발

테스트를 먼저 만들어 테스트가 실패하는 것을 보고 코드 수정

  

- 기능설계를 위한 테스트

추가하고 싶은 기능을 테스트 코드로 표현

1. 가져올 사용자 정보가 존재하지 않는 경우에

2. 존재하지 않은 id로 get을 실행하면

3. 특별한 예외가 던져진다

  

- 테스트 주도 개발 TDD

테스트 코드를 먼저 만들고, 테스트를 성공하게 해주는 코드를 작성하는 방식의 개발 방법, 테스트 우선 개발이라고도 함

- 테스트 코드를 꼼꼼하게 만들고, 코드 작성 후 테스트를 바로해 즉각적인 피드백 가능

- 단위 테스트로 테스트 코드와 기능 구현 사이의 간격이 짧을수록 좋음

### 2.3.5 테스트 코드 개선

코드의 중복 ▶️ JUnit의 @Before로 해결

💡 매번 테스트 메소드를 실행하기 전에 먼저 실행시켜줌

Dao는 테스트 메소드에서 접근할 수 있도록 인스턴스 변수로 선언

~~~java

private  UserDao  dao;

@Before

public  void  setUp() {

ApplicationContext  context = new  GenericXmlApplicationContext("applicationContext.xml");

this.dao = context.getBean("userDao", UserDao.class)

}

~~~

JUnit은 @Test가 붙은 메소드를 실행하기 전과 후에 @Before와 @After가 붙은 메소드를 자동으로 실행.

메소드를 실행할 때마다 테스트 클래스의 오브젝트를 새로 만듦!

❓테스트 클래스마다 하나의 오브젝트만 만들어놓는 것이 성능은 좋지만, 테스트가 서로 영향을 받지 않고 독립적으로 실행됨을 확실히 보장해주기 위해 매번 새로운 오브젝트를 만들었다.

  

- 픽스처

테스트를 수행하는 데 필요한 정보나 오브젝트

-> Dao

~~~java

private  User  user1;

private  User  user2;

private  User  user3;

@Before

public  void  setUp() {

ApplicationContext  context = new  GenericXmlApplicationContext("applicationContext.xml");

this.dao = context.getBean("userDao", UserDao.class);

this.user1 = new  User("gyumee", "박성철", "springno1");

this.user2 = new  User("leegw700", "이길원", "springno2");

this.user3 = new  User("bumjin", "박범진”, "springno3");

}

~~~

## 2.4 스프링 테스트 적용

@Before 메소드가 3번 반복 -> 애플리케이션 컨텍스트로 3번 만들어짐

▶️ 테스트 전체가 공유하는 오브젝트를 만드는게 나음

### 2.4.1 테스트를 위한 애플리케이션 컨텍스트 관리

애노테이션 설정으로 테스트에서 필요로 하는 애플리케이션 컨텍스트를 만들어서 모든 테스트가 공유할 수 있음

  

- 스프링 테스트 컨텍스트 프레임워크 적용

~~~java

// JUnit이 테스트를 진행하는중에 테스트가 사용할 애플리케이션 컨텍스트를 만들고 관리하는 작업을 진행

@RunWith(SpringJUnit4ClassRunner.class)

// 애플리케이션 컨텍스트의 설정파일 위치 지정

@ContextConfiguration(locations="/applicationContext.xml")

public  class  UserDaoTest {

@Autowired

ApplicationContext  context;

. . .

@Before

public  void  setUp() {

this.dao = this.context.getBean("userDao", UserDao.class);

. . .

}

~~~

  

- 테스트 메소드의 컨텍스트 공유

Context는 모두 동일하고, UserDaoTest의 오브젝트는 매번 주소 값이 다름

테스트 실행되기 전에 한 번만 애플리케이션 컨텍스트를 만들어두고, 테스트 오브젝트가 만들어질 때마다 자신을 테스트 오브젝트의 특정 필드에 주입. 일종의 DI

  

- 테스트 클래스의 컨텍스트 공유

같은 설정파일을 가진 애플리케이션 컨텍스트를 사용한다면, 클래스 사이에서도 애플리케이션 컨텍스트를 공유하게 해줌

-> 성능 향상

- @Autowired

스프링 DI에 사용되는 특별한 애노테이션

💡 타입에 의한 자동와이어링: @Autowired가 붙은 인스턴스 변수가 있으면, 테스트 컨텍스트 프레임워크는 변수 타입과 일치하는 컨텍스트 내의 빈을 찾아 주입해줌 (생성자나 수정자 메소드 없어도 됨)

- 애플리케이션 컨텍스트 자체도 빈 -> DI 가능

- UserDao도 애플리케이션 컨텍스트에 등록되어있는 빈 -> getBean() 필요없이 DI 받을 수 있음 -> ApplicationContext를 DI 안 받아도 됨

- 같은 타입의 빈이 두 개 이상 있는 경우 변수의 이름과 같은 이름의 빈을 주입

- 인터페이스 타입으로 주입 가능

### 2.4.2 DI와 테스트

DataSource의 구현 클래스를 바꾸지 않아 사용해도 인터페이스를 사용해 DI주입을 해야하는 이유

1. 언젠가 변경이 필요한 상황이 생길 수 있음

2. 부가 기능 추가 (DB 개수 카운팅 기능)

3. 테스트 : 가능한 작은 단위의 대상에 국한해서 테스트 해야 함

- 테스트 코드에 의한 DI

수정자 메소드는 테스트 코드 내에서 이를 이용하여 DI 해도 됨

->테스트용 DB에 연결해주는 DataSource를 테스트 내에서 직접 만듦

  

~~~ java

// 테스트 메소드에서 애플리케이션 컨텍스트의 구성이나 상태를 변경한다는 것을 테스트 컨텍스트 프레임워크에 알려준다.

@DirtiesContext

public  class  UserDaoTest {

@Autowired

UserDao  dao;

  

@Before

public  void  setUp() {

// 테스트에서 UserDao가 사용할 DataSource 오브젝트를 직접 생성한다.

DataSource  dataSource =new  SingleConnectionDataSource( "jdbc:mysql://localhost/testdb", "spring", "book", true);

dao.setDataSource(dataSource); // 코드에 의한 수동 DI

  

}

~~~

  

애플리케이션 컨텍스트에서 가져온 UserDao의 빈 관계를 강제로 변경 -> 나머지 모든 테스트를 하는 동안 변경된 내용 사용됨

▶️ @DirtiesContext : 해당 클래스의 테스트에서 애플리케이션 컨텍스트의 상태를 병경한다는 것을 알려줌 -> 테스트 컨텍스트는 이 클래스에는 애플리케이션 컨텍스트 공유를 허용하지 않고 메소드 수행 후 매번 새로운 애플리케이션 컨텍스트를 만듦

  

- 테스트를 위한 별도의 DI 설정

문제점 : 수동으로 DI하는 방법은 애플리케이션 컨텍스트를 매번 새로 만들어야 하고 코드도 많아짐

▶️ 테스트 전용 설정 파일을 따로 만듦 (db만 테스트용으로 바꿈)

UserDaoTest의 @ContextConfiguration 애노테이션에 있는 locations 엘리먼트의 값을 새로 만든 테스트용 설정파일로 변경해준다.

  

- 컨테이너 없는 DI 테스트

~~~java

public  class  UserDaoTest {

UserDao  dao; // @Aulowired가 없다.

  

@Before

public  void  setUp() {

. . .

dao = new  UserDao();

DataSource  dataSource = new  SingleConnectionDataSource( "jdbc: mysql://localhost/testdb", "spring", "book", true);

dao.setDataSource(dataSource); //오브젝트 생성,관계설정 직접함

}

~~~

  

- 코드는 단순해지고 이해하기 편해짐

- 테스트 시간 절약됨

- 매번 새로운 UserDao 오브젝트가 만들어짐 그치만 가벼운 오브젝트

  

- DI를 이용한 테스트 방법 선택

스프링 컨테이너 없는 방법을 우선적으로 고려

복잡한 의존관계를 갖고 있는 오브젝트라면 스프링의 설정을 이용한 DI 방식 테스트

  

## 2.5 학습테스트로 배우는 스프링

- 학습테스트 : 자신이 사용한 api나 프레임워크의 기능을 테스트로 보면서 사용 방법을 익히려는 것

### 2.5.1 학습테스트의 장점

수동으로 예제를 만드는 것에 비해 무엇이 좋은지

- 다양한 조건에 따른 기능을 손쉽게 확인해볼 수 있다. (자동화)

- 학습 테스트 코드를 개발 중에 참고할 수 있다. (기록이 남음. 샘플 코드)

- 프레임워크나 제품을 업그레이드할 때 호환성 검증을 도와준다 (학습테스트에만 새로운 버전을 적용해본 후 도입)

- 테스트 작성에 대한 좋은 훈련이 된다 (단순)

- 새로운 기술을 공부하는 과정이 즐거워진다

  

### 2.5.2 학습 테스트 예제

- JUnit 테스트 오브젝트 테스트

JUnit이 정말 테스트 메소드를 수행할 때마다 새로운 오브젝트를 만드는지 확인

~~~ java

public  class  JUnitTest {

static  Set<JUnitTest> testObjects = new  HashSet<JUnitTest>();

@Test  public  void  test1() {

assertThat(testObjects, not(hasItem(this)));

testObjects.add(this);

}

@Test  public  void  test2() {

assertThat(testObjects, not(hasItem(this)));

testObjects.add(this);

}

@Test  public  void  test3() {

assertThat(testObjects, not(hasItem(this)));

testObjects.add(this);

}

}

~~~

HasItem() : 컬렉션의 원소인지를 검사

  

- 스프링 테스트 컨텍스트 테스트

스프링의 테스트용 애플리케이션 컨텍스트는 항상 한 개만 만들어짐

- 새로운 설정 파일을 등록. 빈 등록 x

~~~ java

@RunWith(SpringJUnit4ClassRunner.class)

@ContextConfiguration

public  class  JUnitTest {

@Autowired  ApplicationContext  context;

static  Set<JUnitTest> testObjects = new  HashSet<JUnitTest>();

static  ApplicationContext  contextObject = null;

@Test  public  void  test1() {

assertThat(testObjects, not(hasItem(this)));

testObjects.add(this);

assertThat(contextObject == null || contextObject == this.context, is(true));

contextObject = this.context;

}

@Test  public  void  test2() {

assertThat(testObjects, not(hasItem(this)));

testObjects.add(this);

assertTrue(contextObject == null || contextObject == this.context);

contextObject = this.context;

}

@Test  public  void  test3() {

assertThat(testObjects, not(hasItem(this)));

testObjects.add(this);

assertThat(contextObject, either(is(nullValue())).or(is(this.contextObject)));

contextObject = this.context;

}

}

~~~

contextObject가 null이면 첫 번째 테스트 -> 현재 context 저장 -> 같은 오브젝트인지 검증

1. assertThat() : 매처와 비교할 대상인 첫 번째 파라미터에 boolean 타입의 결과가 나오는 조건문을 넣음. 그 결과를 is() 매처를 써서 비교

2. AssertTrue() : 조건문이 참인지

3. either()와 or()를 사용해 or 연산 수행. 둘 중 하나만 참이여도 성공

  

### 2.5.3 버그 테스트

일단 실패하게 만들고 성공할 수 있도록 애플리케이션 코드를 수정

- 테스트의 완성도를 높여준다.

- 버그의 내용을 명확하게 분석하게 해준다. (실패하게 만들기 위해 분석)

- 기술적인 문제를 해결하는 데 도움이 된다.
