# 1장 오브젝트와 의존관계
- 오브젝트의 설계 
	- 객체지향 설계
	- 디자인 패턴
	- 리팩토링
	- 단위 테스트

## 1.1 초난감 DAO : 앞으로 책에서 이 코드를 수정해가며 내용을 배움
-  DAO : Data access object, db를 사용해 **데이터를 조회하거나 조작**하는 기능을 전담하도록 만든 오브젝트
### 1.1.1 User
사용자 정보를 저장할 때는 자바빈 규약을 따르는 오브젝트를 이용하면 편리하다. (domain)
- 자바빈 : 빈이라고도 부름. 밑에 두가지를 가지고 있음
	- 디폴트 생성자 : 자바빈은 파라미터가 없는 **디폴트 생성자**를 갖고 있어야 한다.
	- 프로퍼티 : 자바빈이 노출하는 이름을 가진 속성을 프로퍼티라고 한다. setter, getter
	
### 1.1.2 UserDao
- DAO안에 담겨야하는 JDBC의 과정
1. db 연결 (db 주소를 입력해서 DriverManager.getConnection 메소드로 연결)
2. SQL 문 담은 statement (쿼리)
3. 실행 (파라메터에 값 넣어서), 조회는 ResultSet 받아서 오브젝트로 옮김 (User user = new User())
4. 다 닫아줌
5. 예외는 throws로

### 1.1.3 main()을 이용한 테스트 코드
위 코드의 문제점을 혼자 생각해보기.
1. User의 속성을 private으로 설정하지 않았다.
2. 어노테이션을 통해 각각의 속성의 특징을 설정하지 않았다.
3. getter와 setter는 어노테이션을 통해 생략가능함.
4. db 연결도 DAO 클래스 외부에서 해주면 좋지 않을까?

## 1.2 DAO의 분리
### 1.2.1 관심사의 분리
미래의 변화를 어떻게 대비할 것인가. -> 디자인 패턴?
- 분리와 확장
	- 모든 변경과 발전은 한 번에 한 가지 관심사항에 집중해서 일어난다.
	-> 관심이 같은 것끼리는 같은 데에 모은다.

관심이 같은 것끼리는 하나의 객체 안으로 또는 친한 객체로 모이게 하고, 다른 것은 떨어져서 서로 영향을 주지 않도록 분리.

### 1.2.2. 커넥션 만들기의 추출
1. UserDao의 DB 연결을 위한 connection 오브젝트를 가져오는 부분
	- 다른 관심사와 섞여서 add()에 있음.
	- add()와 get()에 같은 코드가 중복
	-> 중복 코드의 **메소드 추출** : getConnection 메소드를 따로 private으로 선언하여 코드에서 불러오도록 함.
	- User 테이블의 값을 초기화하고 main() 테스트 코드를 실행하면 정상 작동 (기능에는 바뀐 점이 없기 때문 -> 이걸 **리팩토링**이라고 함)

