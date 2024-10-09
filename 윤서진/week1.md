# 1장. 오브젝트와 의존 관계

과목: Spring
강좌명: 토비의 스프링
날짜: 2024년 9월 28일

# 0. 스프링의 핵심 철학 = 객체지향 back to basic

## ∴ **오브젝트**에 초점!

- 기술적 특징 : 오브젝트의 생성 - 관계 형성 - 사용 - 소멸 주기
- 사용 방법 : 오브젝트 설계 방식, 단위, 등장을 위한 과정
- 설계 : **Object-Oriented Design** 기초/원칙 기반, 오브젝트 설계 및 구현에 관한 여러 응용 기술과 지식(디자인 패턴, 리팩토링, 단위 테스트, … )

스프링의 기능

- 오브젝트 활용에 대한 명확한 기준을 제시
- OOP의 손쉬운 적용 유도

<aside>
💡

### 학습 목표

- 스프링의 관심 대상인 **오브젝트**의 설계와 구현, 동작 원리 파악
</aside>

---

# 1. 초난감 DAO

### Data Access Object 란?

**DB를 사용해 데이터를 조회하거나 조작**하는 기능을 전담하도록 만든 오브젝트

## 1) User

사용자 정보 저장: **자바빈 규약**을 따르는 오브젝트 이용!

### JavaBean 이란?

두 관례를 준수하여 만들어진 오브젝트

- **디폴트 생성자** : 파라미터 없는 디폴트 생성자 보유
    
    ∵ 툴이나 프레임워크에서 리플렉션을 이용해 오브젝트 생성
    
- 프로퍼티(자바빈이 노출하는 이름을 가진 속성) : **setter(수정자)와 getter(접근자) 메소드를 통해 프로퍼티를 수정/조회** 가능

```java
package springbook.user.domain;

public class User {
	String id;
	String name;
	String password;
	
	public String getId() { return id; }
	public void setId(String id) { this.id = id; }
	
	public String getName() { return name; }
	public void setName(String name) { this.name = name; }
	
	public String getPassword() { return password; }
	public void setPassword(String password) { this.password = password; }
}
```

## 2) UserDao

### JDBC 이용 작업의 일반적 순서

1. DB 연결 위한 Connection 가져오기
    - `Connection` : jdbc가 정의한 인터페이스, 오브젝트로 구현해 가져오기
2. SQL 담은 Statement(PreparedStatement) 가져오기
3. 만들어진 Statement 실행
4. <조회> 쿼리 실행 결과를 ResultSet으로 받기 → 정보 저장할 오브젝트(빈)에 옮기기
5. 작업 종료 후, 작업 중 생성된 Connection, Statement, ResultSet 등의 리소스 닫기
6. JDBC API가 생성한 exception을 잡아 직접 처리하거나, 메소드에 throws 선언
    - 메소드 밖으로 던져버리는 편을 추천!

```java
package springbook.user.dao;

import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;
import java.sql.ResultSet;
import java.sql.SQLException;

import springbook.user.domain.User;

public class UserDao {
	public void add(User user) throws ClassNotFoundException, SQLException {
		Class.forName("com.mysql.jdbc.Driver");
		Connection c = DriverManager.getConnection("jdbc:mysql://localhost/springbook?characterEncoding=UTF-8", "spring",
				"book");

		PreparedStatement ps = c.prepareStatement(
			"insert into users(id, name, password) values(?,?,?)");
		ps.setString(1, user.getId());
		ps.setString(2, user.getName());
		ps.setString(3, user.getPassword());

		ps.executeUpdate();

		ps.close();
		c.close();
	}

	public User get(String id) throws ClassNotFoundException, SQLException {
		Class.forName("com.mysql.jdbc.Driver");
		Connection c = DriverManager.getConnection("jdbc:mysql://localhost/springbook?characterEncoding=UTF-8", "spring",
				"book");
		PreparedStatement ps = c
				.prepareStatement("select * from users where id = ?");
		ps.setString(1, id);

		ResultSet rs = ps.executeQuery();
		rs.next();
		User user = new User();
		user.setId(rs.getString("id"));
		user.setName(rs.getString("name"));
		user.setPassword(rs.getString("password"));

		rs.close();
		ps.close();
		c.close();

		return user;
	}

}
```

스프링을 공부하며 이 코드의 문제점을 찾아보자.

# 2. DAO의 분리

## 1) 관심사의 분리(Separation of Concerns)

### 객체 설계 시 가장 염두에 둘 사항

: 미래의 변화에 대한 대비! 오브젝트에 대한 설계와 구현 코드는 무조건 변함

- **OOP의 특징 = 변화에 효과적 대처 가능**
    - 가상의 추상세계를 효과적으로 모델링할 뿐만 아니라, 
    이를 자유롭고 편리하게 변경/발전/확장 가능
- 변화의 폭을 최소한으로 줄이고, 그 변경이 다른 곳에 문제를 일으키지 않도록 확신해야 함

### 분리와 확장을 고려한 설계

- 모든 변경과 발전은 한 번에 **한 가지 concern**에 집중해 발생!
    
    ⇒ 한 가지 interest에 따른 작업이 한 곳에 효과적으로 집중되게 해야 함
          **관심사가 같은 것들끼리 모으는 것!**
    
- 관심이 같은 것끼리 하나의 객체 or 친한 객체로 모이도록
- 관심이 다른 것들은 서로 영향을 주지 않도록 따로 분리

## 2) refactor: ‘커넥션 만들기’ 파트 추출하기

메소드 추출 기법(extract method) : 공통의 기능을 담당하는 메소드로 중복된 코드 뽑아내기

<aside>
💡

### UserDao.add()의 관심사 분리

- DB 연결을 위한 커넥션 생성: DB 종류, 드라이버, 로그인 정보, 커넥션 생성법
    
    ⇒ get()에도 중복
    
- 쿼리 담을 Satement 생성 및 실행: 파라미터의 유저 정보를 바인딩 후 DB 통해 실행
- 작업 후 리소스 닫기
</aside>

### 커넥션 가져오는 코드를 독립 메소드로 분리

- DB 연결과 관련된 부분에 변경이 일어났을 경우, 해당 메소드의 코드만 수정하면 됨
    
    concern에 따라 코드 구분 ⇒ 해당 관심사에 대한 변경 발생 시 그 관심사가 집중되는 부분의 코드만 수정!
    관심사가 다른 코드엔 영향 X, 관심 내용도 독립적으로 존재
    

```java
private Connection getConnection() throws ClassNotFoundException, SQLException {
	Class.forname("com.mysql.jdbc.Driver");
	Connection c = DriverManager.getConnection("jdbc:mysql://localhost/springbook?characterEncoding=UTF-8", "spring",
				"book");
	return c;
}
// Connection C = getConnection(); 와 같이 사용
```

## 3) getConnection()의 확장성 부여하기

변화에 적응하는 DAO 만들기

- 각기 다른 종류의 DB 적용, 커넥션 가져오는 방법 변동 등의 변화에 적응하되, 커넥션 이외 로직은 보호

### sol: 상속을 통한 확장(독립)

getConnection()을 추상 메소드로 생성 → 고객사는 추상 클래스 UserDao를 상속해 사용

- 핵심 기능은 보안 유지, 커넥션 관련 기능만 확장성 open
    - 어떤 방식으로 Connection 기능을 제공할지, 어떤 방법으로 Connection 오브젝트 생성할지 관심
- UserDao()는 Connection 인터페이스에 정의된 메소드를 사용할 뿐(어떤 기능을 ‘사용한다’에만 관심)
    - Connection 끌어오는 과정에 일절 관심 X

### Template Method Pattern

- 스프링애서 애용하는 디자인 패턴!
- 슈퍼 클래스 : 기본적인 로직의 흐름 작성
    
    일부 기능 : 추상 메소드 or `protected` 메소드(overridable)로 구현 <템플릿 메소드>
    cf. 훅 메소드 : 서브 클래스에서 선택적으로 오버라이드할 수 있도록 만든 메소드
    
    서브 클래스 : 오픈한 ‘일부 기능’을 필요에 맞게 구현 - 자주 변경되며 확장할 기능을 만듦
    

### Factory Method Pattern

- 서브클래스에서 **구체적인 오브젝트 생성 방법**을 결정
- 슈퍼클래스에선, 서브클래스에서 구현할 메소드를 호출 → 필요한 타입의 오브젝트를 가져와 사용
    - 펙토리 메소드 : 서브클래스에서 오브젝트 생성 방법 및 클래스를 결정하게끔 하는 선정의 메소드

cf) 펙토리 메소드 : 오브젝트를 생성하는 기능을 가진 메소드. 패턴과는 의미가 상의하므로 주의!

### 단점: 상속 사용

- 다중 상속 불가 이슈: UserDao가 이미 다른 목적으로 상속 사용하고 있을 시 적용 불가
- 상속을 통한 부모-자식 관계가 생각보다 밀접
    - 서로 다른 두 관심사에 대한 긴밀한 결합 허용
        
        서브클래스: 슈퍼클래서의 기능 직접 사용 가능 ⇒ 슈퍼클래스 내부 변동 시 서브클래스 영향 가능성
        