### 1.2.3 DB 커넥션 만들기의 독립
요구사항 : UserDao의 소스코드를 제공하지 않고 고객 스스로 원하는 DB 커넥션 생성 방식을 적용해가며 UserDao를 사용할 수 있게 하자.
- 상속을 통한 확장
	1. getConnection()을 추상 메소드로 만들어 추상 클래스인 UserDao를 판매함. 
	2. UserDao를 상속받아 서브 클래스를 만들어 getConnection() 메소드를 원하는 대로 확장
	
	코드 참조

        // UserDao
        public abstract protected Connection getConnection() throws ClassNotFoundException, SQLException ;
         
        // 상속 받은 서브 클래스
        public class NUserDao extends UserDao {
    	protected Connection getConnection() throws ClassNotFoundException,
    			SQLException {
    		Class.forName("com.mysql.jdbc.Driver");
    		Connection c = DriverManager.getConnection(
    				"jdbc:mysql://localhost/springbook?characterEncoding=UTF-8",
    				"spring", "book");
    		return c;
    	}
    
    - 템플릿 메소드 패턴 : 슈퍼 클래스에 기본적인 로직의 흐름을 만들고, 그 기능의 일부를 추상 메소드나 오버라이딩이 가능한 protected 메소드 등으로 만든 뒤 서브 클래스에서 이런 메소드를 필요에 맞게 구현해서 사용하도록 하는 방법
	    - 변하지 않는 기능은 슈퍼클래스에, 자주 변경되며 확장할 기능은 서브 클랙스에서 만들도록 함.
	    - 훅 메소드 : 선택적으로 오버라이드 가능 (protected)
	    - 추상 메소드 : 반드시 구현해야 함. (abstract)
    - **팩토리 메소드 패턴** : 서브 클래스에서 구체적인 오브젝트 생성 방법을 결정 (Connection 오브젝트를 사용하는 것만 같음).

> 🔼 더 공부가 필요할듯!!

- 문제점
	- 자바는 클래스의 다중 상속을 허용하지 않아 만약 UserDao가 다른 클래스를 상속 받고 있다면 사용할 수 없음
	- 상속은 상하위 클래스가 밀접한 관계를 가짐
	- 다른 DAO 클래스에서 connection 메소드를 사용할 수 없음

## 1.3 DAO의 확장
### 1.3.1 클래스의 분리
독립된 클래스로 분리

    public abstract class UserDao {
	private SimpleConnectionMaker simpleConnectionMaker;
	
	public UserDao() {
		this.simpleConnectionMaker = new SimpleConnectionMaker();
	}
UserDao에서 new를 사용하여 생성자에서 오브젝트를 생성

    public class SimpleConnectionMaker {
    	public Connection getConnection() throws ClassNotFoundException,
    			SQLException {
    		Class.forName("com.mysql.jdbc.Driver");
    		Connection c = DriverManager.getConnection(
    				"jdbc:mysql://localhost/springbook?characterEncoding=UTF-8", "spring", "book");
    		return c;
    	}
    }

추상 클래스로 만들 필요 없음

- 문제점 
	- UserDao의 소스코드를 제공해야함 (simpleConnectionMaker = new SimpleConnectionMaker();)
	- Connection c = this.simpleConnectionMaker.getConnection(); 에서 getConnection의 이름을 다르게 한다면 UserDao 코드 수정해야함

### 1.3.2 인터페이스 도입
두 개의 클래스가 긴밀하게 연결되어 있지 않도록 중간에 추상적인 느슨한 연결고리인 인터페이스 도입! -> 자바의 추상화 도구
인터페이스에 최소한의 정보만 담아서 주는거 : 어떤 일을 하겠다. 구현 방법은 없음

- 인터페이스를 통해 오브젝트에 접근 -> 서브 클래스에서 메소드 이름이 변경되지 않음
- 문제점 : UserDao 클래스에 서브클래스 이름이 나옴 -> UserDao를 수정해야하는 일이 생김

### 1.3.3 관계설정 책임의 분리
- UserDao와 다른 관심사
new DConnectionMaker() : UserDao가 어떤 ConnectionMaker 구현 클래스의 오브젝트를 이용하게 할지를 결정하는 관심사를 갖고 있음
-> 분리해야함 -> UserDao의 클라이언트 오브젝트
- UserDao가 인터페이스 외에 어떤 클래스와도 관계를 가지면 안됨. 위에 코드는 불필요한 의존 관계를 가진 구조.
- 오브젝트끼리 관계를 가져야 함. 오브젝트 사이의 관계는 코드에서 특정 클래스를 알지 못해도 클래스가 구현한 인터페이스를 사용했다면, 그 클래스의 오브젝트를 인터페이스 타입으로 사용할 수 있다 (객체지향 프로그램의 다형성)
- 런타임시 오브젝트 만들어지며 관계 형성
-  결론 : UserDao의 클라이언트 (여기선 main() 메소드 = UserDaoTest) 에서 ConnectionMaker의 구현 클래스를 선택하고, UserDao에선 생성자에 파라메터를 추가해 클래스를 ConnectionMaker의 인터페이스 방식으로 호출한다.
즉,  **UserDaoTest에서 UserDao를 호출할때 ConnectionMaker를 선택할 수 있도록 한다.**  
> 다시 봐야할 페이지 : 78p

### 1.3.4 원칙과 패턴
- 개방 폐쇄 원칙 : 클래스나 모듈은 확장에는 열려 있어야 하고 변경에는 닫혀 있아야 한다. (객체지향 설계 원칙 5가지 중 하나)
- 높은 응집도 : 하나의 모듈, 클래스가 하나의 책임, 관심사에만 집중되어있음
	- 변경이 일어날 때 모듈의 많은 부분이 함께 바뀜.
- 낮은 결합도 : 하나의 변경이 발생할 때 마치 파문이 이는 것처럼 여타 모듈과 객체로 변경에 대한 요구가 전파되지 않는 상태를 말한다.
- 전략 패턴 : 자신의 기능 맥락(UserDao)에서, 필요에 따라 변경이 필요한 알고리즘을 인터페이스를 통해 통째로 외부로 분리시키고(ConnectionMaker), 이를 구현한 구체적인 알고리즘 클래스를 필요에 따라 바꿔서 사용(UserDaoTest, 생성자)할 수 있게 하는 디자인 패턴이다. 

## 1.4 제어의 역전(IoC)
### 1.4.1 오브젝트 팩토리
UserDaoTest의 기능 분리
1. UserDao와 ConnectionMaker 구현 클래스의 오브젝트를 만드는 것
2. 두 개의 오브젝트가 연결돼서 사용될 수 있도록 관계를 맺어주는 것

- 팩토리 : 객체의 생성 방법 결정
	- DaoFactory에 UserDaoTest에 있었던 생성 작업을 옮겨주고 userDao를 반환.
	- 어떤 오브젝트가 어떤 오브젝트를 사용하는지를 정의해놓은 설계도
	- 애플리케이션의 컴포넌트 역할을 하는 오브젝트 (UserDao, ConnectionMaker)와 애플리케이션의 구조를 결정하는 오브젝트 (DaoFactory)를 분리

### 1.4.2 오브젝트 팩토리의 활용
- 문제점 : 여러 개의 Dao를 DaoFacotry에서 생성한다면, ConnectionMaker의 구현 코드가 중복됨
-> ConnectionMaker의 구현 클래스를 결정하고 오브젝트를 만드는 코드를 별도의 메소드로 뽑아냄

        public class UserDaoFactory {
	    	public UserDao userDao() {
	    		UserDao dao = new UserDao(connectionMaker());
	    		return dao;
	    	}
	    
	    	public ConnectionMaker connectionMaker() {
	    		ConnectionMaker connectionMaker = new DConnectionMaker();
	    		return connectionMaker;
	    	}
	    }
  
### 1.4.3 제어권의 이전을 통한 제어관계 역전
- 제어의 역전 : 프로그램의 제어 흐름 구조가 뒤바뀜
	- 오브젝트가 자신이 사용할 오브젝트를 스스로 선택, 생성하지 않고 위임함
	- 템플릿 메소드 패턴 : 서브클래스에선 결정 x
	- 프레임워크 : 라이브러리를 사용하는 애플리케이션 코드는 애플리케이션 흐름을 직접 제어하지만, 프레임워크는 거꾸로 애플리케이션 코드가 프레임워크에 의해 사용된다.
	- UserDao와 ConnectionMaker는 스스로 생성할 수도 언제 쓰일 수도 결정할 수가 없는 수동적인 입장. DaoFactory가 모든걸 결정한다.
	- DaoFactory는 가장 단순한 수준의 IoC 프레임워크, 하지만 애플리케이션 전반에 걸쳐 적용하려면 스프링과 같은 IoC 프레임워크의 도움을 받아야한다.

## 1.5 스프링의 IoC
### 1.5.1 오브젝트 팩토리를 이용한 스프링 IoC
- 애플리케이션 컨텍스트와 설정정보
	- 빈 : 스프링이 제어권을 가지고 직접 만들고 관계를 부여하는 오브젝트
	- 빈 팩토리 : 빈의 생성과 관계설정 같은 제어를 담당하는 IoC 오브젝트
	- 애플리케이션 컨텍스트 :  빈 팩토리와 같은거. 애플리케이션 전반에 걸쳐 모든 구성요소의 제어 작업을 담당하는 IoC 엔진
		- 직접 코드에 설정 정보가 나타나지 않고 설정정보를 가진 다른걸 가져와서 사용

- DaoFactory를 사용하는 애플리케이션 컨텍스트
	- 스프링 전용 설정 정보로 만들어주기 위해 필요한 것
		- @Configuration : 빈 팩토리를 위한 오브젝트 설정을 담당하는 클래스
		- @Bean : 오브젝트를 만들어주는 메소드

코드

    @Configuration
    public class DaoFactory {
    	@Bean
    	public UserDao userDao() {
    		UserDao dao = new UserDao(connectionMaker());
    		return dao;
    	}

	@Bean
	public ConnectionMaker connectionMaker() {
		ConnectionMaker connectionMaker = new DConnectionMaker();
		return connectionMaker;
	}

UserDaoTest에 애플리케이션 컨텍스트를 만듦

    AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(DaoFactory.class);
    UserDao dao = context.getBean("userDao", UserDao.class);
    		
### 1.5.2 애플리케이션 컨텍스트의 동작방식
- 클라이언트는 구체적인 팩토리 클래스를 알 필요가 없다 : XML 방식으로 설정정보 만들 수도 있음
- 애플리케이션 컨텍스트는 종합 IoC 서비스를 제공해준다
- 애플리케이션 컨텍스트는 빈을 검색하는 다양한 방법을 제공한다.

### 1.5.3 스프링 IoC의 용어 정리
- 빈 : 스프링이 직접 그 생성과 제어를 담당하는 오브젝트
- 컨테이너/스프링 컨테이너 : 애플리케이션 컨텍스트, IoC 컨테이너 : 빈팩토리 

## 1.6 싱글톤 레지스트리와 오브젝트 스코프
스프링 컨테이너 (애플리케이션 컨텍스트)는 여러 번에 걸쳐 빈을 요청하더라도 매번 동일한 오브젝트를 돌려줌
### 1.6.1 싱글톤 레지스트리로서의 애플리케이션 컨텍스트
스프링은 별도로 설정하지 않으면 빈 오브젝트를 모두 싱글톤으로 만듦
- 서버 애플리케이션과 싱글톤
	- 스프링은 처음부터 서버를 위해 만들어짐 -> 많은 부하를 감당하기 위해 매번 새로운 오브젝트를 만드는 방식이 아닌 서블릿(서비스 오브젝트) 사용
	- 서블릿은 싱글톤으로 동작
	- 싱글톤 패턴 : 애플리케이션 안에 제한된 수, 대개 한 개의 오브젝트만 만들어서 사용하는 것
- 싱글톤 패턴의 한계
	- private 생성자를 갖고 있기 때문에 상속할 수 없다. : 다형성 x
	- 싱글톤은 테스트하기가 힘들다 
	- 서버 환경에서는 싱글톤이 하나만 만들어지는 것을 보장하지 못한다
	- 싱글톤의 사용은 전역 상태를 만들 수 있기 때문에 바람직하지 못하다

- **싱글톤 레지스트리**
	- 스프링이 직접 싱글톤 형태의 오브젝트를 만들고 관리하는 기능을 제공
	- 스태틱 메소드와 private 생성자 사용 안해도 IoC 방식으로 컨테이너에게 제어권을 넘기면 괜찮다
	- 

### 1.6.2 싱글톤과 오브젝트의 상태
- 싱글톤은 멀티스레드 환경이라면 무상태 방식으로 있어야 함 (여러 스레드가 동시에 값을 변경하거나 읽어가는 경우가 생기기 때문)
- 그래서 인스턴스 변수에 저장하지 않고 파라미터, 로컬 변수, 리턴 값 등을 이용함
- 읽기 전용 변수/자신이 사용하는 다른 싱글톤 빈을 저장하는 용도로는 인스턴스 변수를 사용해도 됨.

### 1.6.3 스프링 빈의 스코프
- 기본적으로 싱글톤 스코프(스프링 컨테이너가 존재하는 동안 계속 유지)

## 1.7 의존관계 주입(DI)
### 1.7.1 제어의 역전과 의존관계 주입
스프링이 요즘엔 다른 프레임워크와의 차별성을 보여주는 의존관계 주입, DI 컨테이너라고 불리고 있음

### 1.7.2 런타임 의존관계 설정
- 의존관계
	- 방향성을 부여해줘야함 (A가 B에 의존하고 있음)
	- A는 B의 변화에 영향을 받음
	- B는 A의 변화에 영향을 받지 않음
- UserDao의 의존 관계
	-  인터페이스를 통해 의존 관계를 제한해둠
	- UserDao는 ConnectionMaker 인터페이스에 의존하여 영향을 받지만, DConnectionMaker 클래스와는 관계가 없어 영향을 받지 않음 -> 결합도가 낮다
- 런타임 의존 관계
	- 클래스 모델이나 코드에는 런타임 시점의 의존 관계가 드러나지 않음
	- 제 3의 존재 (애플리케이션 컨택스트 등)이 결정
	- 의존 오브젝트 : 오브젝트가 만들어지고 나서 런타임 시에 의존관계를 맺는 대상, 즉 실제 사용 대상인 오브젝트
- UserDao의 의존관계 주입
	- **DaoFactory(제 3의 존재, DI 컨테이너)가** 런타임 시점에 UserDao가 사용할 ConnectionMaker 타입의 오브젝트를 결정하고 이를 생성한 후에 UserDao의 생성자 파라미터로 주입(레퍼런스를 넘겨줌)해서 UserDao가 ConnectionMaker의 오브젝트와 런타임 의존관계를 맺게 해줌
	
### 1.7.3 의존관계 검색과 주입
- 외부에서 주입이 아니라 스스로 검색을 이용
- 런타임 시 의존관계를 맺을 오브젝트를 결정하는 것과 오브젝트의 생성 작업은 외부 컨테이너에게 IoC로 맡기지만, 이를 가져올 때는 메소드나 생성자를 통한 주입 대신 스스로 컨테이너에게 요청하는 방법을 사용한다
- getBean()
- 권장하지 않지만, 스프링 컨테이너에 담긴 오브젝트를 사용하려면 적어도 한 번은 의존관계 검색 방식을 사용해 오브젝트를 가져와야함
- 사용하는(검색하는) 오브젝트는 스프링의 빈일 필요가 없음.
- DI는 모두 빈 오브젝트여야함
- DI는 무조건 인터페이스 타입의 파라메터를 이용해 이뤄져야 함

### 1.7.4 의존관계 주입의 응용
- 기능 구현의 교환
	- DI 방식을 사용하면 DB를 로컬에서 운영서버로 바꿀 때 DaoFactory의 코드만 수정하면 됨
- 부가 기능 추가
	- DaoFactory에서는 의존 관계 설정만 해줌

### 1.7.5 메소드를 이용한 의존관계 주입
- setter : 외부에서 오브젝트 내부의 애트리뷰트 값을 변경. 값은 내부의 인스턴스 변수로 저장
- 일반 메소드를 이용해 파라메터 제약 없이 사용가능

## 1.8 XML을 이용한 설정
|  |자바 코드 설정 정보| XML 설정 정보|
|--|--|--|
| 빈의 설정파일| @Configuration | < beans>|
 |빈의 이름|@Bean methodName()|<bean id="methodName"
 |빈의 클래스|return new BeanClass();|class="a.b.c... BeanClass">
	<bean id="myConnectionMaker" class="springbook.user.dao.DConnectionMaker">


- userDao() 전환

수정자 메소드는 xml로 의존 관계 정보를 만들 때 편리

    <bean id="userDao" class="springbook.user.dao.UserDao">
    		<property name="connectionMaker" ref="connectionMaker" />
    </bean>


### 1.8.2 XML을 이용하는 애플리케이션 컨텍스트
- GenericXmlApplicationContext 사용, 생성자 파라미터로 XML 파일의 클래스패스 지정

applicationContext.xml

    <?xml version="1.0" encoding="UTF-8"?>
    <beans xmlns="http://www.springframework.org/schema/beans"
    	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    	xsi:schemaLocation="http://www.springframework.org/schema/beans 
    						http://www.springframework.org/schema/beans/spring-beans-3.0.xsd">
    	<bean id="myConnectionMaker" class="springbook.user.dao.DConnectionMaker"/>
    	<bean id="userDao" class="springbook.user.dao.UserDao">
    		<property name="connectionMaker" ref="myConnectionMaker" />
    	</bean>
    </beans>
- ClassPathXmlApplicationContext 클래스패스에 있는 클래스 오브젝트를 넣어주면 거기에 위치한 xml 파일을 찾을 수 있음

### 1.8.3 DataSource 인터페이스로 변환
- db 커넥션을 가져오는 거(getConnection()) 외에도 여러가지 기능 제공
- db 종류, 아이디, 비밀번호 지정

- 자바 코드 설정 방식

DaoFactory

    @Configuration
	public class DaoFactory {
		@Bean
		public DataSource dataSource() {
			SimpleDriverDataSource dataSource = new SimpleDriverDataSource ();

			dataSource.setDriverClass(com.mysql.jdbc.Driver.class);
			dataSource.setUrl("jdbc:mysql://localhost/springbook?characterEncoding=UTF-8");
			dataSource.setUsername("spring");
			dataSource.setPassword("book");

			return dataSource;
	}

		@Bean
		public UserDao userDao() {
			UserDao userDao = new UserDao();
			userDao.setDataSource(dataSource());
			return userDao;
		}
	}

- XML 설정 방식
### 1.8.4. 프로퍼티 값의 주입까지
	<property name="driverClass" value="com.mysql.jdbc.Driver" />
    <property name="url" value="jdbc:mysql://localhost/springbook?characterEncoding=UTF-8" />
    <property name="username" value="spring" />
    <property name="password" value="book" />
    	

## 질문
### Spring의 싱글톤 패턴에 대해 설명해주세요.

답변 : 스프링에서 bean 생성시 별다른 설정이 없으면 default로 싱글톤이 적용된다. 스프링은 컨테이너를 통해 직접 싱글톤 객체를 생성하고 관리하는데,  
요청이 들어올 때마다 매번 객체를 생성하지 않고, 이미 만들어진 객체를 공유하기 때문에 효율적인 사용이 가능하다.

- 기존 싱글톤 패턴과의 차이
	-   static 메소드나 private 생성자 등을 사용하지 않아 객체지향적 개발을 할 수 있다.
	-  테스트하기 편리하다.