- 해당 서브 클래스는, 자신의 부모 클래스와 유사한 다른 클래스에는 적용 불가
    - 매 클래스마다 해당 서브 클래스를 중복 구현해야 하는 이슈 발생

# 3. DAO의 확장

오브젝트들은 각각 변화의 성격과 시점이 모두 상이함

## 1) refactor: 별도의 클래스로 분리

SimpleConnectionMaker.java

장점 : 상속을 사용하지 않으므로 abstract 관련 기능 사용 X

단점 : 확장 불가

- UserDao에서 SimpleConnectionMaker 內 메소드 사용 ⇒ 메소드 이름 상이할 시 일일이 변경해야 함
- UserDao에서 SimpleConnectionMaker의 인스턴스 정의 ⇒ 다른 클래스 구현 시 UserDao 수정 불가피

∴ UserDao가 ‘별도의 클래스’에 대한 너무 많은 정보를 포함하고 있음 ⇒ 자연스레 종속되는 이슈 발생

## 2) refactor: 인터페이스 사용

추상화 : 어떤 것들의 ‘공통 속성’을 뽑아내 이들만 따로 분리하는 작업
⇒ 자바에선 인터페이스를 통해 수행 가능!

(+) 메소드명 고정
(-) 인스턴스 정의 과정으로 인해 여전히 클래스명 등장 (’어떤 구현 클래스를 사용할 것인가’를 결정하는 부분)

## 3) ‘관계 설정’ 책임의 분리

분리해야 할 관심사: “UserDao - UserDao가 사용할 ConnectionMaker의 특정 구현 클래스 사이 관계 설정”
∵ UserDao는 ConnectionMaker 인터페이스 外 그 어떤 클래스와도 관계를 맺어선 안 됨!

다형성 ⇒ UserDao가 CM 인터페이스 사용했다면, CM 구현한 오브젝트를 인터페이스 타입으로 받아 사용 가능!

오브젝트 간의 관계 설정 : 런타임 시 한 쪽이 다른 쪽의 레퍼런스를 보유하는 방식으로 구현
- 생성자 호출해 직접 오브젝트 만들기
- 외부에서 만든 오브젝트를 메소드 파라미터나 생성자 파라미터로 전달 받기

### 제3의 클래스를 만들어 런타임 오브젝트 관계 설정의 책임을 지우기

- UserDao에 cm 오브젝트 받을 파라미터 추가: `ConnectionMaker cm`

(+) 다른 DAO 클래스에도 ConnectionMaker 인터페이스의 구현 클래스들을 적용 가능해짐
       DAO가 많아져도 DB 연결에 관한 관심사는 cm에게 집중 ⇒ 수정 용이

## 4) 원칙 및 패턴 총정리: 스프링 연관 개념

스프링은 아래 원칙 및 패턴에 나타나는 장점을 개발자들이 활용할 수 있게 돕는 프레임워크

### Open-Closed Principle

클래스나 모듈은 **확장에는 open**, **변경에는 closed** 되어야 함

- 인터페이스를 사용해 확장 기능 정의함으로써 준수
- 객체지향 설계 원칙 SOLID 중 하나
    - Single Responsibility Principle
    - Open-Closed Principle
    - Liskov Substitution Principle
    - Interface Segregation Principle
    - Dependency Inversion Principle

### High Coherence and Low Coupling

**높은 응집도** = 하나의 모듈/클래스가 하나의 책임(관심사)에만 집중

- 클래스 레벨을 넘어 패키지, 컴포넌트, 모듈에서도 적용 가능
- 변화 발생 시 해당 모듈에서 변하는 부분이 큼, 다른 모듈에선 변하는 부분 無
    - 변경 부분이 작을수록, 변경이 미변경 부분에 미치는 영향을 파악해야 하는 부담이 커짐

**낮은 결합도** = 한 모듈에서 변경 발생 시, **관계 맺은 다른 모듈에는 변경 요구가 전파되지 않아야** 함

- 책임(관심사)가 다른 오브젝트/모듈과는 연결이 느슨해야 함
- 관계를 유지하는 데 꼭 필요한 최소한의 방법만 간접적인 형태로 제공,
나머지 부분은 서로 독립적이며 알 필요도, 관심도 없음
    
    (+) 변화 대응 속도 증대, 구성의 간결화, 확장 용이
    
- 결합도: 한 오브젝트에서 변경 발생 시 유관한 다른 오브젝트에 변화를 요구하는 정도
    - 결합도가 높을수록, 변경에 따르는 작업량과 변경으로 인한 버그 발생 가능성 커짐

### Strategy Pattern

필요에 따른 변경(variation)이 필요한 기능 ⇒ 인터페이스 통해 통째로 외부에 분리
⇒ 해당 인터페이스를 필요에 따라 구체적인 클래스로 구현해 사용

- 전략 = 필요에 따라 변경이 필요한 기능

---

# 4. Inversion of Control

## 1) 오브젝트 팩토리

UserDao 클래스, CM 인터페이스 구현 클래스의 오브젝트 각각 생성 후 둘을 연결하는 기능을 UserDaoTest로부터 분리하기

∵ 테스트용 클래스에서 해당 기능은 관심사가 상이함

### 팩토리

오브젝트의 생성 방법 결정 후, 그 방식으로 만들어진 오브젝트를 반환

- 오브젝트 생성 담당 - 오브젝트 사용 담당의 역할 및 책임을 분리
- 설계도; 어플리케이션을 구성하는 컴포넌트의 **구조(구성)**와 **관계** 정의
    
    ↔ 컴포넌트 : 실질적 로직 담당
    
    e.g. userDao는DConnectionMaker를 사용, …
    
- 기능에 변경 필요 시(EConnectionMaker): 팩토리에서 변경된 클래스 인스턴스 생성 및 관계 설정
    
    ⇒ 해당 기능의 자유로운 확장 가능, 핵심 로직은 변경 필요 없으므로 소스코드 보존 가능
    

## 2) 오브젝트 팩토리 활용

메소드 추출 기법 사용; 구현 클래스 결정 & 오브젝트 생성 코드를 별도의 ‘생성용 메소드’로 분리

## 3) 제어권 이전을 통한 제어 관계 역전

### 제어의 역전(IoC) : 모든 제어 권한을 ‘제어 권한 갖는 특별한 오브젝트’에 위임

오브젝트가 프로그램의 흐름을 결정하거나 사용할 오브젝트를 직접 구성하던 흐름을 반전

- 오브젝트는 자신이 사용할 오브젝트를 직접 선택사용 X
- 자기 자신의 생성 과정 및 사용처도 알지 못함
- 어플리케이션 컴포넌트의 생성, 관계 설정, 사용, 생명 주기 관리 등을 관장하는 존재 필요
    - 프레임워크, 컨테이너, …

(+) 깔끔한 설계, 유연성 증가, 확장성 증대

### e.g. 프레임워크: 프로그램의 흐름을 주도하며 코드 사용

애플리케이션 코드가 ‘프레임워크’에 의해 수동적으로 사용됨

↔ 라이브러리 : 코드가 흐름 주도, 필요할 때 라이브러리 사용

# 5. 스프링의 IoC

스프링의 핵심 : 빈 팩토리(어플리케이션 컨텍스트)

## 1) 오브젝트 팩토리를 이용한 스프링 IoC

**스프링 Bean** : 스프링이 제어권을 가지고 직접 생성 & 관계 부여하는 오브젝트

- 오브젝트 단위의 애플리케이션 컴포넌트
- IoC가 적용됨

### Bean Factory : 빈의 생성, 관계 설정 등의 제어 담당 IoC 오브젝트

- 보통 빈 팩토리보단 이를 좀 더 확장한 일종의 빈 팩토리인 application context를 사용함
    - 빈 팩토리 : IoC의 기본 기능(빈 생성 & 관계 설정)에 초점
    - 애플리케이션 컨텍스트 : 애플리케이션 內 모든 구성요소의 제어 작업을 담당하는 IoC 엔진이라는 의미에 초점

### Application Context : 별도 정보 참고해 빈 제어 작업 총괄

- 설정 정보 X, 대신 설정 정보를 담은 무언가를 가져와 활용
- 범용적인 IoC 엔진

### 설정 정보 만들기

팩토리(=설계도) : 어플리케이션 컨텍스트 + 설정 정보

- 어플리케이션 로직을 담당하진 않음
- IoC 방식으로 어플리케이션 컴포넌트 생성 및 관계 설정하는 책임을 담당
    
    ### @Configuration
    
    - 어플리케이션 컨텍스트(빈 팩토리)가 사용할 IoC  설정 정보라는 표식
    - 스프링이, ‘빈 팩토리를 위한 오브젝트 설정 담당 클래스’임을 인식하게 함
    
    ### @Bean
    
    - 오브젝트를 생성해주는 **IoC 메소드**라는 표식

### 어플리케이션 컨텍스트 만들기

`ApplicationContext` 타입 오브젝트

- `AnnotationConfigApplicationContext` : @Configuration이 붙은 자바 코드를 설정 정보로 사용
    - 생성자 파라미터 : 설정 정보 클래스
- `getBean()` : ApplicationContext가 관리하는 오브젝트를 요청
    - 파라미터
        1. ApplicationContext에 등록된 빈의 이름 = 메소드명
            - UserDao를 생성하는 방식이나 구성이 다른 메소드들이 있을 수 있음 ⇒ 선택
        2. 빈의 리턴 타입
            - getBean()의 리턴 타입이 Object이므로, 매번 캐스팅 코드 사용하는 대신 generic 방식 활용해 두 번째 파라미터에 리턴 타입 부여

```java
public class UserDaoTest {
	public static void main(String[] args) throws ClassNotFoundException, SQLException {
		ApplicationContext context = new AnnotationConfigApplicationContext(DaoFactory.class);
		UserDao dao = context.getBean("userDao", userDao.class);
		...
	}
}
```

### 스프링 라이브러리 추가

## 2) 애플리케이션 컨텍스트 동작 방식

### 스프링 컨테이너(IoC 컨테이너) = 어플리케이션 컨텍스트(빈 팩토리)

- `ApplicationContext` 인터페이스 구현
    - `BeanFactory` 상속하므로 일종의 빈 팩토리라 볼 수 있음
- 어플리케이션에서 **IoC 적용해 관리할 모든 오브젝트에 대해** 생성 & 관계 설정 담당
- 직접 오브젝트 생성하고 관계 설정하는 코드는 X
해당 정보는 별도의 설정 정보를 통해 얻음
    - 외부의 오브젝트 팩토리에 해당 작업을 위임하고 결과를 받아 쓰기도

### ↔ 오브젝트 팩토리

- 제한적 역할 수행 : **DAO 오브젝트** 생성, 관계 생성

### 장점

- 클라이언트가 구체적인 팩토리 클래스를 알지 않아도 됨
    - 오브젝트 팩토리가 많아져도 이를 전부 알거나 직접 생성/사용할 필요 X
    - 일관된 방식으로 원하는 오브젝트 가져오기 가능
    - 직접 코드 작성하는 대신 XML 같은 단순한 방법으로 설정 정보 생성 가능
- IoC 서비스를 종합적으로 제공
    - 오브젝트의 생성 방식/시점/전략 다르게 지정 가능
    - 자동 생성, 오브젝트 후처리, 정보 조합, 설정 방식 다변화, 인터셉팅 등
    - 빈이 사용 가능한 기반기술 서비스, 외부 시스템 연동 등의 기능을 컨테이너 차원에서 제공
- 빈을 검색하는 다양한 방법을 제공
    - `getBean()` : 빈의 이름으로 빈 찾기
    - 타입으로 빈 검색
    - 특별한 애노테이션 설정이 되어 있는 빈 찾기

## 3) 스프링 IoC 용어 정리

### Bean

스프링이 IoC 방식으로 관리하는 오브젝트

- managed object라 부르기도
- 스프링에서 만들어지는 모든 오브젝트 ≠ 빈
    - **스프링이 직접 생성 & 제어** 담당해야 Bean!

### Bean Factory

스프링의 IoC를 담당하는 핵심 컨테이너

빈의 등록, 생성, 조회, 반환, 그 외 빈 관리하는 부가 기능 담당

- 보통 빈 팩토리를 바로 사용하지 않고, 이를 확장한 어플리케이션 컨텍스트 이용
- `BeanFactory` : 빈 팩토리가 구현되고 있는 가장 기본적인 인터페이스
    - `getBean()`

### Application Context

빈 팩토리를 확장한 IoC 컨테이너

빈 팩토리 + 스프링이 제공하는 애플리케이션 지원 기능

### Configuration Metadata

빈 팩토리가 IoC를 적용하기 위해 사용하는 메타 정보

- 어플리케이션의 형상 정보, 어플리케이션 전체 청사진
- IoC 컨테이너에 의해 관리되는 애플리케이션 오브젝트를 생성 & 구성할 때
- 컨테이너에 어떤 기능을 세팅하거나 조정할 때

### Container

= 빈 팩토리

- IoC 방식으로 빈을 관리한다는 의미
- 어플리케이션 컨텍스트보다 추상적인 표현
    - 어플리케이션 컨텍스트(인터페이스 구현한 오브젝트) + 한 어플리케이션에서 구현된 여러 어플리케이션 컨텍스트 오브젝트 통틀어 지칭
- 스프링과 같은 의미로 쓰이기도

### 스프링 프레임워크

IoC 컨테이너 + 어플리케이션 컨텍스트 + 기타 스프링 제공 기능 전체
